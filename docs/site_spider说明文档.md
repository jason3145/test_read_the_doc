# Site spider项目说明文档

---
> site spider是基于scrapy1.4框架写的爬虫项目，项目中引入kafka作为初始爬取队列并采用多优先级内存队列对下载失败或发现需要使用代理下载的链接进行调度，后端数据存储可以选用elasticsearch、hbase或kafka；爬取过程中的统计数据，如下载链接总数、请求总大小、响应总大小等数据会直接放入elasticsearch中做统计。

### 项目部署说明

1. 相关环境部署说明
	* kafka： 需要设置初始爬取队列，消息的json格式如下
	
		``` 
		{
		 'url': url, 
		 'meta': {
		     'url_id': _id,  # 存入elasticsearch中的id
		     'NotExtract': True  # 是否不要获取响应内容中的链接（发现新链接）
		  }, 
		 'method': 'HEAD',  # 请求方式
		 '_encoding': 'utf-8'
		}
		  ```
	* elasticsearch：需要的字段如下
	
		```
		status:下载的状态码
		headers:响应headers内容转成json字符串
		body:响应的内容
		description:下载失败的原因描述
		```
2. 配置文件修改
	* 执行脚本的修改
		> 修改项目中task_shell/crawl_spider.sh中的启动参数包括并发数，初始化队列名称，消费者组名称及生产环境中项目virtualenv的位置
	* 配置运行环境
		> 修改项目中site_spider/spiders/crawl_spider.py中custom_settings的内容，包括kafka broker的地址、elasticsearch的地址等信息
3. 项目部署运行
	> 将项目打包到服务器，将virtualenv安装在执行脚本中指定的位置处，然后运行脚本即可完成部署

### 项目统计数据说明
* total_num: 下载链接总数
* total_response_size: 响应总大小
* total_request_size: 请求总大小

### 重要组件实现原理
1. 项目整体执行流程
	![](./resources/scrapy_architecture.png)
2. url调度原理
	* 重要组件说明
		* 调度器（scheduler）
			> 用于初始化各个组件，在url入列时调用去重器，在url出列时添加callback和errback回调方法
		* 三类队列（内存队列、kafka队列、优先级队列）
			> 1. **优先级队列：**会根据入列url的优先级创建内存队列或kafka队列，优先级队列负责维护kafka队列和多个内存队列，包括它们的创建、销毁和消息的出列、入列
			
			> 2. **kafka队列：**为初始队列(优先级为0)会在优先级队列初始化时创建同时不会被回收(kafka队列复写了__len方法永远返回1)
			
			> 3. **内存队列：**根据需要进行创建，队列长度为0时被回收；重试插件抛出request时会降低其优先级，重定向插件抛出request时会升高其优先级
			
			> 4. 若有待入列的request优先级为0时，会自动升高其优先级为1（所有后来生成的url均在内存处理不发送到kafka队列）
		* 去重器（dupefilter）
	* 调度器初始化
		* 根据配置获取三类队列、去重器的构造方法
		* 将_newmq作为参数构造优先级队列的实例，该方法用于根据优先级创建kafka队列和内存队列；_newmq实现如下
		
			```
			    def _newmq(self, priority):
        			if priority != 0:
            			return self.mqclass()
        			else:
            			return self.kqclass(self.settings)
			```
	* 优先级队列调度原理
		* 初始化
			> 根据传入的优先级列表(startprios=())及创建队列的工厂方法初始化队列（此处为0，_newmq），并设置当前使用队列的优先级
			```self.curprio = min(startprios) if startprios else None```
		* 入列
			> 入列时将request根据其优先级push到相应队列，并根据情况改变当前优先级
			
			> 
			``` 
			if self.curprio is None or priority < self.curprio:
            	self.curprio = priority
		```
		* 出列
			> 出列时根据当前优先级（self.curprio）从相应队列中取出url，并根据情况销毁队列、重置当前优先级
			```
			if self.curprio is None:
            	return
        	q = self.queues[self.curprio]
        	m = q.pop()
        	if len(q) == 0:
            	del self.queues[self.curprio]
            	q.close()
            	prios = [p for p, q in self.queues.items() if len(q) > 0]
            	self.curprio = min(prios) if prios else None
        	return m
		```
3. 下载中间件
	> 总结：**返回request**则重新调度，**返回None**继续后续处理，**抛出异常**process_exception或errback将会被调用，**返回response**会继续后续处理或进入process_response回调链
	* process_request方法
		* **返回request**：停止后续process_request的调用，并对该request进行重新调度（入列）
		* **返回response**：不会在进行后续的下载流程（包括剩下的process_request方法和下载器调用），会直接返回该response进入相应处理流程（process_response的按序调用）
		* **返回None**：会继续调用后续中间件的process_request方法，最后会调用下载器下载该url
		* **抛出异常**：IgnoreRequest处理流程，所有下载中间件的process_exception都会被调用，若没有被处理那么该request的errback将会被调用，若还是没有被处理该request将被忽略且无日志记录
	* process_response方法（不能返回None）
		* **返回request：**停止后续的调用，并对该request进行重新调度（入列）
		* **返回response：**会作为参数传入下一个process_response方法
		* **抛出异常：**IgnoreRequest被抛出会直接调用request的errback方法（不会调用process_exception）
	* process_exception方法
		* 处理流程同process_request，若抛出异常则不会被任何函数处理直接被放弃

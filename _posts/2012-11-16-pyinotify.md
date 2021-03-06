---
layout: post
title: pyinotify使用中的陷阱
category: thinking
tags: [pyinotify]
---

在做米聊线上监控报警工具的时候，需要将每次的监控数据进行记录，这里采用了以下流程。

![米聊在线服务预警图](/assets/images/2012-11-16-miliao.png)

探测程序会定期将自己的探测结果以xml文件的形式写入到特定的目录中，监控程序会监控该目录并到新写入的数据，并插入数据库。报警程序会定期扫描数据库中的数据，结合报警策略，决定是否进行报警。

	问：为啥探测程序会是写入特定的目录中，再有监控程序写入数据库，而不是直接写数据库?

	答：1. 保证探测程序的简单性，不做过多的事件。

	2. 解耦合，监控程序类似于一个agent，可以支持更多不同语义的监控。

	3. 监控＋报警这样更容易做成一个服务
	
	
	问：为啥监控程序会将监控数据写入数据库，而不是直接决定是否报警

	答：1. 报警策略可能需要结合监控的多个状态来报警。

	2. 更重要的是，满足可以网页可以浏览在过去一段时间内的监控状态

如何获取一个指定的目录下，是否有新的数据写入？
两种方法：

1. 启动一个线程，定期扫描该文件夹，如果某个文件不存在之前的文件hash表中，则为新增的数据。或者每次写入的文件名相同，此时，比较本次的时间和上次的时间是否相同。
2. 使用inotify，在文件夹下有自己感兴趣的event发生时，则调用回调函数，写入数据库中

方法二相对于方法一，减少了定期的轮询开销，减少了前后两个状态的对比，处理方式相对更加优雅。

监控程序使用python编写，所以使用pyinotify来实现inotify的监控。inotify 既可以监视文件，也可以监视目录，可以监视的文件系统事件包括：

<table>
	<tr>
		<td>Event Name</td><td>Is an Event</td><td> Description</td>
	</tr>
	<tr>
		<td>IN_ACCESS</td><td>YES</td><td>file was accessed</td>
	</tr>
	<tr>
		<td>IN_ATTRIB</td><td>YES</td><td>metadata changed</td>
	</tr>
	<tr>
		<td>IN_CLOSE_WRITE</td><td>YES</td><td>writtable file was closed</td>
	</tr>
	<tr>
		<td>IN_CREATE</td><td>YES</td><td>file/dir was created in watched directory</td>
	</tr>
	<tr>
		<td>IN_DELETE</td><td>YES</td><td>file/dir was deleted in watched directory</td>
	</tr>
	<tr>
		<td>IN_MODIFY</td><td>YES</td><td>file was modified</td>
	</tr>
	<tr>
		<td>IN_OPEN</td><td>YES</td><td>file was opened</td>
	</tr>	
</table>

通过pyinotify来实现对文件系统的监控非常简单，如下是一个demo，只需要重写对应的回调函数既可。

	#!/usr/bin/env python
	# encoding:utf-8
 
	import os
	from  pyinotify import  WatchManager, Notifier, \
	ProcessEvent,IN_DELETE, IN_CREATE,IN_MODIFY
 
	class EventHandler(ProcessEvent):
	    """事件处理"""
	    def process_IN_CREATE(self, event):
	        print   "Create file: %s "  %   os.path.join(event.path,event.name)
	 
	    def process_IN_DELETE(self, event):
	        print   "Delete file: %s "  %   os.path.join(event.path,event.name)
	 
	    def process_IN_MODIFY(self, event):
	        print   "Modify file: %s "  %   os.path.join(event.path,event.name)
	 
	def FSMonitor(path='.'):
	    wm = WatchManager() 
	    mask = IN_DELETE | IN_CREATE |IN_MODIFY
	    notifier = Notifier(wm, EventHandler())
	    wm.add_watch(path, mask,rec=True)
	    print 'now starting monitor %s'%(path)
	    while True:
	        try:
	            notifier.process_events()
	            if notifier.check_events():
	                notifier.read_events()
	        except KeyboardInterrupt:
	            notifier.stop()
	            break
	 
	if __name__ == "__main__":
	    FSMonitor()

放在本项目中，则被hook的函数为IN_CREATE，因为每次探测程序都会写数据到新的文件，然后定期删除监控的数据。但是一个诡异的问题出现了：在IN_CREATE事件发生时，调用程序读取新的数据，读取函数返回错误：
	xml.parsers.expat.ExpatError: no element found: line 1, column 0

google 该错误信息，很多都说是open了file后忘记了关闭，但是我在这里已经做了close的动作。而且这个问题的出现，是偶尔发生的。

思考后，发现是读取element没有读取到，是不是和文件的content有关，没办法，只能抓日志，并查看content的日志是否合法了，有了这个思路后，下一次一发生这个文件，就找到原因了。content的长度为0，难怪会打印这个错误。

于是疑问来了，为啥content的内容会为0？打开文件时，明明长度是存在的。当然，sleep 1s，基本可以规避这个问题。但是明显没有完全解决这个问题。

后面回去的路上，仔细考虑这个问题，并将demo代码中的事件都监控起来，终于找到问题的根源了。

创建文件的方法为：

	f = open(filename, 'w')
	...
	dom.writexml(f, addindent='  ', newl='\n', encoding='utf-8')
	f.close()

在这里，实际发生了三个事件，分别是IN_CREATE, IN_MODIFY, IN_CLOSE_WRITE, 分别对应于这三步，虽然我们平时说的是创建一个文件，但在inotify中，创建文件和我们平时指的还是有较大差别。

修复方法自然也出来了，将监控的事件修改为IN_CLOSE_WRITE，问题解决。sleep 只是一个不治本的方法。

---
layout: post
title: Java监听文件夹
date: 2019-07-09
Author: 来自veryfirefly
categories: 
tags: [工作经验,解决方法]
comments: true
---

最近遇到了一个需求，需要知道文件夹下的某个配置文件是否被修改了，如果修改了，就重载配置文件。当时想用Listener模式实现，通过一个线程一直轮询该文件夹下的所有文件，然后一旦发生了改变，就去触发自定义的事件，但想法很美好，实现起来Bug巨多，于是我选择了“面向百度编程”。查到了在JDK1.7后，NIO有一个`java.nio.file.WatchService` API可以实现该需求，且性能相对于我的那个好很多。


>  注意WatchService监听在系统支持的情况下采用事件驱动机制，可降档为扫描式机制。
>  
>  事件发生后需使用reset()方法重置WatchKey对象，否则事件所属的目录的改动通知只会发生一次。


**WatchService**接口及实现关系图

1. 该类的对象就是操作系统原生的文件系统监控器！我们都知道OS自己的文件系统监控器可以监控系统上所有文件的变化，这种监控是无需遍历、无需比较的，是一种基于信号收发的监控，因此效率一定是最高的；现在Java对其进行了包装，可以直接在Java程序中使用OS的文件系统监控器了。
2. 获取当前OS平台下的文件系统监控器：
	- 1). WatchService watcher = FileSystems.getDefault().newWatchService()。
	- 2). 从FileSystems这个类名就可以看出这肯定是属于OS平台文件系统的，接下来可以看出这一连串方法直接可以得到一个文件监控器。
3. 我们都知道，操作系统上可以同时开启多个监控器，因此在Java程序中也不例外，上面的代码只是获得了一个监控器，你还可以用同样的代码同时获得多个监控器。
4. 监控器其实就是一个后台线程，在后台监控文件变化所发出的信号，这里通过上述代码获得的监控器还只是一个刚刚初始化的线程，连就绪状态都没有进入，只是初始化而已。


	- `AbstractWatchService` 实现WatchService接口。
	- `WindowsWatchService` 具体的实现，启动Poller线程。
	- `AbstractWatchKey` 实现WatchKey接口，作为一个监控池，其中包含了多个WatchEvent事件。
	- `sun.nio.fs.AbstractWatchKey$Event` 实现WatchEvent接口，作为一个监控文件变化的事件。
	- `Poller` 实现AbstractPoller接口，其本质为一个线程(Runnable)，轮询指定的目录。


![watchservice-api](https://veryfirefly.github.io/images/watchservice-api.png)

`java.nio.file.FileSystem`抽象接口中含有`public abstract WatchService newWatchService() throws IOException;`接口，使用改接口创建WatchService实例，而Windows平台下则默认使用的`sun.nio.fs.WindowsFileSystem`子类实现，该子类中创建WatchService方法为：

	public WatchService newWatchService() throws IOException {
        return new WindowsWatchService(this);
    }

直接创建一个WindowsWatchService对象并返回，而如果调用`com.sun.nio.zipfs.ZipFileSystem`子类中的newWatchService方法，则会得到一个不支持该操作的异常：

	 public WatchService newWatchService() {
        throw new UnsupportedOperationException();
     }

在windows平台下，我并为像上图中找到WatchService的LinuxWatchService和PollingWatchService实现子类，这可能会与平台有关。

----------

![watchservice-implement](https://veryfirefly.github.io/images/watchservice-implement.png)

Java API文档描述WatchKey接口中这么说道：

	A token representing the registration of a watchable object with a WatchService.
	A watch key is created when a watchable object is registered with a watch service. The key remains valid until:
	
		1. It is cancelled, explicitly, by invoking its cancel method, or
		2. Cancelled implicitly, because the object is no longer accessible, or
		3. By closing the watch service.

该接口表示使用WatchService注册为可观察对象的标记。当监视服务注册可观察对象时，会创建监视密钥。密钥仍然有效，直到它通过调用取消方法显示取消，当对象不可访问时隐式取消，或手动关闭服务。

文档中还说：WatchKey具有状态，最初创建时，WatchKey状态为ready，检测到事件触发时，会将WatchKey放入`LinkedBlockingDeque`进行排队执行，以便通过调用监视服务的poll()或take()方法来获取该WatchKey，一旦触发，密钥就会保持此状态，知道调用其reset方法将WatchKey返回就绪状态。在WatchKey处于信号状态(AbstractWatchKey.State.SIGNALLED)检测到的事件进行排队但不会导致WatchKey重新排队，此方法检索并删除为对象累积的所有事件。最初创建时，WatchKey没有待处理的事件。

在AbstractWatchKey接口中，提供了一个`List<WatchEvent<?>> events`的集合来装WatchEvent，并通过一个`Map<Object, WatchEvent<?>> lastModifyEvents` 来映射对应的Event。


**PropertyChangesMonitor** 代码如下：

>  watcher.poll() 尝试获取下一个变化信息的监控池，如果没有变化则返回null。
>  
>  watcher.take() 尝试获取下一个变化信息的监控池，如果没有变化则一直等待。
>
>  如果需要长时间一直监控要用`watchService.take()`，而如果只是在某个指定的时间监控则用`watchService.poll()`。

	package cool.likeu;

	import java.io.File;
	import java.io.IOException;
	import java.nio.file.Path;
	import java.nio.file.Paths;
	import java.nio.file.WatchEvent;
	import java.nio.file.WatchKey;
	import java.nio.file.WatchService;
	
	import static java.nio.file.StandardWatchEventKinds.ENTRY_MODIFY;
	import static java.nio.file.StandardWatchEventKinds.OVERFLOW;

	
	public class PropertyChangesMonitor {
		
	    private static final String MONITOR_THREAD_NAME = "Watch-Thread";

	    /**
	     *  对根目录注册监听服务。
	     *  1.ENTRY_CREATE: 创建条目时返回的事件类型
	     *  2.ENTRY_DELETE: 删除条目时返回的事件类型
	     *  3.ENTRY_MODIFY: 修改条目时返回的事件类型
	     *  4.OVERFLOW: 表示事件丢失或被丢弃，不必要注册该事件类型
	     *
	     * @param filepath
	     * @throws Exception
	     */
	    public void doMonitor(String filepath) throws Exception {
	        if (null == filepath || "".equals(filepath))
	            throw new NullPointerException("file path is empty!");
	
	        File file = new File(filepath);
	        Path path = Paths.get(file.toURI());
	        synchronized (this){
	            WatchService watcher = path.getFileSystem().newWatchService();
	            path.register(watcher,ENTRY_MODIFY,
	                                  ENTRY_CREATE,
	                                  ENTRY_DELETE);
	
	            try {
	                Thread watchThread = new Thread(() -> {
	                    while(true){
	                        /**
	                         * WatchKey对象key轮询事件(pollEvents())获得事件队列
	                         * 遍历事件(WatchEvent)获得变化文件的信息
	                         * watcher.poll() 尝试获取下一个变化信息的监控池，如果没有变化则返回null。
	                         * watcher.take() 尝试获取下一个变化信息的监控池，如果没有变化则一直等待。
	                         * 这两个方法都是WatchService的对象方法，从这点看出，获取更新的监控信息其实需要当前Java进程和操作系统进程之间进行交流的，返回的WatchKey是当前Java程序的(内存空间位于当前Java程序中)。
	                         * 如果需要长时间一直监控要用take()，而如果只是在某个指定的时间监控则用poll()。
	                         */
	                        WatchKey key = watcher.poll();
	                        if (Objects.isNull(key)){
	                            continue;
	                        }
                        
	                        for (WatchEvent<?> event : key.pollEvents()){
	                            if (event.kind() == OVERFLOW){
	                                continue;
	                            }
	                            //event.context()返回Object对象，可转换为Path或者String类型。
	                            //event.kind()返回事件种类。
	                            Path filename = (Path) event.context();
	                            // this doChange logic
	                            doChange(filename);
	                            System.out.println("文件更新:"+filename.toFile().getAbsolutePath());
	                        }
	                        /**
	                         * 重置WatchKey对象，使事件对象回到队列。
	                         * 重启该线程，因为处理文件可能是一个耗时的过程，因此调用poll()时需要阻塞监控器线程
	                         */
	                        if (!key.reset()) {
	                            break;
	                        }
	                    }
	                },MONITOR_THREAD_NAME);
	
	                watchThread.start();
	                watchThread.join();
	            } catch (Exception e){
	                if (e instanceof InterruptedException){
	                    throw new Exception(e.getMessage(),e);
	                }
	                throw e;
	            }
	        }
	    }
	
	    public void doChange(Path path){
			// do somethings
	    }
	}
hystrix 限流熔断库

停止故障级联
失败快速恢复
实时监控 预警

在高流量的情况下，单个后端依赖关系变得潜在，可能会导致所有服务器上的所有资源在数秒内饱和。
应用程序中通过网络或进入客户端库的每个点都可能导致网络请求，这是潜在的故障来源。比故障更糟糕的是，这些应用程序还可能导致服务之间的延迟增加，从而备份队列、线程和其他系统资源，从而导致跨系统发生更多的级联故障。

网络连接失败或降级。服务和服务器故障或变慢。新的库或服务部署会改变行为或性能特征。客户端库有bug。
所有这些都表示需要隔离和管理的故障和延迟，以便单个故障依赖项不会导致整个应用程序或系统停机。

Hystrix

防止任何单一依赖耗尽所有容器(如Tomcat)用户线程。
减少负载和快速失败，而不是排队。
在可行的情况下提供后备方案以保护用户不受故障的影响。
使用隔离技术(如 bulkhead 舱壁、swimlane 泳道和 circuit breaker patterns 断路器模式)来限制任何一个依赖项的影响。
通过接近实时的度量、监视和警报来优化发现时间
通过配置更改的低延迟传播和Hystrix在大多数方面的动态属性更改来优化恢复时间，这允许您通过低延迟反馈循环进行实时操作修改。
防止在整个依赖客户端执行过程中发生故障，而不仅仅是在网络流量中。

Hystrix是如何实现
将所有对外部系统(或“依赖”)的调用包装到HystrixCommand或hystrixobservableccommand对象中，这些对象通常在单独的线程中执行(这是command模式的一个例子)。
超时调用的时间超过您定义的阈值。这是默认的，但是对于大多数依赖项，您通过“属性”自定义设置这些超时，因此它们略高于每个依赖项的99.5%的性能。
为每个依赖项维护一个小线程池(或信号量);如果该依赖项已满，那么针对该依赖项的请求将立即被拒绝，而不是排队等待。
度量成功、失败(客户端抛出的异常)、超时和线程拒绝。
触发断路器，在一段时间内停止对某一特定服务的所有请求，如果该服务的错误率超过阈值，可以手动或自动停止。
在请求失败、被拒绝、超时或短路时执行回退逻辑。
以接近实时的方式监控度量和配置更改。

每个依赖都是相互隔离的，当延迟发生时，它可能饱和的资源受到限制，并在回退逻辑中涵盖，决定在依赖中发生任何类型的故障时做出何种响应

流程
1、构造HystrixCommand或者HystrixObservableCommand对象

如果依赖关系预期返回单个响应，则构造一个HystrixCommand对象。例如:
HystrixCommand command = new HystrixCommand(arg1, arg2);
同步实现

如果依赖预期会返回一个发出响应的可观察对象，那么就构造一个hystrixobservableccommand对象。例如:
HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
异步的实现

2、有四种方法可以执行这个命令，通过使用Hystrix命令对象的以下四种方法之一(前两种只适用于简单的HystrixCommand对象，而hystrixobservableccommand不能使用):
execute()——阻塞，然后返回从依赖项接收到的单个响应(或者在出现错误时抛出异常)
queue()——返回一个Future，你可以用它从依赖项中获取单个响应
observe()——订阅表示来自依赖项的响应的Observable，并返回一个复制源Observable的Observable
toObservable()——返回一个可观察对象，当你订阅它时，它会执行Hystrix命令并发出响应

3、如果这个命令启用了请求缓存，并且该请求的响应在缓存中可用，那么这个缓存的响应将立即以一个可观察对象的形式返回

4、检查断路器是否开启
当你执行这个命令时，Hystrix检查断路器，看看电路是否打开。
如果电路断开(或“触发”)，Hystrix将不会执行命令，但会将流程路由到获得回退。
如果电路是闭合的，则流向以检查是否有可用的容量来运行命令。

5. 线程池/队列/信号量是否已满?
如果与该命令相关的线程池和队列(或者信号量，如果不是在线程中运行)已满，Hystrix将不会执行该命令，而是立即将流路由到获取回退。

6. HystrixObservableCommand.construct() or HystrixCommand.run()
在这里，Hystrix通过你为此目的编写的方法调用对依赖的请求，如下之一:
HystrixCommand.run() -返回单个响应或抛出异常
hystrixobservableccommand .construct()——返回一个Observable，它会发出响应或者发送一个onError通知

如果run()或construct()方法超过了命令的超时值，线程将抛出一个TimeoutException(如果命令本身没有在自己的线程中运行，则另一个计时器线程将抛出一个TimeoutException)。在这种情况下，Hystrix将响应路由到。获取回退，如果该方法没有取消/中断，它将丢弃最终的返回值run()或construct()方法。
请注意，没有办法强制潜在线程停止工作——Hystrix在JVM上能做的最好的事情是向它抛出InterruptedException。如果Hystrix包装的工作不尊重interruptedexception, Hystrix线程池中的线程将继续它的工作，尽管客户端已经收到了一个TimeoutException。这种行为会使Hystrix线程池饱和，尽管负载会“正确地释放”。大多数Java HTTP客户端库不会解释interruptedexception。因此，请确保正确配置HTTP客户机上的连接和读写超时。
如果命令没有抛出任何异常并返回响应，Hystrix将在执行一些日志记录和指标报告后返回此响应。在run()的例子中，Hystrix返回一个Observable，它会发出单个响应，然后发出一个onCompleted通知;在construct()的情况下，Hystrix返回的是construct()返回的相同的可观察对象。

7、断路器的健康检查
Hystrix向断路器报告成功、失败、拒绝和超时，断路器维护一组滚动的计数器来计算统计数据。
它使用这些统计数据来确定电路何时应该“跳闸”，在这一点上它会短路任何后续请求，直到恢复周期结束，在恢复周期结束后，它会在首先检查某些健康检查后再次闭合电路。

8、fallback
编写回退，从内存缓存或通过其他静态逻辑提供无任何网络依赖的通用响应。如果必须在回退中使用网络调用，则应该通过另一个HystrixCommand或hystrixobservableccommand来实现。
在HystrixCommand的情况下，为了提供回退逻辑，您实现了HystrixCommand. getfallback()，它返回一个回退值。

在hystrixobservableccommand的例子中，为了提供回退逻辑，你实现了hystrixobservableccommand . resumewithfallback()，它返回一个可观察对象，这个可观察对象可能会发出一个或多个回退值。
如果回退方法返回一个响应，那么Hystrix将把这个响应返回给调用者。在HystrixCommand.getFallback()的例子中，它将返回一个可观察对象，该可观察对象会发出从该方法返回的值。在hystrixobservableccommand . resumewithfallback()的例子中，它会返回从该方法返回的相同的可观察对象。
如果你还没有为Hystrix命令实现一个回退方法，或者回退本身抛出一个异常，Hystrix仍然返回一个可观察对象，但这个可观察对象不发出任何东西，并立即以一个onError通知终止。通过这个onError通知，导致命令失败的异常被传输回调用者。(实现可能失败的回退实现是一个糟糕的实践。你应该实现你的回退，这样它就不会执行任何可能失败的逻辑。)

9、成功返回响应



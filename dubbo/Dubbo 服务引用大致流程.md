上篇文章我们已经了解了 Dubbo 服务暴露全过程，这篇文章我就带着大家再来看看 Dubbo 服务引入全流程，这篇服务引入写完下一篇就要来个全链路打通了，看看大家看完会不会有种任督二脉都被打通的感觉。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrgtSwCBx4xTHgyMoIWWD1avaoFIyMVMGeUnmZEYmG1FmZ5VtpO6B4ZA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在写文章的过程中丙还发现官网的一点**小问题**，下文中会提到。

话不多说，咱们直接进入正题。

## 服务引用大致流程

我们已经得知 `Provider`将自己的服务暴露出来，注册到注册中心，而 `Consumer`无非就是通过一波操作从注册中心得知 Provider 的信息，然后自己封装一个调用类和 Provider 进行深入地交流。

而之前的文章我都已经提到在 `Dubbo`中一个可执行体就是`Invoker`，所有调用都要向 Invoker 靠拢，因此可以推断出应该要先生成一个 Invoker，然后又因为框架需要往不侵入业务代码的方向发展，那我们的 Consumer 需要无感知的调用远程接口，因此需要搞个代理类，包装一下屏蔽底层的细节。

整体大致流程如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrXv1lLAjPPFbhOG7j0n3IU9Ez3V7HDZkeZY2PCufvUPrCEEMxN7MJJQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 服务引入的时机

服务的引入和服务的暴露一样，也是通过 spring 自定义标签机制解析生成对应的 Bean，**Provider Service 对应解析的是 ServiceBean 而 Consumer Reference 对应的是 ReferenceBean**。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrPCk0saf8Zu2LosicdYAfxEuRD3vKy64zUp9gkEaE6B7bKNOyk5rXoKA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

前面服务暴露的时机我们上篇文章分析过了，在 Spring 容器刷新完成之后开始暴露，而服务的引入时机有两种，第一种是饿汉式，第二种是懒汉式。

饿汉式是通过实现 Spring 的`InitializingBean`接口中的 `afterPropertiesSet`方法，容器通过调用 `ReferenceBean`的 `afterPropertiesSet`方法时引入服务。

懒汉式是只有当这个服务被注入到其他类中时启动引入流程，也就是说用到了才会开始服务引入。

**默认情况下，Dubbo 使用懒汉式引入服务**，如果需要使用饿汉式，可通过配置 dubbo:reference 的 init 属性开启。

我们可以看到 `ReferenceBean`还实现了`FactoryBean`接口，这里有个关于 Spring 的面试点我带大家分析一波。

## BeanFactory 、FactoryBean、ObjectFactory

就是这三个玩意，我单独拿出来说一下，从字面上来看其实可以得知`BeanFactory`、`ObjectFactory`是个工厂而`FactoryBean`是个 Bean。

`BeanFactory` 其实就是 IOC 容器，有多种实现类我就不分析了，简单的说就是 Spring 里面的 Bean 都归它管，而`FactoryBean`也是 Bean 所以说也是归 BeanFactory 管理的。

那 `FactoryBean` 到底是个什么 Bean 呢？它其实就是把你真实想要的 Bean 封装了一层，在真正要获取这个 Bean 的时候容器会调用 FactoryBean#getObject() 方法，而在这个方法里面你可以进行一些复杂的组装操作。

这个方法就封装了真实想要的对象**复杂的创建过程**。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrw3KkRtbfeKsNZNnNNFclf9t1rzaaxGWLnLaJzGM0ibwc2XpyNic4gLRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

到这里其实就很清楚了，就是在真实想要的 Bean 创建比较复杂的情况下，或者是一些第三方 Bean 难以修改的情形，使用 FactoryBean 封装了一层，屏蔽了底层创建的细节，便于 Bean 的使用。

而 ObjectFactory 这个是用于延迟查找的场景，它就是一个普通工厂，当得到 ObjectFactory 对象时，相当于 Bean 没有被创建，只有当 getObject() 方法时，才会触发 Bean 实例化等生命周期。

主要用于暂时性地获取某个 Bean Holder 对象，如果过早的加载，可能会引起一些意外的情况，比如当  Bean A 依赖 Bean B 时，如果过早地初始化 A，那么 B 里面的状态可能是中间状态，这时候使用 A 容易导致一些错误。

总结的说 **BeanFactory 就是 IOC 容器，FactoryBean 是特殊的 Bean, 用来封装创建比较复杂的对象，而 ObjectFactory 主要用于延迟查找的场景，延迟实例化对象**。

## 服务引入的三种方式

服务的引入又分为了三种，第一种是本地引入、第二种是直接连接引入远程服务、第三种是通过注册中心引入远程服务。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrmmOgiaU3K060l9w9mWbUOHYtkYUN5CgUFPOZ2NpUK6CT8SlDC8Yzlsg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**本地引入**不知道大家是否还有印象，之前服务暴露的流程每个服务都会通过搞一个本地暴露，走 injvm 协议（当然你要是 scope = remote 就没本地引用了），因为**存在一个服务端既是 Provider 又是 Consumer 的情况，然后有可能自己会调用自己的服务**，因此就弄了一个本地引入，这样就避免了远程网络调用的开销。

所以**服务引入会先去本地缓存找找看有没有本地服务**。

**直连远程引入服务**，这个其实就是平日测试的情况下用用，不需要启动注册中心，由 Consumer 直接配置写死 Provider 的地址，然后直连即可。

**注册中心引入远程服务**，这个就是重点了，Consumer 通过注册中心得知 Provider 的相关信息，然后进行服务的引入，这里还包括多注册中心，同一个服务多个提供者的情况，如何抉择如何封装，如何进行负载均衡、容错并且让使用者无感知，这就是个技术活。

**本文用的就是单注册中心引入远程服务**，让我们来看看 Dubbo 是如何做的吧。

## 服务引入流程解析

默认是懒汉式的，所以服务引入的入口就是 ReferenceBean 的 getObject 方法。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrUbHyCWZvXrQWJlz2gdvjEL3ZD6G4mibcofjIAmPgALtgHdiaqAX2BxIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到很简单，就是调用 get 方法，如果当前还没有这个引用那么就执行 init 方法。

## 官网的一个小问题

这个问题就在 `if (ref == null)` 这一行，其实是一位老哥在调试的时候发现这个 ref 竟然不等于 null，因此就进不到 init 方法里面调试了，后来他发现是因为 IDEA 为了显示对象的信息，会通过 toString 方法获取对象对应的信息。

toString 调用的是 AbstractConfig#toString，而这个方法会通过反射调用了 ReferenceBean 的 getObject 方法，触发了引入服务动作，所以说到断点的时候 `ref != null`。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdr6lECMvsYUAm5HASdDvKHY6dqNQsKAjgBfVTO9L4vdqzqU821j6VJJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到是通过方法名来进行反射调用的，而 getObject 就是 get 开头的，因此会被调用。

所以这个哥们提了个 PR，但是一开始没有被接受，一位 Member 认为这不是 bug， idea 设置一下不让调用 toString 就好了。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrtmiaBh3JJicSqiaq7Y3MQ3fCBPm3IMYdgbL1GhOuWJIOuvpFYlAXBeLug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

不过另一位 Member 觉得这个 PR 挺好的，并且 Dubbo 项目二代掌门人北纬30也发话了，因此这个 PR 被受理了。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrfbVhTAyBSA498I6DQgwr5lQW28ROaiaDy8HzvuL95cd0ibArAe9mrAWg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

至此我们已经知道这个小问题了，然后官网上其实也写的很清楚。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrSMN3u8xlGpu8eLt1Y5Tia5AdqOBYp5YdpFx5j2VrM8k9832u113FyDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但是小问题来了，之前我在文章提到我的源码版本是 **2.6.5**，是在 github 的 releases 里面下的，这个 tostring 问题其实我挺早之前就知道了，我想的是我 2.6.5 稳的一批，谁知道翻车了。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrMG4PB5wrVNuoYR9bLK9j2hCbA7xsZhrGDDc21gAqrMoVYIja4IniaVQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我调试的时候也没进到 init 方法因为 ref 也没等于 null，我就奇怪了，我里面去看了下 toString 方法，**2.6.5**版本竟然没有修改？没有将 getObject 做过滤，因此还是被调用了。

我又打开了**2.7.5**版本的代码，发现是修改过的判断。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdr1PrUJ565qicL524YajjJl4CdEpwiaX55xBM1E0E8VW5v0ibBKhBrPibwKQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我又去特意下了 **2.6.6** 版本的代码，发现也是修改过的，因此这个修改并不是随着 **2.6.5** 版本发布，而是 **2.6.6**，除非我下的是个假包，这就是我说的小问题了，不过影响不大。

其实提到这一段主要想说的是那个 PR，作为一个开源软件的输出者，很多细节也是很重要的，这个问题其实很影响源码的调试，因为对代码不熟，肯定会一脸懵逼，谁知道是不是哪个后台线程异步引入了呢。

提这个 PR 的老哥**花了两个小时才搞清楚真正的原因**，所以说虽然这不是个 bug 但是很影响那些想深入了解 Dubbo 内部结构的同学们，这种改配置去适应的方案是不可取了，还好最终的方案是改代码。

好了让我们回到今天的主题，接下来分析的就是那个不让我进去的 init 方法了。

## 源码分析

init 方法很长，不过大部分就是检查配置然后将配置构建成 map ，这一大段我就不分析了，我们直接看一下构建完的 map 长什么样。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrIwVhwDSUcObSHGDSczzPxzrBf1qStwhgFKxoJB4PicYhdI6tHbjMTBQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后就进入重点方法 createProxy，从名字可以得到就是要创建的一个代理，因为代码很长，我就**一段一段的分析**。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrCZDOy0jWeNn2geruN8EMTAWDh37aLNY4LHMrU98zEJyj6ebsq1I6KA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果是走本地的话，那么直接构建个走本地协议的 URL 然后进行服务的引入，即 refprotocol.refer，这个方法之后会做分析，本地的引入就不深入了，就是去之前服务暴露的 exporterMap 拿到服务。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrI8XQw98uGiaRpRZPzon40Z5QG2CbUMiaeh33H9mSubYxX88IGodadZrA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果不是本地，那肯定是远程了，接下来就是判断是点对点直连 provider 还是通过注册中心拿到 provider 信息再连接 provider 了，我们分析一下配置了 url 的情况，如果配置了 url 那么不是直连的地址，就是注册中心的地址。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrLQD22JzNlt4H9G6ucIfWuAN4ctqg120FfPSbbjylD2gNpm9lRVoD4Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后就是没配置 url 的情况，到这里肯定走的就是注册中心引入远程服务了。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrs9XjIjpL5kibfibLiaBVZ6OA96zRD94KyCussZYvzibqd7ibGrR44xW8HDw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最终拼接出来的 URL 长这样。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrHMTHpA73Z90DOJQ9K1Baaicicn9YHqzgew5rao4A6IJPo3oZSxdlojxQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到这一部分其实就是根据各种参数来组装 URL ，因为我们的自适应扩展都需要根据 URL 的参数来进行的。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdr86F9oNnOozhV7MhENfAQqq22rzoaQK1Qicax0TggDaXsedgqhj8l24g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

至此我先画个图，给大家先捋一下。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrlyHedol8mkDRbeecs6TibqBibWQ4NAlmXT8CStgS4Z1ZJXut3XdqYJRg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这其实就是整个流程了，简述一下就是先检查配置，通过配置构建一个 map ，然后利用 map 来构建 URL ，再通过 URL 上的协议利用自适应扩展机制调用对应的 protocol.refer 得到相应的 invoker 。

在有多个 URL 的时候，先遍历构建出 invoker 然后再由 StaticDirectory 封装一下，然后通过 cluster 进行合并，只暴露出一个 invoker 。

然后再构建代理，封装 invoker 返回服务引用，之后 Comsumer 调用的就是这个代理类。

相信通过图和上面总结性的简述已经知道大致的服务引入流程了，不过还是有很多细节，比如如何从注册中心得到 Provider 的地址，invoker 里面到底是怎么样的？别急，我们继续看。

从前面的截图我们可以看到此时的协议是 registry 因此走的是 RegistryProtocol#refer，我们来看一下这个方法。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdromJ8WP6SvDqslKyibk7Nlg91vmJZCzO4K1t1Uia4NCWcBfC5qC68fzdw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

主要就是获取注册中心实例，然后调用 doRefer 进行真正的 refer。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrZdgkbWZlRjPhgH4sWlicrXF6MvvJ8Ch860Gw0PzPTwZOQsOWQpMbwgg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个方法很关键，可以看到生成了`RegistryDirectory` 这个 directory 塞了注册中心实例，它自身也实现了`NotifyListener` 接口，因此**注册中心的监听其实是靠这家伙来处理的**。

然后向注册中心注册自身的信息，并且向注册中心订阅了 providers 节点、 configurators 节点 和 routers 节点，**订阅了之后 RegistryDirectory 会收到这几个节点下的信息，就会触发 DubboInvoker 的生成了，即用于远程调用的 Invoker**。

然后通过 cluster 再包装一下得到 Invoker，因此一个服务可能有多个提供者，最终在 ProviderConsumerRegTable 中记录这些信息，然后返回 Invoker。

所以我们知道`Conusmer` 是在 RegistryProtocol#refer 中向注册中心注册自己的信息，并且订阅 Provider 和配置的一些相关信息，我们看看订阅返回的信息是怎样的。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrMPHw3C3zPow0ODKgSCduespHTUUodL6EnPsjZzu8nGJ2GSguhGzAng/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

拿到了`Provider`的信息之后就可以通过监听触发 DubboProtocol# refer 了（具体调用哪个 protocol 还是得看 URL的协议的，我们这里是 dubbo 协议），整个触发流程我就不一一跟一下了，看下调用栈就清楚了。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdruxQeyeTV9Z4m3PqB18z4BuKbCM8guHTuZGVDkH9Zr1aN9OmnjlibUibg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

终于我们从注册中心拿到远程`Provider` 的信息了，然后进行服务的引入。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrGjs4RiavT2BF6Ayoib2JgUBrpe4mYVjVLrOcl3Fwib1u6TWA5sgicH3kOw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里的重点在 `getClients`，因为终究是要跟远程服务进行网络调用的，而 getClients 就是用于获取客户端实例，实例类型为 ExchangeClient，底层依赖 Netty 来进行网络通信，并且可以看到默认是共享连接。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrG8bia5sRKH3GooPo03qhFmRF7grN51m7NNUs8k3jnQDH7P0Mw8FTibaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`getSharedClient` 我就不分析了，就是通过远程地址找 client ，这个 client 还有引用计数的功能，如果该远程地址还没有 client 则调用 initClient，我们就来看一下 initClient 方法。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

而这个`connect`最终返回 `HeaderExchangeClient`里面封装的是 `NettyClient` 。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

然后最终得到的 `Invoker`就是这个样子，可以看到记录的很多信息，基本上该有的都有了，我这里走的是对应的服务只有一个 url 的情况，多个 url 无非也是利用 `directory` 和 `cluster`再封装一层。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrTVRmAU5kjbVvDAjorC4gYoibeOkxNeaO4WbsE6DEktRRezRqStz99AA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最终将调用 `return (T) proxyFactory.getProxy(invoker);` 返回一个代理对象，这个就不做分析了。

到这里，整个流程就是分析完了，不知道大家清晰了没？我再补充前面的图，来一个完整的流程给大家再过一遍。

![img](https://mmbiz.qpic.cn/mmbiz_png/uChmeeX1Fpz5d8nUSmg6LId8fd5ibJPdrpk1g5xYFUKsBNiaXQPibnYicl8bhiawpzKlgloDmoAUEZCPSpTDiaWPSkzg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 小结

相信分析下来整个流程不难的，总结地说无非就是通过配置组成 URL ，然后通过自适应得到对于的实现类进行服务引入，如果是注册中心那么会向注册中心注册自己的信息，然后订阅注册中心相关信息，得到远程 `provider`的 ip 等信息，再通过`netty`客户端进行连接。

并且通过`directory` 和  `cluster` 进行底层多个服务提供者的屏蔽、容错和负载均衡等，这个之后文章会详细分析，最终得到封装好的 `invoker`再通过动态代理封装得到代理类，让接口调用者无感知的调用方法。

## 最后

今天这篇文章看下来相信大家对服务的引入应该有了清晰的认识，其实里面还是很多细节我没有展开分析，比如一些过滤链的组装，这其实在服务暴露的文章里面已经说了，同样服务引用也有过滤链，不过篇幅有限就不展开了，抓住主线要紧。

至此我已经带大家先过了一遍 `Dubbo` 的整体概念和大致流程，介绍了 `Dubbo SPI`机制，并且分析了服务的`暴露流程`和`服务引入`流程，具体的细节还是得大家自己去摸索，大致的流程我都讲的差不多了。

dubbo系列也快接近尾声了，虽然我知道每次写硬核技术看的小伙伴就少了很多，但是还是想写完这个系列，感谢大家的支持。
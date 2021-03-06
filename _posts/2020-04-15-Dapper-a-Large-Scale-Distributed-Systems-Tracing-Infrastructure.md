---
title: Dapper, a Large-Scale Distributed Systems Tracing Infrastructure
tags: Dapper
key: Dapper
---

形式上，我们的Dapper traces使用trees（树形结构）、spans、annotations（标注）表示。

##### 2.1 Trace trees and spans

在一个Dapper trace tree中，每个树节点都是一个基本工作但愿，我们称之为s pans。每条边都表示一个span和他父span的随机关系。span本身是一个时间戳记录日志，并与span在trace tree中的位置无关，日志中编码了这个span的开始时间和结束时间。其他RPC相关的时间数据和零到多个基于特定应用程序相关的annotations我们将在2.3节讨论。

图二展示了spans是如何形成一个更大的trace树形结构。Dapper为每个span记录一个人类可读的*span name*、*span id*和*parent id*，以便再将各独立spans重建为一次独立的分布式trace。没有*parent id*的spans称为*root spans*。所有关联在同一个trace的spans共享一个*trace id*（没有在图中展示）。以上这些id都使用全局唯一的64位整型表示。在一个典型的Dapper trace中，我们将每一个RPC与一个span对应，基础实施的每多一层就将trace tree的深度增加一层。

图三展示了一个典型Dapper trace span记录日志的详细信息。这具体的span是图二中同样名为“Helper.Call“的两个span中较长的那个。==span的起始和结束时间以及任何RPC的时间信息都通过Dapper对RPC library 植入被记录下来。==如果应用程序的拥有者选择在trace中增加他们自己的注释（例如图中的“foo”），这些注释同样会和其他的span数据一起被记录下来。

需要特别注意的是，一个span会包含多个hosts的信息，事实上每个RPC span包含的标注来自client和server两个过程，这使得两个主机被span关联起来。既然client和server的时间戳来自不同主机，我们需要去考虑时钟偏差问题（即两个机器的机器时间并不一致）。在我们分析工具中，我们利用了RPC client的发送请求一定是在RPC server收到请求之前这个事实（反过来，server response也是同理）。这样一来，我们就限制主了RPC中server一方span时间戳的上下界。

##### 2.2 植入点

Dapper通过几乎完全依赖对很少的通用library进行植入，使得可以在对应用程序开发者近乎零干扰的程度下，实现对分布式控制路径进行跟踪：

- 当一个线程在处理被跟踪的控制路径时，Dapper会将*trace context*附在thread-local storage上。一个*trace context*就是一个小型容易复制的容器，里面包含了span的属性，例如*trace id*和*span id*等等。
- 当计算过程被延迟或者异步执行时，多数Google开发者会在线程池或者其他执行器中，使用一个通用的control flow library回调这些线程。Dapper可以确保所有这样的回调可以存储*trace context*，以及当回调被触发时，这个*trace context*被关联在合适的线程上。以这种方式，Dapper可以透明的跟踪异步控制路径使其重建trace tree。
- 几乎所有Google的进程间通信都是建立在一个用C++和Java开发的RPC框架。我们将trace植入这个框架来定义所有RPC中的spans。对于被跟踪的RPC，*span id*和*trace id*会从client传到server。因为像这样基于RPC的系统被广泛使用在Google中，所以这是一个重要的植入点。我们计划对非RPC通讯框架进行植入，当他们继续演化并有自己用武之地时。

Dapper跟踪数据时独立与语言的，很多在生产中的跟踪结合了使用C++和Java编写的进程的数据。在3.2节中我们会进一步讨论这种应用透明度在实践中是如何实现的。

##### 2.3 Annotations

以上描述的植入点对推导出一个复杂分布式系统的trace是充足的，可以使Dapper的核心功能可以在不修改Google应用程序的基础上可用。然而，Dapper也支持应用程序开发者使用额外的信息来丰富Dapper的跟踪，这对于监控更高级别的系统行为或者帮助debug故障是非常有用的。我们允许用户通过一个简单的API定义带时间戳的annotations。图四是这个API的一个使用用例。这些annotations可以是任何内容。为了保护Dapper用户意外的过于热衷于添加标注，每一个span都有可以配置annotations的总量上限。应用程序级别的annotations不能取代那些用于重建trace tree的span或者RPC信息。

除了简单的文本annotations，Dapper也支持键值对映射的annotations以提供给开发人员更强的跟踪能力，例如维护一个计时器、记录二进制消息和在一个进程上传输任意用户定义的数据。这些键值对annotations被用来在分布式跟踪的上下文中定义特定于应用程序的等价类。

##### 2.4 Sampling

低能耗是Dapper的一个关键设计目标，因为如果一个新工具会严重影响应用程序性能，服务的运营人员一定会不愿意部署使用它。更多的，我们希望允许开发者使用annotation API而无需担心额外的开支。我们还发现Web服务的一些类确实对植入点开销很敏感。因此，除了使把Dapper收集工作的植入开销限制的尽可能小之外，我们控制开销使用更多的方法就是，遇到大量请求时只记录其中的一小部分。我们将在4.4节详细的讨论跟踪采样机制。

##### 2.5 Trace colletion

Dapper跟踪记录和收集管道的过程分为三个阶段（见图五）。首先，span数据被写入(1)机器的本地日志文件中，然后Dapper的守护进程和收集组件将这些数据从生产环境中的主机中拉出来(2)，最终，(3)这些数据被写入到Dapper Bigtable仓库的一个cell中。每个trace被设计成Bigtable中的一行，每一列对应着一个span。Bigtable支持的这种稀疏表格布局正适合这种情况，因为每个独立的trace可以有任意多个span。跟踪数据收集的延迟，即应用中的二进制数据传输到中心仓库所用时间的中位数小于15s。The 98th percentile latency随着时间的推移呈现双峰型；大约75%的时间，The 98th percentile collection latency小于2min，但是大约25%的时间，他会增长到几个小时。

Dapper还提供了可以简化从我们的仓库中访问跟踪数据的API。Google的开发者使用这个API来构建通用的和基于特定应用程序的分析工具。5.1节包含了如何使用它的详细信息。

###### 2.5.1 Out-of-band trace collection

带外数据：传输层协议使用带外数据（OOB）来发送一些重要的数据，如果通信一方有重要的数据需要通知对方时，协议能够将这些数据使用特殊的通道快速地发送给对方。

正如描述的那样Dapper系统和请求树本身实现跟踪日志和收集带外数据。这样做时为了两个不相关的原因。首先，in-band收集机制，即跟踪数据以RPC响应头的形式被传回，会影响应用程序的网络动态。在许多Google大型系统中，一个trace包含了上千个spans是很常见的。然而，RPC responses的大小即使是接近这样大型分布式trace的根节点，也还是非常小的：通常小于10kb。在这种情况下，in-band Dapper跟踪数据会使应用数据相形见绌，并使后续的分析结果产生误差。其次，in-band收集机制假设了所有的RPC都是完美嵌套的。而我们发现，有许多中间件系统会，在所有后端返回最后结果前，将结果返回给它的调用者。in-band手机系统不能处理这种非嵌套的分布式执行模式。

##### 2.6 Security and privacy considerations

记录一定量的RPC有效负载信息会丰富Dapper的跟踪能力，因为分析工具能够在有效负载的数据（方法传递的参数）中找到相关的patterns，这些patterns可以解释被监控的系统为何会表现异常。然而，有些情况下，有效负载数据可能包含了一些不应该透露给未经授权用户（包括正在debug的工程师）的内部信息。

由于安全和隐私的考虑不能被忽视，Dapper虽然存储RPC方法的名字但是不记录在这个时候不记录任何有效负载。相反，应用程序级别的annotations提供了一个方便的可选机制：应用程序开发者可以选择在span中选择关联那些以后可以为分析提供价值的数据。

Dapper还提供一些安全上的便利，这是它的设计者都有事前意识到的。通过跟踪公开的安全协议参数，Dapper可以用来监控是否应用程序是否满足安全政策，例如，通过合适级别的认证或加密政策。Dapper也可以提供用于确保基于策略的孤立系统按预期执行的信息，例如，包含敏感数据的应用程序不应与未经授权的系统进行交互。这种类型的度量提供了比源代码审核更强大的保障。


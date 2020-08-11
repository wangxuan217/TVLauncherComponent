# TVLauncherComponent
## 信息流组件化之路

#### 需求背景

在电视内容日益丰富的今天，固定的展示方式已满足不了当前的市场需求。例如我们经常使用的淘宝、京东等以内容为基础的项目都喜欢给自己造节(双11、618)，几乎一年四季都会有营销活动，这些活动能带动GMV持续造血。一场大促，通常会分预热期和正式期。预热期用来造势，着重透出主会场、活动等内容；正式期则在接近尾声时，着重透出倒计时内容增强紧迫感。可以看出，从预热期到正式期，着重透出的内容不同，结构也不同。也就是说，`需要足够灵活的页面模板，满足不同时间，不同人群（如多人多面）展示不同结构的页面`。所以我们立足智能电视行业，也应该考虑让内容真正的流动起来。

#### 多结构化的页面存在的问题

首先，在view上的性能消耗通常有以下几种：

- 布局嵌套导致多重measure/layout

可以使用`ConstraintLayout`减少布局嵌套（但在item-view中使用`ConstraintLayout`反而会影响性能）

- view的频繁创建与销毁

列表使用`RecyclerView`来复用布局

- xml转换成view解析过程产生的内存和耗时

如果列表的样式不多，使用`RecyclerView`的复用机制可以避免大量的xml解析；如果样式比较多比如推荐图墙、活动等，则有必要把xml解析提前到编译期，在编译期根据注解将xml转成对应的view类，直接使用view类创建viewHolder，当然这么做会势必会增大包体积，需要克制使用。为了解决这些问题，我们考虑使用vlayout来简化工作量。

#### 什么是vlayout ?

vlayout 是手机天猫 Android 版内广泛使用的一个基础 UI 框架项目 提供了一个用于RecyclerView的自定义的LayoutManger，可以实现不同布局格式的混排，目标是支撑客户端native页面的快速开发（h5退出了直播间）。

在风行TVLauncher项目的设计中已经有这样一个借鉴，我们发现首页所有的fragment都可以抽象成一个`RecyclerView` ,并且所有的`LayoutManager  `都可以用一个 `GridLayoutManager` 来描述，这里只是借鉴了vlyout中`LayoutHelper`中的一个分支，使得vlayout中的 `LayoutManager `直接指向我们自定义的 `GridLayoutManager` 。或许是对valyout应用在TV场景下的一种的精准裁剪。

#### vlayout是如何进行布局的 ？

整个页面树被解析出卡片+组件的数据列表之后，会对块数据做进一步转换。首先提取所有组件model，也就是将组件都打平到同一级别的组件，这个组件会被传递给`RecyclerView`的`Adapter`，因此数据的位置其实就对应了`RecyclerView`看到的组件位置。而卡片model，将会拿来构建一个个`LayoutHelper`，这些`LayoutHelper`是负责具体布局的对象，一种布局类型的卡片对应于一种`LayoutHelper`，而且`LayoutHelper`还包含了它负责的组件的位置起始区域，它们会被传递给自定义的`LayoutManager`。当`RecyclerView`开始渲染页面或者滑动时，它内部维护了一个布局状态，获取当前屏幕范围内还有多少区域是空白的，下一个要加载的`View`的位置是多少，然后把这些信息告诉`LayoutManager`去加载`View`做布局。我们的自定义`LayoutManager`拿到这个位置之后，就反向查找对应的`LayoutHelper`，然后交给`LayoutHelper`去布局，这个过程还会涉及到从回收复用池或者通过`Adapter`获取一个组件实例。不同的`LayoutHelper`会按照约定的协议进行进一步布局。

即使我们对vlayout的裁剪能够满足现有的需求，在业务层面，我们还是面临了在主工程中维护模板带来的工作量，为后续组件化分包也带来很多困难。然后我们需要去考虑能不能在vlayout上层对业务进行解耦。

#### 什么是tangram ？

`Tangram`的意思是七巧板，旨在用七巧板的方式拼凑出各式各样的页面。他抽象了两个概念，`Card`和`Cell`，`Card`用于描述布局方式，`Cell`用于描述在这个布局方式下，用什么样的view去展示，这样的抽象会使得我们描述这个功能暂时不用不关心他的数据是如何来的。

tangram架构图

![](https://raw.githubusercontent.com/yuugu/picGo/master/tangram-img/tangram-scaffold.png)

1.整个 Tangram 框架的页面UI搭建基于 vlayout 和 UltraViewPager。vlayout 刚刚已经提到过了，用来构建多类型、多布局类型的 RecyclerView；[UltraViewPager](https://github.com/alibaba/UltraViewPager) 是 ViewPager 的一种扩展，整合了多种特性，比如横向滑动，竖向滑动，循环滚动等，用来实现 Tangram 内部所需要的轮播滚动卡片；

2.vlayout 主要提供了一个自定义的 LayoutManager，因此 Tangram 还需要提供一个 RecyclerView 和 Adapter 来才能配合 vlayout 运行。这里的 RecyclerView 可以由外部业务方通过 TangramEngine 注入，也可以内部默认构造。GroupBasicAdapter 则封装了 vlayout 所需要的 Adapter 的逻辑，组件的创建、组件数据的绑定、组件类型的定义等都由它负责。

3.在 UI 基础之上的便是各种功能逻辑模块：

- TangramEngine 是核心类，它负责绑定 RecyclerView 到底层 vlayout，绑定页面数据，操作页面数据（包括增、删、改），还提供注册外部服务的接口。
- ServiceManager 是服务管理模块，不轮是内部还是外部功能模块，都可以注册到这里，一方面能被 Tangram 内部的其他模块访问到使用，另一方面解耦了框架与业务模块。
- Bus 是事件总线（其实只建议在框架内部使用bus），它在内部也被注册到 ServieManager，内部模块和业务使用方都可以使用它进行通信，解耦业务代码。
- DataParser 负责解析数据，它将原始数据解析成卡片、组件的 model 对象。框架里提供的是解析 JSON 数据的解析器，也支持扩展解析其他类型的数据。
- DataResolver 负责识别卡片、组件并构建对象，解析器解析数据的时候，需要依赖这些 Resolver去识别数据中的卡片或者组件是否合法，Resolver 识别的方式就是去组件库或者卡片库里寻找这些组件是否已经注册过。

与业务相关性较大的就是组件库、卡片库以及相关业务接口。TangramBuilder 是业务方构建 TangramEngine 的入口。组件库里注册了业务方所需有的组件，Tangram 的实例是一个页面一份，因此每个业务方可以分别注册各自所需要的组件，当业务方使用 Tangram 进行业务开发的时候，主要工作可能就在组件的开发上。卡片库注册的是卡片类型，框架里已经内置了一系列卡片，如果业务方有需要可以单独再注册特殊类型的卡片。而 ClickSupport、ExposureSupport 等都是辅助业务开发的功能模块，前者定义了组件点击处理的接口，后者定义了组件曝光处理的接口。它们都被注册到 ServiceManager 里，业务方在组件或者页面内都可以使用它们。

Tangram的初始化如图：

![](https://raw.githubusercontent.com/yuugu/picGo/master/tangram-img/Tangram-init-pic.png)

Tangram的运行流程：

![](https://raw.githubusercontent.com/yuugu/picGo/master/tangram-img/Tangram-start-pic.png)

1. 整个 Tangram 对界面的动态调整是通过数据来驱动的，所以首先要将原始数据传递给 TangramEngine，由于项目内接口都采用 JSON 数据，Tangram 框架的默认设计也是接收 JSON 格式的数据，不过也支持通过自定义 DataParser 提前将其他格式的数据解析好之后再传给 TangramEngine。以TVLauncher中 JSON 数据为例，一个页面下，挂载了一个卡片数组，每个卡片都定义了 id、type、items节点；items 内部的数组定义的是组件数据，组件也有type、bxxId 等业务字段数据。

2. 不论是传递原始 JSON 数据给 TangramEngine还是通过直接解析原始数据，都是通过 DataParser 来完成的，它会按照树型结构解析出对应的卡片和组件的 model 对象，解析过程依赖于相应的卡片 Resolver 和组件 Resolver 来识别卡片、组件是否已注册，关键点就是识别 type 字段。若碰到无法识别的 type，则不会解析出对应的 model 对象。

3. 解析完成之后会得到一个卡片列表，每个列表的卡片 model 元素里持有它所包含的组件列表。

4. model 列表交给 GroupBasicAdapter 进行处理，首先提取卡片列表，将包含空组件列表的卡片过滤掉，因为它没有东西可以渲染展示，然后创建出 vlayout 所需要的 LayoutHelper 列表，设置它们的样式属性，这样就打通了通过 JSON 数据最终控制布局排版的流程。

5. 同时将所有的组件 model 提取出来成为一个独立的列表，真正交给 GroupBasicAdapter 去渲染数据，组件 model 列表的大小就是 GroupBasicAdapter 的 item 的大小， RecyclerView 也就直接加载组件视图，卡片相对于只负责了布局逻辑的控制，并没有 UI 实体的承载。

6. 数据都准备完毕之后，RecyclerView 就驱动 vlayout 里的 LayoutManager 进行渲染和布局。

7. LayoutManager 首先回调 RecyclerView 内部获取 ViewHolder，若复用池里存在复用的对象，就回调 GroupBasicAdapter 进行数据绑定，否则先回调 GroupBasicAdapter 进行组件 ViewHolder 的创建，然后进行数据绑定。ViewHolder 的创建也是通过 Resolver 内部创建 UI 的模块进行构造。

   这就是 Tangram 渲染页面的整体流程，本身并没有特别复杂的逻辑。是对现有方案的解耦，也是组件化信息流模块的基石。

#### 还能怎么玩 ？

对当前业务整合之后，我们还会发现一些问题。静态的模板想要满足动态的需求绕不开的是新发版本（上述cell不够用）。虽然我们可以考虑热更新、热修复等一系列的手段但是在不同版本的Android平台上表现却不尽人意。无论是andifx方案对AMS的hook还是tinker拦截类加载器在各个平台的表现都差强人意。

针对我们的需求，我们需要的是一种动态下发view的能力。

一个简单的`xml`样式文件，直接把他下发到客户端存在两个问题，一是冗余字符引起的带宽浪费，二是客户端解析耗时和内存，在用户手机内存吃紧时，面对一个样式繁多的`RecyclerView`时，即便存在复用机制也可能因解析引起oom。

那么我们能否约定一种数据格式，每一块分别展示什么信息，如下

![](https://raw.githubusercontent.com/yuugu/picGo/master/tangram-img/vt-view.png)

比如，开头有`版本区`，后面有`组件区`、`组件长度区`、`字符串区`、`字符串长度区`、`表达式区`、`表达式长度区`...这有点像`JVM`校验解析字节码的过程。（其实Google也有使用过类似的方式去进行传递一些期望)

`颜色：转换成4字节整型颜色值，格式 AARRGGBB；`

`枚举：按照预定义的整数转换，比如 gravity 的类型，orientation 的类型；`

`字符串：以 hashCode 值作为它的序列化后整数，并在字符串资源区建立以 hashCode 为索引的列表，在解析的时候从中获取原始的字符串值；`

`逻辑表达式：与字符串的处理类似；`

`数字：直接转换成 4 字节的整型或者浮点型，并支持带单位的类型 `

字符串用hashCode值为索引的列表方案，可以节省重复字符串的空间，表达式是用来绑定动态数据如`${text}`。

#### 什么是VirtualView ？

简单讲，就是我们实现了一系列自定义控件，建立通过自定义 XML 方式引用这些控件来搭建 UI 视图，然后通过引擎解析 XML 数据并渲染出界面的方案。就好比在 Android 里写 XML 布局文件然后渲染展示，或者写 HTML 文件在浏览器里渲染展示这两种方式。

### 使用 VirtualView 开发一个组件

大概需要这么几个过程：编写模板 —— 编译模板 —— 下发到客户端 —— 渲染；

1. 首先通过[virtualview_tools](https://github.com/alibaba/virtualview_tools/blob/master/README-ch.md)编写模板
2. 编译模板，上文提到的引擎加载 XML 并不是直接加载原始 XML 文件，而是先通过 [virtualview_tools](https://github.com/alibaba/virtualview_tools/blob/master/README-ch.md) 编译成一段二进制数据，后缀为 `.out`。
3. 下发到客户端，前两个步骤都是在客户端运行时之外进行的，这里的下发到客户端有两种含义，一种是直接将编译结果打包到客户端里加载，另一种是发布到 cdn 上，让客户端去下载。
4. 渲染，方案引擎会加载这份二进制数据，并绑定数据渲染出来。


#### 更深度的思考

在这些前提下，我们离组件化的业务剥离更进一步了。那我们又可以考虑到是否组件化通信在面向接口编程的基础上能否比市面上的方案更契合TV的场景。未来我们可能在主工程中负责各个流程的Gradle脚本，子模块暴露自己所提供的服务接口，在各自内部进行实现，然后模块之间假如存在依赖的初始化顺序又当如何处理.....我们的路可能才刚刚开始。可能市面上大多数的开源框架都会给我们提供一种思路，我们需要探寻的是更加适合我们场景的一些架构设计。正如《重构》一书中所说：简单和平衡是最好的设计。在众多开源项目中都可以窥见。

### 总结：

1.使用`Tangram`来解耦当前业务与信息流展示的耦合，为组件化做准备。

2.使用`VirtualView`来实现view动态下发需求。

## Future

1.使用mmap+前置流式压缩策略优化日志上报

2.使用mmap替换SharedPreferences

3.组件化通信与生命周期的管理

4.EventBus不可追溯的问题

.....

ps:该文档是我在8月公司技术交流会上所陈述的总结，已经删除了对业务的描述，不涉及内部机密。

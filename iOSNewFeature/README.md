## iOS WidgetKit概览
### 背景
WidgetKit 是 WWDC2020 备受关注的新特性，支持在 iOS、iPadOS 以及 macOS 展示动态信息和个性化内容。Apple 对 iOS 主屏幕的改进一直保持克制，Widget 的加入无疑是一次比较大的破局。Widget 如何提供一致的跨平台体验？如何提供实时的个性化内容？如何引入 Widget 同时最大可能减少主屏幕的性能、电量开销？本文将参照苹果官方文档，快速了解一下上面的内容。

### 什么是Widget？
Widget是动态和个性化的窗口小组件，而它的本质是一个随着时间线而更新的 SwiftUI View。**它应该是将 APP 重要内容投射到主屏幕上的一种方式，而不是充满各种小按钮的 Mini APP **

#### 优秀 Widget 的三要素
- Glanceable 一目了然
- Relevant 高相关性
- Personalized 个性化
#### 为什么需要Widget
用户每天平均打开手机屏幕次数为90次，每次只会停留几分钟（数据来源：Meet WidgetKit 5:19）。 Widget 最小的 size 也是 APP 图标的4倍。在寸土寸金的手机桌面上，一个能提供重要信息、快速跳转、及时刷新、支持配置的的 Widget 可能成为用户进入 APP 的一个重要入口。
#### Widget功能
##### 配置类型
- 静态配置 (Static configuration): 显示内容固定，不需要/不允许用户自己设置
用户不能通过编辑小组件去操作 Widget 显示的内容，但是依然可以通过数据的差异化来驱动展示差异化的界面
- 动态配置 (Intent configuration): 用户可以自己选择当前 Widget 显示的内容
用户可以通过编辑小组件来对 Widget 的显示内容进行配置，并且可以差异化的数据来驱动差异化界面展示

配置类型更多请看：Making a Configurable Widget

##### 大小支持
- systemSmall
- systemMedium
- systemLarge

#### Widget交互
##### 较为傻瓜的 UI
- Widget 不是迷你 APP
- 不支持滑动
- 不支持视频、动态图像和动画
##### 快速触达
- 操作简单，点触区域内任何地方即可触发
- 点触动作直接联入 APP 内对应的页面（通过 widgetURL）

Widget 为用户提供一目了然的内容展示，在用户需要进一步查看或编辑的时候，通过轻点启动 APP 完成顺畅的体验。
用户点击 Widget 中的特定区域后，通过 Deep links的功能，APP 启动后快速展示对应内容，不仅把重要信息展示在里屏幕上，并且提供了快速打开指定内容的方式。

值得注意的是：systemSmall只可以作为一个整体的点击区域，systemMedium、systemLarge可以触发多个操作

### Widget工作原理
Widget 作为一个 Extension 存在，是独立在 APP 进程之外的存在。Widget 是由 Timeline 驱动，TimelineProvider 给 Extension 提供 Timeline Entry 和 Widget 显示的快照；

而时间线是关联时间节点的数据集合，Extension 会跟进时间去更换不同的数据来渲染界面；如下图：

开发者通过 SwiftUI 构建 Views，定义 Timelines 为 Views 提供对应时间所需数据。数据变化时，通过reload 更新数据。
WidgetKit 统一管理多个 Widgets：序列化 Widget Views 和 Timelines 数据，处理预渲染和复用，响应并处理 System reloads、App-driven reloads，为桌面、Widget Gallery预览等场景提供高效、即时的 Widget 体验。
值得一提的是：**WidgetKit 会把 Timelines 所定义的 Entries 对应的 Views 结构信息缓存到磁盘，仅在需要的时候实时渲染特定的 View。这使得系统可以在极低电量开销下为众多 Widgets 处理 Timelines信息。**
#### Timeline刷新机制
项目中的 TimelineProvider 给 Timeline 提供的 Entry 数量是有限的；也就是说在一天内只有一部分有限的时刻 Widget 会根据 TimelineProvider 中的计划刷新显示内容，而当 Entry 消耗殆尽后 Widget 中的显示内容将不再会根据时间而变化了。
那么，当 Timeline 走完，Widget 是怎么继续刷新的呢？
答案是：**Timeline Reload**
##### Timeline Reload分为两种：
- System reloads
  - atEnd
这个就是在 TimelineProvider 提供的所有 Entry 显示完毕之后自动刷新，也就是说只要还有没有显示的 Entry 在就不会刷新当前时间线

  - after(date)
date 这个参数可以是某一个特定时间值，在系统时间到达那一时刻后会使 Timeline 自动刷新

  - never
这个就是字面意思，永远不会刷新 Timeline，最后一个 Entry 也展示完毕之后 Widget 就会一直保持那个 Entry 的显示内容

- App-driven reloads
当应用在后台时，后台推送可以触发 reload；当应用在前台时，应用可以主动触发 reload。进而通过 reload 变更全部或特定 Timeline Entry。
甚至，其他 Extension 也可以刷新 Widget，比如收到 NotificationServiceExtension
这里需要说明的是：
- **在 System reload 的情况下**，Timeline Reload 的时机是由系统统一控制的，而为了保证性能，系统会**根据各个 reload 请求的重要等级**来决定在某一时刻是否按照  APP 要求的刷新时机来刷新Timeline。因此如果过于频繁的请求刷新 Timeline，很有可能会被系统限制从而不能达到理想的刷新效果。换句话说，上面所说的 atEnd, after(date) 中定义的刷新 Timeline 的时机可以看作刷新 Timeline 的最早时间，而根据系统的安排，这些时机可能会被延后。
- 在App-driven reloads的情况下，Widget 会立即刷新，不存在延迟。
Timeline相关更多请看：Keeping a Widget Up To Date
在实验中，
在理想环境下（模拟器、电量充足、连接Xcode），Timeline 刷新的极限频率是1min，当刷新时间小于1min时，系统会统一处理为1min；
在真机正常环境下，Timeline 的刷新极限频率在5min，且会存在2min左右的误差；
Timeline Entry 的最小时间间隔暂未测试；
真机非理想环境下（电量过低，优先级过低等）暂未测试；
Widget中网络连接
在 Widget 中，对于需要加载网络数据的情况可以通过 Background sessions 完成，相关的系统资源占用将计入 Widget 开销，由 WidgetKit 管理对应 Background reload 预算。
### 总结
以上内容，大部分来自 Apple 官方文档和WWDC2020，以及部分实验数据和个人理解；只包含了大致流程和原理，具体技术细节暂未列出，以及一些技术细节，需在实践中继续探索；Widget 设计理念和实现原理很棒，也是 Apple 近些年在手机屏幕上较大的更新，希望大家集思广益，多多创造有趣的 Widgets！



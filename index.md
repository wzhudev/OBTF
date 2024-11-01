# TODO

- [x] 购买透明胶
- [ ] 预约快递时间 2024/11/2 上午
- [ ] 把行李寄回绩溪 2024/11/2 上午
- [ ] 把要带去北京的东西带回家，联系德邦快递 2024/11/2 下午
- [ ] 把行李寄到北京 2024/11/3 上午
- [ ] 出发去北京 2024/11/3 12:55 G16 上海站

## Engineering TODO

- [ ] **EMERGENT** Reverse Linear Sync Engine. It shall be done before [2024/11/30].

## Univer TODO

- [ ] 根据 0.5.0 版本改动内容修改 starter kit 以及 examples
- [ ] 找个时间 review 重构之后的编辑器模块代码

Projects that I would like to read the code:

- Zed, for their rendering engine GPUI, and other interesting data structures’ (such as rope) implementations.
- Typst, for typesetting and rendering.

# Journals

可以等到公式相对引用完成之后再开始进行重构，本次重构公式模块的目标包括：
1. 提升公式模块的可维护性
2. 探索如何实现主线程和 worker 线程的调度和 fallback 机制
3. 尝试解决公式模块的性能问题
	1. 上层业务功能应用公式计算结果的性能问题

#### Univer 公式模块问题整理

整体架构设计问题

* `FeatureFormula` 和 `OtherFormula` 并没有区分这两个概念的必要，也不需要切分很多 module 来设置这两种 formula。
* 很多操作通过 mutation 来进行，这是种反模式
	* 应该提供一个操作 formula 底层模块的中介模块，并负责维护 worker 线程的创建，并将其他插件对公式系统的调用发送到正确的模块
	* 一些修改底层数据的逻辑应该走这个中介模块去 set 而不是通过 mutation
		* 滥用 mutation 会导致调用链路不清晰
* 没有在 worker 崩溃时的处理机制，不能 fallback 到主线程
* formula service 和 other service 在获取公式计算结果的时候需要全量遍历计算结果
* 模块划分过多，依赖关系过于复杂，内聚程度较低

部分模块设计问题

`ActiveDirtyManagerService` 主要用于让业务注册回调，来告诉公式系统某个 mutation 发生的时候可能的脏区是什么。改造点：

- [ ] 不需要作为抽象依赖
- [ ] 重命名为 `DirtyAreaRegistrationService`
- [ ] 暴露出的几个方法是不必要的
- [ ] `RuntimeService` 和 `CalculateFormulaService` 存在大量的单一调用，这两个类应该合并在一起 

`Interpreter` 对于 `FormulaRuntimeService` 的依赖主要在于获取公式运行状态，只需要把这个状态设置到一个底层的状态服务即可解除该依赖

代码风格问题

- [ ] 一些模块没有必要作为抽象依赖

滥用的 mutation 列表

* `SetArrayFormulaDataMutation`
	* trigger: `UpdateFormulaController`
	* listener: `ArrayFormulaCellInterceptorController` `CalculateController`
* `SetArrayFormulaDataMutation`
	* trigger
	* listener
* `RegisterFunctionMutation`
* `SetOtherFormulaMutation`
	* sender: `RegisterOtherFormulaService`
	* listener: `SetOtherFormulaController` `SetDependencyController`
* `OtherFormulaMarkDirty`
	* sender: `RegisterOtherFormulaService`
	* listener: `RegisterOtherFormulaService` (via `activeDirtyManagerService`)

换句话说，在调用的时候需要不区分本分计算还是 web worker 计算调用。


在 λ 公式进行计算的时候，需要传递词法作用域把上层的结果传递给它；和 [[刘洋]] 确认了一下，我们可以试试看

Lambda 公式相关的 node 主要有以下几个：
* lambda-node
* lambda-parameter-node
* lambda-value-object

横向公式引用关系 si，通过这个来复用公式的 node 等等
修改不必要的抽象依赖
移除掉一些未使用、非必要的 API
合并 feature formula 和 other formula 的赋值机制
实现中介接口层，移除掉对 mutation 的滥用，移除掉一系列 SetXXXXController
尽可能地少走 command.onCommandExecute
实现 web worker 线程的维护和 fallback 机制
修改 feature formula 和 other formula 接入计算引擎的方式，移除掉这两种 formula 的概念

#### 2024/11/1

Before 志祥 completes the bundling change and unblock us from updating starter kits and examples, I will just read the source of our formula engine. :cry:

#lexical #editor #oss Some design decisions I personally love with Lexical, Facebook’s open source editor framework:

- Not using the DOM as SSOT but its own state model. ProseMirror uses the first approach and I hate to assemble HTML pieces to get started with an editor.

Other design decisions:

- Mutable state. But mutable actions can only happen in the update callback.
- Double buffering. 

#minimalism #digital #workflow My One Big Text File is an interesting article. I’ll start today with VSCode (yes!) to try and use this method for my own workflow, to avoid sudden sunsets of third-party applications (e.g. Arc Browser and lots of products killed by Google) and information leaks. And also, it works well with my minimalism philosophy - No new application to learn, no new fancy features to get used to. Just plain text.

#### 2024/10/30

Got our first US visa today. #visa #us 

Linear Sync Engine #linear #sync-engine #collaboration
React Helsinki February 2020
Scaling the Linear Sync Engine
设计基本要点
使用 MobX 做前端的状态管理，触发组件自动更新。
数据对象实际存储的数据结构并不是一个 graph，而是一个对象池。从这个对象池中选取合适的对象够造成 graph 给视图层使用。
加载之后会把对象存储在数据库里。数据库名字用很多数据做了 hash。
通过 sync ID 来管理本地数据的版本以及和服务器做数据同步。
❓ 如何应对 schema change，比如对象的字段改变了，增加了新类型的对象 etc
Sync server 是性能瓶颈
严重超时的 client，会停下所有客户端的同步服务，把发送 delta sync 的服务拆解到另外一个服务器
严重超时的 client 会在 fetch miss 之前缓存服务端的 updates（和我们目前的设计一样
从客户端加载数据成为了性能瓶颈
需要加载 mobx，建立数据连接，等等，性能比较差
Partial bootstrap 加载部分数据以初始化
Comment / issue history
Collection -> LazyCollection ，后者是前者的子类，懒加载
React Suspense
Prefetch (Next.js Link like)
优化 Mobx 的初始化过程，只有在获取值的时候才 make them observable
GraphQL 成为性能瓶颈
Delay bootstrap，重要的东西先加载，不重要的资讯后加载（评论、issue 历史记录）
Bad for big operations. Response is a JSON object. Everything is in memory!
Streaming REST, for bootstrap queries
PostgreSQL 成为瓶颈
Bootstrap 的时候会造成大量的访问
通过主从分离降低访问负载，从数据库存在数据延迟
MongoDB 定期存储数据快照，bootstrap 的时候从快照拉取数据，然后通过 syncID 更新到最新状态
Delayed bootstrap 成为瓶颈
Batch loader
Only really necessary data
Stream data on demand
Auto dedupe

#### Univer 2024 Q4 #OKR #Univer #2024Q4 #planning

##### 团队工作重点

- [ ] 交付 Node.js SDK
    - [ ] 拆解设计有问题的包，以实现服务端计算
    - [ ] Facade API 的实现拆分到各个 plugin 内部以支持 Node.js SDK
    - [ ] 底层的数据对象和渲染对象都可以移动到各个业务的包里面去，不过这个做起来会有比较大的 breaking change 而且收益率不高，可以在 1.0 版本做
    - [ ] 发布 presets 包并更新官网的教程和案例
    - 剩余问题

很多权限校验是在 UI 包做的，现在看来不仅是 UI 包会发生权限校验，需要迁移相关逻辑 原彬
不过，我们可以假设服务端有最高权限，因此总是可以直接通过权限校验而无需做权限判断
如果无头服务端需要接入协同编辑，那么 collaboration-client 需要拆一个 UI 包出来，以排除 UI 相关的代码
doc 协同编辑代码里有大量依赖 docs-ui 的内容，以至于不好拆解
message service 和 confirm service 需要抽到更底层的包里去，让 Node.js 也能够提示错误信息以及让用户确认消息 肖志祥
在 Node.js 上，可以让用户对 confirm service 选择 always true 或者 always false 的策略

- [ ]🏃‍♀️ Facade API @[向松](#向松)，能明确归属模块的转交模块 owner 处理，向松处理基础功能的 Facade API
    - [ ] 条件格式 让哲能来做估计是遥遥无期了，向松来顶一顶
    - [ ] 排序 张宇鸿
    - [ ] 本来查找替换也是要做的，但是看了一下要求重构查找替换 plugin， ROI 不高，后续有用户需求再做
- 为下个周期的 Sheet 功能分配 owner
    - [x] styles 重构，支持层级、行列 style 和 default style 闵成成
    - [x] 水印（跨文档类型需求、同时涉及到打印）原彬
    - [x] 自动列宽 罗楠
    - [x] 复选框进入 styles 张威
    - [x] 网格线配置 [胡文召
    - [ ] 🏃‍♀️ 继续补充公式，Q4 结束时补充到 500 个 潘学平
    - [ ] 🏃‍♀️ 图表、迷你图 闵成成 张宇鸿 廖志鹏
    - [ ] 🏃‍♀️ 单元格图片 张威
    - [x] 筛选年月日聚合 原彬
    - [ ] 提及、通知（同时支持 doc 和 sheet） 张威
    - [ ] 行列转置（同时转置粘贴）张宇鸿 原彬
    - [ ] 分列和智能分列 闵成成
    - [ ] 切片器 闵成成
    - [ ] Univer Doc & Editor M7~M12 RoadMap
- [ ] 质量提升
    - [ ] 🏃‍♀️ 编辑器重构
    - [ ] 提升公式引擎的计算性能
- [ ] 易用性提升
    - [ ] 🏃‍♀️ 翻新官网，更好地介绍产品和收费服务，把体验做到和国外同类公司基本一致
    - [ ] 交付优化事项 🏋🏻 Univer Backend Q4 (draft)

##### 个人工作重点

1. 进一步加强对研发同事工作的评审，包括但不限于：Ticket、技术方案、PR、文档、API、examples、单元测试、e2e 测试、Facade API；每天上午都抽出时间来进行 review
2. 技术重点
    - [ ] 实现服务端计算
    - [ ] 掌握公式模块，优化性能和架构 [1](#univer-公式模块问题整理)
    - [ ] 掌握渲染模块

### 2024

2023 年年度工作总结 #2023 #reelection #annual-summary

**Learnings**

Univer 给了我一个实现技术理想的机会。4 个月的时间我带领团队彻底重构了项目，依赖注入、插件化、注册式 UI 等等架构设计发挥了它们的威力，支撑着我们快速迭代，并达成了在 12 月底发布 MVP 的目标。

**正面**

1. 几个架构设计要点发挥了预想中的作用，给了项目一个好的开始
    1. 依赖注入给我们的应用带来的极大的灵活性，测试、开发内部工具都非常方便
    1. 插件化让我们的项目组织很有条理，并且支持了一些功能以闭源的方式实现，不过目前它在性能方面的优势和用户二次开发的便利程度方面的优势还尚未体现，有待观察
    1. 我们的团队分工比较合理，充分结合了每个人的兴趣和优势

**负面**

1. 我们在 MVP 阶段没有足够重视代码质量和文档质量，有一部分同学的代码质量问题缺乏及时的引导
1. 我们在早期没有太重视 Univer 对于用户来说是否容易接入
    1. 文档不够完善，很多文档都是空白的
    1. 我们应该更早地开始开发 Facade API
    1. 部署整套 Univer 系统比较麻烦
    1. 用户对于二次开发仍然存在较多的疑惑

** 如何改善 **

1. 招聘了 QA 赵丽鑫 来负责测试。同时我们要求研发同学在修复 bug 的时候都要尽可能地带上单元测试，并对单元测试覆盖率提出了高的要求。不过目前这一要求并没有被严格执行，需要再好好把关，同时需要加强对团队内工程师素养的培训；
1. 招聘了 向松 来增加在社区上的投入，倾听用户的反馈并解决在易用性方面的问题。我自己也要多了解用户的问题，并亲力亲为地解决。
1. 通过一些管理提升大家对代码质量的重视程度
    1. 在 Code Review 环节提升对大家代码质量的要求
    1. 在 Contributing Guide 里提供优秀实践供大家学习
    1. 将代码质量作为绩效考评的一环
    1. 确立研发团队的价值观

# Univer

## Team members

#### 向松

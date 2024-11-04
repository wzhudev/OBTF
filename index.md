# TODO

🌟 stands for big projects. I should not have more than 2 🌟 at the same time.
🚨 stands for emergent tasks.

- [ ] 计划一个日本 11 天的旅游行程
- 1/8 - 1/19
- 日本 - 东京
- [ ] 提前联系一个搬家公司，看看全包的价格和流程 [2024/11/4](#2024/11/4)

## Engineering TODO

- [ ] 🌟 Reverse Linear Sync Engine. It shall be done before [2024/11/30](#2024/11/30).
- [] 解决 Rust 调试工具中看不到断点 variable 的问题

## Univer TODO

- [ ] 🚨 根据 0.5.0 版本改动内容修改 starter kit 以及 examples
- [ ] 对权限相关的工作进行复盘，可能不进行大范围的复盘，先在一个比较小的 scope 上进行 [#权限复盘]
- [ ] 重构公式模块
- [ ] 找个时间 review 重构之后的编辑器模块代码
- [ ] 🚨 建立 [个人看板](#个人看板)，关注项目和团队的重要信息
- [ ] 明确功能交付标准 2024/11/3

Projects that I would like to read the code:

- Zed, for their rendering engine GPUI, and other interesting data structures’ (such as rope) implementations.
- Typst, for typesetting and rendering.

But I would not read them recently because I need to focus on 🌟 projects.

# Journals

#### Plans to Expand My Capability Set

At present

- Front-end programming languages
- Architecture design
- Collaborative algorithms
- Component kits
- Typesetting

Next

- Rendering based on GPU [1](#gpui-of-zed) 

#### 2024/11/2

#### 2024/11/2

刘洋这下牛逼了，把公式引擎的性能提升到了和竞品齐平甚至在部分场景下超越了竞品。接下来我们需要发挥我们架构的优势把这部分成果商业化。

发现作为 Head of Engineering，我在 Univer 的工作有阶段性的特点：

- 去年 8 月到今年 8 月，主要作为架构师和核心开发，设计架构，编写核心组件和核心功能
- 今年 8 月到 10 月，主要作为 tech leader review 方案和代码，并根据公司的商业策略探索技术方向，更多关心面向交付和开发者体验的问题
- 11 月之后，我可能需要作为 CTO 探索技术方向，更多关心公司的技术战略

#### Univer 公式插件问题整理

**Univer 的公式引擎拆分商业版之后，看来没有将模块改造成抽象模块的必要。**

等到公式性能优化完成之后再开始进行重构，本次重构公式模块的目标包括：

1. 提升公式模块的可维护性
2. 探索如何实现主线程和 worker 线程的调度和 fallback 机制
3. 尝试解决公式模块的性能问题
	1. 上层业务功能应用公式计算结果的性能问题

整体架构设计问题

* `FeatureFormula` 和 `OtherFormula` 并没有区分这两个概念的必要，也不需要切分很多 module 来设置这两种 formula。
* 很多操作通过 mutation 来进行，这是种反模式
	* 应该提供一个操作 formula 底层模块的中介模块，并负责维护 worker 线程的创建，并将其他插件对公式系统的调用发送到正确的模块
	* 一些修改底层数据的逻辑应该走这个中介模块去 set 而不是通过 mutation
		* 滥用 mutation 会导致调用链路不清晰
* 没有在 worker 崩溃时的处理机制，不能 fallback 到主线程
* formula service 和 other service 在获取公式计算结果的时候需要全量遍历计算结果
* 模块划分过多，依赖关系过于复杂，内聚程度较低

##### 部分模块设计问题

`ActiveDirtyManagerService` 主要用于让业务注册回调，来告诉公式系统某个 mutation 发生的时候可能的脏区是什么。改造点：

- [ ] ~~不需要作为抽象依赖~~ 不再需要
- [ ] 重命名为 `DirtyAreaRegistrationService`
- [ ] 暴露出的几个方法是不必要的
- [ ] `RuntimeService` 和 `CalculateFormulaService` 存在大量的单一调用，这两个类应该合并在一起 

`Interpreter` 对于 `FormulaRuntimeService` 的依赖主要在于获取公式运行状态，只需要把这个状态设置到一个底层的状态服务即可解除该依赖。

##### 滥用的 mutation 列表

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

在 λ 公式进行计算的时候，需要传递词法作用域把上层的结果传递给它；和刘洋确认了一下，我们可以试试看

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

I should really not start lots of things at the same time.

Before 志祥 completes the bundling change and unblock us from updating starter kits and examples, I will just read the source of our formula engine.

#lexical #editor #oss Some design decisions I personally love with Lexical, Facebook’s open source editor framework:

- Not using the DOM as SSOT but its own state model. ProseMirror uses the first approach and I hate to assemble HTML pieces to get started with an editor.

Other design decisions:

- Mutable state. But mutable actions can only happen in the update callback.
- Double buffering. 

*#minimalism #digital #workflow* My One Big Text File is an interesting article. I’ll start today with VSCode (yes!) to try and use this method for my own workflow, to avoid sudden sunsets of third-party applications (e.g. Arc Browser and lots of products killed by Google) and information leaks. And also, it works well with my minimalism philosophy - No new application to learn, no new fancy features to get used to. Just plain text.

What is especially good with **VSCode**?

1. It is still pure text files that can live forever.
2. You can benefit from VSCode’s top-notch performance and rich extension ecosystem, especially Vim.
3. It is very likely something your are already using, so no new learning curve.

#### 2024/10/30

_#visa #US_ Got our first US visa today.

#### Linear Sync Engine _#linear #sync-engine #collaboration_

Linear Sync Engine
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

#### Univer 2024 Q4 _#OKR #Univer #2024Q4 #planning_

##### 团队工作重点

- [ ] 交付 Node.js SDK
    - [x] 拆解设计有问题的包，以实现服务端计算
    - [x] Facade API 的实现拆分到各个 plugin 内部以支持 Node.js SDK
    - [ ] 底层的数据对象和渲染对象都可以移动到各个业务的包里面去，不过这个做起来会有比较大的 breaking change 而且收益率不高，可以在 1.0 版本做
    - [ ] 🏃‍♀️ 发布 presets 包并更新官网的教程和案例
    - 剩余问题
        - 很多权限校验是在 UI 包做的，现在看来不仅是 UI 包会发生权限校验，需要迁移相关逻辑 @原彬；不过，我们可以假设服务端有最高权限，因此总是可以直接通过权限校验而无需做权限判断
        - 如果无头服务端需要接入协同编辑，那么 collaboration-client 需要拆一个 UI 包出来，以排除 UI 相关的代码
        - doc 协同编辑代码里有大量依赖 docs-ui 的内容，以至于不好拆解
        - message service 和 confirm service 需要抽到更底层的包里去，让 Node.js 也能够提示错误信息以及让用户确认消息 @肖志祥；在 Node.js 上，可以让用户对 confirm service 选择 always true 或者 always false 的策略
- [ ] 🏃‍♀️ Facade API @[向松](#向松)，能明确归属模块的转交模块 owner 处理，向松处理基础功能的 Facade API
    - [ ] 条件格式 让哲能来做估计是遥遥无期了，向松来顶一顶
    - [ ] 排序 张宇鸿
    - [ ] 本来查找替换也是要做的，但是看了一下要求重构查找替换 plugin，ROI 不高，后续有用户需求再做
- 交付 Sheet 功能
    - [x] styles 重构，支持层级、行列 style 和 default style @闵成成
    - [x] 水印（跨文档类型需求、同时涉及到打印）@原彬
    - [x] 自动列宽 @罗楠
    - [x] 复选框进入 styles @张威
    - [x] 网格线配置 @胡文召
    - [ ] 🏃‍♀️ 继续补充公式，Q4 结束时补充到 500 个 @潘学平
    - [ ] 🏃‍♀️ 图表、迷你图 @闵成成 @张宇鸿 @廖志鹏
    - [ ] 🏃‍♀️ 单元格图片 @张威
    - [x] 筛选年月日聚合 @原彬
    - [ ] 提及、通知（同时支持 doc 和 sheet） @张威
    - [ ] 行列转置（同时转置粘贴）@张宇鸿 or @原彬
    - [ ] 分列和智能分列 @闵成成
    - [ ] 切片器 !闵成成
- [ ] Univer Doc & Editor M7~M12 RoadMap
- [ ] 质量提升
    - [ ] 🏃‍♀️ 编辑器重构
    - [ ] 提升公式引擎的计算性能
- [ ] 易用性提升
    - [ ] 🏃‍♀️ 翻新官网，更好地介绍产品和收费服务
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

**如何改善**

1. 招聘了 QA 赵丽鑫 来负责测试。同时我们要求研发同学在修复 bug 的时候都要尽可能地带上单元测试，并对单元测试覆盖率提出了高的要求。不过目前这一要求并没有被严格执行，需要再好好把关，同时需要加强对团队内工程师素养的培训；
1. 招聘了 向松 来增加在社区上的投入，倾听用户的反馈并解决在易用性方面的问题。我自己也要多了解用户的问题，并亲力亲为地解决。
1. 通过一些管理提升大家对代码质量的重视程度
    1. 在 Code Review 环节提升对大家代码质量的要求
    1. 在 Contributing Guide 里提供优秀实践供大家学习
    1. 将代码质量作为绩效考评的一环
    1. 确立研发团队的价值观

# Univer

## 流程管理

### 功能交付标准

 

## Univer Management

### 个人看板

## Team members

#### 向松

# Techniques

## Rendering

DOING: 在学习任何渲染引擎时需要搞明白的几件事情：

1. 管理内部状态的方法
2. 状态变更触发响应式更新的办法，比如在哪里启动的渲染循环
3. 通过几棵树来管理渲染对象
4. 性能优化的方法
5. 如何实现排版

### GPUI of Zed

Some blogs about GPUI on Zed's official website.

* [GPUI 2 is now in production](https://zed.dev/blog/gpui-2-on-preview)
* [Ownership and data flow in GPUI](https://zed.dev/blog/gpui-ownership) 这篇文章讲解了如何在 Rust 里构造一个响应式更新模型。

在 Zed 当中一共有四种不同的 Context:

- AppContext 这个是在 `App::new()` 时创建的，会在应用程序的整个生命周期中持续存在
- WindowContext 这是在初始化或者 `upddat` 是创建的，每次更新过后都会被抛弃

```rs
let result = update(root_view, &mut WindowContext::new(cx, &mut window));
```


一个简单的用例如下所示，这里演示的是启动应用和开启窗口的部分：

```rs
struct HelloWorld {
    text: SharedString,
}

// 需要绘制的组件需要实现 Render trait
impl Render for HelloWorld {
    // render 方法返回一个实现了 IntoElement trait 的对象
    // 同时 Div 也实现了 Element trait
    // 被送到 GPU 渲染管线里面的是 Element
    fn render(&mut self, _cx: &mut ViewContext<Self>) -> impl IntoElement {
        // 链式调用语法
        // 其实不用考虑 tree-shaking 的语言用起来真的挺爽的
        div()
            .flex()
            .bg(rgb(0x2e7d32))
            .size(Length::Definite(Pixels(300.0).into()))
            .justify_center()
            .items_center()
            .shadow_lg()
            .border_1()
            .border_color(rgb(0x0000ff))
            .text_xl()
            .text_color(rgb(0xffffff))
            .child(format!("Hello, {}!", &self.text))
    }
}

fn main() {
    App::new().run(|cx: &mut AppContext| {
        let bounds = Bounds::centered(None, size(px(300.0), px(300.0)), cx);

        // 业务代码需要自己去创建窗口
        // 也合理，或许这样可以支持多窗口
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            // 根 view，需要返回一个带有 Render trait 的对象
            |cx| {
                // 在 GPUI 里面，所有对状态的修改都是通过 cx 来进行的
                cx.new_view(|_cx| HelloWorld {
                    text: "World".into(),
                })
            },
        )
        .unwrap();
    });
}
```

```rs
    pub fn open_window<V: 'static + Render>(
        &mut self,
        options: crate::WindowOptions,
        build_root_view: impl FnOnce(&mut WindowContext) -> View<V>,
    ) -> anyhow::Result<WindowHandle<V>> {
        self.update(|cx| {
            let id = cx.windows.insert(None);
            let handle = WindowHandle::new(id);
            match Window::new(handle.into(), options, cx) {
                Ok(mut window) => {
                    let root_view = build_root_view(&mut WindowContext::new(cx, &mut window));
                    window.root_view.replace(root_view.into());
                    WindowContext::new(cx, &mut window).defer(|cx| cx.appearance_changed());
                    cx.window_handles.insert(id, window.handle);
                    cx.windows.get_mut(id).unwrap().replace(window);
                    Ok(handle)
                }
                Err(e) => {
                    cx.windows.remove(id);
                    Err(e)
                }
            }
        })
    }
```

* `WindowHandle` 是用于操作窗口的句柄

到这里 Window 和 App 相关的 Context 和 handle 全部创建完毕，接下来就应该启动第一次渲染了。

注意两个属性：

1. dirty
2. needs_present 好像是为了避免掉帧引入的一个状态标志， `draw` 之后即会标记为 `needs_present`


两个绘制方法：

1. draw 里面触发整个 View 树绘制的是这一行，`self.draw_roots();`
2. present

#### 绘制过程和性能优化

绘制过程分为以下几个阶段

```rs
pub(crate) enum DrawPhase {
    None,
    Prepaint,
    Paint,
    Focus,
}
```

Prepaint 阶段所做的事情：

1. pre_paint root element
2. pre_paint deferred draws
3. 处理正在进行中的 prompt drag 或者 tooltip

Questions: 

- [ ] What will be deferred to draw?
- [ ] pre-paint 具体在做些什么？

pre-paint 具体在做什么？

1. 根据窗口约束 layout，`AnyElement` 对所有 Element 都包裹了 `Drawable` struct 从而拥有了 `layout_as_root` 方法

每个元素绘制也有若干个阶段。从 `layout_as_root` 方法的实现来看，一个 `Drawable` 可能会多次进入 `ElementDrawPhase::LayoutComputed` 状态，排版时常会出现稳定性的问题。

走到孩子节点去继续计算 layout，如果 element 有孩子的话，就会去递归算孩子的 layout，例如 `Div` view。

```rs
self.element.request_layout(global_id.as_ref(), cx);
```

```rs
enum ElementDrawPhase<RequestLayoutState, PrepaintState> {
    #[default]
    Start, // layout_as_root, 在这个阶段似乎每个 layout 树上的节点都会用一个相同的 layout_id
    RequestLayout {
        layout_id: LayoutId,
        global_id: Option<GlobalElementId>,
        request_layout: RequestLayoutState,
    },
    LayoutComputed {
        layout_id: LayoutId,
        global_id: Option<GlobalElementId>,
        available_space: Size<AvailableSpace>,
        request_layout: RequestLayoutState,
    },
    Prepaint {
        node_id: DispatchNodeId,
        global_id: Option<GlobalElementId>,
        bounds: Bounds<Pixels>,
        request_layout: RequestLayoutState,
        prepaint: PrepaintState,
    },
    Painted,
}
```

`AnyView` 本身就实现了 `Element` trait

`AnyView` 里面实现了一个 `ArenaBox`，这个一定就是上文中说的对 `Rc` 的拓展了？

```rs
let mut root_element = self.window.root_view.as_ref().unwrap().clone().into_any();
```

Paint 阶段所做的事情


# Management

## Reading

### 技术为径 *技术管理*

#### Chapter 4 管理员工

这个章节覆盖的主题：

1. 如何和员工建立合作关系
2. 如何与员工定期进行一对一会议
3. 如何对员工进行有效授权，同时确保项目进展可控
3. 如何想员工提供职业发展方面的反馈，如何向员工反馈工作结果
4. 如何帮助员工识别其能力短板，并提升其能力

和员工在任何可能出现问题的领域坦率地沟通。

#thinking #reflection 目前我在管理上的问题之一是对下属的工作进展关心比较少，虽然没干预，但是也没监督。

- ticket
- 技术方案
- 复盘

有效授权的要点：

1. 明确何时应该介入：
2. 尽可能从现有系统中搜寻数据，然后再找员工询问
3. 根据项目的不同阶段调整自己不同的关注点
4. 为代码和系统建立标准

#### Chapter 5 管理团队

这个章节覆盖的主题：

1. 保持技术投入
2. 识别团队问题
3. 作为经理推动更大团队的决策
4. 管理团队中的冲突
5. 项目管理

书里导出的一个建议是不要脱离一线的研发工作，只有仍然承担一定的开发工作，才能有效识别目前面临的障碍以及流程上真正的问题。之所以字节会搞一堆对质量毫无作用的所谓专项优化活动，就是因为大老板无法真正感知到真正的问题，只能按照所谓“业界共识”、“最佳实践”机械地提一些要求，但是软件工程并无银弹，仍然非常依赖领导者的判断力。

研发经理保持一线工作的好处（反之则是问题）：

1. 更好地监督下属员工对技术决策负责，确保这些技术决策符合基本的技术要求，评估决策和当下团队情况
2. 赢得下属员工的尊敬
3. 识别影响团队的真正的问题
4. 更准确的技术可行性评估和工作量评估，以更好地为产品策略提供技术支持

#thinking 如果没有受到员工认可的技术能力，但是又喜欢微管理下属的技术决策，只会进一步削弱已经不多的认可。

技术团队可能存在的问题：

1. **无法交付** 
2. 人际关系紧张
3. 因超负荷工作所带来的不满
4. 合作困难

#thinking 经过这一年多的管理工作，我认为的管理工作要点主要在于三个：识别问题、凝聚共识、推动解决。

经理如何帮助团队进行更好的决策：

1. 创造数据驱动决策的团队文化 不过这一点经常会导致管理者不愿意承担决策责任，以数据不足为由延迟甚至是拒绝决策
2. 掌握一定的产品话语权
3. 前瞻性
4. 复盘技术决策和技术项目 #thinking 权限相关的工作就需要进行一次复盘，为什么三番五次出现沟通不到位，反复修改代码的情况 [#权限复盘](#权限复盘)
5. 对团队日常工作和流程进行复盘

> 管理层承担不确定性是责无旁贷的，管理者不应该将不确定性统统甩给团队成员来承担。

笑死，这不就是字节的大老板们最爱干的事情吗？

# Programming Language

## Rust

#debugging #vscode 解决了在 Rust debugger 中看不到 local variable 的问题，原来在一个 Rust 项目中，不只是需要修改子 crate `cargo.toml` 文件中的 `profile.dev` 配置，也需要修改整个项目的配置，添加：

```toml {2,3}
[profile.dev]
debug = "full"
opt-level = 0
```


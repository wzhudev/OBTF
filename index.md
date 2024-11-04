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
        - [ ] 目前被 [肖志祥](#肖志祥) 的打包进度严重阻塞
    - 剩余问题
        - 很多权限校验是在 UI 包做的，现在看来不仅是 UI 包会发生权限校验，需要迁移相关逻辑 @原彬；不过，我们可以假设服务端有最高权限，因此总是可以直接通过权限校验而无需做权限判断
        - 如果无头服务端需要接入协同编辑，那么 collaboration-client 需要拆一个 UI 包出来，以排除 UI 相关的代码
        - doc 协同编辑代码里有大量依赖 docs-ui 的内容，以至于不好拆解
        - message service 和 confirm service 需要抽到更底层的包里去，让 Node.js 也能够提示错误信息以及让用户确认消息 @肖志祥；在 Node.js 上，可以让用户对 confirm service 选择 always true 或者 always false 的策略
- [ ] 🏃‍♀️ Facade API @[向松](#向松)，能明确归属模块的转交模块 owner 处理，向松处理基础功能的 Facade API
    - [ ] 条件格式 让哲能来做估计是遥遥无期了，向松来顶一顶
    - [ ] 排序 张宇鸿
    - 本来查找替换也是要做的，但是看了一下要求重构查找替换 plugin，ROI 不高，后续有用户需求再做
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
    - [ ] 行列转置（转置粘贴）@张宇鸿
    - [ ] 分列和智能分列 @闵成成 @原彬
    - [ ] 切片器 @闵成成 @原彬
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

# Collaboration / Collaborative Editing

## Linear Sync Engine

[blog](#a-reverse-study-of-linears-sync-engine)

# Rendering

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

# Rust

#debugging #vscode 解决了在 Rust debugger 中看不到 local variable 的问题，原来在一个 Rust 项目中，不只是需要修改子 crate `cargo.toml` 文件中的 `profile.dev` 配置，也需要修改整个项目的配置，添加：

```toml {2,3}
[profile.dev]
debug = "full"
opt-level = 0
```

# vscode 

通过 `alt+z` 键盘就可以切换 vscode 的 word wrap 配置，这样写 markdown table 的时候临时 nowrap 就很方便了！

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

---

# A Reverse Study of Linear's Sync Engine

I work on collaborative software, primarily rich text editors and spreadsheets. The **collaborative engines**, also known as **data sync engines**, in these applications enhance user experience by enabling real-time, simultaneous edits on the same file, and offering features like offline availability and file history. Earlier this year, our team at [Univer](https://www.univer.ai/) developed a [collaborative engine](https://www.univer.ai/blog/post/ot) from the ground up, supporting various document types including documents, spreadsheets, and (in the near future) slides. It also has offline availability, permission controls, and file history. Everything looks great.

However, developing a collaborative engine remains a significant challenge in our industry, especially when using operational transformation (OT). This complexity arises from the diverse data models and operation sets of different applications, which demand substantial effort to implement accurately and to create effective transformation methods for managing conflicts. While OT is effective and precise for synchronizing documents, preserving user intents, and handling data conflicts, it might be overly complex for simpler tasks such as synchronizing file names or comments.

When we developed the first version of our SaaS platform, we used an approach very similar to [Figma's legacy](https://www.figma.com/blog/livegraph-real-time-data-fetching-at-figma/#challenges-in-real-time). However, as the blog author noted, it feels verbose when synchronizing many models. It is difficult to maintain, and the data can become outdated very easily. Therefore, I am seeking a generic data sync engine that simplifies the synchronization of various models. While CRDTs appear promising, they come with metadata overhead and can become complex when partial syncing or permission control is required (for example, when a user is allowed to view only a subset of the data). These challenges often arise because CRDTs are designed for decentralized collaborative software. However, most applications still rely on centralized servers. Although I am personally a fan of CRDTs, it seems that a different approach is necessary.

[Linear Sync Engine](https://linear.app/docs/offline-mode) (LSE) effectively addresses the discussed topics. It supports bi-directional syncing between client and server, offers offline mode and action history, and automatically updates the UI when data changes. Additionally, it integrates all these features within a straightforward API, streamlining the development of new features. For example, to update the title of an issue, you simply need to:

```jsx
issue.title = "New Title";
issue.save();
```

I believe LSE is what I had been searching for, so I chose to reverse-engineer its frontend code to understand its workings. Additionally, I am documenting my findings, aiming to assist others who are planning to build a similar data sync engine.

_By the way, Figma's approach appears quite interesting as well. I will certainly explore it further if time permits._

In this post, we will explore how Linear:

- Defines models, their properties, and the relationships between them
- Makes models observable with MobX
- Bootstrap by fully loading data from the server and constructs models in memory
- Builds a local database and populates it with data
- Bootstrap from the local IndexedDB
- Hydrate lazily-loaded data
- Executes transactions and sends them to the server
- Applies updates received from the server

To assist you in understanding the raw code, I've uploaded a copy of Linear's code for your reference, complete with comments. The comments include many details not covered in this post. At the end of the post, I've also provided a table that maps minimized names to their possible original names. For an optimal experience, I suggest reading this article on a desktop or laptop, as viewing the code side by side may be necessary.

## Introduction

If you haven't watched Tuomas' [two](https://www.youtube.com/watch?v=WxK11RsLqp4&t=2175s) [talks](https://linear.app/blog/scaling-the-linear-sync-engine), a [podcast](https://www.devtools.fm/episode/61), and a [recent talk](https://www.youtube.com/watch?v=VLgmjzERT08) at Local First Conf about LSE, I highly recommend doing so before continuing. But here are core concepts in LSE:

![](./images/introduction.png)

### Model

Entities such as "Issue", "Team", "Organization", "Comment" etc. are **models** in LSE. Models have **properties** and **references** to other models, and many properties and references are observable (via MobX) to make views update automatically. Models can be bootstrapped from the local database (IndexedDB) or from the server (a full bootstrapping). Operations (additions, deletions, updates) to these models (and/or their properties and references) will be sent to the server and then broadcasted to other connected clients to make sure that models are eventually consistent among multi duplicates.

### Transaction

Operations sent to the server will be encapsulate as a "Transaction". Transactions are meant to execute only on the server and may fail to execute, so transactions need to be reversible. If the client loose connect to the server, transactions would be saved into the local database and resent to the server when the client goes online.

### Delta

When transactions are executed on the server, server would then broadcast "deltas" to all clients (including the one send the transactions) to update models. Deltas are not identical to what the client sent as transactions because the server need to perform side effects (such as generating history).

### Sync Engine

The **Sync Engine** is a set of critical modules that are responsible for tasks such as:

1. Bootstrap from local database or the server.
2. Manage local database, transactions, deltas.
3. Figure out what have been changed on models and send transactions to the server.

In the following parts, we will discuss these core concepts in detail, starting with **"Model"**.

## Models

### Model Registry

When Linear starts, the initial step involves **declaring models, along with their properties, actions, and computed values**.

LSE maintains a detailed dictionary referred to as `ModelRegistry` (`rr`) that records metadata of models and their properties. Various parts of the application utilize this dictionary to determine how to load, bootstrap, or hydrate models.

![[Pasted image 20241004162637.png]]

_Here is a screenshot displaying the models, their properties, references, and metadata. Note that the minimized names in screenshots differ from those in the GitHub repos._

`ModelRegistry` has several maps storing metadata for different purposes and corresponding methods to register and retrieve metadata, for example:

- `modelLookup` to map a model's name to the constructor
- `modelPropertyLookup` to register metadata of models properties
- `modelReferencedPropertyLookup` to register references

`ModelRegistry` has a special property `__schemaHash` which is used to determine the version of all models and whether the local database should be migrated. We will cover this in the bootstrapping section.

### Model

![[Pasted image 20241004165751.png]]

LSE utilizes JavaScript `class` to define models. Models such as `Issue`, `Comment`, and `Project` extend `BasicModel`, which has the following properties and methods which we will cover in detail later:

- `id` - A unique UUID for every model, used as the map key to retrieve a model from the "Object Pool".
- `_mobx` - An empty object that is essential for making the model observable. Will be explained in the "Observability" section.
- `store` - The property holds the reference to `SyncedStore`, which we would cover in detail in later chapters.
- `makeObservable` - Make the model observable.
- `propertyChanged`, `markPropertyChanged`, `changeSnapshot` - Used to figure out what properties have been changed and then to generate a transaction.
- `save` - Calling it would generate an `UpdateTransaction`. More on it later.
- Methods ending with `Mutation`, such as `updateMutation`, are used to generate GraphQL mutating queries. More on it later.
- `updateFromData` - Dumps serialized values into a model. When creating a model, LSE does not pass in arguments to the constructor. Instead, it calls this method.
- `attachToReferencedProperties` - Attaches all reference properties of a model.

Models have **model metadata**. Some fields of model metadata include:

- [ ] These metadata needs more explanations.

1. `usedForPartialIndexes`,
2. `loadStrategy`. There are five different load strategies: `instant`, `lazy`, `partial`, `explicitlyRequested`, and `local`.
   1. `instant` - These models should be loaded into memory when the application bootstraps. This is the default strategy.
   2. `lazy` - These models can be loaded into memory when they are used. However, if they are used, all instances of that model should be loaded at the same time. For example, `ExternalUser`.
   3. `partial` - These models can be loaded on demand. This strategy is used quite often. For example, `DocumentContent`.
   4. `explicitlyRequested` - These models should be loaded only when requested. For example, `DocumentContentHistory`.
   5. `local` - These models are only persisted in the local database. I cannot find any model that uses this.
3. `partialLoadMode`

> [!NOTE]
> During his presentation at Local First Conf, Tuomas explained how features can be developed without altering the server-side code. IMO, they achieve this by setting the load strategy to `local` for any new model, ensuring it persists or bootstraps only to the local IndexedDB. Once satisfied with the new model, they begin implementing syncing for this model by changing its load strategy from `local` to others.

### Properties

Models have properties. There are **seven types of properties** (defined in enum `vn`):

1. `property` - A property considers as "owned" by that model. For example, `title` is a `property` of `Issue`.
2. `ephemeralProperty` - A property as owned like `property`, but the property will not be persisted in the database. It's rarely used. All I could found is that `lastUserInteraction` of `User` is an ephemeral property.
3. `reference` - A property used when a model holds reference to another model. Its value is usually the id of the referenced model. Reference can be lazy, meaning the system will not load the referenced model until this property is accessed. For example, `subscription` is a `reference` of `Team`.
4. `referenceModel` - Decorators that register `reference` or `backReference` property will always register a `referenceModel` property as well. This property defines a getter and setter to access the model using the value of its corresponding `reference` or `backReference`.
5. `referenceCollection` - Similar to `reference`, but an array. For example, `templates` is a `referenceCollection` of `Team`.
6. `backReference` - For example, `favorite` is a `backReference` of `Issue`. The difference between a `reference` and `backReference` is that: assuming A is a `reference` of B, we consider A is “owned” by B. When B is deleted,
7. `referenceArray` - Used for many-to-many references. For example, `members` of `Project` is a `referenceArray` that references `Users`. Users can be members of multiple `Projects`.

Each property has **property metadata**. Property metadata instructs LSE how to deal with this property in scenarios like hydrating and data-fetching. Some fields of property metadata are:

1. `serializer` - Defines how to serialize the property for data transfer or storage.
2. `type` - Specifies the type of this property.
3. `persistence` - Indicates how this property should be stored in the database. Options include `none`, `createOnly`, `updateOnly`, and `createAndUpdate`.
4. `indexed` - Determines where this property should be indexed in the database.
5. `lazy` - Specifies if this property should be loaded only when accessed.
6. `referenceOptional` - Its difference from `referenceNullable` is unclear.
7. `referenceNullable` - Unknown.
8. `referencedClassResolver` - A function that returns the referenced model’s constructor.
9. `referencedProperty` - If the referenced model also has a property that references back, this would be the name of that property.
10. `cascadeHydration` - Indicates if LSE should hydrate referenced models in a cascading manner.
11. `onDelete` - Defines how to handle the referenced model when the model is deleted. Options include `CASCADE`, `NO ACTION`, `SET NULL`, and `REMOVE AND CASCADE WHEN EMPTY`.
12. `onArchive` - Defines how to handle the referenced model when the model is archived.

### Registration of Properties

LSE uses decorators to register properties and models. When code runs, properties are registered before the corresponding model. There are different decorators for different kinds of properties. I will cover two of them in this post.

#### Decorator `Property` (`w`)

Lets take `Issue` model for an example. `priority` is declared as a `property` of `Issue`:

```tsx
Pe(
  [
    w({
      serializer: P_,
    }),
  ],
  re.prototype,
  "priority",
  void 0
);

// In the source code, it may looks like:
@Model("Issue")
class Issue {
  @Property({ serializer: PrioritySerializer })
  priority: Priority;
}
```

```jsx
function w(t = {}) {
  return (e, n) => {
    M1(e, n, t.serializer !== void 0 && t.shallowObservation !== !0);
    const r = t.persistance !== void 0 ? t.persistance : ee.createAndUpdate,
      s = {
        type: vn.property,
        persistance: r,
      };
    t.serializer && (s.serializer = t.serializer),
      t.optimizer && (s.optimizer = t.optimizer),
      t.enum && (s.enum = t.enum),
      t.indexed && (s.indexed = t.indexed),
      t.multiEntry && (s.multiEntry = t.multiEntry),
      t.shallowObservation && (s.shallowObservation = t.shallowObservation),
      Me.registerProperty(e.constructor.name, n, s);
  };
}
```

In `Property` 's implementation:

1. The property is made observable by calling `M1`, which we would cover in detail in section "[Observability](#observability)".
2. The metadata is generated, and this property is registered to `ModelRegistry`.

#### `Reference` (`pe`) & `LazyReferenceCollection` (`Nt`)

For example, `assignee` is a `reference` of `Issue`, and `assignedIssues` is a `LazyReferenceCollection` of `User`.

```tsx
Pe([pe(()=>K, "assignedIssues", {
    nullable: !0,
    indexed: !0
})], re.prototype, "assignee", void 0);

st([Nt()], K.prototype, "assignedIssues", void 0);

// In the source code, it may looks like:
@Model("Issue")
class Issue {
	@Reference(() => User, "assignedIssues", {
	  nullable: true,
	  indexed: true,
	})
	assignee: User | null;
}

@Model("User")
class User {
  @LazyReferenceCollection()
  assignedIssues: LazyReferenceCollectionImpl;

  constructor() {
    this.assignedIssues = new LazyReferenceCollectionImpl(Issue, this, "assigneeId", undefined, {
      canSkipNetworkHydration: () => this.canSkipNetworkHydration(Issue)
    }),
  }
}
```

In the implementation of `Reference`, it will actually register two properties: `assignee` and `assigneeId`.

1. The `assignee` is not saved in the database; only `assigneeId` is. Their `type`s are different.
2. LSE uses a getter and setter to link `assigneeId` and `assignee`. When the `assignee` value is set, it updates `assigneeId` with the new value's ID. Similarly, when `assignee` is retrieved, it fetches the corresponding record from the data store using the given ID.
3. Furthermore, `assigneeId` is made observable with `M1`.

### Registration of Models (`We`)

There is a decorator named `Model` used to declare models, such as "Issue".

```tsx
re = Pe([We("Issue")], re);

// In the source code, it may be something like:
@Model("Issue")
class Issue {}
```

In the implementation of `Model`:

1. The model's name and the constructor function are registered to `ModelRegistry`'s `modelLookup`.
2. The model's name, its schema version, and the names of its properties are combined into a **hash value**. This hash value is registered to `ModelRegistry` and will be used to check the database's schema.
3. If the model's `loadStrategy` is `partial`, this information is also encoded into the hash.

### Observability (`M1`)

In his talks Tuomas mentioned that they are using [Mobx](https://mobx.js.org/README.html) to make the UI responsive to data changes. `M1` function performs the necessary operations to achieve observability. It uses `Object.defineProperty` to define a getter and setter for the property that needs to be observable.

When a value is assigned to the property, the setter checks whether it needs to create a Mobx box value on `_mobx` and sets the value to the box.

The same logic applies to the getter. When other code attempts to access the property, the getter checks whether it should create a Mobx box and retrieves the value from that box.

```jsx
function M1(t, e, n, r) {
  // `t` for the model's prototype,
  // `e` for the property's name,
  // `n` for deep observation,
  // `r` for deserializer
  const s = e + "_o", // key for observable value,
    i = e + "_v", // key for the plain value
    a = e + "_d"; // key for the deserializer
  Object.defineProperty(t, e, {
    get: function () {
      return (
        this.__mobx[a] &&
          (this.__mobx[i] !== void 0 &&
            (this.__mobx[i] = this.__mobx[a].deserialize(this.__mobx[i])),
          delete this[a]),
        this.madeObservable
          ? (this.__mobx[s] ||
              ((this.__mobx[s] = ut.box(this.__mobx[i], {
                deep: n,
              })),
              delete this.__mobx[i]),
            this.__mobx[s].get())
          : this.__mobx[i]
      );
    },
    set: function (o) {
      // when the value is set, we always remove the deserializer
      if ((delete this.__mobx[a], !this.madeObservable))
        // if this model is not observable, directly set the value to __mobx[i]
        r && this.__mobx[i] !== o && delete this[r], (this.__mobx[i] = o);
      else if (!this.__mobx[s])
        r && this.__mobx[i] !== o && delete this[r],
          (this.__mobx[s] = ut.box(o, {
            deep: n,
          })), // if there is no observable and the teh value changes, we should create an observable value
          this.propertyChanged(e, this.__mobx[i], o),
          delete this.__mobx[i];
      // remain the plain value
      else {
        const l = this.__mobx[s].get();
        this.__mobx[s].set(o),
          l !== o && (this.propertyChanged(e, l, o), r && delete this[r]);
      }
    },
    enumerable: !0,
    configurable: !0,
  });
}
```

By wrapping React components with `observer`, MobX can identify which components subscribe to the observables and refresh them when the observable values change.

Another important aspect is that when setting the value, the `propertyChanged` method will be called to register which property has changed, along with the old and new values. This will be used to create a transaction, which we will discuss in a later section.

### Actions (`rt`) & Computed (`O`)

Let's take `moveToTeam` and `parents` of `Issue` for example, there is `Action` decorator and `Computed` decorator.

```jsx
Pe([rt], re.prototype, "moveToTeam", null);
Pe([O], re.prototype, "parents", null);

// The source code would be something like:
@Model("Issue")
class Issue {
  @Action
  moveToTeam() {
    // implementation
  }

  @Computed
  get parents() {
    // implementation
  }
}
```

Action and computed are primitives of MobX. During the bootstrapping, these properties would be made observable by directly calling MobX's `makeObservable` API.

### Takeaway

- Models and properties have metadata which determines how they would behave in LSE.
- LSE leverages decorators to register models, properties, references, actions, and computed values to the `ModelRegistry`.
- LSE employs `Object.defineProperty` to implement getters and setters for building references and/or making models observable.

## Bootstrapping

## The Whole Picture

1. `StoreManager` creates `PartialStore` or `FullStore` for each model to synchronize in-memory data with the IndexedDB (1). Related functions: .
2. Generate metadata of the database to store the current user's data. Then connect to IndexedDB and determine if the database need to be migrated. Metadata would be written into the database as well (2.1) (2.2).
3. Identify the bootstrap type to decide the bootstrapping method (3). Related method `Database.requiredBootstrap`.
4. Bootstrap the database (4). Related function: `Database.bootstrap`.
5. For a full or partial bootstrap, retrieve the model from the server (5). Related function: `BootstrapHelper.fullBootstrap`
6. Store model data for persistence (6).
7. Load data that needs immediate hydration into memory. Initialize in-memory model objects, start observability, apply delta for local bootstrapping, load persisted transactions, save data into IndexedDB by flushing stores, monitor remote changes, and schedule the execution of persisted transactions (7).

## Create Stores

During the bootstrapping process, the initial step is to construct the `StoreManager`. In the `constructor` of `StoreManager`, it constructs `Store` instances for each model registered on the `ModelRegistry`. `Store` manages a model's corresponding table in IndexedDB.

Models have different `loadStrategy`, so LSE generates different stores for them. One is the `PartialStore`, which manages models using `partial` load strategy. The other is the `FullStore`, used for all other models. Both classes extend `Store`.

![[Pasted image 20241005161425.png]]

When a `Store` is created, it calculates the model's hash. The hash will be used a the name of the table in the database.

```jsx
this.storeName = $1(e + Me.propertyHashOfModel(e)); // bf9... for "Issue"
```

For example, `Comment` model has `storeName` `1a1def71af54d8ede0e6fb8a0b8e71b6`, and there will be a table with the same hash.

![[Pasted image 20241005162006.png]]

## Create Databases

Assuming you have one account logged in and a single workspace loaded, LSE will maintain two databases in IndexedDB, `linear_databases` and `linear_(hash)` (in my case `linear_07c3f5816b277c8158a9f6526b158ba6`). `linear_databases` contains metadata of other databases. `linear_07c3f5` stores my data in my private workspace..

Metadata is generated in the `databaseInfo` method and includes:

During the bootstrapping process, LSE will determine whether to create or migrate these databases and establish connections to them.

1. `name` - The name of the database. Its hash is related to `userId`, `version` and `userVersion`. So apparently, if you have different user identity, you would have multi databases.
2. `schemaHash` - This metadata is used for database migration. The next section gives a simple introduction of how it is calculated.
3. `schemaVersion` - This is a local incremental counter used to determine if a database migration is needed. If the new `schemaHash` differs from what is stored in indexDB, the counter will increment. The updated version is then passed to [`IndexDB.open`](http://indexdb.open/) as its second parameter to check if the database requires a migration.

`linear_07c3f5` contains tables each dedicated to a specific type of model. Additionally, there are two special tables: `_meta` and `_transaction`. The `_meta` table stores persistence information for each model and other important information like `lastSyncId`, while the `_transaction` table contains mutations that have not yet been sent to the server.

## Determined Bootstrapping Type

`Database.requiredBootstrap` would determine which type of bootstrapping should be performed. There are three types of bootstrapping:

1. `full`: LSE would load everything from the server.
2. `local`: The application should load from the local database.
3. `partial` : The application should .

We would only cover full bootstrapping in this blog.

### `lastSyncId`

## Full Bootstrapping

- `Database.bootstrap`
- `BootstrapHelper.fullBootstrap`

When Linear does a full bootstrapping, it would send a request like this:

```
https://client-api.linear.app/sync/bootstrap?type=full&onlyModels=WorkflowState,IssueDraft,Initiative,ProjectMilestone,ProjectStatus,TextDraft,ProjectUpdate,IssueLabel,ExternalUser,CustomView,ViewPreferences,Roadmap,RoadmapToProject,Facet,Project,Document,Organization,Template,Team,Cycle,Favorite,CalendarEvent,User,Company,IssueImport,IssueRelation,TeamKey,UserSettings,PushSubscription,Activity,ApiKey,EmailIntakeAddress,Emoji,EntityExternalLink,GitAutomationTargetBranch,GitAutomationState,Integration,IntegrationsSettings,IntegrationTemplate,NotificationSubscription,OauthClientApproval,Notification,OauthClient,OrganizationDomain,OrganizationInvite,ProjectLink,ProjectUpdateInteraction,InitiativeToProject,Subscription,TeamMembership,TimeSchedule,TriageResponsibility,Webhook,WorkflowCronJobDefinition,WorkflowDefinition,ProjectRelation,DiaryEntry,Reminder
```

Note that there are two parameters:

1. `type`. It's either `full` or `partial`.
2. `onlyModels`. Names of the models that are going to load.

The response is a stream of objects:

```jsx
{"id":"8ce3d5fe-07c2-481c-bb68-cd22dd94e7de","createdAt":"2024-07-03T11:37:04.865Z","updatedAt":"2024-07-03T11:37:04.865Z","userId":"4e8622c7-0a24-412d-bf38-156e073ab384","issueId":"01a3c1cf-7dd5-4a13-b3ab-a9d064a3e31c","events":[{"type":"issue_deleted","issueId":"01a3c1cf-7dd5-4a13-b3ab-a9d064a3e31c","issueTitle":"Load data from remote sync engine."}],"__class":"Activity"}
{"id":"ec9ec347-4f90-465c-b8bc-e41dae4e11f2","createdAt":"2024-07-03T11:37:06.944Z","updatedAt":"2024-07-03T11:37:06.944Z","userId":"4e8622c7-0a24-412d-bf38-156e073ab384","issueId":"39946254-511c-4226-914f-d1669c9e5914","events":[{"type":"issue_deleted","issueId":"39946254-511c-4226-914f-d1669c9e5914","issueTitle":"Reverse engineering Linear's Sync Engine"}],"__class":"Activity"}
// many lines omitted here
_metadata_={"method":"mongo","lastSyncId":2326713666,"subscribedSyncGroups":["89388c30-9823-4b14-8140-4e0650fbb9eb","4e8622c7-0a24-412d-bf38-156e073ab384","AD619ACC-AAAA-4D84-AD23-61DDCA8319A0","CDA201A7-AAAA-45C5-888B-3CE8B747D26B"],"databaseVersion":948,"returnedModelsCount":{"Activity":6,"Cycle":2,"DocumentContent":5,"Favorite":1,"GitAutomationState":3,"Integration":1,"Issue":3,"IssueLabel":4,"NotificationSubscription":2,"Organization":1,"Project":2,"ProjectStatus":5,"Team":1,"TeamKey":1,"TeamMembership":1,"User":1,"UserSettings":1,"WorkflowState":7,"Initiative":1,"SyncAction":0}}
```

Each line (expect for the last) would be an instance of a model. Take an object for example:

```json
{
  "id": "556c8983-ca05-41a8-baa6-60b6e5d771c8",
  "createdAt": "2024-01-22T01:02:41.099Z",
  "updatedAt": "2024-05-16T08:23:31.724Z",
  "number": 1, // created sequence?
  "title": "Welcome to Linear ðŸ‘‹", // text encoding problem
  "priority": 1,
  "boardOrder": 0,
  "sortOrder": -84.71,
  "startedAt": "2024-05-16T08:16:57.239Z",
  "labelIds": ["30889eaf-fac5-4d4d-8085-a4c3bd80e588"],
  "teamId": "89388c30-9823-4b14-8140-4e0650fbb9eb",
  "projectId": "3e7ada3c-f833-4b9c-b325-6db37285fa11",
  "projectMilestoneId": "397b95c4-3ee2-47b0-bad1-d6b1c7003616",
  "subscriberIds": ["4e8622c7-0a24-412d-bf38-156e073ab384"],
  "previousIdentifiers": [],
  "assigneeId": "4e8622c7-0a24-412d-bf38-156e073ab384",
  "stateId": "030a7891-2ba5-4f5b-9597-b750950cd866",
  "reactionData": [],
  "__class": "Issue"
}
```

It is an `Issue`. The `id` is the primary key in the issue table. Additionally, they use UUIDs to generate IDs and establish references among models using these IDs. Also note that LSE employs fractional indexing to handle sorting.

The last line of the response is a metadata object. The `lastSyncId` `databaseVersion` would later be saved into metadata.

```json
{
  "method": "mongo",
  "lastSyncId": 2326713666,
  "subscribedSyncGroups": [
    "89388c30-9823-4b14-8140-4e0650fbb9eb",
    "4e8622c7-0a24-412d-bf38-156e073ab384",
    "AD619ACC-AAAA-4D84-AD23-61DDCA8319A0",
    "CDA201A7-AAAA-45C5-888B-3CE8B747D26B"
  ],
  "databaseVersion": 948,
  "returnedModelsCount": {
    "Activity": 6,
    "Cycle": 2,
    "DocumentContent": 5,
    "Favorite": 1,
    "GitAutomationState": 3,
    "Integration": 1,
    "Issue": 3,
    "IssueLabel": 4,
    "NotificationSubscription": 2,
    "Organization": 1,
    "Project": 2,
    "ProjectStatus": 5,
    "Team": 1,
    "TeamKey": 1,
    "TeamMembership": 1,
    "User": 1,
    "UserSettings": 1,
    "WorkflowState": 7,
    "Initiative": 1,
    "SyncAction": 0
  }
}
```

## Init In-Memory Data and Database

### Object Pool

## Lazy Loading & Hydration

LSE doesn't load everything into memory at once. When you lick on issues or projects, there might be network requests to load additional data from the server (sometimes from IndexedDB). This process is known as hydration.

Classes with `hydrate` method could be hydrated, including `Model` `LazyReferenceCollection` `LazyReference` `RequestCollection` and `LazyBackReference` .

Hydration for lazy reference collections are much more complex than

`processPartialIndexInfoForModel` generates an array that displays which models have back references to a given model. This process is performed recursively up to three layers deep.

```jsx
[
  {
    path: "issueId",
    referencedModelName: "Issue",
  },
  {
    path: "issue.teamId",
    referencedModelName: "Team",
  },
  {
    path: "issue.cycleId",
    referencedModelName: "Cycle",
  },
  {
    path: "issue.projectId",
    referencedModelName: "Project",
  },
  {
    path: "issue.projectMilestoneId",
    referencedModelName: "ProjectMilestone",
  },
  {
    path: "issue.creatorId",
    referencedModelName: "User",
  },
  {
    path: "issue.assigneeId",
    referencedModelName: "User",
  },
  {
    path: "issue.subscriberIds",
    referencedModelName: "User",
  },
  {
    path: "issue.stateId",
    referencedModelName: "WorkflowState",
  },
];
```

![[Pasted image 20241005165418.png]]

- [ ] this part has not finished yet

`resolveCoveringPartialIndexValues`

```jsx
[
  "issueId-c775f0f8-7c85-4da6-998a-939eccd337ed",
  "369af3b8-7d07-426f-aaad-773eccd97202",
  "issue.subscriberIds-e86b9ddf-819e-4e77-8323-55dd488cb17c",
  "issue.stateId-1595ecd3-0185-4b65-b172-141040c346aa",
];
```

#### BatchUpdater

### Takeaway

## Part III: Syncing

In the final part, we would talk about how LSE sync between clients and the server. What exactly happens when we change the assignee of an issue? How does LSE manage undo/redo functionality, networking, error handling, offline caching, and many other details within these two lines?

```jsx
issue.assignee = user;
issue.save();
```

## The Whole Picture

First, let's review the entire process, then we can delve into the details.

![[Pasted image 20241009161445.png]]

1. When a property is assigned a value, decorators will bookkeeper what property has been changed and what the new value and the old value are (1), and perhaps also update back references.
2. Once `save()` is called, the model call `SyncedStore` , subsequently `SyncedClient` and `TransactionQueue` to generate a n `Transaction` (2).
3. The generated transaction is pushed into a queue (sometimes sent immediately) and saved into the `__transactions` table in IndexedDB (3). `TransactionQueue` schedule a timer to repeatedly sending these pending transactions to the server.
4. Later, it is sent to the backend along with other transactions in a batch, a process known as transaction execution (4).
5. If the transaction is accepted by the backend, it is removed from the `__transactions` table (5) (6).
6. Subsequently, the backend sends a delta to `SyncClient` (7), informing clients of changes made on the server. `SyncClient` rebases transactions that haven't been accepted by the server, modifies in-memory models (8), and updates the tables in IndexedDB (9).

## Find Out What Has Been Changed

- `M1`
- `as#propertyChanged`
- `as#markPropertyChanged`
- `as#referencedPropertyChanged`
- `as#updateReferencedModel`

As we have discussed in section "Observability", LSE uses a decorator `M1` to make a property observable. It also plays a critical part in generating transactions. When a property of a model is assigned with a new value, the setter would intercept the assignment and call `propertyChanged` to record the name of that property, the old value and the new value. `markPropertyChanged` would then be called to serialize the old value and store it in `modifiedProperties`.

<aside>
💡 In addition to 
</aside>

## Generate Transactions

There are various types of transactions:

| Minimized name | Possible original names | Description                                                             |
|----------------|-------------------------|-------------------------------------------------------------------------|
| `M3` `Zo`      | `BaseTransaction`       | The base class of all transactions.                                     |
| `Hu`           | `CreationTransaction`   | The transaction to add an model.                                        |
| `zu`           | `UpdateTransaction`     | The transaction to update properties on an existing model.              |
| `g3`           | `DeleteTransaction`     | The transaction to delete a model. E.g. deleting a comment of an issue. |
| `m3`           | `ArchiveTransaction`    | The transaction to archive a model. E.g. deleting an issue.             |
| `y3`           | `UnarchiveTransaction`  | The transaction to unarchive a model.                                   |

`uce` (`TransactionQueue`) provides methods to create and execute transactions. But how does LSE determine which type of transaction it should generate, and what is the content of a transaction?

When the model object's `save` method is called, it invokes the `save` method of `SyncedStore`. If the model exists in `SyncClient`, an `UpdateTransaction` is generated. During the construction of `UpdateTransaction`, the model’s `changeSnapshot` function is called. Ultimately, an object is generated to represent the changes:

```jsx
{
  assigneeId: {
    original: this.modifiedProperties[n],
    updatedFrom: this.modifiedProperties[n],
    updated: i,
    unoptimizedUpdated: s
  }
}
```

## Transaction Queue

`TransactionQueue` use four arrays to manage transactions.

1. `createdTransactions` When a transaction is queued, it first goes to `createdTransactions`. A `commitCreatedTransactions` scheduler periodically moves all transactions in this array to the end of `queuedTransactions`.
2. `queueTransactions` These transactions are pushed to a queue, waiting to be executed. Transactions would all be saved into `__transactions` when are are move to `queuedTransactions` .
3. `executingTransactions` These transactions have been sent to the server but have not been accepted yet. The `TransactionQueue` has a `dequeueTransaction` scheduler that periodically moves transactions from `queuedTransactions` to `executingTransactions` in a FIFO manner.
4. `completedButUnsyncedTransactions` These transactions have been sent to the server and accepted, but the corresponding delta has not been received. When a set of transactions from `executingTransactions` is executed, they are removed from `executingTransactions` and pushed to `completedButUnsyncedTransactions`. Note that `completedButUnsyncedTransactions` are removed from `__transactions` in IndexedDB.
5. `persistedTransactionsEnqueue` When the database bootstraps, transactions saved in `__transactions` would be loaded into this array. After remote updates have been processed, they are moved to `queueTransactions` and waiting to be executed. During processing remove updates, these transactions may need to rebase on deltas.

At any given moment, transactions in `completedButUnsyncedTransactions` must be created earlier than those in `executingTransactions`, and the same applies to any other two arrays. This is important because the sequence plays a role when rebasing transactions.

## Generating GraphQL Mutating Queries

Before sending a batch of transactions to the server, it is converted into GraphQL mutating queries. Each transaction calls the `prepare` function to generate parameters (there are a lot of details but I am not going to cover in this post). These parameters are then used in `TransactionExecutor.execute` to create the query path and parameters.

```jsx
mutation IssueUpdate($issueUpdateInput: IssueUpdateInput!) {
  issueUpdate(id: "a8e26eed-7ad4-43c6-a505-cc6a42b98117", input: $issueUpdateInput
) { lastSyncId } }
```

```jsx
{
    "issueUpdateInput": {
        "assigneeId": "e86b9ddf-819e-4e77-8323-55dd488cb17c"
    }
}
```

## Delta, Actions and Rebasing

Delta from the server would be executed in the `applyDelta` method. After we modified the assignee, the delta would look like this:

A delta includes one or more actions, each identified by an `action` type. The current types are:

1. `I` for insertion
2. `U` for updating
3. `A` for archiving
4. `D` for deleting
5. `C` remains unknown
6. `G` remains unknown
7. `S` remains unknown
8. `V` remains unknown

For the updating action, LSE broadcasts all properties of the model, not just the changed ones. The model’s `updateFromData` would be called to update all properties.

```jsx
[
  {
    id: 2361610825,
    modelName: "Issue",
    modelId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
    action: "U",
    data: {
      id: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
      title: "Connect to Slack",
      number: 3,
      teamId: "369af3b8-7d07-426f-aaad-773eccd97202",
      stateId: "28d78a58-9fc1-4bf1-b1a3-8887bdbebca4",
      labelIds: [],
      priority: 3,
      createdAt: "2024-05-29T03:08:15.383Z",
      sortOrder: -12246.37,
      updatedAt: "2024-07-13T06:25:40.612Z",
      assigneeId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
      boardOrder: 0,
      reactionData: [],
      subscriberIds: ["e86b9ddf-819e-4e77-8323-55dd488cb17c"],
      prioritySortOrder: -12246.37,
      previousIdentifiers: [],
    },
    __class: "SyncAction",
  },
  {
    id: 2361610826,
    modelName: "IssueHistory",
    modelId: "ac1c69bb-a37e-4148-9a35-94413dde172d",
    action: "I",
    data: {
      id: "ac1c69bb-a37e-4148-9a35-94413dde172d",
      actorId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
      issueId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
      createdAt: "2024-07-13T06:25:40.581Z",
      updatedAt: "2024-07-13T06:25:40.581Z",
      toAssigneeId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
    },
    __class: "SyncAction",
  },
  {
    id: 2361610854,
    modelName: "Activity",
    modelId: "1321dc17-cceb-4708-8485-2406d7efdfc5",
    action: "I",
    data: {
      id: "1321dc17-cceb-4708-8485-2406d7efdfc5",
      events: [
        {
          type: "issue_updated",
          issueId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
          issueTitle: "Connect to Slack",
          changedColumns: ["subscriberIds", "assigneeId"],
        },
      ],
      userId: "e86b9ddf-819e-4e77-8323-55dd488cb17c",
      issueId: "a8e26eed-7ad4-43c6-a505-cc6a42b98117",
      createdAt: "2024-07-13T06:25:41.924Z",
      updatedAt: "2024-07-13T06:25:41.924Z",
    },
    __class: "SyncAction",
  },
];
```

When applying a delta, it might conflict with local changes (transactions). For updating actions and transactions, LSE needs to rebase transactions onto the model modified by the action. This starts with the `TransactionQueue.rebaseTransactions` method.

LSE rebase all transactions in a specific order: `completedButUnsyncedTransactions`, `executingTransactions`, `queuedTransactions`, and `persistedTransactionsEnqueue`, as mentioned earlier. In terms of rebasing, it:

1. Adjusts the original transaction values to those specified by the action. For example, if I change the title of an issue from "A" to "B", and my colleagues change it from “A” to “C”, during rebasing on my device, my transaction would update from `A -> B` to `C -> B`.
2. Updates the model’s properties again since they have been altered by the action. This inevitably leads to some back-and-forth value assignments.

As Tuomas mentioned in his talk, this follows a last-writer-wins strategy.

### Side effects on the server

Issue history.

If you made any changes offline and then reload the page, it may seem that the changes are lost. However, they are not. The transactions are stored in the database and will be sent to the server once you are back online. **Linear does not apply these transactions locally as they are designed to be executed only on the backend.** In other words, the frontend and backend use different methods to update data. The backend needs to check authentication, generate history records, send emails, and so on. Linear refers to these as 'side effects'.

<aside>
💡 Linear has to do some side effects on the server.
</aside>

## Error handling

## Undo & redo

Undo and redo operations in LSE are based on transactions. Each transaction type has a specific `undoTransaction` method. This method executes the undo logic and returns another transaction for redo purposes. For example, the `undoTransaction` method of `UpdateTransaction` reassigns the model's property with its previous values and returns another `UpdateTransaction` to the `UndoQueue`. It is important to note that when a transaction performs its undo logic, a new transaction is created and added to the `queuedTransactions` to ensure synchronization.

But how does the `UndoManager` determine which transactions should be appended to the undo redo stack? It turns out that Linear UI is responsible for identifying the differences.

```jsx
n.title !== d &&
  o.undoQueue.addOperation(
    s.jsxs(s.Fragment, {
      children: [
        "update title of issue ",
        s.jsx(Le, {
          model: n,
        }),
      ],
    }),
    () => {
      //
      (n.title = d), n.save();
    }
  );
```

When an edit is made, the UI calls `UndoQueue.addOperation` to have `UndoQueue` subscribe to the next `transactionQueuedSignal` and create an undo item. This signal is triggered when transactions are added to `queuedTransactions`. The subscription ends when a callback finishes execution. What is in that callback? It is `save()`, our old friend! So when we perform an undo, although the signal is triggered, since `UndoQueue` is not listening to it, no extra undo item will be created.

## Add, Delete and Archive

For `DeleteTransaction`, `ArchiveTransaction`, and `UnarchiveTransaction`, there are specific methods on models to generate these transactions. We would not take a deeper investigation into them.

## Takeaway

A quick summary of this part:

1. **Client-side operations will never directly alter the tables in the local database!** Only data synced from the server would do that. You can consider
2. Unlike OT, which only sends an acknowledgment signal to the mutator client, **LSE consistently sends all properties of modified models to all clients, regardless of whether one of them is the mutator**. It really simplifies things.
3. LSE use simple **last-writer-win strategy to handle conflicts**, and only handle conflicts for updating actions.

## Conclusion

## Lookup

| Minimized names                                                                                                                                         | Possible original names                                                     | A short description                                                                                                                                                            |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Me` `rr`                                                                                                                                               | `ModelRegistry`                                                             |                                                                                                                                                                                |
| `w`                                                                                                                                                     | `ApplicationStore`                                                          |                                                                                                                                                                                |
| `Ln`                                                                                                                                                    | `makeObservable`                                                            | `MobX` API to make an object observable.                                                                                                                                       |
| `sg` `hf` `km`                                                                                                                                          | `SyncedStore`                                                               | It is the `store` in models. It provides lots of methods to get or manipulate models.                                                                                          |
| `Hr`                                                                                                                                                    | `SyncedClient`                                                              |                                                                                                                                                                                |
| `Dt`                                                                                                                                                    | `Database`                                                                  |                                                                                                                                                                                |
| `jn`                                                                                                                                                    | `DatabaseManager`                                                           |                                                                                                                                                                                |
| `cce`                                                                                                                                                   | `StoreManager`                                                              | The in-memory store manager. It manages lots of object stores. Each store maps to a model, and maps to a table in the database.                                                |
| `TE`                                                                                                                                                    | `FullStore`                                                                 | A store which content should be loaded all-at-once.                                                                                                                            |
| `Jm` `p3`                                                                                                                                               | `PartialStore`                                                              | A store which content can be partially loaded. It extends `FullStore` .                                                                                                        |
| `uce`                                                                                                                                                   | `TransactionQueue`                                                          |                                                                                                                                                                                |
| `A8` `sd`                                                                                                                                               | `GraphQLClient`                                                             | A GraphQL client. It would be used by many classes.                                                                                                                            |
| `w`                                                                                                                                                     | `Property`                                                                  | This refers to a self-owned property of a model. For example, `title` is a property of `Issue`.                                                                                |
| `_g`                                                                                                                                                    | `EphemeralProperty`                                                         | This is also a self-owned property of a model but is not stored in the database. For example, `lastUserInteraction` is an ephemeral property of `User`.                        |
|                                                                                                                                                         |                                                                             | For example, `documentContentId` of `Issue` refers to the document model's ID of the issue's description.                                                                      |
| `pe`                                                                                                                                                    | `Reference`                                                                 |                                                                                                                                                                                |
| `xe` `Nt`                                                                                                                                               | `ReferenceCollection` `LazyReferenceCollection`                             | This is used for 1 to many relationships. For example, `issues` is a lazy reference collection of `IssueLabel`. And `notifications` is a reference collection of `Initiative`. |
| `pe` `hr` `Ue` `g5` `dt` `kl`                                                                                                                           | `WithBackReference` `LazyWithBackReference` `Reference` `LazyBackReference` | These 6 decorators are used to declare reference and back references with different options.                                                                                   |
| `ReferenceArray`                                                                                                                                        | `ii`                                                                        |                                                                                                                                                                                |
| Many to many relationships. For example, `labels` are reference array of `Issue`. (But `issues` are lazy reference collection of `Label`).              |                                                                             |                                                                                                                                                                                |
| `as`                                                                                                                                                    | `Model`                                                                     |                                                                                                                                                                                |
| The base class that each model will inherit. The class declares a `store` and `__mobx` . The are the critical parts of data fetching and observability. |                                                                             |                                                                                                                                                                                |
| The class provides lots of static methods to register models, properties and their metadata.                                                            |                                                                             |                                                                                                                                                                                |
| `Qc`                                                                                                                                                    | `UUID`                                                                      | A helper function to generate UUID.                                                                                                                                            |

--- 


# React 实践揭秘之旅，中高级前端必备(下)

## 引言

上一篇文章我们主要实现了 **JSX** 在 `WebGL` 上的渲染与更新，对 **虚拟DOM** 和 **Diff** 有了更深的了解，但相比于我们使用的 `React`，还缺乏了之中很重要的一环 --- **组件模式**。

想必大家能认同，React组件(`Component`)具有 **强大的功能，高拓展性和高解耦性**，在其基础上构建的各种 `UI` 组件框架完全改变了传统的 Web 开发模式，成为了Web 中的大型复杂应用提供了一种很好的构建模式和保障，也让我们的开发效率也有了质的变化。

在这篇文章中，我们会在上一篇的实现基础上，加入 `React` 组件模式，并在实现的过程中适时的去讲解一些原理和思维，也有利于大家 **由浅入深的理解** 和 **编程思维上的提升**。

由于下篇是完全以上篇作为基础的。所以如果你还没看过上篇，请优先猛戳:

**[React 实践揭秘之旅，中高级前端必备(上) ->](https://github.com/xd-tayde/blog/blob/master/ReactGL-1.md)**

## 第七站: 破碎之墟 - Component

**代码复用性**，向来是编程领域一个核心的理念。我们最常使用 **函数、类** 方式进行代码的封装，但这里有个痛点: 

**Web 中 UI 与 逻辑 的分离的特性，导致较难优雅地整合封装。**

通常我们需要引 `JS`、`CSS`，再在 `HTML` 中按规定书写结构，最后再初始化库。整个过程十分割裂，且不优雅。

而 `React Component` 帮我们解决了这个痛点，即保持了 **动态UI** 与 **逻辑** 的分离，又有了全新的整合方式。从而让 UI 组件变得非常的 **高效易用**，只需要在结构中以标签的形式引入即可。

我们就继续在上篇文章的基础上，加入 `Component` 的特性。基于 `JSX` 的变量传递，我们只需要实现一个 `Component` 类，并针对性地去完成组件的 **渲染和更新** 即可。

> TIPs: 
> 
> 由于组件的加入，这时我们的 **虚拟DOM(VNode)** 就包含了两种类型: **组件节点(compVNode)** 和 **元素节点(elVNode)**。后续都会以此区分，便于讲解。

```js
// 组件基类
class Component {
    // 通过保持三份不同的时期的快照
    // 有利于对组件的状态及属性的管理和追踪

    // 属性
    public __prevProps
    public props = {}
    public __nextProps
    
    // 状态
    public __prevState
    public state = {}
    public __nextState
    
    // 储存当前组件渲染出的 VNode
    public __vnode
    
    // 储存 组件节点
    public __component
    
    constructor(props) {
        // 初始化参数
        this.props = props
    }
    public render(): any { return null }
    public __createVNode() {
        // 该方法用于封装重新执行 render 的逻辑
    
        // state 与 props 状态变更的逻辑
        this.__prevProps = this.props
        this.__prevState = this.state

        this.props = this.__nextProps
        this.state = this.__nextState

        this.__nextState = this.__nextProps = undefined
        
        // 重新执行 render 生成 VNode
        this.__vnode = this.render()
        
        return this.__vnode
    }
}
```

有了这个类后，我们就可以使用 `React` 的方式继承出 **自定义组件** 进行渲染:

```js
class App extends Component {
    constructor(props) {
        super(props)
        this.state = {
            content: 'ReactWebGL Component, Hello World',
        }
    }
    public render() {
        return (
            <Container name="parent">
                {this.state.content}
            </Container>
        )
    }
}

render(<App />, game.stage)
```

由于我们之前提过，`JSX` 是以 **变量传递** 的形式编译的。因此将 `<App />` 转换成 `VNode` 后，其 **`vnode.type` 的值便是 `App` 这个类**，并不是类似 `div` 那样的 **字符串标签名**。所以接下来我们需要实现一个 **组件初始化** 的函数(`Component`)。这个函数的主要功能是:  

**实例化组件并获取组件 `render` 方法中返回的 元素节点。**

```js
// 渲染组件
function renderComponent(compVNode) {
    const { type: Comp, props } = compVNode
    let instance = compVNode.instance
    
    // 当 instance 已经存在时，则不需要重新创建实例
    if (!instance) {
        // 传入 props，初始化组件实例
        // 支持 类组件 或者 函数组件
        if (Comp.prototype && Comp.prototype.render) {
            instance = new Comp(props)
        } else {
            instance = new Component(props)
            instance.constructor = Comp
            instance.render = () => instance.constructor(props)
        }
        
	    // 初次渲染时
	    // 将来的属性与状态其实便是与当前值一致
	    instance.__nextProps = props
	    instance.__nextState = instance.state
    }    
    
    // 调用 render 获取 VNode
    const vnode = instance.__createVNode()

    // 组件、元素、实例之间保持相互引用，有利于双向连接整棵虚拟树
    instance.__component = compVNode
    compVNode.instance = instance
    compVNode.vnode = vnode
    vnode.component = compVNode

    return vnode
}
```

接下来我们只需要在 `createElm` 的函数加入: 当传入的为 **组件节点** 时调用函数初始化生成 **元素节点**，后续只需要继续原有的逻辑继续创建，便能正确渲染组件。

```js
function createElm(vnode) {
	// 当为组件时，初始化组件
	// 重新赋值成 元素节点
    if (typeof vnode.type === 'function') {
        vnode = renderComponent(vnode) 
    }
    
    // 维持原有逻辑
    ...
    
    return vnode.elm
}
```

保存执行，页面中已经能正确的渲染出 `<App>` 组件了。延续之前的逻辑，在完成初次渲染后，接下来就是组件的更新。这就是我们最常使用的 `this.setState`了。

## 第八站: 净化之匙 - setState

在上一篇文章中，我们实现了 **虚拟DOM** 更新函数 `diff`。参数为 **新旧虚拟节点(`oldVNode`、`newVNode`)**。所以组件更新的原理也一样: 

**获取组件实例先后渲染的新旧 `VNode` 再触发 `diff` 函数。**

在刚才 `Component `的渲染中，我们已经把 `render` 生成的 `VNode` 保存在 `this.__vnode` 上，这便是初始化时生成的 **旧虚拟节点(`oldVNode`)**。所以我们要做的就是: 

**`setState` 中 更新状态，调用 `render` 生成 新虚拟节点(`newVNode`)，触发 `diff` 更新。**

因此我们在类上新增两个方法: `setState` 和 `__update`:

```js
class Component {
	// 其余方法
	...
	
	// 更新函数
	public __update = () => {
        // 临时存储 旧虚拟节点 (oldVNode)
        const oldVNode = this.__vnode
        
        this.__nextProps = this.props
        if (!this.__nextState) this.__nextState = this.state

        // 重新生成 新虚拟节点(newVNode)
        this.__vnode = this.__createVNode()

        // 调用 diff，更新 VNode
        diff(oldVNode, this.__vnode)
    }
    
    // 更新状态
    public setState(partialState, callback?) {
        // 合并状态, 暂存于 即将更新态 中
        if (typeof partialState === 'function') {
            partialState = partialState(this.state)
        }
        this.__nextState = {
            ...this.state,
            ...partialState,
        }

        // 调用更新，并执行回调
        this.__update()
        callback && callback()
    }
}
```

### 更新优化
 
到这里我们可以看出: `setState` 封装了 `diff` 方法。但由于 `diff` 的复杂度，性能的优化会是一个我们需要着重考虑的点。每执行一次 `setState`，就需要重新生成 `newVNode` 进行 `diff`。因此，当组件非常复杂时或者连续更新时，可能会导致 **主进程的阻塞，造成页面的卡死。**

这里我们需要有两个优化:

- **`setState` 异步化，避免阻塞主进程；**

- **`setState` 合并，多次连续调用会被最终合并成一次；**

	- 同一组件 **连续多次更新**；
	
	- 父子级连续触发更新，由于 **父级更新其实已经包含子级的更新**，此时如果子级再自我更新一次，则就变成了一种无谓消耗； 

为了这个优化，我们首先需要一个更新队列功能: 

- **可以异步化调用更新**；

- **为组件标注，保证一次循环中单个组件只会更新一次**；

我们先来实现个 异步执行队列: 

```js
// 更新异步化，采用属于微任务的 Promise，兼容性使用 setTimeout
// 这里使用微任务，可以保证宏任务的优先执行
// 保证例如 UI渲染 等更为重要的任务，避免页面卡顿；
const defer = typeof Promise === 'function' 
    ? Promise.prototype.then.bind(Promise.resolve())
    : setTimeout

// 更新队列
const updateQueue: any[] = []

// 队列更新 API
export function enqueueRender(updater) {

    // 将所有 updater 同步推入更新队列中
    // 为实例添加一个属性 __dirty，标识是否处于待更新状态
    // 初始 和 更新完毕，该值会被置为 false
    // 推入队列时，标记为 true
    if (
        !updater.__dirty && 
        (updater.__dirty = true) && 
        updateQueue.push(updater) === 1
    ) {
        // 异步化冲洗队列
        // 最终只执行一次冲洗
        defer(flushRenderQueue)
    }
}

// 合并一次循环中多次 updater
function flushRenderQueue() {
    if (updateQueue.length) {
        // 排序更新队列
        updateQueue.sort()

        // 循环队列出栈
        let curUpdater = updateQueue.pop()
        while (curUpdater) {
        
            // 当组件处于 待更新态 时，触发组件更新
            // 如果该组件已经被更新完毕，则该状态为 false
            // 则后续的更新均不再执行
            if (curUpdater.__dirty) {
                // 调用组件自身的更新函数
                curUpdater.__update()

                // 执行 callback
                flushCallback(curUpdater)
            }
            
            curUpdater = updateQueue.pop()
        }
    }
}

// 执行缓存在 updater.__setStateCallbacks 上的回调
function flushCallback(updater) {
    const callbacks = updater.__setStateCallbacks
    let cbk
    if (callbacks && callbacks.length) {
        while (cbk = callbacks.shift()) cbk.call(updater)
    }   
}
```

完成这个方法后，我们来修改下上面的 `setState` 函数: 

```js
class Component {
	// 其余方法
	...
	
	public setState(partialState = {}, callback?) {
        // 合并状态, 暂存于 即将更新态 中
        // 处理参数为函数的形式
        if (typeof partialState === 'function') {
            partialState = partialState(this.state, this.props)
        }
        
        this.__nextState = {
            ...this.state,
            ...partialState,
        }

        // 缓存回调
        callback && this.__setStateCallbacks.push(callback)
        // 把组件自身先推入更新队列
        enqueueUpdate(this)
	}
}
```

此时，由于 **更新队列** 为异步的，因此当多次连续调用 `setState` 时，组件的状态会被 **同步合并**，待全部完成后，才会进入更新队列的冲洗并最终只执行一次组件更新。

React 中组件还有个更新的方法: `forceUpdate(callback)`，该方法的功能其实是与 `this.setState({}, callback)` 相同的，唯一有个需要注意的点就是: **触发的更新不需要经过 `shouldComponentUpdate` 的判断**，实现只需要加个标识位即可。

### 优化策略

回到性能优化这个点，从这里的简单实现我们可以看出: 虽然异步化了更新流程，但本质上仍然没有解决 复杂的组件 `diff` 带来长时间执行阻塞主进程。我记得以前文章有说过: **最有效的性能优化方式就是 异步、任务分割 和 缓存策略。**

#### 1. 异步化: 

通过把同步的代码执行变成异步，把串行变成并行，可以有效提高 **执行的时间利用率** 和 **保证代码优先级**。从这里可以延伸出两种优化方向:

- 1. **异步**: 如我们上面所做的优化，这样能保证主进程的执行优先级，保证页面渲染或者更主要任务的优先执行，避免卡顿；
- 2. **并行**: 通过把某些高消耗的操作放到 **非主进程** 上执行，例如 worker 线程。不过由于 `diff` 本身就较为复杂，还要需要处理好主进程与线程之间的交互，会导致复杂度极高，但也并非不可行，后续也许是个优化方向。
	- 例如我就在思考在这里引入 wasm 的可能性，代价与收益比如何，有兴趣的童鞋可以一起探讨。  
	
#### 2. 任务分割

将原本会阻塞主进程的 **大块逻辑执行进行拆解，分割成一个个小任务**。从而可以在逻辑中找到合适的时机点       **分段执行**，即 **不会阻塞主进程，又可以让代码快速高效的执行，最大化利用物理资源。**

Facebook 的大神们选择了这条优化方向，这就是 React 16 新引入的 `Fiber` 理念的最主要目的。上面我们实现的 `diff` 中，有着一个很大的障碍: 

**一棵完整 虚拟DOM树 更新，必须一次性更新完成，中间无法被暂停，也无法被分割。**

而 `Fiber` 最主要的功能就是 **指针映射，保存上一个更新的组件与下一步需要更新的组件**，从而完成 **可暂停可重启**。计算进程的运行时间，利用浏览器的 `requestIdleCallback` 与 `requestAnimationFrame` 接口，当有优先级更高的任务时，优先执行，暂停下一个组件的更新。待空闲时再重启更新。

`Fiber` 算是一种编程思想，在其它语言中也有许多应用(`Ruby Fiber`)。核心思想是:

**任务拆分和协同，主动把执行权交给主线程，使主线程有时间空挡处理其他高优先级任务。**

但实现复杂度较高，为了本文便于理解，暂时并没有引入。等以后有机会我们再来一起深挖 `Fiber` 的实现，也许能成为更多使用场景的性能优化手段。

### 子组件更新

在上篇文章中，我们优先实现了 `diffVNode` 方法用于更新 **元素节点**，但组件节点的更新与元素节点的更新是不同的。当出现组件嵌套的情况时，我们就需要一个新的方法(`diffComponent`)用于组件节点的更新。

与 **元素节点** 不同，组件节点之间的更新重要的是重渲染，类似于我们上面的 `setState`。

**复用已创建好的组件实例，根据新的 状态(`state`)与 属性(`props`) 重新执行 `render` 生成 元素节点，再递归比对**，

也就是说，我们需要在 `diffVNode` 外围再做一层判断处理:

```js
function diff(oldVNode, newVNode) {
	if (isSameVNode(oldVNode, newVNode)) {
        if (typeof oldVNode.type === 'function') {
            // 组件节点
            diffComponent(oldVNode, newVNode)
        } else {
            // 元素节点，
            // 直接执行比对
            diffVNode(oldVNode, newVNode)
        }
	} else {
		// 新节点替换旧节点
		...
	}
}


// 组件比对
function diffComponent(oldCompVNode, newCompVNode) {
    const { instance, vnode: oldVNode, elm } = oldCompVNode
    const { props: nextProps } = newCompVNode

    if (instance && oldVNode) {
        instance.__dirty = false
        
        // 更新状态和属性
        instance.__nextProps = nextProps
        if (!instance.__nextState) instance.__nextState = instance.state
        
        // 复用旧组件实例和元素
        newCompVNode.instance = instance
        newCompVNode.elm = elm

        // 使用新属性、新状态，旧组件实例
        // 重新生成 新虚拟DOM
        const newVNode = initComponent(newCompVNode)
        
        // 递归触发 diff
        diff(oldVNode, newVNode)
    }
}
```

## 第九站: 生命之泉 - Lifecycle

组件有一个相当重要的特征，便是具有 **生命周期**。不同的函数钩子对应了组件 **从初始化到销毁** 的各个关键时间点。主要是为了让业务方有能力 **插入组件的渲染工作流** 中，编写业务逻辑。我们先来简单梳理下最新 React 组件的生命周期:

### **首次渲染**:

- **`constructor`**

	- 即组件的 **实例化时机**，通常可以用来设置初始化 `state`；
	
- **`static getDerivedStateFromProps(nextProps, prevState)`**

	- 在组件的模板渲染中，我们通常使用的数据为 `state` 和 `props`，而 `props` 由父级传入，组件本身并无法直接修改，因此唯一的常见需求就是: **根据父级传入的 `props` 动态修改 `state`**。该生命周期就是为此而生；
	
	- 大家可能会有疑问: **该方法为什么为静态方法? 而不是常规的实例方法呢？**
	
		- 先肯定一点: **使用实例方法肯定也是能满足需求的**。但这个钩子比较特殊，它的执行时机是位于 **新状态合并之后，重渲染之前**，而且该方法会 **侵入更新机制** 中。如果在之中做例如修改状态之类的操作是十分不可控的。当设计为静态方法后，函数内部便无法访问组件实例，成为一个 **纯函数**，便能保证更新流程的安全与稳定。
		
- **`render()`**

	- 根据 `state` 和 `props`，生成 **虚拟DOM**；
	
- **`componentDidMount()`**

	- **组件被创建成真实元素并渲染后** 被调用，此时可获取真实的元素状态，主要用于业务逻辑的执行，例如数据请求，事件绑定等；

### **更新阶段**:

- **`static getDerivedStateFromProps(nextProps, prevState)`**

- **`shouldComponentUpdate(nextProps, nextState)`**

	- 上篇文章中的 `diff` 优化策略中有提到: 为了减少 **无谓的更新消耗**，赋予组件一个可以 **主动中断更新流** 的 `API`。根据参数中的 **更新属性** 和 **更新状态**，业务方自行判断是否需要继续往下执行 `diff`，从而能有效地提升 **更新性能**；
	
	- 大家记得 React 中有种组件叫 **纯组件(`PureComponent`)** 吧，其实这个类继承于普通的 `Component` 上封装的，可以减少多余的 `render`，提升性能。

		- 默认使用 `shouldComponentUpdate` 函数设定更新条件: **仅当 `props` 和 `state` 发生改变时，才会触发更新**。这里使用了 `Object` 浅层比对，也就是仅做第一层比对，即 **1. key 是否完全匹配；2. value 是否全等；** 所以如果需要超过一层的数据变动，纯组件即无法正确更新了；
		
		- 这也是为什么 React 提倡使用 **不变数据** 的原理，能有效地使用浅层比对；

		- **不变数据**: 提倡数据不可变，任何修改都需要返回一个新的对象，不能直接修改原对象，这样能有效提高比对效率，减少无谓性能损耗。
		
- **`render()`**

- **`getSnapshotBeforeUpdate(prevProps, prevState)`**

	- 替换旧版的 `componentWillUpdate`，触发时机点: 在数据状态已更新，最新 `VNode` 已生成，但 **真实元素还未被更新**；
	
	- 可以用来在 **更新之前** 从真实元素或状态中获取计算一些信息，便于在更新后使用；
	
- **`componentDidUpdate(prevProps, prevState, snapshot)`**

	- 组件更新完成后调用；
	
	- 可以用于 **监听数据变化**，使用 `setState` 时必须加条件，避免无限循环；

### **卸载阶段**:

- **`componentWillUnmount()`**
	- 组件即将被销毁。通常可以用于 **解绑事件**、**清除数据**、**释放内存** 等功能； 

我们也按这样的目标来在我们的 `Component` 上实现这些生命周期，那如何更好的组织生命周期呢？这里我考虑到的是: 

**组件作为元素的容器，生命周期的本质其实是 其渲染出的元素节点的生命周期**。
	
也就是说，关键点还是在于 **元素在视图中的工作流，何时被挂载 - 更新 - 卸载**。所以为了更好的维护性和拓展性，更理想的方式应该是 **为元素节点统一添加生命周期**，而不是单独为组件，这样可大大降低复杂度，增加可拓展性。

### 节点生命周期

那我们第一步先根据上面需要的生命周期来理一下需要哪些时机:

- **创建后(`create`)**；
- **挂载后(`insert`)**；
- **更新前(`willupdate`)**；
- **更新后(`update`)**；
- **删除前(`willremove`)**；

原理就很简单了，只需要在 `VNode` 工作流中的对应时期调用相应的生命周期函数即可。那我们现在 `VNode` 上新增一个属性 `hooks`，用于 **储存** 对应的生命周期函数:

```js
interface VNode {
	...
	
	hooks: {
        create?: (vnode) => void
        insert?: (vnode) => void
        willupdate?: (oldVNode, newVNode) => void
        update?: (oldVNode, newVNode) => void
        willremove?: (vnode) => void
    }
}
```

新增一个触发的方法(`fireVNodeHook`):

```js
function fireVNodeHook(vnode, name, ...data) {
    // 根据 生命周期名称
    // 执行储存在 VNode 上的对应函数即可
    const { hooks: _hooks } = vnode
    if (_hooks) {
        hook = _hooks[name]
        hook && hook(...data)
    }
}
```

有了这层基础方法后，我们只需要分别在之前所写的 **渲染与更新** 流程中的各个函数适时地触发就行了。

#### 1. `create`

该时机是在 **元素被创建后，但还未被挂载之前**。由于我们之前将逻辑统一收归为 `createElm`，因此只需要在该函数末尾统一加入触发即可。

```js
function createElm(vnode) {
	// 创建元素逻辑
	...
	
	// 触发 虚拟DOM 上储存的 钩子函数
	fireVNodeHook(vnode, 'create', vnode)
	
	return vnode.elm
}
```

#### 2. `insert`

**元素被挂载到视图** 上的时机。从元素的角度来看，就是被 `append` 到父级中的时机点。这个时机点比较分散，但也比较好加入，找到我们使用 `Api` 中的 `append` 方法加入，总共有三个地方:

- `render` 函数中加入对 **根节点** 的触发；
- `createElm` 函数中加入对 **所有子级** 的触发；
- `diffChildren` 列表比对中 **新增列表项** 的触发；

#### 3. `willupdate` 与 `update`

**更新之前** 与 **更新之后**，对应的便是我们的 `diff` 函数。由于最终均需走到 `diffVNode` 中，因此只需要在 `diffVNode` 开头和末尾触发即可。

#### 4. `willremove`

**元素被卸载时**，其实与 `insert` 类似，只需要关注 `Api` 中 `removeChild` 的调用时机即可。在 `diff` 列表比对期间，当新列表中不存在时，我们需要删除旧列表中的元素，也就是之前写的业务函数 `removeVNodes`。

### 组件生命周期

由于 **元素节点** 才是贯穿整棵 **虚拟DOM** 渲染与更新的关键，因此我们上面先实现的是对 **元素节点** 的生命周期触发。但是我们最终需要是 **组件节点** 的生命周期。由于 **组件节点** 与 **元素节点** 为一一对应的 **上下层级关系**，因此这里我们还需要做一层转接: 

**把组件节点的生命周期赋值给其生成的元素节点**。

首先我们先来为组件定义上生命周期，并定义一个中转对象 `__hooks`，实现 **组件节点周期与元素节点周期的转换**:

```js
class Component {
    public __hooks = {
        // 元素节点 插入时机，触发 didMount
        insert: () => this.componentDidMount(),

        // 元素节点 更新之前，触发 getSnapshotBeforeUpdate
        willupdate: (vnode) => {
            this.__snapshot = this.getSnapshotBeforeUpdate(this.__prevProps, this.__prevState)
        },

        // 元素节点 更新之后， 触发 didUpdate
        update: (oldVNode, vnode) => {
            this.componentDidUpdate(this.__prevProps, this.__prevState, this.__snapshot)
            this.__snapshot = undefined
        },

        // 元素节点 卸载之前， 触发 willUnmount
        willremove: (vnode) => this.componentWillUnmount(),
    }

    // 默认生命周期函数	
    // getDerivedStateFromProps(nextProps, state)
    public getSnapshotBeforeUpdate(prevProps, prevState) { return undefined }
    public shouldComponentUpdate(nextProps, nextState) { return true }
    public componentDidMount() { }
    public componentDidUpdate(prevProps, prevState, snapshot) { }
    public componentWillUnmount() { }
}
```

然后我们只需要在 `__createVNode` 方法中将 `this.__hooks` 赋值给生成出的 `VNode` 即可:

```js
class Component {
	...
	
	public __createVNode() {
	    // ...
	    this.__vnode = this.render()
	    
	    // 赋值给对应的 元素节点，
	    // 实现该 元素节点 与 组件 之间生命周期的绑定
	    this.__vnode.hooks = this.__hooks
	    
	    return this.__vnode
	}
}
```

最后，大家可能发现我们还有两个钩子没有实现: `getDerivedStateFromProps` 和 `shouldComponentUpdate`，这是因为这两个生命周期会影响到更新结果，因此需要 **深入到更新流程中**，无法单纯的通过 **元素节点** 的生命周期来实现。

但其实也很简单，就是在 **更新之前**，需要根据这两个函数的返回结果，适当调整下更新逻辑即可:

```js
// Componet 中的 __update 方法
class Component {
	// ...

	public __update = () => {
        // 临时存储 旧虚拟节点 (oldVNode)
        const oldVNode = this.__vnode
        
        this.__nextProps = this.props
        if (!this.__nextState) this.__nextState = this.state
        
        // 执行 getDerivedStateFromProps
        // 更新 state 状态
        const cls = this.constructor
        if (cls.getDerivedStateFromProps) {
            const state = cls.getDerivedStateFromProps(this.__nextProps, this.state)
            if (state) {
                this.__nextState = Object.assign(this.__nextState, state)
            }
        }
        
        // 在 diff 之前调用 shouldComponentUpdate 进行判断
        // true: 生成新 VNode，继续 diff
        // false: 清空状态
        if (this.shouldComponentUpdate(this.props, this.__nextState)) {
            // 重新生成 新虚拟节点(newVNode)
            this.__vnode = this.__createVNode()

            // 调用 diff，更新 VNode
            diff(oldVNode, this.__vnode)
        } else {
            // 清空状态更新
            this.__nextProps = this.__nextState = undefined
        }
        
        // 刚才 异步更新队列 中标识的组件 待更新状态
        // 在更新完后置为 false
        this.__dirty = false
	}
}
```

组件更新还有另外一个地方，即 `diffComponent`，也需要加入上述类似的执行和判断。完成这部分代码后，我们来简单测试个 DEMO:

- 1. `<App>`、`<BBB txt={this.state.txt} />` **正确渲染**；
- 2. 双组件 **渲染生命周期** 与预期一致；
- 3. **触发更新**，调用 `<App> setState`，`<BBB>` 文字元素正确更新；
- 4. 双组件 **更新生命周期** 与预期一致；

![](https://user-gold-cdn.xitu.io/2020/3/24/1710a381f500d6d3?w=550&h=417&f=gif&s=4875150)

<p style="text-align: center;font-weight: bold;">图1. 生命周期演示DEMO</p>

## 最后一站: 旅途之末

在这系列文章中，我们实现了 React 最核心的部分: **JSX、组件、渲染、更新**。我们基于 **动手实践** 的方式，循序渐进地探讨了一些原理与策略，得出一些最佳实践。相信走完这遍旅程后，大家能对 React 有了更深层次的了解，能够给到各位小伙伴启发与帮助。其实我也一样，也是在这个旅程中跟大家一起共同学习，共同成长。

**[React 实践揭秘之旅，中高级前端必备(上) ->](https://github.com/xd-tayde/blog/blob/master/ReactGL-1.md)**

还有许多模块，如 `Context`、`Refs`、`Fragment` 和 一些全局API，如 `cloneElement` 等，还有代码中一些更严谨的判断及边界情况的处理，并没有在文章中体现，主要是由于这些部分更多的是纯逻辑的扩展，同时也是为了便于理解。如果童鞋们有兴趣，可以到 github 中查看完整版代码:

**[react-webgl.js ->](https://github.com/xd-tayde/react-webgl.js)**

另外我也想稍微唠嗑下关于 React-WebGL 这个想法。

近阶段我接触了一些 Web 游戏的开发，有了一些从前端开发者出发的思考与理解。在游戏开发领域，传统的游戏开发者有着一套与前端领域完全不同的思维编程模式。随着 Web 的发展，使他们需要拓展到 Js 的环境中。所以出现了一系列的游戏引擎库，本质上是从其它平台的库移植过来的。当我从一个前端开发者的角度在进行开发时，其实并不是说入门难，学习成本高，而是给我的感觉是: 类似用纯原生 js 在写页面，觉得效率低下。所以这也是 React-WebGL 的出发点，期望能将现在 Web 中更优秀的理念运用到游戏开发，甚至找到一种更高效的开发模式，提升效率，完善生态。

当然，这仅仅只是一个起点， 游戏开发 与 界面开发 确实有着许多异同点，**如何找到一种更现代化更高效的 Web 游戏开发模式**，这还需要很长的一段旅程。我也一直在思考，一直在摸索，相信能有一些好玩的东西。**没有尝试，没有努力，就千万别在起点就放弃了**。有什么问题，有什么想法，直接找我一起探讨哈。🙃~~

> Tips:
> 
> 看博主写得这么辛苦下，跪求点赞、关注、Star！[更多文章猛戳 ->](https://github.com/xd-tayde/blog) 
> 
> 邮箱: 159042708@qq.com  微信/QQ: 159042708

**[祝福#感恩#武汉加油##RIP KOBE#]()**




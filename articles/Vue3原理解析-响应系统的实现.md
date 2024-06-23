# 概要
#### ```Vue3```框架的```响应系统```是其核心特性之一，文章将会简单介绍其背景，然后阐述```响应系统```的实现思路并给出基础版本代码。接下来会在基础版本代码之上逐渐迭代，达到完善形态。
#### 文章的主要代码来自于《Vue.js 设计与实现》一书。但是不同的是，这里给出的代码是```TypeScript```版本，而非书中的```JavaScript```。通过```TypeScript```类型机制，以便实现对代码更好的理解。

# 背景
Vue3设计```响应系统```的主要作用是实现```数据```和```视图```的```自动同步```，能简化开发过程和提升应用的性能。具体来说有两大好处：
* ```自动化视图更新```：当```响应式数据```发生变化时，相关的视图会自动更新
* ```组件颗粒度级别```的视图更新：精确追踪每个组件对```响应式数据```的依赖关系，从而只更新受影响的组件部分，避免不必要的全局重新渲染

# 响应式数据的基础实现
### 响应式数据的目标
正如在使用```Vue3```框架时，修改```响应式数据```后，视图会自动更新。基础版```响应式数据```的实现目标是：修改了某个对象的属性后，使用了该属性值的```副作用函数```会自动执行。

以下面代码为例，当修改```obj.text```的值后，我们希望effect函数会自动执行。

```TypeScript
const obj = {
    text: 'hello world'
}

const effect = () => {
    const text = obj.text
    console.log('effect1', text)
}

// 修改obj.text后，希望effect能执行
obj.text = 'foo'
```
### 实现思路
要实现上述逻辑，需要做到两件事情：
1. 知道```对象的属性值```对应哪些```副作用函数```
2. 在```对象的属性值```更新后，触发其对应的```副作用函数```的执行

通过```ES6```的```Prxoy```创建对象的`代理`能实现这一点：
1. 当对象的`属性值`被`副作用函数`访问，就在`Prxoy`的`get`中收集`副作用函数`
2. 当对象的`属性值`发生修改，就在`Prxoy`的`set`触发该属性收集的所有`副作用函数`

可以用这段简短的代码来辅助理解
```JS
// 存储副作用函数的桶
const bucket = new Set()
// 原始数据
const data = { text: 'hello world' }

// 对原始数据的代理
const obj = new Proxy(data, {
    // 拦截读取操作
    get(target, key) {
        // 将副作用函数 effect 添加到存储副作用函数的桶中
        bucket.add(effect)
        // 返回属性值
        return target[key]
    },
    // 拦截设置操作
    set(target, key, newVal) {
        // 设置属性值
        target[key] = newVal
        // 把副作用函数从桶里取出并执行
        bucket.forEach(fn => fn())
    }
})

// 副作用函数
function effect () {
    const text = obj.text
    console.log(text)
}
effect()

// 修改 obj.text 的值，会触发 effect 的执行
obj.text = 'foo'
```

这段代码是```响应式数据```的最基础实现逻辑，目前还存在两个问题：
1. 代码`bucket.add(effect)`是通过`函数名`去获取```副作用函数```，这种`硬编码`方式很不灵活。
2. 存储``副作用函数``的`bucket`，直接存入了所有副作用函数。但我们要做到`对象的属性`与其对应的`副作用函数`精细映射，否则这会导致一个`属性值`变化后，与该`属性`的无关的`副作用函数`也会执行。

### 具体实现
通过如下措施解决上述两个问题：
1. 通过一个全局变量记录当前的`副作用函数`，解决`硬编码`方式获取```副作用函数```的问题。
2. 建立更精细的`bucket`数据结构，实现`对象的属性`到其对应的`副作用函数`的映射。即建立`对象`映射`属性`，`属性`映射`副作用函数`的数据结构。细节如下：
    * 用`WeakMap`作为`bucket`，`WeakMap`的`key`是对象，`value`是一个`Map`。`Map`的`key`是`对象的属性`，`value`是一个存储`副作用函数`的`Set`。
    * 用`TypeScript`伪代码描述`bucket`的数据类型：
```ts
WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
```

接下来是实现响应式数据的运行逻辑：
1. 通过一个叫做`effect`的函数来注册副作用函数，`副作用函数`是`effect`函数的参数。在`effect`内部，有一个全局变量`activeEffect`会记录传入`effect`的`副作用函数`，并且`副作用函数`会被调用，从而触发`Prxoy`的`get`。
2. `Prxoy`的`get`方法的前两个参数分别是`目标对象`和`被获取的属性名`，再加上记录当前`副作用函数`的变量`activeEffect`，就能在`get`方法中更新`bucket`的数据，从而建立`对象->属性名->副作用函数`的映射关系。
3. 当修改`Prxoy`实例的属性，会触发`Prxoy`的`set`方法，该方法的前两个参数分别是`目标对象`和`被获取的属性名`，通过`bucket`记录的映射关系，配合前两个参数，就能获取并执行该`属性`对应的`副作用函数`。

### 完整代码
下面是具体实现，也是理解Vue3响应系统的`关键`。虽然看起来有100多行，但是`核心代码`不到`20行`：

```ts
/**
 * @desc 基础版响应式数据的实现
 */

/**
 * @Type 属性值为任意类型的对象
 */
interface AnyObj {
    [prop: string | symbol]: any
}

/**
 * @Type 副作用函数
 */
type EffectFn = () => any

/**
 * 用一个WeakMap记录响应式对象的属性变化后, 需要执行的副作用函数
 * 具体映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
 */
const bucket: WeakMap<AnyObj, Map<string | symbol, Set<EffectFn>>> = new WeakMap()

// 通过全局变量, 记录当前的副作用函数
let activeEffect: EffectFn | undefined

/**
 * 注册副作用函数
 */
const effect = (fn: EffectFn) => {
    activeEffect = fn
    fn()
}

// 响应式对象的原始值
const data: AnyObj = {
    ok: true,
    text: 'hello world'
}

// 将原始数据转换为响应式对象
const obj = new Proxy(data, {
    // 在get中收集对象属性对应的副作用函数
    get(target, key) {
        track(target, key)
        return target[key]
    },

    // 在set中触发对象属性对应的副作用函数
    set(target, key, newVal) {
        target[key] = newVal
        trigger(target, key)
        return true
    }
})

// Proxy的 get 中会调用 track 函数, 从而建立 对象属性 到 副作用函数 的映射
const track = (target: AnyObj, key: string | symbol) => {
    /**
     * 从映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 读取 Map<原始对象的属性, Set<该属性对应的副作用函数>>
     */
    if (!activeEffect) return
    let depsMap = bucket.get(target)
    // 要考虑初始化
    if(!depsMap) {
        depsMap = new Map
        bucket.set(target, depsMap)
    }

    /**
     * 从映射关系: Map<原始对象的属性, Set<该属性对应的副作用函数>>
     * 读取 Set<该属性对应的副作用函数>
     */
    let deps = depsMap.get(key)
    // 要考虑初始化
    if (!deps) {
        deps = new Set()
        depsMap.set(key, deps)
    }
    // 将副作用函数存入 Set<该属性对应的副作用函数>
    deps.add(activeEffect)
}

// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)
    effects && effects.forEach(fn => fn())
}

// 注册一个的副作用函数
effect(() => {
    const text = obj.text
    console.log('effect1', text)
})

// 注册一个的副作用函数
effect(() => {
    const text = `${obj.text} ---`
    console.log('effect2', text)
})

// 修改响应式数据, 会触发obj.text对应的副作用函数
obj.text = 'foo'
```


# 分支切换与cleanup
### 分支切换带来的问题
我们已经实现了一个较基础的`响应式数据`，但存在一个缺陷：副作用函数内部可能存在`条件判断`逻辑，有些在原本会执行的`响应式数据`的`属性`访问，由于`条件判断`状态的变化，导致不再会被访问。但由于`bucket`已经建立了依赖关系，当修改该`响应式数据`的`属性`后，副作用函数还是会被执行，这个执行是`多余`的。

可以用这个例子辅助理解，当`obj.ok`设置为`false`后，`obj.text`不会再被读取，按照预期我们不希望修改`obj.text`的值后，触发该副作用函数。但是由于先前已经建立了依赖关系，会导致`副作用函数`被执行。
```ts
// 注册一个副作用函数
effect(() => {
    // obj.ok的初始值是true，obj.text会被读取，会在bucket中建立obj.text对该副作用函数的依赖
    const text = obj.ok ? obj.text : 'not'
    console.log('effect1', text)
})

obj.ok = false
// obj.flag已经是false, 上述副作用函数的 三元运算 不会进入ture分支从而读取 obj.text
// 但是修改 obj.text, 副作用还是会执行, 这是不必要的
obj.text = 'foo'
```
### 解决思路
这就是`分支切换`带来的问题，解决方案如下：
1. 除了之前`bucket`建立的`属性`到`副作用函数`的映射，还需要建立`副作用函数`到`收集依赖数据的集合`的映射。换个说法：`bucket`的数据结构是`WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>`，要建立`副作用函数`到`Set<该属性对应的副作用函数>`的映射。
2. 在`副作用函数`执行前，先清除该`副作用函数`在所有`收集依赖数据的集合`中的存在，然后执行`副作用函数`，这样又能重新收集对`副作用函数`的`依赖`。

### 具体实现

对之前[响应式数据的基础实现](#heading-8)的代码稍加修改就能实现：
#### 修改`effect`函数
1. 修改`effect`函数：
    * 在`effect`内部用一个叫做`effectFn`的函数对`副作用函数`进行封装。并给`effectFn`添加一个叫做`deps`属性，其值为数组，数组的元素是收集该`副作用函数`的`依赖集合`。
    * 先遍历`effectFn.deps`中的`依赖集合`，这些`依赖集合`移除对`effectFn`的收集。这个逻辑封装在需要新增的函数`cleanup`中。
    * 执行`effectFn`，触发副作用并重新收集依赖

这么说有点绕，看代码比较好理解：

```ts
/**
 * 遍历所有收集了 effectFn 的依赖集合, 在每个依赖集合中都将 effectFn 移除
 */
const cleanup = (effectFn: EffectFn) => {
    for(const deps of effectFn.deps) {
        deps.delete(effectFn)
    }
    // 重置 effectFn.deps 数组长度
    effectFn.deps.length = 0
}

/**
 * 注册副作用函数
 */
const effect = (fn: () => any) => {
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        fn()
    }) as unknown as EffectFn

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 执行副作用函数
    effectFn()
}
```

#### 修改track函数
2.修改track函数，实现`effectFn.deps`对`依赖集合`的记录。只需在`track`函数最后加一行代码：
``` ts
const track = (target: AnyObj, key: string | symbol) => {
    /**
     * 从映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 读取 Map<原始对象的属性, Set<该属性对应的副作用函数>>
     */
    if (!activeEffect) return
    let depsMap = bucket.get(target)
    // 要考虑初始化
    if(!depsMap) {
        depsMap = new Map
        bucket.set(target, depsMap)
    }

    /**
     * 从映射关系: Map<原始对象的属性, Set<该属性对应的副作用函数>>
     * 读取 Set<该属性对应的副作用函数>
     */
    let deps = depsMap.get(key)
    // 要考虑初始化
    if (!deps) {
        deps = new Set()
        depsMap.set(key, deps)
    }
    // 将副作用函数存入 Set<该属性对应的副作用函数>
    deps.add(activeEffect)

    /**
     * @新增代码 将deps添加到 activeEffect.deps中
     */
    activeEffect.deps.push(deps)
}
```
#### 新的问题：死循环
这样看来好像已经解决`分支切换`问题了，但是实际运行代码我们会发现程序进入`死循环`。问题出在`
trigger`函数
```ts
// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)
    effects && effects.forEach(fn => fn())
}
```
`trigger`中的最后一行代码`effects && effects.forEach(fn => fn())`是问题所在：
* `forEach`里面的`fn`就是`effect`函数中的`effectFn`，`effectFn`会调用`cleanup`清除自身在`effects`中的存在，但是`effectFn`内部接下来会执行`副作用函数`，导致`effects`又添加`effectFn`。
* 总的来说就是：`effects`中刚被清除`effectFn`，马上又被添加`effectFn`，形成了`死循环`。

#### 解决死循环问题
我们通过构造一个新的Set来解决这个问题：
```ts
// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set(effects) // 新增
    effectsToRun.forEach(effectFn => effectFn()) // 新增
    // effects && effects.forEach(fn => fn()) 删除
}
```

#### 完整代码
至此，`分支切换`问题解决。完整代码如下：
```ts
/**
 * @Type 属性值为任意类型的对象
 */
interface AnyObj {
    [prop: string | symbol]: any
}

/**
 * @Type 副作用函数
 */
interface EffectFn extends Function {
    // EffectFn.deps 用来存储所有与该副作用函数相关联的依赖集合
    deps: Set<EffectFn>[];
}


/**
 * 用一个WeakMap记录响应式对象的属性变化后, 需要执行的副作用函数
 * 具体映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
 */
const bucket: WeakMap<AnyObj, Map<string | symbol, Set<EffectFn>>> = new WeakMap()

// 通过全局变量, 记录当前的副作用函数
let activeEffect: EffectFn | undefined

/**
 * 注册副作用函数
 */
const effect = (fn: () => any) => {
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        fn()
    }) as unknown as EffectFn

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 执行副作用函数
    effectFn()
}

/**
 * 遍历所有收集了 effectFn 的依赖集合, 在每个依赖集合中都将 effectFn 移除
 */
const cleanup = (effectFn: EffectFn) => {
    for(const deps of effectFn.deps) {
        deps.delete(effectFn)
    }
    // 重置 effectFn.deps 数组长度
    effectFn.deps.length = 0
}

// 响应式对象的原始值
const data: AnyObj = {
    ok: true,
    text: 'hello world'
}

// 将原始数据转换为响应式对象
const obj = new Proxy(data, {
    // 在get中收集对象属性对应的副作用函数
    get(target, key) {
        track(target, key)
        return target[key]
    },

    // 在set中触发对象属性对应的副作用函数
    set(target, key, newVal) {
        target[key] = newVal
        trigger(target, key)
        return true
    }
})

// Proxy的 get 中会调用 track 函数, 从而建立 对象属性 到 副作用函数 的映射
const track = (target: AnyObj, key: string | symbol) => {
    /**
     * 从映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 读取 Map<原始对象的属性, Set<该属性对应的副作用函数>>
     */
    if (!activeEffect) return
    let depsMap = bucket.get(target)
    // 要考虑初始化
    if(!depsMap) {
        depsMap = new Map
        bucket.set(target, depsMap)
    }

    /**
     * 从映射关系: Map<原始对象的属性, Set<该属性对应的副作用函数>>
     * 读取 Set<该属性对应的副作用函数>
     */
    let deps = depsMap.get(key)
    // 要考虑初始化
    if (!deps) {
        deps = new Set()
        depsMap.set(key, deps)
    }
    // 将副作用函数存入 Set<该属性对应的副作用函数>
    deps.add(activeEffect)

    /**
     * @新增代码 将deps添加到 activeEffect.deps中
     */
    activeEffect.deps.push(deps)
}

// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set(effects) // 新增
    effectsToRun.forEach(effectFn => effectFn()) // 新增
    // effects && effects.forEach(fn => fn()) 删除
}

// 注册一个的副作用函数
effect(() => {
    const text = obj.ok ? obj.text : 'not'
    console.log('effect1', text)
})

obj.ok = false
// obj.flag已经是false, 上述副作用函数的 三元运算 不会进入ture分支从而读取 obj.text
// 修改 obj.text, 副作用不会执
obj.text = 'foo'
```

# 嵌套的effect与effect栈
#### 背景
`effect`可以发生`嵌套`。`Vue.js`中`组件`的渲染函数`render`是在一个`effect`中执行的，`组件`的`嵌套`意味着`effect`的`嵌套`。比如一个叫`Foo`的组件嵌套了一个叫`Bar`的组件：

```tsx
<Foo>
    <Bar/>
</Foo>
```
对应的`effect`嵌套就是:

```ts
effect(() => {
    Foo.render()
    effect(() => {
        Bar.render()
    })
})
```
#### 理解问题
用下面代码来说明`effect`嵌套带来的问题：

```ts
// 原始数据
const data: AnyObj = {
    foo: true,
    bar: true
}

// 代理对象
const obj = new Proxy(data, { /* ... */ })

let temp1
let temp2
effect(function effectFn1() {
    console.log('effectFn1 执行')
    effect(function EffectFn2() {
        console.log('effectFn2 执行')
        temp2 = obj.bar
    })
    temp1 = obj.foo
})

obj.foo = false
```

再回顾一下`effect`函数中的这行代码：`activeEffect = effectFn`
```ts
/**
 * 注册副作用函数
 */
function effect (fn: () => any) {
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        fn()
    }) as unknown as EffectFn

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 执行副作用函数
    effectFn()
}
```

现在能看出问题了，当上面实例代码执行到`temp1 = obj.foo`时，变量`activeEffect`的值还是`EffectFn2`。这会导致`proxy`的`get`中，`obj.foo`收集的副作用函数是`EffectFn2`，也就是说修改`obj.foo`的值，会导致`EffectFn2`的执行。但我们的预期是让`EffectFn1`执行，因为`temp2 = obj.bar`对应的副作用函数是`EffectFn1`。
#### 问题解决思路
通过一个叫`effectStack`的`栈`来维护`activeEffect`值的变化：
* 在`effectFn`的内部，执行`fn()`之前，把`activeEffect`赋值`effectFn`的同时，还要把`effectFn`入栈`effectStack`
* 在`effectFn`的内部，在执行`fn()`之后，对`effectStack`弹出栈顶元素，这样`effectStack`的栈顶元素就变回调用当前`effectFn`的外层`effectFn`了。把`activeEffect`赋值为新的栈顶元素，这样`activeEffect`的值就变成了外层`effectFn`。

通过维护`effectStack`，就解决了`effect`嵌套导致`activeEffect`指向不正确的问题。

#### 具体实现
代码中通过`// 新增`注释标记了改动之处：

```ts
// 使用stack维护`activeEffect`值的变化
const effectStack: EffectFn[] = [] // 新增

/**
 * 注册副作用函数
 */
const effect = (fn: () => any) => {
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        // 执行 fn() 之前，把 effectFn 入栈 effectStack
        effectStack.push(effectFn) // 新增
        fn()
        // 执行 fn() 之后，将当前 effectFn 出栈，这样栈顶元素就变成了调用当前effectFn的上层effectFn
        effectStack.pop() // 新增
        activeEffect = effectStack[effectStack.length - 1] // 新增
    }) as unknown as EffectFn

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 执行副作用函数
    effectFn()
}
```

# 避免无限递归循环
#### 理解问题
如果`副作用函数`中存在对同一个`响应式对象`的`属性`的`读取`与`赋值`，就会发生`无限递归循环`。
比如这个例子：

```ts
const data = {
    foo: 1
}

const obj = new Proxy(data, { /* ... */ })

effect(function effectFn1() {
    obj.foo = obj.foo + 1
})
```
对`obj.foo`的`读取`会触发`track`，接着对`obj.foo`的`赋值`会触发`trigger`，这就导致`effectFn1`无限递归调用自己，产生函数的`调用栈溢出`。
#### 解决问题
这个问题很好解决：如果`trigger`中执行的`副作用函数`与`activeEffect`相同，就不执行。代码如下：

```ts
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set(effects)
    effectsToRun.forEach(effectFn => {
        // 如果 trigger 中执行的 副作用函数 与 activeEffect相同，就不执行
        if(effectFn !== activeEffect) { // 新增
            effectFn()
        }
    })
}
```

# 调度执行
`可调度性`是响应式系统的关键特性，`可调度`指的是在`tigger`触发`副作用函数`重新执行时，有能力决定`副作用函数`的执行时机、次数以及方式。
#### 举例说明
```ts
const data = { foo: 1 }
const obj = new Proxy(data, { /* ... */ })

effect(() => {
    console.log(obj.foo)
})
obj.foo++
console.log('end')
```

这段代码的输出顺序是
```ts
1
2
'end'
```
如果我们希望改变打印顺序为`1 'end' 2`，并且不调整代码。这就需要响应系统支持`调度`，我们希望这样使用调度：
```ts
const data = {foo: 1}
const obj = new Proxy(data, { /* ... */})

effect(
    () => {
        console.log(obj.foo)
    },
    // options 用于配置effect的选项
    {
        // 指定调度器的实现
        scheduler(fn) {
            setTimeout(fn)
        }
    }
)

obj.foo++
console.log('end')
```
给`effect`增加第二个参数，用于设置`effect`的选项，其中`scheduler`配置项就是调度器。在这个例子中，`调度器`通过`setTimeout`实现了`obj.foo++`的异步打印，从而让打印顺序变为`1 'end' 2`。

#### 设计调度器
明确用法后，实现方案也很明显了：
1. 给`effect`函数增加一个`options`参数，类型是对象，表示配置项
2. 在`effect`中，把`options`挂载到`副作用函数`上，即：`effctFn.options = options`
3. 在`trigger`中，执行`副作用函数`时，判断一下这个`副作用函数`有没有`调度器`，即`effctFn?.options?.scheduler`

#### 具体实现
修改`effect`和`trigger`
```ts
/**
 * 注册副作用函数
 */
const effect = (fn: () => any, options: EffectOptions = {}) => { // 新增代码：添加 options 参数
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        // 执行 fn() 之前，把 effectFn 入栈 effectStack
        effectStack.push(effectFn)
        fn()
        // 执行 fn() 之后，将当前 effectFn 出栈，这样栈顶元素就变成了调用当前effectFn的上层effectFn
        effectStack.pop()
        activeEffect = effectStack[effectStack.length - 1]
    }) as unknown as EffectFn

    //
    effectFn.options = options

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 执行副作用函数
    effectFn()
}

// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set(effects)
    effectsToRun.forEach(effectFn => {
        // 新增：如果 副作用函数存在调度器，就通过调度器执行副作用函数
        if(effectFn.options.scheduler) {
            effectFn.options.scheduler(effectFn)
        } else {
            effectFn()
        }
    })
}
```
完整代码
```ts
/**
 * @desc 基础版响应式数据的实现
 */

/**
 * @Type 属性值为任意类型的对象
 */
interface AnyObj {
    [prop: string | symbol]: any
}

// effect函数的配置项参数
interface EffectOptions {
    // 调度器
    scheduler?: (effectFn: EffectFn) => any
}

/**
 * @Type 副作用函数
 */
interface EffectFn extends Function {
    // EffectFn.deps 用来存储所有与该副作用函数相关联的依赖集合
    deps: Set<EffectFn>[];

    options: EffectOptions
}

/**
 * 用一个WeakMap记录响应式对象的属性变化后, 需要执行的副作用函数
 * 具体映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
 */
const bucket: WeakMap<AnyObj, Map<string | symbol, Set<EffectFn>>> = new WeakMap()

// 通过全局变量, 记录当前的副作用函数
let activeEffect: EffectFn | undefined

// 使用stack维护`activeEffect`值的变化
const effectStack: EffectFn[] = []

/**
 * 注册副作用函数
 */
const effect = (fn: () => any, options: EffectOptions = {}) => { // 新增代码：添加 options 参数
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        // 执行 fn() 之前，把 effectFn 入栈 effectStack
        effectStack.push(effectFn)
        fn()
        // 执行 fn() 之后，将当前 effectFn 出栈，这样栈顶元素就变成了调用当前effectFn的上层effectFn
        effectStack.pop()
        activeEffect = effectStack[effectStack.length - 1]
    }) as unknown as EffectFn

    //
    effectFn.options = options

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 执行副作用函数
    effectFn()
}

/**
 * 遍历所有收集了 effectFn 的依赖集合, 在每个依赖集合中都将 effectFn 移除
 */
const cleanup = (effectFn: EffectFn) => {
    for(const deps of effectFn.deps) {
        deps.delete(effectFn)
    }
    // 重置 effectFn.deps 数组长度
    effectFn.deps.length = 0
}

// 原始值
const data: AnyObj = {foo: 1}

// 将原始数据转换为响应式对象
const obj = new Proxy(data, {
    // 在get中收集对象属性对应的副作用函数
    get(target, key) {
        track(target, key)
        return target[key]
    },

    // 在set中触发对象属性对应的副作用函数
    set(target, key, newVal) {
        target[key] = newVal
        trigger(target, key)
        return true
    }
})

// Proxy的 get 中会调用 track 函数, 从而建立 对象属性 到 副作用函数 的映射
const track = (target: AnyObj, key: string | symbol) => {
    /**
     * 从映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 读取 Map<原始对象的属性, Set<该属性对应的副作用函数>>
     */
    if (!activeEffect) return
    let depsMap = bucket.get(target)
    // 要考虑初始化
    if(!depsMap) {
        depsMap = new Map
        bucket.set(target, depsMap)
    }

    /**
     * 从映射关系: Map<原始对象的属性, Set<该属性对应的副作用函数>>
     * 读取 Set<该属性对应的副作用函数>
     */
    let deps = depsMap.get(key)
    // 要考虑初始化
    if (!deps) {
        deps = new Set()
        depsMap.set(key, deps)
    }
    // 将副作用函数存入 Set<该属性对应的副作用函数>
    deps.add(activeEffect)

    // 将deps添加到 activeEffect.deps中
    activeEffect.deps.push(deps)
}

// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set(effects)
    effectsToRun.forEach(effectFn => {
        // 新增：如果 副作用函数存在调度器，就通过调度器执行副作用函数
        if(effectFn.options.scheduler) {
            effectFn.options.scheduler(effectFn)
        } else {
            effectFn()
        }
    })
}

effect(
    () => {
        console.log(obj.foo)
    },
    // options 用于配置effect的选项
    {
        // 指定调度器的实现
        scheduler(fn) {
            setTimeout(fn)
        }
    }
)

obj.foo++
console.log('end')
```


#### 调度的意义
Vue3 `调度`执行机制的主要目的是**优化组件更新的性能和响应速度**。具体包括以下几个关键目的：

1. ***避免重复更新***：合并多次状态变化，确保组件只进行必要的更新，减少不必要的重复渲染。
2. ***批量更新***：在同一个事件循环内的多次状态变化，不会立即触发更新，而是等所有变化完成后，再进行统一的批量更新，减少 DOM 操作频率。
3. ***优先级调度***：为不同的更新任务设置优先级，确保高优先级任务优先执行，提升用户体验的流畅性。

#### 使用调度避免重复更新
如果一个状态多次更新，我们希望只执行最后一次的副作用函数。以下面代码为例：
```ts
const data = {foo: 1}
const obj = new Proxy(data, { /* ... */ })

effect(() => {
   console.log(obj.foo)
})

obj.foo++
obj.foo++
```
obj.foo会执行两次自增，打印结果是:
```ts
1
2
3
```
我们不关心过渡状态，打印`2`是多余的`副作用函数`执行，我们期望的打印是：
```ts
1
3
```
为了实现这一目标，基于`调度器`，利用`微任务`就能解决，代码如下：
```ts
// 定义一个任务队列
const jobQueue:Set<EffectFn> = new Set()
// 利用 Promise.resolve().then 可以将任务添加到微任务队列
const p = Promise.resolve()

// 一个标志代表是否正在刷新队列
let isFlushing = false
// 刷新任务队列
function flushJob() {
    // 如果队列正在刷新，则什么也不做
    if(isFlushing) return
    // 设置未true，表示正在刷新
    isFlushing = true
    // 在微任务中刷新 jobQueue 队列
    p.then(() => {
        jobQueue.forEach(job => job())
    }).finally(() => {
        // 结束后重置 isFlushing
        isFlushing = false
    })
}

effect(
    () => {
        console.log(obj.foo)
    },
    {
        scheduler(fn: EffectFn) {
            // 每次调度时，将副作用函数添加到jobQueue
            jobQueue.add(fn)
            // 刷新任务队列
            flushJob()
        }
    }
)

obj.foo++
obj.foo++
```

# computed 与 lazy
#### 给`effect`函数加上`lazy`功能
这里要实现的目标是，给`effect`的`options`参数添加一个`lazy`配置项，如果`lazy`为`true`,`effect`函数会不直接调用`副作用函数`，而是返回`副作用函数`。代码如下：
```ts
const effect = <T>(fn: () => T, options: EffectOptions = {}) => {
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        // 执行 fn() 之前，把 effectFn 入栈 effectStack
        effectStack.push(effectFn)
        const res = fn() // @新增
        // fn() 删除
        // 执行 fn() 之后，将当前 effectFn 出栈，这样栈顶元素就变成了调用当前effectFn的上层effectFn
        effectStack.pop()
        activeEffect = effectStack[effectStack.length - 1]
        return res // @新增
    }) as unknown as EffectFn

    effectFn.options = options

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 如果 lazy 为 false, 返回 执行 effectFn()
    // 如果 lazy 为 true, 返回 effectFn
    if(!options.lazy) { // @新增
        // 执行副作用函数
        effectFn()
    } else {
        // 返回 effectFn
        return effectFn
    }
}
```
值得注意的是，这里不仅增加了`lazy`对应的`if`条件语句，`effectFn`也做了修改：`effectFn`是对`原初副作用函数fn`的包装，`effectFn`需要返回`原初副作用函数`的返回值。

#### 把`lazy`视作`getter`
通过`lazy`，可以把`effect`函数的返回视作一个`getter`，这个`getter`可以返回一些有趣的内容：

```ts
// effect 返回的 effectFn 可以视为一个 getter
const effectFn = effect(
    () => obj.foo + obj.bar,
    {
        lazy: true
    }
)

// 获取 getter 的返回值
const value = effectFn()
```

#### 利用`lazy`实现`computed`
利用`lazy`和对象的`get属性访问器`，就实现了一个基本的computed

```ts
function computed<T>(getter: (...args:  any[] ) => T): {value: T} {
    const effectFn = effect(getter, { lazy: true })
    const obj = {
        get value() {
            return effectFn() as T
        }
    }
    return obj
}

const data = { foo: 1, bar: 2}
const obj = new Proxy(data, { /* ... */ })
const sumRes = computed(() => obj.foo + obj.bar)

console.log(sumRes.value) // 3
```
接着通过一个`dirty`标识来实现`computed`的对结果缓存功能
```ts
function computed<T>(getter: (...args:  any[] ) => T): {value: T} {
    // 缓存上一次计算的结果
    let value: T // @新增
    // 使用dirty来标记是否需要重新计算值, true意味"脏"，需要重新计算
    let dirty = true // @新增

    const effectFn = effect(getter, { lazy: true })
    const obj = {
        // @重写get
        get value() {
            // "脏"时需要重新计算结果并缓存
            if(dirty) {
                // 缓存结果到dirty
                value = effectFn() as T
                // dirty 设置为 false，下一次访问可以直接使用 value 的缓存值
                dirty = false
            }
            return value
        }
    }
    return obj
}

const data = { foo: 1, bar: 2}
const obj = new Proxy(data, { /* ... */ })
const sumRes = computed(() => {
    console.log('运行计算')
    return obj.foo + obj.bar
})

console.log(sumRes.value) // 3
// 多亏computed的缓存功能，多次读取 sumRes.value，也不会发生重复计算
console.log(sumRes.value) // 3
```

然而上面的代码有一个问题，考虑这种情况：当`obj.foo`自增后，`sumRes.value`应该要变成**4**，但实际打印输出来是**3**。

```ts
const data = { foo: 1, bar: 2}
const obj = new Proxy(data, { /* ... */ })
const sumRes = computed(() => {
    console.log('运行计算')
    return obj.foo + obj.bar
})

console.log(sumRes.value) // 3
obj.foo++
console.log(sumRes.value) // 打印出来依旧为3，预期是4
```
问题出在`obj.foo`自增后，`dirty`的值没有正确重置为`true`，通过在`scheduler`中重置`dirty`解决这个问题：

```ts
function computed<T>(getter: (...args:  any[] ) => T): {value: T} {
    // 缓存上一次计算的结果
    let value: T

    // 使用dirty来标记是否需要重新计算值, true意味"脏"，需要重新计算
    let dirty = true

    const effectFn = effect(getter, {
        lazy: true,
        // 添加调度器 ，在调度器中重置 dirty 为 true
        scheduler() {
            dirty = true
        }
    })
    const obj = {
        get value() {
            // "脏"时需要重新计算结果并缓存
            if(dirty) {
                // 缓存结果到dirty
                value = effectFn() as T
                // dirty 设置为 false，下一次访问可以直接使用 value 的缓存值
                dirty = false
            }
            return value
        }
    }
    return obj
}
```
对`computed`的设计，到这里已经快完善了，但还有一个缺陷：如果有另一个`effect`调用了`计算属性`的值，当`计算属性`的值发生变化，`effect`中的`副作用函数`不会重新执行。以这段代码为例：
```ts
const sumRes = computed(() => {
    return obj.foo + obj.bar
})

effect(() => {
    // 在副作函数中读取计算属性的值
    console.log(sumRes.value)
})

// 修改obj.foo不会导致副作用函数中的console重新执行
obj.foo++
```

这个问题的本质两个`effect`的嵌套。但是内层的`effect`设置了`lazy`。有两个原因造成了这个问题：
* 如果是两个没设置`lazy`为`true`的`effect`发生`嵌套`，内层`effect`的`副作用函数`执行，不会强行伴随外层的`effect`的`副作用函数`执行。可以类比这种情况：子组件视图更新，父组件视图不一定要更新。但是计算的情况不太一样，计算属性的值（在上面的例子中对应`sumRes.value`）会被外层`effect`的副作用函数使用，而`sumRes`本身只是一个普通对象，不会收集外层`effect`的`副作用函数`。
* 当`计算属性的值`发生变化时，由于`lazy`的原因，如果不主动读取`计算属性的值`，是不会触发副作用函数的。

所以我们通过手动调用track和trigger来解决这两个问题：
* 在`get属性访问器`中调用track收集外层`effect`的`副作用函数`的依赖
* 在`scheduler`中调用`trigger`，触发外层`effect`的`副作用函数`的执行
```
function computed<T>(getter: (...args:  any[] ) => T): {value: T} {
    // 缓存上一次计算的结果
    let value: T //

    // 使用dirty来标记是否需要重新计算值, true意味"脏"，需要重新计算
    let dirty = true //

    const effectFn = effect(getter, {
        lazy: true,
        // 添加调度器 ，在调度器中重置 dirty 为 true
        scheduler() {
            if(!dirty) {
                dirty = true
                // 计算属性发生变化时，手动调用trigger触发响应
                trigger(obj, 'value') // @新增
            }
        }
    })
    const obj = {
        get value() {
            // "脏"时需要重新计算结果并缓存
            if(dirty) {
                // 缓存结果到dirty
                value = effectFn() as T
                // dirty 设置为 false，下一次访问可以直接使用 value 的缓存值
                dirty = false
            }
            // 当读取 value 时，手动调用 track 函数进行跟踪
            track(obj, 'value') // @新增
            return value
        }
    }
    return obj
}
```
#### 完整代码
这是解决了上述问题之后，实现`computed`的完整代码
```ts
/**
 * @Type 属性值为任意类型的对象
 */
interface AnyObj {
    [prop: string | symbol]: any
}

// effect函数的配置项参数
interface EffectOptions {
    // 调度器
    scheduler?: (effectFn: EffectFn) => any
    lazy?: boolean
}

/**
 * @Type 副作用函数
 */
interface EffectFn extends Function {
    // EffectFn.deps 用来存储所有与该副作用函数相关联的依赖集合
    deps: Set<EffectFn>[];

    options: EffectOptions
}

/**
 * effect的返回类型
 * 如果 EffectOptions中的 lazy 是 true，返回类型就是EffectFn，否则是 undefined
 */
type EffectReturn<K extends EffectOptions> = K["lazy"] extends true ? EffectFn : undefined;

/**
 * 用一个WeakMap记录响应式对象的属性变化后, 需要执行的副作用函数
 * 具体映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
 */
const bucket: WeakMap<AnyObj, Map<string | symbol, Set<EffectFn>>> = new WeakMap()

// 通过全局变量, 记录当前的副作用函数
let activeEffect: EffectFn | undefined

// 使用stack维护`activeEffect`值的变化
const effectStack: EffectFn[] = []

/**
 * 注册副作用函数
 */
const effect = <T, K extends EffectOptions>(
    fn: () => T, options: Partial<K> = {}
): EffectReturn<K> => {
    const effectFn = (() => {
        // 清除 effectFn 在所有依赖集合中的存在
        cleanup(effectFn)
        activeEffect = effectFn
        // 执行 fn() 之前，把 effectFn 入栈 effectStack
        effectStack.push(effectFn)
        const res = fn() //
        // fn() @删除

        // 执行 fn() 之后，将当前 effectFn 出栈，这样栈顶元素就变成了调用当前effectFn的上层effectFn
        effectStack.pop()
        activeEffect = effectStack[effectStack.length - 1]
        return res
    }) as unknown as EffectFn

    effectFn.options = options

    // effectFn.deps用来存储所有与effectFn相关联的依赖集合
    effectFn.deps = [] as Set<EffectFn>[]

    // 如果 lazy 为 false, 返回 执行 effectFn()
    // 如果 lazy 为 true, 返回 effectFn
    if(!options.lazy) {
        // 执行副作用函数
        effectFn()
        return undefined as EffectReturn<K>;
    } else {
        // 返回 effectFn
        return effectFn as EffectReturn<K>
    }
}

/**
 * 遍历所有收集了 effectFn 的依赖集合, 在每个依赖集合中都将 effectFn 移除
 */
const cleanup = (effectFn: EffectFn) => {
    for(const deps of effectFn.deps) {
        deps.delete(effectFn)
    }
    // 重置 effectFn.deps 数组长度
    effectFn.deps.length = 0
}

// 原始值
const data: AnyObj = { foo: 1, bar: 2}

// 将原始数据转换为响应式对象
const obj = new Proxy(data, {
    // 在get中收集对象属性对应的副作用函数
    get(target, key) {
        track(target, key)
        return target[key]
    },

    // 在set中触发对象属性对应的副作用函数
    set(target, key, newVal) {
        target[key] = newVal
        trigger(target, key)
        return true
    }
})

// Proxy的 get 中会调用 track 函数, 从而建立 对象属性 到 副作用函数 的映射
const track = (target: AnyObj, key: string | symbol) => {
    /**
     * 从映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 读取 Map<原始对象的属性, Set<该属性对应的副作用函数>>
     */
    if (!activeEffect) return
    let depsMap = bucket.get(target)
    // 要考虑初始化
    if(!depsMap) {
        depsMap = new Map
        bucket.set(target, depsMap)
    }

    /**
     * 从映射关系: Map<原始对象的属性, Set<该属性对应的副作用函数>>
     * 读取 Set<该属性对应的副作用函数>
     */
    let deps = depsMap.get(key)
    // 要考虑初始化
    if (!deps) {
        deps = new Set()
        depsMap.set(key, deps)
    }
    // 将副作用函数存入 Set<该属性对应的副作用函数>
    deps.add(activeEffect)

    // 将deps添加到 activeEffect.deps中
    activeEffect.deps.push(deps)
}

// Proxy的 set 中会调用 track 函数, 触发副作用函数
const trigger = (target: AnyObj, key: string | symbol) => {
    /**
     * 根据映射关系: WeakMap<原始对象, Map<原始对象的属性, Set<该属性对应的副作用函数>>>
     * 执行 Set<该属性对应的副作用函数> 中收集的副作用函数
     */
    const depsMap = bucket.get(target)
    if(!depsMap) return
    const effects = depsMap.get(key)

    const effectsToRun = new Set(effects)
    effectsToRun.forEach(effectFn => {
        // 如果 副作用函数 存在 调度器，就通过调度器执行副作用函数
        if(effectFn.options.scheduler) {
            effectFn.options.scheduler(effectFn)
        } else {
            effectFn()
        }
    })
}

// 定义一个任务队列
const jobQueue:Set<EffectFn> = new Set()
// 利用 Promise.resolve().then 可以将任务添加到微任务队列
const p = Promise.resolve()

// 一个标志代表是否正在刷新队列
let isFlushing = false
// 刷新任务队列
function flushJob() {
    // 如果队列正在刷新，则什么也不做
    if(isFlushing) return
    // 设置未true，表示正在刷新
    isFlushing = true
    // 在微任务中刷新 jobQueue 队列
    p.then(() => {
        jobQueue.forEach(job => job())
    }).finally(() => {
        // 结束后重置 isFlushing
        isFlushing = false
    })
}

function computed<T>(getter: (...args:  any[] ) => T): {value: T} {
    // 缓存上一次计算的结果
    let value: T

    // 使用dirty来标记是否需要重新计算值, true意味"脏"，需要重新计算
    let dirty = true //

    const effectFn = effect(getter, {
        lazy: true,
        // 添加调度器 ，在调度器中重置 dirty 为 true
        scheduler() {
            if(!dirty) {
                dirty = true
                // 计算属性发生变化时，手动调用trigger触发响应
                trigger(obj, 'value')
            }
        }
    })
    const obj = {
        get value() {
            // "脏"时需要重新计算结果并缓存
            if(dirty) {
                // 缓存结果到dirty
                value = effectFn() as T
                // dirty 设置为 false，下一次访问可以直接使用 value 的缓存值
                dirty = false
            }
            // 当读取 value 时，手动调用 track 函数进行跟踪
            track(obj, 'value')
            return value
        }
    }
    return obj
}

const sumRes = computed(() => {
    return obj.foo + obj.bar
})

effect(() => {
    // 在副作函数中读取计算属性的值
    console.log(sumRes.value)
})

// 会打印出预期结果 4
obj.foo++
```

# watch的实现原理
#### 初步实现
`watch`的本质就是观测一个数据，数据发生变化时，通知并执行相应回调函数

这是有两个使用`watch`的例子，他们的区别在于第一个参数的不同：
* watch第一个参数是`() => obj.foo`的情况，watch会监听`obj.foo`的变化
* watch第一个参数是obj的情况，watch会深度监听obj的所有属性值

```ts
watch(
    () => obj.foo,
    (oldValue, newValue) => {
        console.log(oldValue, newValue)
    }
)

watch(
    obj,
    (oldValue, newValue) => {
        console.log(oldValue, newValue)
    }
)

```
`watch`的实现，直接看代码就能理解了：

```ts
// 封装一个能遍历对象属性的函数
function traverse(value: any, seen = new Set()) {
    // 如果要读取的数据是原始值或者已经被读取过，那么什么都不用做
    if(typeof value !== 'object' || value === null || seen.has(value)) return
    // 标记数据的读取状态，避免循环引用导致的死循环
    seen.add(value)
    // 先简单假设value是一个对象
    for (const k in value) {
        traverse(value[k], seen)
    }

    return value
}

function watch(source: any, cb: Function) {
    let getter: Function
    if(typeof source === 'function') {
        getter = source
    } else {
        getter = () => traverse(source)
    }

    // 定义旧值和新值
    let oldValue: any
    let newValue: any

    // 使用 effect 注册副作用函数时，开启 lazy 选项，并把返回值存储到effectFn 中，以便后续调用
    const effectFn = effect(
        () => getter(),
        {
            lazy: true,
            scheduler() {
                newValue = effectFn()
                cb(newValue, oldValue)
                oldValue = newValue
            }
        }
    )

    // 手动调用副作用函数，effectFn()返回的就是旧值
    oldValue = effectFn()
}
```

这里有两个关键点：
* `traverse`函数：递归遍历对象属性，确保它们被响应式系统跟踪，同时还要注意循环引用问题。
* `watch`函数做了两件事情：
    * 处理 `source` ：如果是函数，直接使用；如果是对象，使用`traverse`函数进行深度遍历。
    * 注册一个`lazy`副作用函数，并在`scheduler`中维护`oldValue`和`newValue`

#### 更完善的实现
上面的`watch`实现是最基本的形态，其实还有`立即执行watch`、`解决watch的竞态问题`需要实现，暂时先到这里。

# 后续
到这里响应系统的实现已经初步完成了，但实际的实现比这复杂的多，比如拦截和追踪```for in```循环、对数组的代理、深响应、浅响应，都是要解决的问题。




## 1. 什么是虚拟dom

1. 真实的dom就是document下面的节点，属性会有297个。而虚拟dom是一个JS对象，属性大大减少。标签名、属性名、内容。





## 2. 为什么要创建一个虚拟dom

1. jquery + 模板字符穿，每次修改数据，删除原来所有，增加新的，无法去跟踪旧的状态，进行新旧的对比。频繁刷新页面
3. 虚拟dom的方式，状态更新 不需要立即更新真实dom，而是创建虚拟dom，虚拟dom内部会去分析如何更新，通常是通过diff算法来进行更新。虚拟dom，会跟踪程序的状态，记录旧的状态，比较两次的差异，来更新真实的dom。而且对象的属性也大大减少了





## 3. 虚拟dom不适合的场景

1. 非常简单的场景，而不是“复杂”的场景时，使用虚拟dom，进行“新旧状态的对比”，反而会降低性能





## 4. snabbdom的使用

1. 下包 parce-bundler



虚拟dom，内部使用了 snabbdom这个库

核心：

1. h函数， 创建虚拟dom
2. init 返回的是一个高阶函数
3. thunk 是一种优化的策略



```js
import {h, thunk, init} from 'snabbdom'

// 1. 调用 init函数 
const patch = init([]) // init里面跟的是数组 里面的参数是模块 我们暂时不用模块 返回的是一个 高阶的函数 patch
// patch 是对比旧的和新的dom的差异,然后更新到真实的dom中


// 2. 创建虚拟节点
// h函数第二个参数 如果是字符串 就是内容
const Vnode = h('div#container.cls', 'hello world') 
// 获取容器
const app = document.querySelector('#app')


// 3. 通过patch函数 对比新旧虚拟dom 渲染虚拟dom 为另一个虚拟dom
// patch的第一个dom也可以是真实的dom patch会将真实的dom转化为虚拟dom
const oldVnode = patch(app, Vnode)

```





## 5. 模拟服务器数据 

>记住，创建vnode 用let 不然，vnode就不能够修改了

```js
import {h, init} from 'snabbdom'

let patch = init([])
// 之前用const 报错了
+let vnode = h('div#container', [
    h('h1', '我是h1'),
    h('p', '我是小p')
])

let app = document.querySelector('#app')
let oldVnode = patch(app, vnode)

// 模拟从服务器获取节点
setTimeout(() => {
    // 修改 vnode的值 容器还是那个 但是里面的节点内容改变了
    vnode = h('div#container', [
        h('h1', '我是服务器来的h1'),
        h('p', '我是服务器来的p')
    ])
    // 利用patch 进行新旧dom节点的重新对比
    patch(oldVnode, vnode)
}, 2000)

```



## 6. 模块的使用

- 因为patch不能处理 样式 事件 属性设置这些
- 所以snabbdom新增了 6个模块



```js
1. attributes属性，能够处理布尔值(selected checked)。 调用 setAttribute设置属性
2. prop：属性，不能设置布尔 element[attr] = value
3. class：类名
4. dataset：自定义属性
5. eventListeners: 事件
6. style: 样式
```



注意点就是，模块的写法：

1. 引入
2. init注册
3. 在h的第二个参数里面使用

```diff
// 模块的使用
import {h, init} from 'snabbdom'
import style from 'snabbdom/modules/style'
import eventListeners from 'snabbdom/modules/eventlisteners'

// 注册模块
let patch = init([style, eventListeners])

// 创建虚拟dom 使用模块
let vnode = h(
    'div',
    {
+        style:{
            backgroundColor: 'red'
        },
+        on: {
            click: eventHandler
        }，
        // 自定义属性的设置
+        dataset: {
            time: new Date(),
            value: '123'
        }
    },
    // 子元素 记得用数组
+子元素    [
        h('p', '我是小p'),
        h('h1', '我是小h1')
    ]
)

function eventHandler() {
    console.log('点击了我');
}

let app = document.querySelector('#app')
patch(app, vnode)
```



## 7. snabbdom源码

### 7.1 h函数

作用：

>1. 创建JS对象， 创建虚拟dom。h函数最关键的就是利用`vnode方法`，创建返回一个`虚拟dom`

1. 知道h函数在vue里面做了修改，使得可以在组件里面使用；在snabbdom里面只能用来创建虚拟dom
2. 理解函数重载概念

- 函数名相同
- 函数的参数不同。在js里面，会进行覆盖，无法实现重载。可以利用 arguments来实现
- 但是在ts里面是可以实现的

```js
function fn (a,b) {

​	return a + b

}

function fn (a, b, c) {

​	return a + b + c

}
```



```diff
export function h(sel: string): VNode;
export function h(sel: string, data: VNodeData): VNode;
export function h(sel: string, children: VNodeChildren): VNode;
export function h(sel: string, data: VNodeData, children: VNodeChildren): VNode;
+//  表示 b和c的参数都是可选的 返回值也是虚拟dom
export function h(sel: any, b?: any, c?: any): VNode {
  var data: VNodeData = {}, children: any, text: any, i: number;  
  if (c !== undefined) {
    data = b;
+    // 有三个参数 参数c如果是数组，就赋值给children 参数c如果是字符串 就赋值给text 参数c如果有sel
    if (is.array(c)) { children = c; }
    else if (is.primitive(c)) { text = c; }
+    // sel是什么意思呢？表示c是虚拟dom，就要转化为数组，再给children
    else if (c && c.sel) { children = [c]; }
  } else if (b !== undefined) {
+  // 如果只有2个参数 
  // 参数b如果是数组 就赋值给children
  // 字符串 就给text
  // 是虚拟dom 转化为数组 再给children
  // 否则直接给data
    if (is.array(b)) { children = b; }
    else if (is.primitive(b)) { text = b; }
    else if (b && b.sel) { children = [b]; }
    else { data = b; }
  }
+  // 判断 children里面是否有值 -> 如果有值 -> 遍历
  if (children !== undefined) {
    for (i = 0; i < children.length; ++i) {
+    // 判断里面是否是 string/number -> 如果是的话 就创建文本/字符串的虚拟节点
      if (is.primitive(children[i])) children[i] = vnode(undefined, undefined, undefined, children[i], undefined);
    }
  }
  if (
+  // 如果是 svg 就添加命名空间
    sel[0] === 's' && sel[1] === 'v' && sel[2] === 'g' &&
    (sel.length === 3 || sel[3] === '.' || sel[3] === '#')
  ) {
    addNS(data, children, sel);
  }
+  // 最后返回的就是一个虚拟节点 调用的是vnode方法，
  return vnode(sel, data, children, text, undefined);
};
export default h;
```



### 7.2 vnode函数的使用

```js
// vnode函数返回的js对象 可以有的这些内容
export interface VNode {
    // 选择器 h函数传递的第一个参数
    sel: string | undefined;
    // 跟模块的很多内容相关 类 属性 自定义属性 事件 样式 
    data: VNodeData | undefined;
    // 子节点 和 children 是不会同时存在的
    children: Array<VNode | string> | undefined;
    // 虚拟dom -》 转化为真实dom 信息存储到 elm里面去
    elm: Node | undefined;
    // 文本内容
    text: string | undefined;
    // diff算法优化时使用的
    key: Key | undefined;
}
```

如果有text，那么children就是undefined



```js
export declare function vnode(sel: string | undefined, data: any | undefined, children: Array<VNode | string> | undefined, text: string | undefined, elm: Element | Text | undefined): VNode;
```





### 7.3 h函数和vnode函数的执行流程的模拟

```js
// 创建虚拟dom 使用模块
let vnode = h(
    'div#app',
    {
        style:{
            backgroundColor: 'red'
        },
        on: {
            click: eventHandler
        },
        // 自定义属性的设置
        dataset: {
            time: new Date(),
            value: '123'
        }
    },
    // 子元素 用数组也可以 也可以用字符串 字符串就是表示div的内容
    // 三个参数 第三个是数组的情况
    [
        h('p', {style: { backgroundColor: 'pink' }}, '你好我是小p'),
        '132',
        h('h1',{attrs: {name: '123'}}, '我是小h1')
    ]
)
```





1. 理解，h函数执行，是从内到外执行的，也就是 先执行里面的h函数，再开始执行外面的h函数

```diff
h('p', {style: { backgroundColor: 'pink' }}, '你好我是小p')
-》
'132',
-》
h('h1',{attrs: {name: '123'}}, '我是小h1')
-》
最外层的h('div#app', {}, [])
```



2. 理解如下的执行流程

```diff
h('p', {style: { backgroundColor: 'pink' }}, '你好我是小p')
```



第三个参数不是undefined,进入第一个if语句，

第三个参数是字符串，`is.primitive(c)`

```diff
function h(sel, b, c) {
    var data = {}, children, text, i;
+    if (c !== undefined) {
        data = b;
        if (is.array(c)) {
            children = c;
        }
+        else if (is.primitive(c)) {
+            text = c;
        }
        else if (c && c.sel) {
            children = [c];
        }
    }
    else if (b !== undefined) {
        if (is.array(b)) {
            children = b;
        }
        else if (is.primitive(b)) {
            text = b;
        }
        else if (b && b.sel) {
            children = [b];
        }
        else {
            data = b;
        }
    }
    if (children !== undefined) {
        for (i = 0; i < children.length; ++i) {
            if (is.primitive(children[i]))
                children[i] = vnode_1.vnode(undefined, undefined, undefined, children[i], undefined);
        }
    }
    if (sel[0] === 's' && sel[1] === 'v' && sel[2] === 'g' &&
        (sel.length === 3 || sel[3] === '.' || sel[3] === '#')) {
        addNS(data, children, sel);
    }
+    return vnode_1.vnode(sel, data, children, text, undefined);
}
```



最终，直接返回`vnode函数`的调用，传入的参数，

```diff
// h('p', {style: { backgroundColor: 'pink' }}, '你好我是小p')
vnode_1.vnode(sel, data, children, text, undefined);
sel是标签，
data是{style: { backgroundColor: 'pink' }}，
children是undefined
text是 '你好我是小p'
elm: undefined
```



'132',

```diff
并没有调用h函数，最终值还是 '132'
```



当h函数渲染下一个标签时

```diff
// h('h1',{attrs: {name: '123'}}, '我是小h1')

vnode_1.vnode(sel, data, children, text, undefined);
sel是标签h1，
data是{attrs: {name: '123'}}，
children是undefined
text是 '我是小h1'
elm: undefined
key: undefined
```





而 p和h1返回的都是对象，因为

```diff
export interface VNode {
    // 选择器 h函数传递的第一个参数
    sel: string | undefined;
    // 跟模块的很多内容相关 类 属性 自定义属性 事件 样式 
    data: VNodeData | undefined;
    // 子节点 
    children: Array<VNode | string> | undefined;
    // 虚拟dom -》 转化为真实dom 信息存储到 elm里面去
    elm: Node | undefined;
    // 文本内容
    text: string | undefined;
    // diff算法优化时使用的
    key: Key | undefined;
}

export declare function vnode(sel: string | undefined, data: any | undefined, children: Array<VNode | string> | undefined, text: string | undefined, elm: Element | Text | +undefined): VNode;

// 最终的返回值是 :VNode
// 而VNode是一个对象
```







此时最终的结果就是

```diff
let vnode = h(
    'div#app',
    {
        style:{
            backgroundColor: 'red'
        },
        on: {
            click: eventHandler
        },
        // 自定义属性的设置
        dataset: {
            time: new Date(),
            value: '123'
        }
    },
    // 子元素 用数组也可以 也可以用字符串 字符串就是表示div的内容
    // 三个参数 第三个是数组的情况
    [
        {sel: 'p', data: {……}, ele: undefined, children: undefined, text:'你好我是小p', key: undefined}
        '132',
        {……text: '我是小h1', key: undefined}
    ]
)
```



然后执行最外层的h函数，

```diff
function h(sel, b, c) {
    var data = {}, children, text, i;
+    if (c !== undefined) {
        data = b;
+        if (is.array(c)) {
+            children = c;
        }
        else if (is.primitive(c)) {
            text = c;
        }
        else if (c && c.sel) {
            children = [c];
        }
    }
    else if (b !== undefined) {
        if (is.array(b)) {
            children = b;
        }
        else if (is.primitive(b)) {
            text = b;
        }
        else if (b && b.sel) {
            children = [b];
        }
        else {
            data = b;
        }
    }
+    if (children !== undefined) {
        for (i = 0; i < children.length; ++i) {
            if (is.primitive(children[i]))
                children[i] = vnode_1.vnode(undefined, undefined, undefined, children[i], undefined);
        }
    }
    if (sel[0] === 's' && sel[1] === 'v' && sel[2] === 'g' &&
        (sel.length === 3 || sel[3] === '.' || sel[3] === '#')) {
        addNS(data, children, sel);
    }
    return vnode_1.vnode(sel, data, children, text, undefined);
}
```



此时，走到if(children !== undefined)时，要遍历children里面的值,判断是否是字符串

```js
    if (children !== undefined) {
        for (i = 0; i < children.length; ++i) {
            if (is.primitive(children[i]))
                children[i] = vnode_1.vnode(undefined, undefined, undefined, children[i], undefined);
        }
    }
```



而，只有数组的索引为1的值，才会通过if语句，最终调用vnode_1.vnode的方法

```diff
    [
        {sel: 'p', data: {……}, ele: undefined, children: undefined, text:'你好我是小p', key: undefined}
+        '132',
        {……text: '我是小h1', key: undefined}
    ]
```

生成

```diff
    [
        {sel: 'p', data: {……}, ele: undefined, children: undefined, text:'你好我是小p', key: undefined}
        {sel: undefined， data: , ele: undefined, children: undefined, text:'132', key: undefined}
        {……text: '我是小h1', key: undefined}
    ]
```





最终返回对象

```diff
{sel: 'div#app', 
 data: {
        style:{
            backgroundColor: 'red'
        },
        on: {
            click: eventHandler
        },
        // 自定义属性的设置
        dataset: {
            time: new Date(),
            value: '123'
        }
  },
  ele: 会记录真实dom的信息 -》 undefined，
  children: [
        {sel: 'p', data: {……}, ele: undefined, children: undefined, text:'你好我是小p', key: undefined}
        {sel: undefined， data: , ele: undefined, children: undefined, text:'132', key: undefined}
        {……text: '我是小h1', key: undefined}
  ],
  text: undefined
 }
```



后续会调用patch函数 渲染为真实的dom。上面只是渲染为虚拟dom



### 7.4 patch函数的使用

```diff
1. patch函数，会比较新旧节点的值，然后把更新后的内容，更新到真实的dom节点上面去。patch的返回值就是一个，新的节点
2. vnode是一个js对象，里面有sel、key这两个关键的属性
3. 根据“sel”属性，如果说，选择器不同，就创建新的节点
4. 选择器相同，就比较text属性，看看是不是内容不同，
5. 如果text相同，看看有没有chidlren属性，有的话，如果值不同，就要依靠diff算法来进行比较
```





如何获取到patch函数的呢？

```diff
const patch = init([])
```





### 7.5 查看init源码

1. 首先，注意，init方法，返回的是一个patch函数，是一个高阶函数。

```diff
return function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
let i: number, elm: Node, parent: Node;
```





2. 查看init函数，在snabbdom.ts文件



他会判断init有没有传入domapi，如果有传入的话，就用传进来的

没有就用domApi

```diff
export function init(modules: Array<Partial<Module>>, domApi?: DOMAPI) {
  let i: number, j: number, cbs = ({} as ModuleHooks);

+  const api: DOMAPI = domApi !== undefined ? domApi : htmlDomApi;
```



htmlDomApi,在另一个文件里面声明了dom的操作方法

```diff
export const htmlDomApi = {
  createElement,
  createElementNS,
  createTextNode,
  createComment,
  insertBefore,
  removeChild,
  appendChild,
  parentNode,
  nextSibling,
  tagName,
  setTextContent,
  getTextContent,
  isElement,
  isText,
  isComment,
} as DOMAPI;
```





3. 理解如下代码，

遍历了hooks数组，这个数组声明的create update remove会在稍后使用，作为对象的属性

声明了cbs对象，

cbs对象里面的每一个属性都是二维数组

```diff
cbs = {

update: [],

create: [],

remove: []
……
}
```

`snabbdom/snabbdom.js`

```diff

+const hooks: (keyof Module)[] = ['create', 'update', 'remove', 'destroy', 'pre', 'post'];
……
export function init(modules: Array<Partial<Module>>, domApi?: DOMAPI) {
+  let i: number, j: number, cbs = ({} as ModuleHooks);

  const api: DOMAPI = domApi !== undefined ? domApi : htmlDomApi;

+  for (i = 0; i < hooks.length; ++i) {
    // cbs = {
    //   create: [fn1, fn2],
    //   update: [fn1, fn2]
    //   ……
    // }
    cbs[hooks[i]] = [];
    for (j = 0; j < modules.length; ++j) {
      // 这里是什么意思呢？
      调用init方法，传入比如[style, eventListeners]
      而modules是传进来的数组，
      而modules[j]就是style或者 eventListeners 
      style[create]就是去 modules/style.js文件夹下面的方法
      // const patch = init([style, eventListeners])
      // modules[j] style -> style[create]
      const hook = modules[j][hooks[i]];
      if (hook !== undefined) {
        (cbs[hooks[i]] as Array<any>).push(hook);
      }
    }
  }
```

`modules/style.js`

```diff
export const styleModule = {
  pre: forceReflow,
  create: updateStyle,
  update: updateStyle,
  destroy: applyDestroyStyle,
  remove: applyRemoveStyle
} as Module;
```





### 7.6 patch函数

```diff
+ 传入新节点和旧的节点，注意 旧节点oldVnode -> 可能是一个真实的dom元素
return function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
    let i: number, elm: Node, parent: Node;
+ 记录被插入的 vnode队列 批量触发 insert更新 而不是 每次有更新就渲染
    const insertedVnodeQueue: VNodeQueue = [];
+ 判断cbs里面有没有pre函数 有的话就调用全局pre钩子函数 ？
    for (i = 0; i < cbs.pre.length; ++i) cbs.pre[i]();
+ 判断oldVnode 是否是虚拟节点 
    if (!isVnode(oldVnode)) {
+ 如果不是，就利用emptyNodeAt函数，将其转化为虚拟节点, 可以查看下面的 emptyNodeAt的代码片段
      oldVnode = emptyNodeAt(oldVnode);
    }
+ 判断是否是相同的节点，如果是，
+ 比较内容 根据sel和key属性
    if (sameVnode(oldVnode, vnode)) {
      patchVnode(oldVnode, vnode, insertedVnodeQueue);
    } else {
+ 如果不是同一个节点 获取 旧节点的elm属性
      elm = oldVnode.elm as Node;
+ 获取elm的父亲
      parent = api.parentNode(elm);
+ 根据vnode 创建一个新的dom节点
      createElm(vnode, insertedVnodeQueue);
+ 如果parent不是空的值
      if (parent !== null) {
+ 在parent父亲里面，把vnode.elm节点插入到 api.nextSibling(elm) 也就是 elm的下一个兄弟节点上
        api.insertBefore(parent, vnode.elm as Node, api.nextSibling(elm));
+ 删除父节点里面的 oldVnode
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }
+ 遍历插入的队列
    for (i = 0; i < insertedVnodeQueue.length; ++i) {
      (((insertedVnodeQueue[i].data as VNodeData).hook as Hooks).insert as any)(insertedVnodeQueue[i]);
    }
    for (i = 0; i < cbs.post.length; ++i) cbs.post[i]();
    return vnode;
  };
```



### emptyNodeAt

```diff
  function emptyNodeAt(elm: Element) {
+ 查看 有没有id属性，有的话 就拼上#号
    const id = elm.id ? '#' + elm.id : '';
+ 查看是否有类名 className 有的话 就拼上. 并且如果有多个类名 就要用split 分割开空格转化为数组，再转化为字符串
    const c = elm.className ? '.' + elm.className.split(' ').join('.') : '';
+ 返回 调用 vnode函数 将 真实dom -> 虚拟dom，
+ 传入选择器的名字， api.tagName(elm).toLowerCase() + id + c
+ 真实的dom节点 “ele”， 这个ele是传入进来的
    return vnode(api.tagName(elm).toLowerCase() + id + c, {}, [], undefined, elm);
  }
```







### createEle函数

```diff
function createElm(vnode: VNode, insertedVnodeQueue: VNodeQueue): Node {
+赋值 vnode.data给data
    let i: any, data = vnode.data;
+如果data不是undefined 就接受 data.hook -> i -> i.init -> i 意思就是如果vnode里面的data有hook并且里面有init函数
    if (data !== undefined) {
      if (isDef(i = data.hook) && isDef(i = i.init)) {
+调用init函数 i(vnode)
        i(vnode);
+为什么要重新赋值 因为调用init函数可能会影响 vnode.data的值
        data = vnode.data;
      }
    }
    let children = vnode.children, sel = vnode.sel;
+如果选择器是 ! 表明是注释节点
    if (sel === '!') {
+如果test是undefined 就vnode.text = ''等于空
      if (isUndef(vnode.text)) {
        vnode.text = '';
      }
+创建注释节点
      vnode.elm = api.createComment(vnode.text as string);
    } else if (sel !== undefined) {
+如果 sel不是 !， 而sel也不是 undefined
      // Parse selector
+解析div#name.cls
      const hashIdx = sel.indexOf('#');
+先id 后class
      const dotIdx = sel.indexOf('.', hashIdx);
      const hash = hashIdx > 0 ? hashIdx : sel.length;
      const dot = dotIdx > 0 ? dotIdx : sel.length;
      const tag = hashIdx !== -1 || dotIdx !== -1 ? sel.slice(0, Math.min(hash, dot)) : sel;
+要么创建“有命名空间的元素”，要么创建普通的dom元素，判断条件是 data里面是否有ns data就是vnode.data
      const elm = vnode.elm = isDef(data) && isDef(i = (data as VNodeData).ns) ? api.createElementNS(i, tag)
                                                                               : api.createElement(tag);
      if (hash < dot) elm.setAttribute('id', sel.slice(hash + 1, dot));
      if (dotIdx > 0) elm.setAttribute('class', sel.slice(dot + 1).replace(/\./g, ' '));
+触发全局的钩子函数“create”
      for (i = 0; i < cbs.create.length; ++i) cbs.create[i](emptyNode, vnode);
+判断是否有孩子 如果有
      if (is.array(children)) {
        for (i = 0; i < children.length; ++i) {
          const ch = children[i];
          if (ch != null) {
+孩子插入到当前的Vnode,并且这个子元素节点要在 insertedVnodeQueue 里面做记录
+这里使用了递归
            api.appendChild(elm, createElm(ch as VNode, insertedVnodeQueue));
          }
        }
+如果没有孩子 children 就创建文本节点 
      } else if (is.primitive(vnode.text)) {
        api.appendChild(elm, api.createTextNode(vnode.text));
      }
+到这里 dom节点创建完毕 但是没有渲染，注意，没有渲染到页面 只是创建好


      i = (vnode.data as VNodeData).hook; // Reuse variable
      if (isDef(i)) {

        if (i.create) i.create(emptyNode, vnode);
+把insert钩子函数 加入到 insertedVnodeQueue 队列 到时候是一次性insert很多个dom，不是一个一个触发
        if (i.insert) insertedVnodeQueue.push(vnode);
      }
    } else {
+sel是undefined 没有选择器 -> 就是文本节点
      vnode.elm = api.createTextNode(vnode.text as string);
    }
    return vnode.elm;
  }
```



### removeVNodes函数

这个函数 用来删除节点，结合`invokeDestroyHook`函数和`createRmCb`函数

```diff
+传入四个参数，parentElm是父节点，从这个父节点里面删除vnodes，注意 vnodes 是数组
function removeVnodes(parentElm: Node,
                        vnodes: Array<VNode>,
                        startIdx: number,
                        endIdx: number): void {
+进行遍历
    for (; startIdx <= endIdx; ++startIdx) {
+listeners是计数器 -> 给全局的 remove方法计数 cbs.remove.length

+ch的值就是 [oldVNodes]里面的酒节点
      let i: any, listeners: number, rm: () => void, ch = vnodes[startIdx];
      if (ch != null) {
        if (isDef(ch.sel)) {
+调用destroy钩子函数
          invokeDestroyHook(ch);
          listeners = cbs.remove.length + 1;
+调用 createRmCb函数 返回的也是一个函数
+传入旧节点和计数器
          rm = createRmCb(ch.elm as Node, listeners);
+调用全局的remove钩子 每次调用 就listeners减一
          for (i = 0; i < cbs.remove.length; ++i) cbs.remove[i](ch, rm);
+调用ch.data.hook方法里面的remove,传入纠结点ch和rm函数
          if (isDef(i = ch.data) && isDef(i = i.hook) && isDef(i = i.remove)) {
            i(ch, rm);
          } else {
+如果说 ch里面都没有data.hook.remove 就直接调用 rm函数删除 确保能够删除
也就是看 vnodes[startIndex].data.hook.remove
            rm();
          }
        } else { // Text node
+文本节点 直接删除
          api.removeChild(parentElm, ch.elm as Node);
        }
      }
    }
  }
```





### createRmCb函数

主要是利用这个函数 进行节点的删除

```diff
  function createRmCb(childElm: Node, listeners: number) {
    return function rmCb() {
      if (--listeners === 0) {
        const parent = api.parentNode(childElm);
        api.removeChild(parent, childElm);
      }
    };
  }
```







### addVNodes

作用：把真实的dom 插入到 指定的父节点之前

```diff
  function addVnodes(parentElm: Node,
                     before: Node | null,
                     vnodes: Array<VNode>,
                     startIdx: number,
                     endIdx: number,
                     insertedVnodeQueue: VNodeQueue) {
    for (; startIdx <= endIdx; ++startIdx) {
+拿到 vnodes[startIndex] -> oldVnode
      const ch = vnodes[startIdx];
+ch不是null 就调用 createElm(ch, insertedVnodeQueue) 把 ch这个虚拟dom 创建为真实的dom -> 插入到 before前面
      if (ch != null) {
        api.insertBefore(parentElm, createElm(ch, insertedVnodeQueue), before);
      }
    }
  }
```





### patchVNode

1. patchVNode和patch有什么区别 -> 在patchVNode函数里面调用了 patch函数

>作用：
>
>最最重要的就是，进行当前新的节点和旧的节点 + 当前新的节点的children 与 旧的节点的 children的比较，后者里面利用了 updateChidlren的diff算法的比较

```diff
function patchVnode(oldVnode: VNode, vnode: VNode, insertedVnodeQueue: VNodeQueue) {
    let i: any, hook: any;
    if (isDef(i = vnode.data) && isDef(hook = i.hook) && isDef(i = hook.prepatch)) {
      i(oldVnode, vnode);
    }
    const elm = vnode.elm = (oldVnode.elm as Node);
    /* 
    请知道 oldVnode -> 旧节点的信息
    vnode -> 新节点的信息
    oldCh -> 旧节点的children
    ch -> vnode.children 新节点的children
    */
    let oldCh = oldVnode.children;
    let ch = vnode.children;
+1. 比较的是新旧节点的内存地址 如果相同 说明内容根本没变化 -》直接return
    if (oldVnode === vnode) return;
+2. vnode是新的节点 如果data不是 undefined
    if (vnode.data !== undefined) {
+2.1 调用全局的update函数 cbs[i].update()方法
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
      i = vnode.data.hook;
+2.2 如果用户传递进来了hook在data里面，也就是h函数的第二个参数，
      if (isDef(i) && isDef(i = i.update)) i(oldVnode, vnode);
    }
+3. vnode 新的节点没有text 
    if (isUndef(vnode.text)) {
+3.1 新旧节点 是否都有children 如果都有的话
      if (isDef(oldCh) && isDef(ch)) {
+3.2 判断 新节点和旧节点的children是否相等 不相等 就 updateChildren
+updateChildren这个方法 非常重要 里面涉及diff算法，是更新子节点的重中之重
        if (oldCh !== ch) updateChildren(elm, oldCh as Array<VNode>, ch as Array<VNode>, insertedVnodeQueue);
+3.3 ch - 新的节点； oldCh - 旧的节点 如果说新的节点有children 大那是 oldCh没有children
      } else if (isDef(ch)) {
+3.4 旧的节点 有text -> 清空旧节点的内容，elm来自oldVnode
+    const elm = vnode.elm = (oldVnode.elm as Node);
        if (isDef(oldVnode.text)) api.setTextContent(elm, '');
+    add -》添加所有的子节点 ch是一个数组
        addVnodes(elm, null, ch as Array<VNode>, 0, (ch as Array<VNode>).length - 1, insertedVnodeQueue);
      } else if (isDef(oldCh)) 
+  	 老节点有children 但是新节点没有 删除老节点里面的children
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
+ 	 新旧都没有children 老节点里面有text 清空它
      } else if (isDef(oldVnode.text)) {
        api.setTextContent(elm, '');
      }
+    新节点有text -> 老节点也有
    } else if (oldVnode.text !== vnode.text) {
+ 	 如果老节点有 children 删除干掉他
      if (isDef(oldCh)) {
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      }
+     修改内容
      api.setTextContent(elm, vnode.text as string);
    }
    if (isDef(hook) && isDef(i = hook.postpatch)) {
      i(oldVnode, vnode);
    }
  }
```









### updateChildren方法

>注意，`patch`和`updateChildren`是diff算法里面的重中之重

四种比较情况的列举：

`updateChildren`源码，我把上面的代码拆成了6个部分，一个部分一个部分的去分析，不用害怕

```diff
function updateChildren(parentElm: Node,
                          oldCh: Array<VNode>,
                          newCh: Array<VNode>,
                          insertedVnodeQueue: VNodeQueue) {
+1 
    let oldStartIdx = 0, newStartIdx = 0; // 旧dom树的起始索引，新dom树的起始索引
    let oldEndIdx = oldCh.length - 1; // 旧dom树的结尾索引
    let oldStartVnode = oldCh[0]; // oldCh是旧的节点数，oldStartVnode是旧的节点的第一个
    let oldEndVnode = oldCh[oldEndIdx]; // 旧dom节点的最后一个
    let newEndIdx = newCh.length - 1; // 新dom的最后一个节点索引
    let newStartVnode = newCh[0]; // 新的起始节点
    let newEndVnode = newCh[newEndIdx]; // 新dom的树的最后一个节点
    let oldKeyToIdx: any;
    let idxInOld: number;
    let elmToMove: VNode;
    let before: any;
+2  while循环 注意start都必须小于等于end
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // oldStartVnode如果是空，就让索引++获取新的节点 下面的类推
      if (oldStartVnode == null) {
        oldStartVnode = oldCh[++oldStartIdx]; // Vnode might have been moved left
      } else if (oldEndVnode == null) {
        oldEndVnode = oldCh[--oldEndIdx];
      } else if (newStartVnode == null) {
        newStartVnode = newCh[++newStartIdx];
      } else if (newEndVnode == null) {
        newEndVnode = newCh[--newEndIdx];
+3
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node));
        oldStartVnode = oldCh[++oldStartIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
        api.insertBefore(parentElm, oldEndVnode.elm as Node, oldStartVnode.elm as Node);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      } else {
+4
        if (oldKeyToIdx === undefined) {
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
        }
        idxInOld = oldKeyToIdx[newStartVnode.key as string];
        if (isUndef(idxInOld)) { // New element
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          newStartVnode = newCh[++newStartIdx];
        } else {
+5
          elmToMove = oldCh[idxInOld];
          if (elmToMove.sel !== newStartVnode.sel) {
            api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          } else {
            patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
            oldCh[idxInOld] = undefined as any;
            api.insertBefore(parentElm, (elmToMove.elm as Node), oldStartVnode.elm as Node);
          }
          newStartVnode = newCh[++newStartIdx];
        }
      }
    }
+6
    if (oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx) {
      if (oldStartIdx > oldEndIdx) {
        before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm;
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
      } else {
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
      }
    }
  }
```





diff算法：

1. 依据真实的dom，生成虚拟的dom树
2. 依照层级，一层一层对比新旧的dom，比较的一句依靠sameVnode方法(每个Vnode的 sel和key)，
3. 如果有不同，就会进行更新。更新依赖的是 patchVnode方法





#### 第一种情况

1. 比较新dom的(新的虚拟dom树的某一个层级)开始节点和旧dom树的开始节点

- 如果相同(sameVnode)，就更新索引值，旧dom(oldStartIdx)和新dom(newStartIdx)的索引值都加1

- 如果不相同，就转化为比较， 旧dom树的结尾节点和新dom树的结尾节点，如果相同
  - 索引值都减1
  - 如果不相同 进行下一种情况

如下图，第一个红色框，表示首先比较起始节点，利用sameVnode,如果相同，就走patchVnode

如果不相同，就去第二个红色框。相同，就让 startIndex





![image-20220903184203717](https://typora-1309613071.cos.ap-shanghai.myqcloud.com/typora/image-20220903184203717.png)

序号3的代码，对应上图，

```diff
      // 起始新节点和起始旧节点是否同？
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 相同就更新
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        // 索引都++,获取到下一组dom，然后下一次循环，再次判断是否相等
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 看看结尾新节点和结尾旧节点是否相同，相同就调用patchVnode
        // 索引都--，往前面去看
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
```





#### 第二种情况

旧dom的开始节点 vs 新dom树的结尾节点

如果旧dom树的开始节点和新dom树的结尾节点相同，调用patchVnode,旧dom树的`oldStartIdx` 指向的节点 移动到最右侧，如下图所示。`oldStartIdx的索引要++，newEndIdx索引要--`

![image-20220903185236204](https://typora-1309613071.cos.ap-shanghai.myqcloud.com/typora/image-20220903185236204.png)

```diff
      // 判同
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      // 打补丁，进行更新
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        // 移动节点 把oldStartVnode 移动到 oldEndVnode 的兄弟的前面，
        api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node));
        // oldStartIdx索引++，获取下一个节点
        oldStartVnode = oldCh[++oldStartIdx];
        // newEndIdx -- 往后退
        newEndVnode = newCh[--newEndIdx];
      }
```



存在疑问：此时，原本的位置的节点是否还会存在？

insertBefore的理解？

nextSibling的使用



#### 第三种情况

如果说，上面的情况不满足，

就要比较，旧的dom树的结尾节点 vs 新的dom树的开始节点。如果相等：

- 调用patch
- 旧dom树结尾节点 移动到最前面
- 旧dom的 index--，而新dom的index++

![image-20220903185813142](https://typora-1309613071.cos.ap-shanghai.myqcloud.com/typora/image-20220903185813142.png)

```diff
     
     } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
     // 打补丁
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
     // 把旧dom树结尾节点 插入到 旧dom树的最开头的前面
        api.insertBefore(parentElm, oldEndVnode.elm as Node, oldStartVnode.elm as Node);
      // 索引更新
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      } 
```





#### 第四种情况：

1. 拿到新dom树里面的节点的key

2. 拿着这个key去旧的dom树对象里面去找，是否有相同的节点

3. 如果找到相同的节点

   3.1 判断是否有 同样的选择器

   3.2 如果key相同，但是选择器不同，这样也不行，要重新创建dom节点，插入到旧dom树的最开头的前面

   3.3 如果key相同，选择器也相同，就是同一个节点，patchVnode进行更新，把这个节点 插入到最前面 然后让newStartIdx的索引值++

4. 如果找不到相同的节点

   4.1 说明是新的节点 直接创建新的 并且插入到旧dom树的起始节点

   4.2 newStartIdx++即可

```diff
else {
        if (oldKeyToIdx === undefined) {
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
        }
        idxInOld = oldKeyToIdx[newStartVnode.key as string];
        if (isUndef(idxInOld)) { // New element
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          newStartVnode = newCh[++newStartIdx];
        } else {
        // key 找到了 
        // 取名 给 oldCh[idxInOld]取名Wie elmToMove
          elmToMove = oldCh[idxInOld];
          // 选择器
          if (elmToMove.sel !== newStartVnode.sel) {
            api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          } else {
          // 同一个节点
            patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
            oldCh[idxInOld] = undefined as any;
            api.insertBefore(parentElm, (elmToMove.elm as Node), oldStartVnode.elm as Node);
          }
          newStartVnode = newCh[++newStartIdx];
        }
      }
```





#### 第五种情况

这里的if判断存在的情况是

1. oldStartIdx 《 oldEndIdx 但是 newStartIdx > newEndIdx 也能够满足  => 这种情况是 新dom树的子节点遍历完了

   旧dom树剩下的就删掉 remove

2. oldStartIdx > oldEndIdx 但是 newStartIdx 《 newEndIdx 也能够满足 => 这种情况是 旧dom树的子节点遍历完了

   新dom树剩下的都是要创建 插入的

3. oldStartIdx 《 oldEndIdx 且 newStartIdx 《 newEndIdx 也能够满足 => 这种情况是 新旧dom树都没有遍历完 这里也要删除

4. 只有 oldStartIdx > oldEndIdx 并且 newStartIdx > newEndIdx 是不能够满足的 => 这种情况是 新旧dom树同时都遍历完了 就不额外操作

```diff
    if (oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx) {
      if (oldStartIdx > oldEndIdx) {
      // 新dom树剩下的都是要创建 插入的
        before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm;
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
      } else {
      // 旧dom树剩下的就删掉 remove
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
      }
    }
```





>困惑：
>
>1. oldStartIdx 《 oldEndIdx 且 newStartIdx 《 newEndIdx  这种情况 旧dom也会删除完毕？
>2. oldCh[idxInOld] = undefined as any; 第四种情况 这句代码不理解

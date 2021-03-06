## 前情提要 —— 一种常见的Event对象的实现：
```javascript
function initEventMap(instance, eventName) {
	const { enents } = instance;
	events[eventName] = events[eventName] || [];
	return events[eventName]; 
}
class FEvent {
	constructor() {
		this.events = {}; // 存储绑定的事件
	}
	$on(eventName, handle) {
		initEventMap(this.events, eventName).push(handle);
	}
	$emit(eventName, data) {
		initEventMap(this.events, eventName).forEach(handle => handle(data));
	}
	$off(eventName) {}
}
```


## 使用方式分析

### 全局方式
```javascript
[global].event = new FEvent();

//or in event module file
export new FEvent();
```

历史上我干过不少这种事，比如在Vue框架下，以`export new Vue()`的方式作为事件模块使用，这样所有事件都指向同一个事件实例。一般倒也没什么问题，但是中大型应用中事件的管理可能会有点麻烦：
	
1. 需要留意 eventName 的定义：为了避免重名可能通常会需要加上一些前缀，eg:
```javascript
event.$on('article-update', doSth)
event.$on('article-tag-update', doSth)
event.$on('article-title-update', doSth)
```
2. 关注事件解绑：事件实例是一个全局的实例，被绑定的事件及其引用的上下文环境，在不解绑的情况下，将不能被释放，这可能会引起内存泄漏。例子参考：[记一次网页内存溢出分析及解决实践](https://juejin.im/post/5c3dce07e51d4551e960d840)

		不过以上两点，总是需要人为去注意。多人协作、或者个人状态不好，难免出现差错。

### 与实例绑定
DOM、VueComponent 等，都是这种方式

```javascript
// extends 方式
class Article extends FEvent {
	constructor{
		this.events = {}; // 用于存储实例上的事件
	}
}

// mixin 方式
function mixinEvent(target) {
    const protos = FEvent.prototype;
    Object.getOwnPropertyNames(protos).forEach((key) => {
        if (key !== 'constructor') {
            target.prototype[key] = protos[key];
        }
    });
    target.prototype.events = {};
}
class Article {}
mixinEvent(Article);

// decorator 方式
@mixinEvent // 实现同 mixin
class A {}

// use
const article = new A();
article.$on(...);
article.$emit(...);
```

除了class 定义上麻烦一点，但是避免了全局事件方式的一些困扰：
1. 一般不用担心事件绑定带来的内存泄漏问题，只要实例不被引用，不用担心绑定的事件会驻留内存
2. 不用担心事件重名问题
3. 特定场景下减少事件参数的判断，比如以下场景：
```javascript
event.$on('article-update', (article) => {
	if (article === this.currentArticle) {
		// doSth
	}
});
```


## 在实际使用时遇到的一些困扰以及处理

### extens/mixin 时需要声明 events 属性
之前期望借用 Vue 的 event 功能，代码模板长这样：
```javascript
class A extends Vue {
	constructor(){
		this._events = {}; // 这里就有点别扭了，毕竟不是公开接口
	} 
}
```
但是这不是标准用法，_events 属性还是看了源码才知道的。

所以，对于 Event 的实现，需要做一些调整：
```javascript
function initEventMap(instance, eventName) {
	// Step1: 在使用时进行检查并初始化
	if (!instance.events) {
		instance.events = {}; // 或者利用 WeakMap私有化 events
	}
	
	const { enents } = instance;
	events[eventName] = events[eventName] || [];
	return events[eventName]; 
}

// Step2: 移除 FEvent 的 constructor
```


### 事件的 Promise 化
在某些场景下，可能会期望 $emit 之后，拿到被触发函数的执行结果。比如在 Vue 中有这样的嵌套模板:
```html
<template>
	<compA>
		<compB>
	</compA>
</template>
```
期望利用 event 的方式代替 ref 调用 compB 中方法，并得到该方法的执行结果。于是可以有这样的方式：
```javascript
// in compB
ins.$on('compBMethod', async ({ data, callback }) => {
	const res = await this.compBMethod(data);
	callback(res);
});

// use
new Promise((resolve) => {
	ins.$emit('compBMethod', { data, resolve });
}).then(doSth);
```

或者，给 FEvent 扩展一个实例方法：
```javascript
{
	$promiseEmit(eventName, ...args) {
		const events = initEventMap(this, eventName);
        const defers = events.map(
					  async handler => await handler.apply(this, [...args])
				);
		return Promise.all(defers);
	}
	...
}

// use
ins.$promiseEmit('init', data).then(...)
```

__不过emit 的 Promise化可能没有普适场景。事件的绑定顺序，会影响.then 的接收参数的顺序；并且按Promise.all 的工作方式，如果有任何地方绑定的事件执行出错，都会影响resolve的执行。所以，仅在特殊场景下，在明确event实例的使用范围的时候才考虑使用__


## 附相关代码
FEvent最终实现： [FEvent](https://github.com/feirpri/notes/blob/master/demo/event/index.js)
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkzMDc4MTU4NywxNjg0MjgxMDQ4LC0xOD
AyMTIzODI0XX0=
-->
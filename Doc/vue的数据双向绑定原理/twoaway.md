# 从源码角度看vue的数据双向绑定原理

> vue的数据双向绑定大家都应该十分的了解了，利用数据劫持结合发布者-订阅模式的方式，通过Object.defineProperty()来劫持各个属性的getter/setter，在数据变动的时候发布消息给订阅者，触发相应的监听回调。

### vue数据响应系统包含三个方面 observer，dep，watcher，我们先看下initData中的内容（initData函数位于src/core/instance/state.js文件中），我们可以看一下initData函数的源码：
```javascript
function initData(vm:Component){
    let data = vm.$options.data
    data = vm._data = typeof data == 'function'
     ?getData(data,vm)
     :data || {}
    if(!isPlainObject(data)){//isPlainObject判断data是否是一个对象
        data = {}
        process.env.NODE_ENv !== 'production' && warn(
            `data functions should return an object:\n`+
            `https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function`,
            vm
        )
    }
    //proxy data on instance
    const keys = Object.keys(data)
    const props = vm.$options.props
    const methods = vm.$options.methods
    let i = keys.length
    //while循环第一个作用就是防止data，props，methods的键值名称有冲突；第二个作用是将key值绑定到vm对象上
    while(i--){
        const key = keys[i]
        if(process.env.NODE_ENV !== 'production'){
            if(methods && hasOwn(methods,key)){//hasOwn函数的作用是判断key值在methods中有没有出现
                warn(
                    `method "${key}" has already been defined as a data property.`,
                    vm
                )
            }
        }
        if(props && hasOwn(props,key)){//hasOwn函数的作用是判断key值在props中有没有出现
            process.env.NODE_ENV !== 'production' && warn(
                `the data property "${key}" is already declared as a prop.`+
                `Use prop default value instead.`,
                vm
            )
        }else if(!isReserved(key)){//isReserved是判断key是否为保留的关键字
            proxy(vm,'_data',key)
        }
    }
    //observe data
    observe(data,true/*asRootData*/)
}
```
* initData函数的主要作用就是1.防止data,props,methods中的键值冲突；2.将_data上的数据代理到vm上；3.通过observe将所有数据变成observable

### 接下来看一下怎么进行observe，这个函数是定义在core/observer/index.js文件中

```javascript
/**
 * Attemp to create an observer instance for a value
 * returns the new observer if successfully observed
 * or the existing observer if the value already has one
*/
export default observe(value:any,asRootDara:?boolean):Observer | void{
    if(!isObject(value) || value instanceof VNode){
        return
    }
    let ob:Observer | void
    //判断传进来的value有没有__ob__属性，如果有，直接赋值给ob，没有的话就new一个Observer对象
    if(hasOwn(value,'__ob__') && value.__ob__ instanceof Observer){
        ob = value.__ob__
    }else if(
        // 这里的判断是为了确保value是单纯的对象，而不是函数或者是Regexp等情况。
        shouldObserve &&
        !isServerRendering() &&
        (Array.isArray(value) || isPlainObject(value)) &&
        Object.isExtensible(value) &&
        !value._isVue
    ){
        ob = new Observer(value)
    }
    if(asRootData && ob){// 如果是根数据则计数，后面Observer中的observe的asRootData非true
        ob.vmCount++
    }
    return ob
}
```
* observe的作用就是返回一个Observe对象

### 接下来看一下Observer构造函数，Observer函数的主要作用就是遍历对象的所有属性，将其进行双向绑定。

```javascript
export default Observer{
    value:any;
    dep:Dep;
    vmCount:number;//number of ams that has this object as root $data

    constructor(value:any){
        this.value = value
        this.dep = new Dep()
        this.vmCount = 0;
        def(value,'__ob__',this)//ref函数的作用：在value对象上定义__ob__属性
        if(Array.isArray(value)){//如果是数组的话进行一下的操作
            const augment = hasProto//can we use __proto__?
             ? protoAugment
             : copyAugment
            augment(value,arrMethods,arrayKeys)//主要对完对数组方法 push pop shift unshift reverse sort splice进行重写，使其能够在数组改变时进行动态响应 
            this.observeArray(value)
        }else{
            this.walk(value)//如果是对象，则直接调用函数walk进行绑定
        }
    }

    /**
     * Walk through each property and convert them into
     * getter/setters.This method should only be called when
     * value type is Object
    */
    walk(obj:Object){
        const keys = Object.keys(obj)
        for(let i = 0; i<keys.length;i++){
            defineReactive(obj,keys[i])//调用defineReactive进行双向绑定
        }
    }

    /**
     * Observe a list of Array items
    */
    observeArray(items:Array<any>) {
        for(let i = 0,l = items.length; i<l; i++){
            observe(item[i])
        }
    }
}
```
* 对data数据中传进来的数据进行判断



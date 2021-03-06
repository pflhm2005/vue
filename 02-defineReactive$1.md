# 浏览器相关/监视器







## 检测浏览器环境





### UA



```javascript
// 判断是否在浏览器环境
var inBrowser = typeof window !== 'undefined';
// window.navigator.userAgent => 获取浏览器信息
// Google Chrome返回下列字符串
// Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36
// IE8返回下列字符串
// Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; Win64; x64; Trident/4.0; .NET CLR 2.0.50727; SLCC2; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET4.0C; .NET4.0E; InfoPath.3)
var UA = inBrowser && window.navigator.userAgent.toLowerCase();
```



#### IE



```javascript
// 判断是否有msie或trident字符串
var isIE = UA && /msie|trident/.test(UA);
```



#### IE9



```javascript
// 判断 msie 9.0
var isIE9 = UA && UA.indexOf('msie 9.0') > 0;
```



#### Edge



```javascript
// 判断 edge/
var isEdge = UA && UA.indexOf('edge/') > 0;
```



#### Android



```javascript
// 判断 android
var isAndroid = UA && UA.indexOf('android') > 0;
```



#### IOS



```javascript
// 判断 iphone,ipad,ipod,ios
var isIOS = UA && /iphone|ipad|ipod|ios/.test(UA);
```



---





## fn







### nextTick(异步执行器)



```javascript
// 异步方法优先级 Promise => MutationObserver => setTimeout
var nextTick = (function(){
  var callback = [],	// 执行函数队列
      pending = false,	// 是否执行中
      timerFunc;		// 异步执行函数
  function nextTickHandler(){
    pending = false;
    // 拷贝函数队列
    var copies = callback.slice(0);
    // 重置函数队列
    callbacks.length = 0;
    // 依次执行函数
    for(var i = 0; i < copies.length; i++){
      copies[i]();
    }
  }
  
  // Promise已定义且是内置方法
  if(typeof Promise !== 'undefined' && isNative(Promise)){
    //决议一个promise对象
    var p = Promise.resolve();
    var logError = function(err){console.log(err);};
    timeFunc = function(){
      p.then(nextTickHandler).catch(logError);
      // IOS有BUG 调用Promise.then不会完成调用 传空定时器可以解决
      if(isIOS){setTimeout(noop);}
    };
  }
  
  // 使用MutationObserver
  // 该对象是H5新特性 在DOM更新完触发 仅触发一次
  else if(typeof MutationObserver !== 'undefined' && 
          (isNative(MutationObserver) || 
           // IOS 7.x
           MutationObserver.toString() === '[object MutationObserverConstructor]')){
    var counter = 1;
    var observer = new MutationObserver(nextTickHandler);
    var textNode = document.createTextNode(String(counter));
    observer.observer(textNode,{
      characterData: true
    });
    timerFunc = function(){
      //这个方法会手动触发DOM更新事件
      counter = (counter + 1) % 2;
      textNode.data = String(counter);
  	};
  }
  
  // 都不行就用setTimeout()方法
  else{
    timerFunc = function(){
      setTimeout(nextTickHandler,0);
    };
  }
  // 自执行 返回这个函数
  // 接受两个参数 函数与上下文
  return function queueNextTick(cb,ctx){
    var _resolve;
    // 定义函数队列
    callback.push(function(){
      if(cb){cb.call(ctx);}
      if(_resolve){_resolve(ctx);}
    });
    if(!pending){
      pending = true;
      // 一开始觉得这里会执行 后面的if会忽略
      // 然而promise是microtask 此时主线程没跑完 所以promise挂起
      timerFunc();
    }
    // 这里应对不传参数的情况 决议一个空promise
    if(!cb && typeof Promise !== 'undefined'){
      return new Promise(function(resolve){
        _resolve = resolve;
      });
    }
  }
})();
```



---



### Set(图)



```javascript
// ES6新类型Set
// 成员唯一 加入值不会进行类型转换
var _Set;
// 优先使用原生Set方法
if(typeof Set !== 'undefined' && isNative(Set)){
  _Set = Set;
}
// 造一个Set 只支持基本类型的key
else{
  _Set = (function(){
    // 生成无原型对象
    function Set(){
      this.set = Object.create(null);
    }
    // 判断是否有对应的键
    Set.prototype.has = function has(key){
      return this.set[key] === true;
    };
    // 添加 key:true
    Set.prototype.add = function add(key){
      this.set[key] = true;
    };
    // 清空Set
	Set.prototype.clear = function clear(){
      this.set = Object.create(null);
	};
    return Set;
  }())
}
```



---



### Dep(监视器)



```javascript
// 全局变量
var uid$1 = 0;

// 构造函数 里面只有一个数组 装依赖对象
// 每生成一个Dep对象 属性id+1
var Dep = function Dep(){
  this.id = uid$1++;
  this.subs = [];
}
// 全局监视器对象
// 同一时间只有一个监视器
Dep.target = null;
// 储存监视对象的栈
var targetStack = [];

// 弹入依赖项 并将该项设为当前监视值
function pushTarget(_target){
  if(Dep.target){
    targetStack.push(Dep.target);
  }
  Dep.target = _target;
}

// 弹出一个依赖项 并设监视值
function popTarget(){
  Dep.target = targetStack.pop();
}
```



#### addSub



```javascript
Dep.prototype.addSub = function addSub(sub){
  // 弹入元素
  this.subs.push(sub);
};
```



#### removeSub



```javascript
Dep.prototype.removeSub = function removeSub(sub){
  // 删除数组中对应元素
  remove$1(this.subs,sub);
}
```



#### depend



```javascript
Dep.prototype.depend = function depend(sub){
  // 如果有依赖对象 添加依赖
  if(Dep.target){
    Dep.target.addDep(this);
  }
};
```



#### dependArray



```javascript
function dependArray(value) {
  for (var e = (void 0), i = 0, l = value.length; i < l; i++) {
    e = value[i];
    e && e.__ob__ && e.__ob__.dep.depend();
    if (Array.isArray(e)) {
      dependArray(e);
    }
  }
}
```



#### notify



```javascript
// 通知所有依赖项
Dep.prototype.notify = function notify(){
  var subs = this.subs.slice();
  for(var i = 0, l = subs.length; i < l; i++){
    subs[i].update();
  }
};
```



---



### def



```javascript
// 对象 键 值 是否可枚举
function def(obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  });
}
```



---



### arrayProto



```javascript
var arrayProto = Array.prototype;
// 构建一个以数组为原型的对象 即arrayMethods.__proto__ => Array.prototype
// 保留对象属性 并拥有数组的方法
// 强 无敌
var arrayMethods = Object.create(arrayProto);
['push','pop','shift','unshift','splice','sort','reverse'].forEach(function(method){
  // 缓存原生方法 比如function push(){[native code]}
  var original = arrayProto[method];
  // 调用Object.defineProperty定义对象 可枚举默认为false
  // Object.defineProperty(obj,key,{value:val})
  // 函数def() => arrayMethods = {method:fn(){}}
  def(arrayMethods,method,function mutator(){
    // 先将参数列表转换成数组
    var arguments$1 = arguments;
    var i = arguments.length;
    var args = new Array(i);
    while(i--){
      args[i] = arguments$1[i];
    }
    
    //调用数组原生方法并获得结果
    var result = original.apply(this,args);
    // this指向
    var ob = this.__ob__;
    var inserted;
    // 弹入一个元素时 标记为插入元素
    switch(method){
      case 'push':
      case 'unshift':
        inserted = args;
        break;
      case 'splice':
        // splice(index,del_num,...,insert_data) 第三个参数开始才是插入的值
        inserted = args.slice(2);
        break;
    }
    // 如果有插入元素 对每个元素调用observe()方法
    if(inserted){
      ob.observeArray(inserted);
    }
    // 广播变化
    ob.dep.notify();
    return result;
  });
});
```



---



### Observer(观察者)



```javascript
var arrayKeys = Object.getOwnPropertyNames(arrayMethods);

// 组件单向数据流
// 默认设为false
var observerState = {
  shouldConvert: true,
  isSettingProps: false
};

// 观测者构造函数
// value为data对应的对象 假设为{msg:'Hello World'}
var Observer = function Observer(value){
  this.value = value;
  // 生成一个新监视器
  this.dep = new Dep();
  this.vmCount = 0;
  // 此处value为传进来的参数
  // 给value对象绑定属性__ob__绑定自己...
  // value:{__ob__:this}不可枚举
  def(value,'__ob__',this);
  // 总结一下目前的this
  // this={dep:{id:0,subs:[]},
  //	  value:{__ob__:this,msg:'Hello World'},
  //	  vmCount:0}
  
  if(Array.isArray(value)){
    // hasProto = '__proto__' in {} 能否使用__proto__
    var augment = hasProto ? 
        protoAugment : copyAugment;
    // 可用__proto__就构造原型链 value => {} =>Array.prototype
    // arrayKeys = Object.getOwnPropertyNames(arrayMethods) 
    // 获取数组方法名 ['push','pop'...]
    // 不可用__proto__进行方法强绑def(target, key, src[key])
    augment(value,arrayMethods,arrayKeys);
    // 遍历数组元素调用observe()
    this.observeArray(value);
  }
  // 对象
  else{
    // 调用自定义的defineReactive$$1(obj,key,value)动态添加属性
    // value:{msg:'Hello World'}
    this.walk(value);
  }
};
```



#### walk



```javascript
Observer.prototype.walk = function walk(obj){
  // Object.keys()返回一个由键组成的数组
  var keys = Object.keys(obj);
  for(var i = 0; i < keys.length; i++){
    // 挨个对值进行动态绑定
    defineReactive$$1(obj, key[i], obj[keys[i]]);
  }
};
```



#### observeArray



```javascript
// 监视一个数组
Observer.prototype.observeArray = function observerArray(items){
  for(var i = 0, l = items.length; i < l; i++){
    observe(items[i]);
  }
};
```



#### protoAugment / copyAugment



```javascript
// 通过__proto__扩展原型链
function protoAugment(target, src){
  target.__proto__ = src;
}

// 通过别的对象扩展方法
function copyAugment(target, src, keys){
  for(var i = 0; l = keys.length; i < l; i++){
    var key = keys[i];
    // target:{key:src[key]}
    def(target, key, src[key]);
  }
}
```



#### observe(核心方法)



```javascript
function observe(value, asRootData){
  // 只接受对象
  if(!isObject(value)){
    return ;
  }
  var ob;
  if(hasOwn(value, '__ob__') && value.__ob__ instanceof Observer){
    ob = value.__ob__;
  }
  // 如果没有__ob__属性 new一个 
  else if(
    observerState.shouldConvert && !isServerRendering() && 
    (Array.isArray(value) || isPlainObject(value)) && 
    Object.isExtensible(value) && !value._isVue){
      ob = new Observer(value);
    }
  if(asRootData && ob){
    ob.vmCount++;
  }
  return ob;
}
```



---



### defineReactive$1



```javascript
// 动态添加属性
// 继续假设obj={msg:'Hello World'} key='msg' val='Hello World'
function defineReactive$$1(obj,key,val,customSetter){
  // 这里又生成一个新的 {id:1, sub:[]}
  var dep = new Dep();
  // Object.getOwnPropertyDescriptor(obj,key)方法以对象形式返回对象的四个特殊属性
  var property = Object.getOwnPropertyDescriptor(obj,key);
  // 不存在或者不可更改 返回
  if(property && property.configurable === false){
    return ;
  }
  // 获取值的get/set
  var getter = property && property.get;
  var setter = property && property.set;
  
  // val = 'Hello World'
  // 只接受对象 对子对象进行递归
  var childOb = observe(val); 
  Object.defineProperty(obj,key,{
    enumerable: true,
    configurable: true,
    get: function reactiveGetter(){
      var value = getter ? getter.call(obj) : val;
      if(Dep.target){
        // 添加依赖
        dep.depend();
        // 子对象
        if(childOb){
          childOb.dep.depend();
        }
        // 数组值
        if(Array.isArray(value)){
          dependArray(value);
        }
      }
      return value;
    },
    set: function reactiveSetter(newVal){
      var value = getter ? getter.call(obj) : val;
      if(newVal === value || (newVal !== newVal && value !== value)){
        return ;
      }
      if('development' !== 'production' && customSetter){
        customSetter();
      }
      if(setter){
        setter.call(obj,newVal);
      }
      else{
        val = newVal;
      }
      childOb = observe(newVal);
      dep.notify();
    }    
  });
}
```



---



### set$1



```javascript
// 为一个对象设置一个属性 属性不存在时添加并触发监视器
function set$1(obj,key,val){
  // 数组
  if(Array.isArray(obj)){
    // 1.如果obj比较长 替换key位置的value
    // 2.如果key比length长 
    obj.length = Math.max(obj.length,key);
    obj.splice(key,1,val);
    return val;
  }
  if(hasOwn(obj,key)){
    obj[key] = val;
    return ;
  }
  var ob = obj.__ob__;
  if(obj._isVue || (ob && ob.vmCount)){
    'development' !== 'production' && warn('Avoid adding reactive properties to a Vue instance or its root $data at runtime - declare it upfront in the data option.');
    return ;
  }
  if(!ob){
    obj[key] = val;
    return;
  }
  defineReactive$$1(ob.value,key,val);
  ob.dep.notify();
  return val;
}
```



---



### del



```javascript
function del(obj,key){
  var ob = obj.__ob__;
  if(obj._isVue || (ob && ob.vmCount)){
    'development' !== 'prodution' && warn('Avoid deleting properties on a Vue instance or its root $data - just set it to null.');
    return ;
  }
  if(!hasOwn(obj,key)){
    return;
  }
  delete obj[key];
  if(!ob){
    return;
  }
  ob.dep.notify();
}
```









# init





## proxy

```javascript
var sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
};

// Proxy是ES6的代理
// 小写的不是关键字！
function proxy(target, sourceKey, key){
  sharedPropertyDefinition.get = function proxyGetter(){
    return this[sourceKey][key];
  };
  sharedPropertyDefinition.set = function proxySetter(val){
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```



---



## initState

```javascript
// 所有状态初始化
function initState(vm){
  vm._watchers = [];
  var opts = vm.$options;
  if(opts.props){ initProps(vm, opts.props); }
  if(opts.methods){ initMethods(vm, opts.methods); }
  if(opts.data){
    initData(vm);
  } else{
    observe(vm._data = {}, true);
  }
  if(opts.computed){ initComputed(vm, opts.computed); }
  if(opts.watch){ initWatch(vm, opts.watch); }
}
```



---



### initProps

```javascript
// 保留字
var isReservedProp = {key: 1, ref: 1, slot: 1};

// props初始化
function initProps(vm, propsOptions){
  var propsData = vm.$options.propsData || {};
  var props = vm._props = {};
  // 缓存prop keys
  var keys = vm.$options._propKeys = [];
  var isRoot = !vm.$parent;
  // 根实例props转换
  observerState.shouldConvert = isRoot;
  var loop = function(key){
    keys.push(key);
    var value = validateProp(key, propsOptions, propData, vm);
    
    // ignore
    if(isReservedProp[key]){
      warn(
        ('"' + key + '" is a reserved attribute and cannot be used as component prop.'), 
        vm
      );
    }
    defineReactive$$1(props, key, value, function(){
      if(vm.$parent && !observerState.isSettingProps){
        warn(
          'Avoid mutating a prop directly since the value will be' + 
          'overwritten whenever the parent component re-renders.' + 
          "Instead, use a data or computed property based on the prop's " + 
          'value. Prop being mutated: "' + key +'"', vm
        );
      }
    });
    
    // 
    if(!(key in vm)){
      proxy(vm, '_props', key);
    }
  };
  for(var key in propsOptions){
    loop(key);
  }
  observerState.shouldConvert = true;
}
```



---



### initData

```javascript
function initData(vm){
  var data = vm.$options.data;
  data = vm._data = typeof data === 'function' ? 
    getData(data, vm) : data || {};
  // data必须返回一个对象
  if(!isPlainObject(data)){
    data = {};
    'development' !== 'production' && warn(
      'data functions should return an object:\n' + 
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    );
  }
  
  // 代理data属性
  var keys = Object.keys(data);
  var props = vm.$options.props;
  var i = keys.length;
  while(i--){
    if(props && hasOwn(props, keys[i])){
      'development' !== 'production' && warn(
        'The data property "' + (keys[i]) + '" is already declared as a prop. ' + 
        'Use prop default value instead.',
        vm
      );
    } else if(!isReserved(keys[i])){
      proxy(vm, '_data', keys[i]);
    }
  }
  // 观测
  observe(data, true);
}
```



#### getData

```javascript
function getData(data, vm){
  try{
    return data.call(vm);
  } catch(e){
    handleError(e, vm, 'data()');
    return {};
  }
}
```



---



### initComputed

```javascript
var computedWatcherOptions = {lazy: true };

function initComputed(vm, computed){
  var watcher = vm._computedWatchers = Object.create(null);
  
  for(var key in computed){
    var userDef = computed[key];
    var getter = typeof userDef === 'function' ? userDef : userDef.get;
    if(getter === undefined){
      warn(
        ('No getter function has been defined for computed property "' + key + '".'),
        vm
      );
      getter = noop;
    }
    // 给computed属性创建内部watcher
    watchers[key] = new Watcher(vm, getter, noop, computedWatcherOptions);

    // 组件部分已经被定义 仅需要computed属性的实例化
    if(!(key in vm)){
      defineComputed(vm, key, userDef);
    }
  }
}
```



#### defineComputed

```javascript
// computed属性实例化
function defineComputed(target, key, userDef){
  if(typeof userDef === 'function'){
    // computed为被动更改 不需要set方法
    sharedPropertyDefinition.get = createComputedGetter(key);
    sharedPropertyDefinition.set = noop;
  } else{
    // userDef.get不存在 => noop
    // userDef.get存在 => userDef.cache不存在 => userDef.get
    // userDef.get存在 => userDef.cache存在 => createComputedGetter(key)
    sharedPropertyDefinition.get = userDef.get ? 
      userDef.cache !== false ? 
      createComputedGetter(key) : 
      userDef.get : noop;
    // userDef.set不存在 => noop
    sharedPropertyDefinition.set = userDef.set ? 
      userDef.set : noop;
  }
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```



#### createComputedGetter

```javascript
function createComputedGetter(key){
  return function computedGetter(){
    var watcher = this._computedWatchers && this._computedWatchers[key];
    if(watcher){
      if(watcher.dirty){
        watcher.evaluate();
      }
      if(Dep.target){
        watcher.depend();
      }
      return watcher.value;
    }
  }
}
```



---



### initMethods

```javascript
function initMethods(vm, methods){
  var props = vm.$options.props;
  for(var key in methods){
    vm[key] = method[key] == null ? noop : bind(methods[key], vm);
    // 
    if(methods[key] == null){
      warn(
      	'method "' + key + '" has an undefined value in the component definition. ' + 
        'Did you reference the function correctly?',
        vm
      );
    }
    // 方法已经被定义
    if(props && hasOwn(props, key)){
      warn(
        ('method "' + key + '" has already been defined as a prop'),
        vm
      );
    }
  }
}
```



---



### initWatch

```javascript
function initWatch(vm, watch){
  for(var key in watch){
    var handler = watch[key];
    if(Array.isArray(handler)){
      for(var i = 0; i < handler.length; i++){
        createWatcher(vm, key, handler[i]);
      }
    } else{
      createWatcher(vm, key, handler);
    }
  }
}
```



#### createWatcher

```javascript
function createWatcher(vm, key, handler){
  var options;
  if(isPlainObject(handler)){
    options = handler;
    handler = handler.handler;
  }
  if(typeof handler === 'string'){
    handler = vm[handler];
  }
  vm.$watch(key, handler, options);
}
```



---



### stateMixin

```javascript
// state相关原型方法混入
// 注释说Object.defineProperty直接定义会出问题
function stateMixin(Vue){
  // 定义get方法
  var dataDef = {};
  dataDef.get = function(){ return this._data };
  var propsDef = {};
  propDef.get = function(){ return this._props };
  
  // 调用set会触发警告
  dataDef.set =  function(newData){
    warn(
      'Avoid replacing instance root $data. ' + 
      'Use nested data properties instead.',
      this
    );
  };
  propsDef.set = function(){
    warn('$props is readonly.', this);
  }
  
  // 定义原型方法
  Object.defineProperty(Vue.prototype, '$data', dataDef);
  Object.defineProperty(Vue.prototype, '$props', propsDef);
  
  Vue.prototype.$set = set;
  Vue.prototype.$delete = del;
  
  Vue.prototype.$watch = function(expOrFn, cb, options){
    var vm =  this;
    options = options || {};
    options.user = true;
    var watcher = new Watcher(vm, expOrFn, cb, options);
    if(options.immediate){
      cb.call(vm,watcher.value);
    }
    return function unwatchFn(){
      watcher.teardown();
    }
  }
}
```






















# 初始化







## initProvide

```javascript
function initProvide(vm){
  var provide = vm.$options.provide;
  if(provide){
    vm._provided = typeof provide === 'function' ? 
      provide.call(vm) : provide;
  }
}
```



---



## initInjections

```javascript
function initInjections(vm){
  var inject = vm.$options.inject;
  if(inject){
    var isArray = Array.isArray(inject);
    // inject是数组 => inject
    // inject不是数组 && 支持ES6 => Reflect.ownKeys(inject) => 返回inject自身属性(包括symbol)
    // inject不是数组 && 不支持ES6 => Object.keys(inject)
    var keys = isArray ? 
        inject : hasSymbol ? 
        Reflect.ownKeys(inject) : Object.keys(inject);
    
    var loop = function(i){
      var key = keys[i];
      var provideKey = isArray ? key : inject[key];
      var source = vm;
      while(source){
        if(source._provided && provideKey in source._provided){
          defineReactive$$1(vm, key, source._provided[provideKey], function(){
            warn(
              'Avoid mutating an injected value directly since the changes will be ' + 
              'overwritten whenever the provided component re-renders. ' + 
              'injection being mutated: "' + key + '"',
              vm
            );
          });
          break;
        }
        source = source.$parent;
      }
    }; // loop
    for(var i = 0; i < keys.length; i++) loop(i);
  }
}
```



---



## initMixin

```javascript
var uid = 0;	// 记录vue实例个数
function initMixin(Vue){
  Vue.prototype._init = function(options){
    var vm = this;
    vm._uid = uid++;
    
    var startTag, endTag;
    if('development' !== 'production' && config.performance && mark){
      startTag = 'vue-pref-init:' + (vm._uid);
      endTag = 'vue-pref-end:' + (vm._uid);
      mark(startTag);
    }
    
    // 表示是vue实例 避免被observe
    vm._isVue = true;
    // 合并参数
    if(options && options._isComponent){
      // 优化内部组件实例化
      initInternalComponent(vm, options);
    } else{
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constuctor),
        options || {},
        vm
      );
    }
    
    // 小分支
    initProxy(vm);
    
    // 属性对自身的引用
    vm._self = vm;
    
    // 初始化
    initLifecycle(vm);
    initEvents(vm);
    initRender(vm);
    callHook(vm, 'beforeCreate');
    initInjections(vm);
    initState(vm);
    initProvide(vm);
    callHook(vm, 'created');
    
    if('development' !== 'production' && config.performance && mark){
      vm._name = formatComponentName(vm, false);
      mark(endTag);
      measure(((vm._name) + 'init'), startTag, endTag);
    }
    
    if(vm.$options.el){
      vm.$mount(vm.$options.el);
    }
  };
}
```



---



## initInternalComponent

```javascript
function initInternalComponent(vm, options){
  // 这个原型对象真是长又长
  var opts = vm.$options = Object.create(vm.constructor.options);
  // 比动态枚举快
  // 应该是指的for-in
  opts.parent = options.parent;
  opts.propsData = options.propsData;
  opts._parentVnode = options._parentVnode;
  opts._parentListeners = options._parentListeners;
  opts._renderChildren = options._renderChildren;
  opts._componentTag = options._componentTag;
  opts._parentElm = options._parentElm;
  opts._refElm = options._refElm;
  if(options.render){
    opts.render = options.render;
    opts.staticRenderFns = options.staticRenderFns;
  }
}
```



---



## resolveConstructorOptions

```javascript
function resolveConstrutorOptions(Ctor){
  var modified;
  var latest = Ctor.options;
  // 密封
  var sealed = Ctor.sealedOptions;
  for(var key in latest){
    if(latest[key] !== sealed[key]){
      if(!modified) { modified = {}; }
      modified[key] = dedupe(latest[key], sealed[key]);
    }
  }
  return modified;
}
```



### dudepe

```javascript
function dudepe(latest, sealed){
  // 比较两者防止lifeCycle钩子不会重复调用
  if(Array.isArray(latest)){
    var res = [];
    sealed = Array.isArray(sealed) ? sealed : [sealed];
    for(var i = 0; i < latest.length; i++){
      if(sealed.indexOf(latest[i]) < 0){
        res.push(latest[i]);
      }
    }
    return res;
  } else{
    return latest;
  }
}
```



---



## vue$3

```javascript
function Vue$3(options){
  if('development' !== 'production' && !(this instanceof Vue$3)){
    warn('Vue is a constructor and should be called with the "new" keyword');
  }
  this._init(options);
}
```



---



## 原型方法混入

```javascript
initMixin(Vue$3);
stateMixin(Vue$3);
eventsMixin(Vue$3);
lifecycleMixin(vue$3);
renderMixin(Vue$3);
```



---



## initUse

```javascript
// 第三方插件
function initUse(Vue){
  Vue.use = function(plugin){
    if(plugin.installed){
      return ;
    }
    // 参数处理
    var args = toArray(arguments, 1);
    args = unshift(this);
    if(typeof plugin.install === 'function'){
      plugin.install.apply(plugin, args);
    } else if(typeof plugin === 'function'){
      plugin.apply(null, args);
    }
    plugin.installed = true;
    return this;
  }
}
```



---



## initMixin$1

```javascript
function initMixin$1(Vue){
  Vue.mixin = function(mixin){
    this.options = mergeOptions(this.options, mixin);
  };
}
```



---



## initExtend

```javascript
function initExtend(Vue){
  Vue.cid = 0;
  var cid = 1;
  
  // 类继承
  Vue.extend = function(extendOptions){
    extendOptions = exntendOptions || {};
    var Super = this;
    var SuperId = Super.cid;
    var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
    if(cachedCtors[SuperId]){
      return cachedCtors[SuperId];
    }
    
    var name = extendOptions.name || Super.options.name;
    // 命名检测
    if(!/^[a-zA-Z][\w-]*$/.test(name)){
      warn(
        'Invalid component name: "' + name + '". Component names' + 
        'can only contain alphanumeric characters and the hyphen, ' + 
        'and must start with a letter.'
      );
    }
    
    var Sub = function VueComponent(options){
      this._init(options);
    };
    Sub.prototype = Object.create(Super.prototype);
    Sub.prototype.constructor = Sub;
    Sub.cid = cid++;
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    );
    Sub['super'] = Super;
  }
}
```
















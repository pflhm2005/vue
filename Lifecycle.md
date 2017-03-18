# Lifecycle



### initLifecycle

```javascript
var activeInstance = null;
// vue对象初始化的第一步
function initLifecycle(vm){
  // 这里的$options是{el:'',data:{}}等等
  var options = vm.$options;
  var parent = options.parent;
  if(parent && !options.abstract){
    while(parent.$options.abstract && parent.$parent){
      parent = parent.$parent;
    }
    parent.$children.push(vm);
  }
  
  vm.$parent = parent;
  vm.$root = parent ? parent.$root : vm;
  
  vm.$children = [];
  vm.$refs = {};;
  
  vm._watcher = null;
  vm._native = null;
  vm._directInactive = false;
  vm._isMounted = false;
  vm._isDestroyed = false;
  vm._isBeingDestroyed = false;
}
```



### lifecycleMixin



```javascript
function lifecycleMixin(Vue){
  // _update
  Vue.prototype._update = function(vnode, hydrating){
    var vm = this;
    if(vm._isMounted){
      callHook(vm, 'beforeUpdate');
    }
    var prevEl = vm.$el;
    var prevVnode = vm._vnode;
    var prevActionInstance = activeInstance;
    activeInstance = vm;
    vm._vnode = vnode;
    
    if(!prevVnode){
      // 初始化渲染
      vm.$el = vm.__patch__(vm.$el,vnode,hydrating,false,vm.$options._parentElm,vm.$options._refElm);
    }
    else{
      vm.$el = vm.__patch__(prevVnode, vnode);
    }
    activeInstance = prevActiveInstance;
    //
    if(prevEl){
      prevEl.__vue__ = null;
    }
    if(vm.$el){
      vm.$el.__vue__ = vm;
    }
    // HOC是什么？？？
    if(vm.$vnode && vm.$parent && (vm.$vnode === vm.$parent._vnode)){
      vm.$parent.$el = vm.$el;
    }
  }
  
  // 强制更新
  // $forceUpdate
  Vue.prototype.$forceUpdate = function(){
    var vm = this;
    if(vm._watcher){
      vm._watcher.update();
    }
  };
  
  // 生命周期的最后阶段 销毁
  // $destroy
  Vue.prototype.$destroy = function(){
    var vm = this;
    if(vm._isBeingDestroyed){
      return
    }
    callHook(vm, 'beforeDestroy');
    vm._isBeingDestroyed = true;
    // 从父元素分离
    var parent = vm.$parent;
    if(parent && !parent._isBeingDestroyed && !vm.$options.abstract){
      remove(parent.$children, vm);
    }
    // 肢解watcher!
    if(vm._watcher){
      vm._watcher.teardown();
    }
    var i = vm._watchers.length;
    while(i--){
      vm._watchers[i].teardown();
    }
    // 
    if(vm._data.__ob__){
      vm._data_.__ob__.vmCount--;
    }
    // 调用最后一个钩子函数
    vm._isDestroyed = true;
    callHook(vm, 'destroyed');
    // 移除所有实例监听者
    vm.$off();
    // 移除vue引用
    if(vm.$el){
      vm.$el.__vue__ = null;
    }
  }
}
```



### mountComponent



```javascript
// 挂载组件
function mountComponent(vm, el, hydrating){
  vm.$el = el;
  if(!vm.$options.render){
    vm.$options.render = createEmptyVNode;
    if((vm.$options.template && vm.$options.template.charAt(0) !== '#') || vm.$options.el || el){
      warn('You are using the runtime-only build of Vue where the template ' + 
          'compiler is not available. Either pre-compiler the template into ' + 
          'render functions, or use the compiler-included build.', vm);
    }
    else{
      warn('Failed to mount component: template or render function not defined.', vm);
    }
  }
  callHook(vm, 'beforeMount');
  
  var updateComponent;
  if('development' !== 'production' && config.performance && perf){
    updateComponent = function(){
      var name = vm._name;
      var startTag = 'start' + name;
      var endTag = 'end' + name;
      perf.mark(startTag);
      var vnode = vm._render();
      perf.mark(endTag);
      perf.measure((name + ' render'), startTag, endTag);
      perf.mark(startTag);
      vm._update(vnode, hydrating);
      perf.mark(endTag);
      perf.measure((name + ' patch'), startTag, endTag);
    };
  }
  else{
    updateComponent = function(){
      vm._update(vm._render(), hydrating);
    };
  }
  vm._watcher = new Watcher(vm, updateComponent, noop);
  hydrating = false;
  
  //
  if(vm.$vnode == null){
    vm._isMounted = true;
    callHook(vm, 'mounted');
  }
  return vm;
}
```



### updateChildComponent



```javascript
function updateChildComponent(vm, propsData, listeners, parentVnode, renderChildren){
  // 检测是否有slot标签
  var hasChildren = !!(
  renderChildren || 
  vm.$options._renderChildren || 
  parentVnode.data.scopedSlots || 
  vm.$scopedSolts !== emptyObject
  );
  
  vm.$options._parentVnodes = parentVnode;
  vm.$vnode = parentVnode;
  if(vm._vnode){
    vm._vnode.parent = parentVnode;
  }
  vm.$options._renderChildren = renderChildren;
  // 更新props
  if(propsData && vm.$options.props){
    observerState.shouldConvert = false;
    observerState.isSettingProps = true;
    var props = vm._props;
    var propKeys = vm.$options._propKeys || [];
    for(var i = 0; i < propKeys.length; i++){
      var key = propKeys[i];
      props[key] = validateProp(key, vm.$options.props, propsData, vm);
    }
    observerState.shouldConvert = true;
    observerState.isSettingProps = false;
    // 保存一份原始副本
    vm.$options.propsData = propsData;
  }
  if(listeners){
    var oldListeners = vm.$options._parentListeners;
    vm.$options._parentListeners = listeners;
    updateComponentListeners(vm, listeners, oldListeners);
  }
  //
  if(hasChildren){
    vm.$slots = resolveSlots(renderChildren, parentVnode.context);
    vm.$forceUpdate();
  }
}
```












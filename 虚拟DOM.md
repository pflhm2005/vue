# 虚拟DOM



## VNode



### 构造函数



```javascript
var VNode = function(tag,data,children,text,elm,context,componentOptions){
  this.tag = tag;
  this.data = data;
  this.children = children;
  this.text = text;
  this.elm = elm;
  this.ns = undefined;
  this.context = context;
  this.functionalContext = undefined;
  this.key = data && data.key;
  this.componentOptions = componentOptions;
  this.componentInstance = undefined;
  this.parent = undefined;
  this.raw = false;
  this.isStatic = false;
  this.isRootInsert = true;
  this.isComment = false;
  this.isCloned = false;
  this.isOnce = false;
};
```



### prototypeAccessors



```javascript
var prototypeAccessors = {child: {}};
//注释讲这里是为了向下兼容
prototypeAccessors.child.get = function(){
  return this.componentInstance;
};
Object.defineProperties(VNode.prototype, prototypeAccessors);
```



### createVNode



#### createEmptyVNode



```javascript
var createEmptyVNode = function(){
  var node = new VNode();
  node.text = '';
  node.isComment = true;
  return node;
}
```



#### createTextVNode



```javascript
//仅传text参数
function createTextVNode(val){
  return new VNode(undefined, undefined, undefined, String(val));
}
```



#### cloneVNode



```javascript
function cloneVNode(vnode){
  var cloned = new VNode(
    vnode.tag,
    vnode.data,
    vnode.children,
    vnode.text,
    vnode,elm,
    vnode.context,
    vnode.componentOptions
  );
  cloned.ns = vnode.ns;
  cloned.isStatic = vnode.isStatic;
  cloned.key = vnode.key;
  cloned.isCloned = true;
  return cloned;
}

//批量clone虚拟节点 包装在数组中
function cloneVNodes(vnodes){
  var len = vnodes.length;
  var res = new Array(len);
  for(var i = 0; i < len; i++){
    res[i] = cloneVNode[vnodes[i]];
  }
  return res;
}
```



### normalizeEvent



```javascript
var normalizeEvent = cached(function(name){
  //检查前缀
  var once$$1 = name.charAt(0) === '~';
  //去掉~
  name = once$$1 ? name.slice(1) : name;
  var capture = name.charAt(0) === '!';
  name = capture ? name.slice(1) : name;
  return {
    name: name,
    once: once$$1,
    capture: capture
  }
})
```



### createFnInvoker



```javascript
//函数调用器
function createFnInvoker(fns){
  function invoker(){
    //内部参数
    var arguments$1 = arguments;
    //外部参数
    var fns = invoker.fns;
    //外部参数为函数数组时 遍历依次执行 参数为内部参数
    if(Array.isArray(fns)){
      for(var i = 0; i < fns.length: i++){
        fns[i].apply(null, arguments$1);
      }
    } else {
      //外部参数为单一函数
      return fns.apply(null, arguments)
    }
  }
  //参数保存为函数属性
  invoker.fns = fns;
  return invoker;
}
```



### updateListeners



```javascript
function updateListeners(on,oldOn,add,remove$$1,vm){
  var name, cur, old, event;
  for(name in on){
    cur = on[name];
    old = oldOn[name];
    event = normalizeEvent(name);
    if(!cur){
      'development' !== 'production' && 
        warn('Invalid handler for event "' + event.name + '": got ' + String(cur), vm);
    } else if (!old) {
      if(!cur.fns){
        cur = on[name] = createFnInvoker(cur);
      }
      add(event.name, cur, event.once, event.capture);
    } else if (cur !== old) {
      old.fns = cur;
      on[name] = old;
    }
  }
  for(name in oldOn){
    if(!on[name]){
      event = normalizeEvent(name);
      remove$$1(event.name, oldOn[name], event.capture);
    }
  }
}
```



### mergeVNodeHook



```javascript
function mergeVNodeHook(def, hookKey, hook){
  var invoker;
  var oldHook = def[hookKey];
  
  function wrappedHook(){
    hook.apply(this, arguments);
    //移除钩子保证只被调用一次 防止内存泄漏
    remove(invoker.fns, wrappedHook);
  }
  if(!oldHook){
    //没有钩子
    FnInvoker([wrappedHook]);
  } else {
    if (oldHook.fns && oldHook.merged){
      invoker = oldHook;
      invoker.fns.push(wrappedHook);
    } else {
      //空钩子
      invoker = createFnInvoker([oldHook, wrappedHook]);
    }
  }
  invoker.merged = true;
  def[hookKey] = invoker;
}
```



### NormalizeChildren



#### simpleNormalizeChildren



```javascript
//尤大是这样说的
//一般情况下都是纯粹HTML标签 调用这个就行了
function simpleNormalizeChildren(children){
  for(var i = 0; i < children.length; i++){
    if(Array.isArray(children[i])){
      return Array.prototype.concat.apply([], children);
    }
  }
  return children;
}
```



#### normalizeChildren



```javascript
function normalizeChildren(children){
  //简单解释下就是 :
  //[*] => createTextVNode(*)
  //基本类型 => [*]
  //数组 => 递归 => [*]
  //非数组非基本类型 => undefined
  return isPrimitive(children) ? [createTextVNode(children)] : 
  Array.isArray(children) ? normalizeArrayChildren(children) : undefined;
}
```



#### normalizeArrayChildren



```javascript
//这TM
function normalizeArrayChildren(children, nestedIndex){
  var res = [];
  var i, c, last;
  for(i = 0; i < children.length; i++){
    c = children[i];
    if( c == null || typeof c === 'boolean'){continue}
    last = res[res.length-1];
    //嵌套
    if(Array.isArray(c)){
      res.push.apply(res, normalizeArrayChildren(c, ((nestedIndex || '') + '_' + i)));
    } else if(isPrimitive(c)){
      if(last && last.text){
        last.text += String(c);
      } else if(c !== ''){
        //转换基本数据类型
        res.push(createTextVNode(c));
      }
    } else {
      if(c.text && last && last.text){
        res[res.length - 1] = createTextVNode(last.text + c.text);
      } else{
        //
        if(c.tag && c.key == null && nestedIndex != null){
          c.key = '__vlist' + nestedIndex + '_' + i + '__';
        }
        res.push(c);
      }
    }
  }
  return res;
}
```



#### getFirstComponentChild



```javascript
function getFirstComponentChild(children){
  return children && children.filter(function(c) {return c && c.componentOptions;})[0]
}
```



### Event



#### initEvents



```javascript
function initEvents(vm){
  vm._events = Object.create(null);
  vm._hasHookEvent = false;
  //初始化父元素相关事件
  var listeners = vm.$options._parentListeners;
  if(listeners){
    updateComponentListeners(vm, listeners);
  }
}

var target;
//添加
function add(event, fn, once$$1){
  if(once$$!){
    target.$once(event, fn);
  } else{
    target.$on(event, fn);
  }
}
//删除
function remove$1(event, fn){
  target.$off(event, fn);
}

function updateComponentListeners(vm, listeners, oldListeners){
  target = vm;
  updateListeners(listeners, oldListeners || {}, add, remove$1, vm);
}
```



#### eventsMixin



```javascript
function eventsMixin(Vue){
  var hookRE = /^hook:/;
  //绑定事件
  Vue.prototype.$on = function(event, fn){
    var this$1 = this;
    var vm = this;
    if(Array.isArray(event)){
      for(var i = 0, l = event.length; i < l; i++){
        this$1.$on(event[i], fn);
      }
    } else{
      (vm._events[event] || (vm._events[event] = [])).push(fn);
      //
      if(hookRE.test(event)){
        vm._hasHookEvent = true;
      }
    }
    return vm;
  };
  //绑定一次事件
  Vue.prototype.$once = function(event, fn){
    var vm = this;
    function on(){
      vm.$off(event, on);
      fn.apply(vm, arguments);
    }
    on.fn = fn;
    vm.$on(event, on);
    return vm;
  };
  //解绑事件
  Vue.prototype.$off = function(event, fn){
    var this$1 = this;
    var vm = this;
    //all
    if(!arguments.length){
      vm._events = Object.create(null);
      return vm;
    }
    //事件数组
    if(Array.isArray(event)){
      //好恶心的命名
      for(var i$1 = 0, l = event.length; i$1 < l; i$1++){
        this$1.$off(event[i$1], fn);
      }
      return vm;
    }
    //特定事件
    var cbs = vm._events[event];
    if(!cbs){
      return vm;
    }
    if(arguments.length === 1){
      vm._events[event] = null;
      return vm;
    }
    //特定处理器
    var cb;
    var i = cbs.length;
    while(i--){
      cb = cbs[i];
      if(cb === fn || cb.fn){
        cbs.splice(i, 1);
        break;
      }
    }
    return vm
  };
  //触发事件
  Vue.prototype.$emit = function(event){
    var vm = this;
    var cbs = vm._events[event];
    if(cbs){
      cbs = cbs.length > 1 ? toArray(cbs) : cbs;
      var args = toArray(arguments, 1);
      for(var i = 0, l = cbs.length; i < l; i++){
        cbs[i].apply(vm, args);
      }
    }
    return vm;
  };
}
```



### Slot



#### resolveSlots



```javascript
function resolveSlots(children,context){
  var slots = {}
  if(!children){
    return slots;
  }
  var defaultSlot = [];
  var name, child;
  for(var i = 0, l = children.length; i < l; i++){
    child = children[i];
    //
    if((child.context === context || child.functionalContext === context) && 
       child.data && (name = child.data.slot)){
      var slot = (slots[name] || (slots[name] = []));
      if(child.tag === 'template'){
        slot.push.apply(slot, child.children);
      } else{
        slot.psuh(child);
      }
    } else{
      defaultSlot.push(child);
    }
  }
  //忽略空格
  if(!defaultSlot.every(isWhitespace)){
    slots.default = defaultSlot;
  }
  return slots;
}
```



#### isWhitespace



```javascript
function isWhitespace(node){
  return node.isComment || node.text === ' ';
}
```



#### resolveScopedSlots



```javascript
function resolveScopedSlots(fns){
  var res = {};
  for(var i = 0; i < fns.length; i++){
    res[fns[i][0]] = fns[i][1];
  }
  return res;
}
```






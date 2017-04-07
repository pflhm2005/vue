# 内置指令



## createElement



### createElement

```javascript
var SIMPLE_NORMALIZE = 1;
var ALWAYS_NORMALIZE = 2;

// 包装函数提供更弹性的接口
// 这函数就是用来做参数修正
function createElement(context, tag, data, children, normalizationType, alwaysNormalize){  
  // data是数组或基本数据类型时
  // 3,4,5参数往后推 undefined => data, data => children, children => normalizationType 
  if(Array.isArray(data) || isPrimitive(data)){
    normalizationType = children;
    children = data;
    data = undefined;
  }
  // 如果提供最后一个参数 normalizationType = 2
  if(alwaysNormalize) { normalizationType = ALWAYS_NORMALIZE; }
  return _createElement(context, tag, data, children, normalizationType);
}
```



### _createElement

```javascript
function _createElement(context, tag, data, children, normalizationType){
  // data是响应式数据
  if(data && data.__ob__){
    'development' !== 'production' && warn(
      'Avoid using observed data object as vnode data: ' + (JSON.stringify(data)) + '\n' + 
      'Always create fresh vnode data objects in each render!',
      context
    );
    return createEmptyVNode();
  }
  // 没有tag参数
  if(!tag){
    return createEmptyVNode();
  }
  // children是数组且第一个元素是函数
  if(Array.isArray(children) && typeof children[0] === 'function'){
    data = data || {};
    data.scopedSlots = { default: children[0] };
    // 清空
    children.length = 0;
  }
  // 根据参数调用对应的函数
  if(normalizationType === ALWAYS_NORMALIZE){
    children = normalizeChildren(children);
  } else if(normalizationType === SIMPLE_NORMALIZE){
    children = simpleNormalizeChildren(children);
  }
  
  var vnode, ns;
  if(typeof tag === 'string'){
    var Ctor;
    ns = config.getTagNamespace(tag);
    if(config.isReservedTag){
      // 内置元素
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      );
    } else if((Ctor = resolveAsset(context.$options, 'component', tag))){
      // 组件
      vnode = createComponent(Ctor, data, context, children, tag);
    } else{
      // 其他未知元素
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      );
    }
  } else{
    vnode = createComponent(tag, data, context, children);
  }
  if(vnode){
    if(ns){ applyNS(vnode, ns); }
    return vnode;
  } else{
    return createEmptyVNode();
  }
}
```



### applyNS

```javascript
function applyNS(vnode, ns){
  vode.ns = ns;
  if(vnode.tag === 'foreignObject'){
    return ;
  }
  if(vnode.children){
    for(var i = 0; l = vnode.children.length; i < l; i++){
      var child = vnode.children[i];
      if(child.tag && !child.ns){
        applyNS(child, ns);
      }
    }
  }
}
```



---



## v-model



### transformModel

```javascript
// 分别将v-model信息转换成prop与event
function transformModel(options, data){
  // 默认 prop = value
  var prop = (options.model && options.model.prop) || 'value';
  // 默认 event = input
  var event = (options.model && options.model.event) || 'input';
  (data.props || (data.props = {}))[prop] = data.model.value;
  var on = data.on || (data.on = {});
  if(on[event]){
    on[event] = [data.model.callback].concat(on[event]);
  } else{
    on[event] = data.model.callback;
  }
}
```



---



## v-for



### renderList

```javascript
function renderList(val, render){
  var ret, i, l, keys ,key;
  // 数组或字符串
  if(Array.isArray(val) || typeof val === 'string'){
    ret = new Array(val.length);
    for(var i = 0, l = val.length; i < l; i++){
      ret[i] = render(val[i], i);
    }
  }
  // 数字
  else if(typeof val === 'number'){
    ret = new Array(val);
    for(var i = 0; i < val; i++){
      ret[i] = render(i + 1, i);
    }
  }
  // 对象
  else if(isObject(val)){
    keys = Object.keys(val);
    ret = new Array(keys.length);
    for(var i = 0, l = keys.length; i < l; i++){
      key = keys[i];
      ret[i] = render(val[key], key, i);
    }
  }
  return ret;
}
```



---



## <slot>



### renderSlot

```javascript
function renderSlot(name, fallback, props, bindObject){
  var scopedSlotFn = this.$scopedSlots[name];
  // scoped属性的slot
  if(scopedSlotFn){
    props = props || {};
    if(bindObject){
      extend(props, bindObject);
    }
    return scopedSlotFn(props) || fallback;
  } else{
    var slotNodes = this.$slots[name];
    // 警告重复slot使用
    if(slotNodes && 'development' !== 'production'){
      slotNodes._rendered && warn(
        'Duplicate presence of slot "' + name + '" found in the same render tree' + 
        '- this will likely cause render errors.',
        this
      );
      slotNodes._rendered = true;
    }
    return slotNodes || fallback;
  }
}
```



---



## filters



### resolveFilter

```javascript
function resolveFilter(id){
  return resolveAsset(this.$options, 'filters', id, true) || identity;
}
```



---



## keyCodes



### checkKeyCodes

```javascript
// 判断是否内置键盘指令别名
function checkKeyCodes(eventKeyCode, key, builtInAlias){
  var keyCodes = config.keyCodes[key] || builtInAlias;
  if(Array.isArray(keyCodes)){
    return keyCodes.indexOf(eventKeyCode) === -1;
  } else{
    return keyCodes !== eventKeyCode;
  }
}
```



---



## v-bind



### bindObjectProps

```javascript
function bindObjectProps(data, tag, value, asProp){
  if(value){
    if(!isObject(value)){
      'development' !== 'production' && warn(
        'v-bind without argument expects an Object or Array value',
        this
      );
    } else{
      // 将数组转换成对象统一处理
      if(Array.isArray(value)){
        value = toObject(value);
      }
      var hash;
      for(var key in value){
        if(key === 'class' || key === 'style'){
          hash = data;
        } else{
          var type = data.attrs && data.attrs.type;
          hash = asProp || config.mustUseProp(tag, type, key) ? 
            data.domProps || (data.domProps = {}) : 
            data.attrs || (data.attrs = {});
        }
        // value的key与hash同步
        if(!(key in hash)){
          hash[key] = value[key];
        }
      }
    }
  }
  return data;
}
```



----



## static-trees



### renderStatic

```javascript
function renderStatic(index, isInFor){
  var tree = this._staticTrees[index];
  // 非v-for情况下重复利用静态树
  if(tree && !isInFor){
    return Array.isArray(tree) ? 
      cloneVNodes(tree) : cloneVNode(tree);
  }
  // 否则渲染一个新树
  tree = this._staticTrees[index] = 
    this.$options.staticRenderFns[index].call(this._renderProxy);
  markStatic(tree, ('__static__' + index), false);
  return tree;
}
```



---



## v-once



### markOnce

```javascript
function markOnce(tree, index, key){
  // 有key markStatic(tree,__once__ + index + _ + key,true)
  // 无key markStatic(tree,__once__ + index,true)
  markStatic(tree, ('__once__' + index + (key ? ('_' + key) : '')), true);
  return tree;
}
```



---



## markStatic

```javascript
function markStatic(tree, key, isOnce){
  if(Array.isArray(tree)){
    for(var i = 0; i < tree.length; i++){
      if(tree[i] && typeof tree[i] !== 'string'){
        markStaticNode(tree[i], (key + '_' + i), isOnce);
      }
    } 
  } else{
    markStaticNode(tree, key, isOnce);
  }
}
```



## markStaticNode

```javascript
// 添加属性的方法
function markStaticNode(node, key, isOnce){
  node.isStatic = true;
  node.key = key;
  node.isOnce = isOnce;
}
```



---





## initRender

```javascript
function initRender(vm){
  vm.$vnode = null;
  vm._vnode = null;
  vm._staticTrees = null;
  var parentVnode = vm.$options._parentVnode;
  var renderContext = parentVnode && parentVnode.context;
  vm.$slots = resolveSlots(vm.$options._renderChildren, renderContext);
  vm.$scopedSlots = emptyObject;
  // 
  vm._c = function(a, b, c, d) { return createElement(vm, a, b, c, d, false); };
  // 
  vm.$createElement = function(a, b, c, d) { return createElement(vm, a, b, c, d, true); };
}
```



## renderMixin

```javascript
function renderMixin(Vue){
  // 几百年前定义的方法在这里混入了
  Vue.prototype.$nectTick = function(fn){
    return nextTick(fn, this);
  }
  
  // 
  Vue.prototype._render = function(){
    var vm = this;
    var ref = vm.$options;
    var render =  ref.render;
    var staticRenderFns = ref.staticRenderFns;
    var _parentVnode = ref._parentVnode;
    
    if(vm._isMounted){
      // 克隆slot节点便于重复渲染
      for(var key in vm.$slots){
        vm.$slots[key] = cloneVNodes(vm.$slots[key]);
      }
    }
    
    vm.$scopedSlots = (_parentVnode && _parentVnode.data.scopedSlots) || emptyObject;
    
    if(staticRenderFns && !vm._staticTrees){
      vm._staticTrees = [];
    }
    // 设置父节点
    vm.$vnode = _parentVnode;
    
    var vnode;
    try{
      vnode = render.call(vm._renderProxy, vm.$createElement);
    } catch(e){
      handleError(e, vm, 'render function');
      // 返回错误渲染结果
      vnode = vm.$options.renderError ? 
        vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e) : 
        vm._vnode;
    }
    
    // 当返回错误的渲染结果
    if(!(vnode instanceof VNode)){
      if('development' !== 'production' && Array.isArray(vnode)){
        warn(
          'Multiple root nodes returned from render function. Render function' + 
          'should return a single root node.',
          vm
        );
      }
      vnode = createEmptyVNode();
    }
    // 设置parent
    vnode.parent = _parentVnode;
    return vnode;
  };
  
  // 内部方法缩写
  
  // v-once
  Vue.prototype._o = markOnce;
  // parseFloat
  Vue.prototype._n = toNumber;
  // String()或JSON.stringify()
  Vue.prototype._s = _toString;
  // v-for
  Vue.prototype._l = renderList;
  // slot
  Vue.prototype._t = renderSlot;
  // _toString后判断是否全等
  Vue.prototype._q = looseEqual;
  // looseEqual判断索引是否存在
  Vue.prototype._i = looseIndexOf;
  // 静态树
  Vue.prototype._m = renderStatic;
  // filters
  Vue.prototype._f = resolveFilter;
  // keyCode
  Vue.prototype._k = checkKeyCodes;
  // v-bind
  Vue.prototype._b = bindObjectProps;
  // 纯文本虚拟节点
  Vue.prototype._v = createTextVNode;
  // 空虚拟节点
  Vue.prototype._e = createEmptyVNode;
  Vue.prototype._u = resolveScopedSlots;
}
```




# component&&props





## component



### componentVNodeHooks

```javascript
var componentVNodeHooks = {
  // init
  init: function init(vnode, hydrating, parentElm, refElm){
    if(!vnode.componentInstance || vnode.componentInstance._isDestroyed){
      // initLifecycle之前
      // var activeInstance = null;
      // lifecycleMixin方法中
      // var prevActiveInstance = activeInstance;
      // activeInstance = vm;
      // activeInstance = prevActiveInstance;
      var child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance,
        parentElm,
        refElm
      );
      child.$mount(hydrating ? vnode.elm : undefined, hydrating);
    } else if(vnode.data.keepAlive){
      // keep-alive组件 当做补丁
      var mountedNode = vnode;
      componentVNodeHooks.prepatch(mountedNode, mountedNode);
    }
  },
  
  // prepatch
  prepatch: function prepatch(oldVnode, vnode){
    var options = vnode.componentOptions;
    var child = vnode.componentInstance = oldVnode.componentInstance;
    
    // 更新子组件
    updateChildComponent(
      child,
      options.propsData, // 更新props
      options.listeners, // 更新监听者
      vnode, // new parent vnode
      options.children // new children
    );
  },
  
  // insert
  insert: function insert(vnode){
    if(!vnode.componentInstance._isMounted){
      vnode.componentInstance._isMounted = true;
  	  callHook(vnode.componentInstance, 'mounted');
    }
    if(vnode.data.keepAlive){
      activateChildComponent(vnode.componentInstance, true);
    }
  },
  
  // destroy
  destroy: function destroy(vnode){
    // 查看对应属性
    if(!vnode.componentInstance._isDestroyed){
      if(!vnode.data.keepAlive){
        vnode.componentInstance.$destroy();
      } else{
        deactivateChildComponent(vnode.componentInstance, true);
      }
    }
  }
};
```



### createComponent

```javascript
function createComponent(Ctor, data, context, children, tag){
  if(!Ctor){
    return;
  }
  
  var baseCtor = context.$options._base;
  if(isObject(Ctor)){
    Ctor = baseCtor.extend(Ctor);
  }
  
  if(typeof Ctor !== 'function'){
    warn(('Invalid Component definition: ' + (String(Ctor))), context);
    return ;
  }
  
  // 异步组件
  if(!Ctor.cid){
    if(Ctor.resolved){
      Ctor = Ctor.resolved;
    } else{
      Ctor = resolveAsyncComponent(Ctor, baseCtor, function(){
        context.$forceUpdate();
      });
      if(!Ctor){
        return ;
      }
    }
  }
  
  resolveConstructorOptions(Ctor);
  
  data = data || {};
  
  // 转换组件v-model
  if(data.model){
    transformModel(Ctor.options, data);
  }
  
  // 提取props
  var propsData = extractProps(data, Ctor, tag);
  
  // 函数组件
  if(Ctor.options.functional){
    return createFunctionalComponent(Ctor, propsData, data, context, children);
  }
  
  // 
  var listeners = data.on;
  data.on = data.nativeOn;
  
  if(Ctor.options.abstract){
    // 抽象组件没数据
    data = {};
  }
  
  // 
  mergeHooks(data);
  
  // 返回一个占位虚拟节点
  var name = Ctor.options.name || tag;
  var vnode = new VNode(
    ('Vue-component-' + (Ctor.cid) + (name ? ('-' + name) : '')),
    data, undefined, undefined, undefined, context, { Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children}
  );
  return vnode;
}
```



### createFunctionalComponent

```javascript
function createFunctionalComponent(Ctor, propsData, data, context, children){
  var props = {};
  var propOptions = Ctor.options.props;
  if(propOptions){
    for(var key in propOptions){
      props[key] = validateProp(key, propOptions, propsData);
    }
  }
  
  // 
  var _context = Object.create(context);
  var h = function(a,b,c,d){ return createElement(_context, a, b, c, d, true); };
  var vnode = Ctor.options.render.call(null, h, {
    props: props,
    data: data,
    parent: context,
    children: children,
    slots: function(){ return resolveSlots(children, context); }
  });
  if(vnode instanceof VNode){
    vnode.functionalContext = context;
    if(data.slot){
      (vnode.data || (vnode.data = {})).slot = data.slot;
    }
  }
  return vnode;
}
```



### createComponentInstanceForVnode

```javascript
function createComponentInstanceForVnode(vnode, parent, parentElm, refElm){
  var vnodeComponentOptions = vnode.componentOptions;
  var options = {
    _isComponent: true,
    parent: parent,
    propsData: vnodeComponentOptions.propsData,
    _componentTag: vnodeComponentOptions.tag,
    _parentVnode: vnode,
    _parentListeners: vnodeComponentOptions.listeners,
    _renderChildren: vnodeComponentOptions.children,
    _parentElm: parentElm || null,
    _refElm: refElm || null
  };
  
  // 检查内置模板渲染函数
  var inlineTemplate = vnode.data.inlineTemplate;
  if(inlineTemplate){
    options.render = inlineTemplate.render;
    options.staticRenderFns = inlineTemplte.staticRenderFns;
  }
  return new vnodeComponentOptions.Ctor(options);
}
```



### resolveAsyncComponent

```javascript
function resolveAsyncComponent(factory, baseCtor, cb){
  if(factory.requested){
    // 回调池
    factory.pendingCallbacks.push(cb);
  } else{
    factory.requested = true;
    var cbs = factory.pendingCallbacks = [cb];
    var sync = true;
    
    var resolve = function(res){
      if(isObject(res)){
        res = baseCtor.extend(res);
      }
      // 缓存resolved
      factory.resolved = res;
      // 非同步时调用回调
      if(!sync){
        for(var i = 0, l = cbs.length; i < l; i++){
          cbs[i](res);
        }
      }
    };
    var res = factory(resolve, reject);
    
    // 处理promise
    if(res && typeof res.then === 'function' && !factory.resolved){
      res.then(resolve, reject);
    }
    
    sync = false;
    
    return factory.resolved;
  }
}
```



---



## props



### extractProps

```javascript
function extractProps(data, Ctor, tag){
  // 仅提取原生值
  var propOptions = Ctor.options.props;
  if(!propOptions){
    return;
  }
  var res = {};
  var attrs = data.attrs;
  var props = data.props;
  var domProps = data.domProps;
  if(attrs || props || domProps){
    for(var key in propOptions){
      var altKey = hyphenate(key);
      
      var keyInLowerCase = key.toLowerCase();
      if(key !== keyInLowerCase && attrs && attrs.hasOwnProperty(keyInLowerCase)){
        tip(
          'Prop "' + keyInLowerCase + '" is passed to component' + 
          (formatComponentName(tag || Ctor)) + ', but the declared prop name is' + 
          ' "' + key + '". ' + 
          'Note that HTML attributes are case-insensitive and camelCased ' + 
          'props need to use their kebab-case equivalents when using in-DOM ' + 
          'templates. You should probably use "' + altKey + '" instead of "' + key + '".'
        );
      }
      checkProp(res, props, key, altKey, true) || 
        checkProp(res, attrs, key, altKey) || 
        checkProp(res, domProps, key, altKey);
    }
  }
  return res;
}
```



### checkProp

```javascript
function checkProp(res, hash, key, altKey, preserve){
  if(hash){
    if(hasOwn(hash, key)){
      res[key] = hash[key];
      if(!preserve){
        delete hash[key];
      }
      return true;
    }
  }
  return false;
}
```



### mergeHooks

```javascript
var hooksToMerge = Object.keys(componentVNodeHooks);

function mergeHooks(data){
  // 默认参数
  if(!data.hook){
    data.hook = {};
  }
  for(var i = 0; i < hooksToMerge.length; i++){
    var key = hooksToMerge[i];
    var fromParent = data.hook[key];
    var ours = componentVNodeHooks[keys];
    data.hook[key] = fromParent ? mergeHook$1(ours, fromParent) : ours;
  }
}
```



### mergeHook$1

```javascript
// 这函数……
function mergeHook$1(one, two){
  return function(a, b, c, d){
    one(a, b, c, d);
    two(a, b, c, d);
  }
}
```




























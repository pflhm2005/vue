# Transition





---



### Transition



```javascript
var transition = inBrowser ? {
  create: _enter,
  activate: _enter,
  remove: function remove$$1(vnode, rm){
    if(!vnode.data.show){
      leave(vnode, rm);
    } else{
      rm();
    }
  }
} : {};
```





#### resolveTransition

```javascript
function resolveTransition(def$$1){
  if(!def$$1){
    return ;
  }
  if(typeof def$$1 === 'object'){
    var res = {};
    if(def$$1.css !== false){
      extend(res, autoCssTransition(def$$1.name || 'v'));
    }
    extend(res, def$$1);
    return res;
  } else if(typeof def$$1 === 'string'){
    return autoCssTransition(def$$1);
  }
}
```



#### autoCssTransition

```javascript
var autoCssTransition = cached(function(name){
  return {
    enterClass: (name + '-enter'),
    enterToClass: (name + '-enter-to'),
    enterActiveClass: (name + '-enter-active'),
    leaveClass: (name + '-leave'),
    leaveToClass: (name + '-leave-to'),
    leaveActiveClass: (name + '-leave-active')
  }
})
```



#### sniffing

```javascript
// transition兼容IE10+
var hasTransition = inBrowser && !isIE9;
var TRANSITION = 'transition';
var ANIMATION = 'animation';

var transitionProp = 'transition';
var transitionEndEvent = 'transitionend';
var animationProp = 'animation';
var animationEndEvent = 'animationend';
if(hasTransition){
  if(window.ontransitionend === undefined && window.onwebkittransitionend !== undefined){
    transitionProp = 'WebkitTransition';
    transitionEndEvent = 'webkitTransitionEnd';
  }
  if(window.onanimationend === undefined && window.onwebkitanimationend !== undefined){
    animationProp = 'WebkitAnimation';
    animationEndEvent = 'webkitAnimationEnd';
  }
}
```



#### raf

```javascript
// 强绑window是为了兼容IE的严格模式
var raf = inBrowser && window.requestAnimationFrame ? 
    window.requestAnimationFrame.bind(window) : 
	setTimeout;
```



#### nextFrame

```javascript
function nextFrame(fn){
  raf(function(){
    raf(fn);
  });
}
```



#### addTransitionClass

```javascript
function addTransitionClass(el, cls){
  (el._transitionClasses || (el._transitionClasses = [])).push(cls);
  addClass(el, cls);
}
```



#### removeTransitionClass

```javascript
function removeTransitionClass(el, cls){
  if(el._transitionClasses){
    remove(el._transitionClasses, cls);
  }
  removeClass(el, cls);
}
```



#### whenTransitionEnds

```javascript
function whenTransitionEnds(el, expectedType, cb){
  var ref = getTransitionInfo(el, expectedType);
  var type = ref.type;
  var timeout = ref.timeout;
  var propCount = ref.propCount;
  if(!type){ return cb() }
  var event = type === TRANSITION ? transitionEndEvent : animationEndEvent;
  var ended = 0;
  var end = function(){
    el.removeEventListener(event, onEnd);
    cb();
  };
  var onEnd = function(e){
    if(e.target === el){
      if(++ended >= propCount){
        end();
      }
    }
  };
  setTimeout(function(){
    if(ended < propCount){
      end();
    }
  },timeout + 1);
  el.addEventListener(event, onEnd);
}
```



#### getTransitionInfo

````javascript
// 匹配 tranform,*或all,*
var transformRE = /\b(transform|all)(,|$)/;

// transition: property duration timing-function delay
function getTransitionInfo(el, expectedType){
  var styles = window.getComputedStyle(el);
  var transitionDelays = styles[transitionProp + 'Delay'].split(', ');
  var transitionDurations = styles[transitionProp + 'Duration'].split(', ');
  var transitionTimeout = getTimeout(transitionDelays, transitionDurations);
  var animationDelays = styles[animationProp + 'Delay'].split(', ');
  var animationDurations = styles[animationProp + 'Duration'].split(', ');
  var animationTimeout = getTimeout(animationDelays, animationDurations);
  
  var type;
  var timeout = 0;
  var propCount = 0;
  
  if(expectedType === TRANSITION){
    if(transitionTimeout > 0){
      type = TRANSITION;
      timeout = transitionTimeout;
      propCount = transitionDurations.length;
    }
  } else if(expectedType === ANIMATION){
    if(animationTimeout > 0){
      type = ANIMATION;
      timeout = animationTimeout;
      propCount = animationDurations.length;
    }
  } else{
    timeout = Math.max(transitionTimeout, animationTimeout);
    type = timeout > 0 ? 
      transitionTimeout > animationTimeout ? 
      TRANSITION : ANIMATION : null;
    propCount = type ? 
      type === TRANSITION ? 
      transitionDurations.length : animationDurations.length : 0;
  }
  var hasTransform = type === TRANSITION && transformRE.test(styles[transitionProp + 'Property']);
  return {
    type: type,
    timeout: timeout,
    propCount: propCount,
    hasTransform: hasTransform
  }
}
````



##### getTimeout

```javascript
function getTimeout(delays, durations){
  while(delays.length < durations.length){
    delays = delays.concat(delays);
  }
  return Math.max.apply(null, durations.map(function(d, i){
    return toMs(d) + toMs(delays[i]);
  }));
}
```



##### toMs

```javascript
function toMs(s){
  // 尝试转换一个字符串 遇到非数字返回NaN
  return Number(s.slice(0, -1)) * 1000;
}
```



---



### enter

```javascript
function enter(vnode, toggleDisplay){
  var el = vnode.elm;
  
  if(el._leaveCb){
    el._leaveCb.cancelled = true;
    el._leaveCb();
  }
  
  var data = resolveTransition(vnode.data.transition);
  if(!data){
    return;
  }
  
  if(el._enterCb || el.nodeType !== 1){
    return;
  }
  
  var css = data.css;
  var type = data.type;
  var enterClass = data.enterClass;
  var enterToClass = data.enterToClass;
  var enterActiveClass = data.enterActiveClass;
  var appearClass = data.appearClass;
  var appearToClass = data.appearToClass;
  var appearActiveClass = data.appearActiveClass;
  var beforeEnter = data.beforeEnter;
  var enter = data.enter;
  var afterEnter = data.afterEnter;
  var enterCancelled = data.enterCancelled;
  var beforeAppear = data.beforeAppear;
  var appear = data.appear;
  var afterAppear = data.afterAppear;
  var appearCancelled = data.appearCancelled;
  var duration = data.duration;
  
  
  var context = activeInstance;
  var transitionNode = activeInstance.$vnode;
  while(transitionNode && transitionNode.parent){
    transitionNode = transitionNode.parent;
    context = transitionNode.context;
  }
  
  var isAppear = !context._isMounted || !vnode.isRootInsert;
  
  if(isAppear && !appear && appear !== ''){
    return;
  }
  
  var startClass = isAppear && appearClass ? appearClass : enterClass;
  var activeClass = isAppear && appearActiveClass ? appearActiveClass : enterActiveClass;
  var toClass = isAppear && appearToClass ? appearToClass : enterToClass;
  
  var beforeEnterHook = isAppear ? (beforeAppear || beforeEnter) : beforeEnter;
  var enterHook = isAppear ? (typeof appear === 'function' ? appear : enter) : enter;
  var afterEnterHook = isAppear ? (afterAppear || afterEnter) : afterEnter;
  var enterCancelledHook = isAppear ? (appearCancelled || enterCancelled) : enterCancelled;
  
  var explicitEnterDuration = toNumber(isObject(duration) ? duration.enter : duration);
  
  if('development' !== 'production' && explicitEnterDuration != null){
    checkDuration(explicitEnterDuration, 'enter', vnode);
  }
  
  var expectsCSS = css !== false && isIE9;
  var userWantsControl = getHookArgumentsLength(enterHook);
  
  var cb = el._enterCb = once(function(){
    if(expectsCSS){
      removeTransitionClass(el, toClass);
      removeTransitionClass(el, activeClass);
    }
    if(cb.cancelled){
      if(expectsCSS){
        removeTranstionClass(el, startClass);
      }
      enterCancelledHook && enterCancelledHook(el);
    } else{
      afterEnterHook && afterEnterHook(el);
    }
    el._enterCb = null;
  });
  
  if(!vnode.data.show){
    mergeVNodeHook(vnode.data.hook || (vnode.data.hook = {}), 'insert', function(){
      var parent = el.parentNode;
      var pendingNode = parent && parent._pending && parent._pending[vnode.key];
      if(pendingNode && 
         pendingNode.tag === vnode.tag && 
         pendingNode.elm._leaveCb){
        pendingNode.elm._leaveCb();
      }
      enterHook && enterHook(el, cb);
    });
  }
  
  beforeEnterHook && beforeEnterHook(el);
  if(expectsCSS){
    addTransitionClass(el, startClass);
    addTransitionClass(el, activeClass);
    nextFrame(function(){
      addTransitionClass(el, toClass);
      removeTransitionClass(el, startClass);
      if(!cb.cancelled && !userWantsControl){
        if(isValidDuration(explicitEnterDuration)){
          setTimeout(cb, explicitEnterDuration);
        } else{
          whenTransitionEnds(el, type, cb);
        }
      }
    });
  }
  
  if(vnode.data.show){
    toggleDisplay && toggleDisplay();
    enterHook && enterHook(el, cb);
  }
  
  if(!expectsCSS && !userWantsControl){
    cb();
  }
}
```



### leave

```javascript
function leave(vnode, rm){
  var el = vnode.elm;
  
  if(el._enterCb){
    el._enterCb.cancelled = true;
    el._enterCb();
  }
  
  var data = resolveTransition(vnode.data.transition);
  if(!data){
    return rm();
  }
  
  if(el._leaveCb || el.nodeType !== 1){
    return;
  }
  
  var css = data.css;
  var type = data.type;
  var leaveClass = data.leaveClass;
  var leaveToClass = data.leaveToClass;
  var leaveActiveClass = data.leaveActiveClass;
  var beforeLeave = data.beforeLeave;
  var leave = data.leave;
  var afterLeave = data.afterLeave;
  var leaveCancelled = data.leaveCancelled;
  var delayLeave = data.delayLeave;
  var duration = data.duration;
  
  var expectsCSS = css !== false && !isIE9;
  var userWantsControl = getHookArgumentsLength(leave);
  
  var explicitLeaveDuration = toNumber(
    isObject(duration) ? duration.leave : durantion
  );
  
  if('development' !== 'production' && explicitLeaveDuration != null){
    checkDuration(explicitLeaveDuration, 'leave', vnode);
  }
  
  var cb = el._leaveCb = once(function(){
    if(el.parentNode && el.parentNode._pending){
      el.parentNode._pending[vnode.key] = null;
    }
    if(expectsCSS){
      removeTransitionClass(el, leaveToClass);
      removeTransitionClass(el, leaveActiveClass);
    }
    if(cb.cancelled){
      if(expectsCSS){
        removeTransitionClass(el, leaveClass);
      }
      leaveCancelled && leaveCancelled(el);
    } else{
      rm();
      afterLeave && afterLeave(el);
    }
    el._leaveCb = null;
  });
  
  if(delayLeave){
    delayLeave(performLeave);
  } else{
    performLeave();
  }
  
  function performLeave(){
    if(cb.cancelled){
      return;
    }
    
    if(!vnode.data.show){
      (el.parentNode._pending || (el.parentNode._pending = {}))[vnode.key] = vnode;
    }
    beforeLeave && beforeLeave(el);
    if(expectsCSS){
      addTransitionClass(el, leaveClass);
      addTransitionClass(el, leaveActiveClass);
      nextFrame(function(){
        addTransitionClass(el, leaveToClass);
        removeTransitionClass(el, leaveClass);
        if(!cb.cancelled && !userWantsControl){
          if(isValidDuration(explicitLeaveDuration)){
            setTimeout(cb, explicitLeaveDuration);
          } else{
            whenTransitionEnds(el, type, cb);
          }
        }
      });
    }
    leave && leave(el, cb);
    if(!expectsCSS && !userWantsControl){
      cb();
    }
  }
}
```



#### checkDuration

```javascript
// 检测duration值 仅用于dev模式
function checkDuration(val, name, vnode){
  if(typeof val !== 'number'){
    warn(
      '<transition> explicit ' + name + ' duration is not a valid number - ' + 
      'got ' + (JSON.stringify(val)) + '.',
      vnode.context
    );
  }
}
```



#### isValidDuration

```javascript
function isValidDuration(val){
  return typeof val === 'number' && !isNaN(val);
}
```



#### getHookArgumentsLength

```javascript
function getHookArgumentsLength(fn){
  if(!fn){ return this; }
  var invokerFns = fn.fns;
  if(invokerFns){
    return getHookArgumentLength(
      Array.isArray(invokerFns) ? invokerFns[0] : invokerFns
    )
  } else{
    return (fn._length || fn.length) > 1
  }
}
```



### _enter

```javascript
function _enter(_, vnode){
  if(!vnode.data.show){
    enter(vnode);
  }
}
```



---





## platformModules





```javascript
var platformModules = [
  attrs,
  klass,
  events,
  domProps,
  style,
  transition
];

var modules = platformModules.concat(baseModules);

var patch = createPatchFunction({ nodeOps: nodeOps, modules: modules });

if(isIE9){
  document.addEventListener('selectionchange', function(){
    var el = document.activeElement;
    if(el && el.vmodel){
      trigger(el, 'input');
    }
  });
}
```





## platformDirectives





```javascript
var platformDirectives=  {
  model: model$1,
  show: show
}
```





#### model$1

```javascript
var model$1 = {
  inserted: function inserted(el, binding, vnode){
    if(vnode.tag === 'select'){
      var cb = function(){
        setSelected(el, binding, vnode.context);
      };
      cb();
      if(isIE || isEdge){
        setTimeout(cb, 0);
      }
    } else if(vnode.tag === 'textarea' || el.type === 'text' || el.type === 'password'){
      el._vModifiers = binding.modifiers;
      if(!binding.modifiers.lazy){
        // change事件在输入框内容改变且失去焦点时触发
        // input事件在输入内容改变时触发
        el.addEventListener('change',onCompositionEnd);
        if(!isAndroid){
          // compositionstart,compositionupdate,compositionend为DOM3的复合事件
          // 类似于keydown 处理一系列的IME输入序列
          // 根本测试不出来 不知道什么效果
          el.addEventListener('compositionstart', onCompositionStart);
          el.addEventListener('compositionend', onCompositionEnd);
        }
        if(isIE9){
          el.vmodel = true;
        }
      }
    }
  },
  componentUpdated: function componentUpdated(el, binding, vnode){
    if(vnode.tag === 'select'){
      setSelected(el, binding, vnode.context);
      var needReset = el.multiple ? 
          binding.value.some(function(v){ return hasNoMatchingOption(v, el.options); }) : 
      binding.value !== binding.oldValue && hasNoMatchingOption(binding.value, el.options);
      if(needReset){
        trigger(el, 'change');
      }
    }
  }
}
```



##### setSelected

```javascript
function setSelected(el, binding, vm){
  var value = binding.value;
  var isMultiple = el.multiple;
  // value必须为数组
  if(isMultiple && !Array.isArray(value)){
    'development' !== 'production' && warn(
      '<select multiple v-model="' + (binding.expression) + '"> ' + 
      'expects an Array value for its binding, but got ' + (Object.prototype.toString.call(value).slice(8, -1)),
      vm
    );
    return ;
  }
  var selected, option;
  for(var i = 0, l = el.options.length; i < l; i++){
    option = el.options[i];
    if(isMultiple){
      selected = looseIndexOf(value, getValue(option)) > -1;
      if(option.selected !== selected){
        option.selected = selected;
      }
    } else{
      if(looseEqual(getValue(option), value)){
        if(el.selectedIndex !== i){
          el.selectedIndex = i;
        }
        return ;
      }
    }
  }
  if(!isMultiple){
    el.selectedIndex = -1;
  }
}
```



##### hasNoMatchingOption

````javascript
function hasNoMatchingOption(value, options){
  for(var i = 0, l = options.length; i < l; i++){
    if(looseEqual(getValue(options[i]), value)){
      return false;
    }
  }
  return true;
}
````



##### getValue

```javascript
function getValue(option){
  return '_value' in option ? option._value : option.value;
}
```



##### onCompositionStart

```javascript
function onCompositionStart(e){
  e.target.composing = true;
}
```



##### onCompositionEnd

```javascript
function onCompositionEnd(e){
  e.target.composing = false;
  trigger(e.target, 'input');
}
```



##### trigger

```javascript
function trigger(el, type){
  var e = document.createEvent('HTMLEvents');
  e.initEvent(type, true, true);
  el.dispatchEvent(e);
}
```



#### locateNode

```javascript
function locateNode(vnode){
  return vnode.componentInstance && (!vnode.data || !vnode.data.transition) ? 
    locateNode(vnode.componentInstance.._vnode) : vnode;
}
```





#### show

```javascript
var show = {
  bind: function bind(el, ref, vnode){
    var value = ref.value;
    vnode = locateNode(vnode);
    var transition = vnode.data && vnode.data.transition;
    var originalDisplay = el._vOriginalDisplay = el.style.display === 'none' ? '' : el.style.display;
    if(value && transition && !isIE9){
      vnode.data.show = true;
      enter(vnode, function(){
        el.style.display = originalDisplay;
      });
    } else{
      el.style.display = value ? originalDisplay : 'none';
    }
  },
  update: function update(el, ref, vnode){
    var value = ref.value;
    var oldValue = ref.oldValue;
    
    if(value === oldValue){ return }
    vnode = locateNode(vnode);
    var transition = vnode.data && vnode.data.transition;
    if(transition && !isIE9){
      vnode.data.show = true;
      if(value){
        enter(vnode, function(){
          el.style.display = el._vOriginalDisplay;
        });
      } else{
        leave(vnode, function(){
          el.style.display = 'none';
        });
      }
    } else{
      el.style.display = value ? el._vOriginalDisplay : 'none';
    }
  },
  unbind: function unbind(el, binding, vnode, oldVnode, isDestroy){
    if(!isDestroy){
      el.style.display = el._vOriginalDisplay;
    }
  }
};
```



---



### transitionProps



```javascript
var transitionProps = {
  name: String,
  appear: Boolean,
  css: Boolean,
  mode: String,
  type: String,
  enterClass: String,
  leaveClass: String,
  enterToClass: String,
  leaveToClass: String,
  enterActiveClass: String,
  leaveActiveClass: String,
  appearClass: String,
  appearActiveClass: String,
  appearToClass: String,
  duration: [Number, String, Object]
};
```



#### getRealChild

```javascript
function getRealChild(vnode){
  var compOptions = vnode && vnode.componentOptions;
  if(compOptions && compOptions.Ctor.options.abstract){
    return getRealChild(getFirstComponentChild(compOptions.children));
  } else{
    return vnode;
  }
}
```



#### extractTransition

```javascript
function extractTransitionData(comp){
  var data = {};
  var options = comp.$options;
  
  // props
  for(var key in options.propsData){
    data[key] = comp[key];
  }
  
  var listeners = options._parentListeners;
  for(var key$1 in listeners){
    data[camelize(key$1)] = listeners[key$1];
  }
  return data;
}
```



#### placeholder

```javascript
function placeholder(h, rawChild){
  if(/\d-keep-alive$/.test(rawChild.tag)){
    return h('keep-alive', {
      props: rawChild.componentOptions.propsData
    });
  }
}
```



#### hasParentTransition

```javascript
function hasParentTransition(vnode){
  // 向上遍历 直到父元素有vnode.data.transition属性
  while((vnode = vnode.parent)){
    if(vnode.data.transition){
      return true;
    }
  }
}
```



#### isSameChild

```javascript
function isSameChild(child, oldChild){
  // key,tag相等
  return oldChild.key === child.key && oldChild.tag === child.tag;
}
```



---





## platformComponent





```javascript
var platformComponent = {
  Transition: Transition,
  TransitionGroup: TransitionGroup
}
```





### Transition

```javascript
var Transition = {
  name: 'transiiton',
  props: transitionProps,
  abstract: true,
  
  render: function render(h){
    var this$1 = this;
    
    var children = this.$slots.default;
    if(!children){
      return ;
    }
    
    // 过滤文本节点(空格)
    children = children.filter(function(c){ return c.tag; });
    if(!children.length){
      return ;
    }
    
    if('development' !== 'production' && children.length > 1){
      warn(
        '<transition> can only be used on a single element. Use' + 
        '<transition-group> for lists.',
        this.$parent
      );
    }
    var mode = this.mode;
    
    if('development' !== 'production' && mode && mode !== 'in-out' && mode !== 'out-in'){
      warn(
        'invalid <transition> mode: ' + mode,
        this.$parent
      );
    }
    var rawChild = children[0];
    
    // 如果这是组价根节点 且父节点也有transition 跳过
    if(hasParentTransition(this.vnode)){
      return rawChild;
    }
    
    // 
    var child = getRealChild(rawChild);
    if(!child){
      return rawChild;
    }
    
    
    if(this._leaving){
      return placeholder(h, rawChild);
    }
    
    var id = '_transition-' + (this.id) + '-';
    child.key = child.key == null ? 
      id + child.tag : 
      isPrimitive(child.key) ? 
      (String(child.key).indexOf(id) === 0 ? child.key : id + child.key) : 
      child.key;
    
    var data = (child.data || (child.data = {})).transition = extractTransitionData(this);
    var oldRawChild = this._vnode;
    var oldChild = getRealChild(oldRawChild);
    
    // 标记v-show
    if(child.data.directives && child.data.directives.somw(function(d){ return d.name === 'show'; })){
      child.data.show = true;
    }
    
    if(oldChild && oldChild.data && !isSameChild(child, oldChild)){
      var oldData = oldChild && (oldChild.data.transition = extend({}, data));
      if(mode === 'out-in'){
        this._leaving = true;
        mergeVNodeHook(oldData, 'afterLeave', function(){
          this$1._leaving = false;
          this$1.$forceUpdate();
        });
        return placeholder(h, rawChild);
      } else if(mode == 'in-out'){
        var delayedLeave;
        var performLeave = function(){ delayedLeave(); };
        mergeVNodeHook(data, 'afterEnter', performLeave);
        mergeVNodeHook(data, 'enterCancelled', performLeave);
        mergeVNodeHook(oldData, 'delayLeave', function(leave){ delayedLeave = leave; });
      }
    }
    return rawChild;
  }
};
```



---



### props



```javascript
var props = extend({
  tag: String,
  moveClass: String
}, transitionProps);

delete props.mode;
```



### TransitionGroup

```javascript
var TransitionGroup = {
  props: props,
  
  render: function render(h){
    var tag = this.tag || this.$vnode.data.tag || 'span';
    var map = Object.create(null);
    var prevChildren = this.prevChildren = this.children;
    var rawChildren = this.$slots.default || [];
    var children  = this.children = [];
    var transitionData = extractTransitionData(this);
    
    for(var i = 0; i < rawChildren.length; i++){
      var c = rawChildren[i];
      if(c.tag){
        if(c.key != null && String(c.key).indexOf('__vlist') !== 0){
          children.push(c);
          map[c.key] = c;
          (c.data || (c.data = {})).transition = transitionData;
        } else{
          var opts = c.componentOptions;
          var name = opts ? (opts.Ctor.options.name || opts.tag || '') : c.tag;
          warn(('<transition-group> children must be keyed: <' + name + '>'));
        }
      }
    }
    
    if(prevChildren){
      var kept = [];
      var removed = [];
      for(var i$1 = 0; i$1 < prevChildren.length; i$1++){
        var c$1 = prevChildren[i$1];
        c$1.data.transition = transitionData;
        // getBoundingClientRect()返回一个对象 包含元素各边与页面上边和左边的距离
        // IE有BUG 默认从(2,2)开始计算
        c$1.data.pos = c$1.elm.getBoundingClientRect();
        if(map[c$1.key]){
          kept.push(c$1);
        } else{
          removed.push(c$1);
        }
      }
      this.kept = h(tag, null, kept);
      this.removed = removed;
    }
    return h(tag, null, children);
  },
  // beforeUpdate
  beforeUpdate: function beforeUpdate(){
    this._patch(this._vnode, this.kept, false, true);
    this,_vnode = this.kept;
  },
  // updated
  updated: function updated(){
    var children = this.prevChildren;
    var moveClass = this.moveClass || ((this.name || 'v') + '-move');
    if(!children.length || !this.hasMove(children[0].elm, moveClass)){
      return ;
    }
    
    // 将渲染分为三步执行
    children.forEach(callPendingCbs);
    children.forEach(recordPosition);
    children.forEach(applyTranslation);
    
    // 强制重排
    var body = document.body;
    var f = body.offsetHeight;
    
    children.forEach(function(c){
      if(c.data.moved){
        var el = c.elm;
        var s = el.style;
        addTransitionClass(el, moveClass);
        s.transform = s.WebkitTransform = s.transitionDuration = '';
        el.addEventListener(transitionEndEvent, el._moveCb = function cb(e){
          if(!e || /transform$/.test(e.propertyName)){
            el.removeEventListener(transitionEndEvent, cb);
            el._moveCb = null;
            removeTransitionClass(el, moveClass);
          }
        });
      }
    });
  },
  // methods
  methods: {
    hasMove: function hasMove(el, moveClass){
      if(!hasTransition){
        return false;
      }
      if(this._hasMove != null){
        return this._hasMove;
      }
      // 
      var clone = el.cloneNode();
      if(el._transitionClasses){
        el._transitionClasses.forEach(function(cls){ removeClass(clone, cls); });
      }
      addClass(clone, moveClass);
      clone.style.display = 'none';
      this.$el.appendChild(clone);
      var info = getTransitionInfo(clone);
      this.$el.removeChild(clone);
      return (this._hasMove = info.hasTransform);
    }
  }
};
```



#### callPendingCbs

```javascript
function callPendingCbs(c){
  if(c.elm._moveCb){
    c.elm._moveCb();
  }
  if(c.elm._enterCb){
    c.elm._enterCb();
  }
}
```



#### recordPosition

```javascript
function recordPosition(c){
  // 获取新位置
  c.data.newPos = c.elm.getBoundingClientRect();
}
```



#### applyTranslation

```javascript
function applyTranslation(c){
  var oldPos = c.data.pos;
  var newPos = c.data.newPos;
  var dx = oldPos.left - newPos.left;
  var dy = oldPos.top - newPos.top;
  if(dx || dy){
    c.data.moved = true;
    var s = c.elm.style;
    s.transform = s.WebkitTransform = 'translate(' + dx + 'px,' + dy + 'px)';
    s.transitionDuration = '0s';
  }
}
```












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



### platformModules



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



### locateNode

```javascript
function locateNode(vnode){
  return vnode.componentInstance && (!vnode.data || !vnode.data.transition) ? 
    locateNode(vnode.componentInstance.._vnode) : vnode;
}
```





### show

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
    
  }
}
```


































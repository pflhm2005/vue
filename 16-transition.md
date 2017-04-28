# event&&style&&class







### Event



```javascript
// 两个一样的搞毛
var events = {
  create: updateDOMlisteners,
  update: updateDOMlisteners
}
```



#### normalizeEvents

```javascript
function normalizeEvents(on){
  var event;
  if(on[RANGE_TOKEN]){
    // IE的input[type=range]仅支持change事件
    event = isIE ? 'change' : 'input';
    on[event] = [].concat(on[RANGE_TOKEN], on[event] || []);
    delete on[RANGE_TOKEN];
  }
  if(on[CHECKBOX_RADIO_TOKEN]){
    event = isChrome ? 'click' : 'change';
    on[event] = [].concat(on[CHECKBOX_RADIO_TOKEN], on[event] || []);
    delete on[CHECKBOX_RADIO_TOKEN];
  }
}
```



```javascript
var target$1;
```



#### add$1

```javascript
function add$1(event, handler, once, capture){
  if(once){
    var oldHandler = handler;
    var _target = target$1;
    handler = function(ev){
      var res = arguments.length === 1 ? oldHandler(ev) : oldHandler.apply(null, arguments);
      if(res !== null){
        remove$2(event, handler, capture, _target);
      }
    };
  }
  target$1.addEventListener(event, handler, capture);
}
```





#### remove$2

```javascript
function remove$2(event, handler, capture, _target){
  (_target || target$1).removeEventListener(event, handler, capture);
}
```



#### updateDOMlisteners

```javascript
function updateDOMlisteners(oldVnode, vnode){
  if(!oldVnode.data.on && !vnode.data.on){
    return ;
  }
  var on = vnode.data.on || {};
  var oldOn = oldVnode.data.on || {};
  target$1 = vnode.elm;
  normalizeEvents(on);
  updatelisteners(on, oldOn, add$1, remove$2, vnode, context);
}
```



------



### domProps



```javascript
var domProps = {
  create: updateDOMProps,
  update: updateDOMProps
}
```



#### updateDOMProps

```javascript
function updateDOMProps(oldVnode, vnode){
  if(!oldVnode.data.domProps && !vnode.data.domProps){
    return ;
  }
  var key, cur;
  var elm = vnode.elm;
  var oldProps = oldVnode.data.domProps || {};
  var props = vnode.data.domProps || {};
  // 复制观测对象
  if(props.__ob__){
    props = vnode.data.domProps = extend({}, props);
  }
  
  for(key in oldProps){
    if(props[key] == null){
      elm[key] = '';
    }
  }
  for(key in props){
    cur = props[key];
    // 判断子元素是否有内容
    if(key === 'textContent' || key === 'innerHTML'){
      if(vnode.children) { vnode.children.length = 0; }
      if(cur === oldProps[key]) { continue; }
    }
    if(key === 'value'){
      // 将value保存为_value属性
      elm._value = cur;
      // 避免当值一样时重置鼠标位置
      var strCur = cur = null ? '' : String(cur);
      if(shouldUpdateValue(elm, vnode, strCur)){
        elm.value = strCur;
      }
    } else{
      elm[key] = cur;
    }
  }
}
```



#### shouldUpdateValue

```javascript
function shouldUpdateValue(elm, vnode, checkVal){
  return (!elm.composing && (
    vnode.tag === 'option' || 
    isDirty(elm, checkVal) || 
    isInputChanged(elm, checkVal)
  ));
}
```



#### isDirty

```javascript
function isDirty(elm, checkVal){
  // 当输入框失去焦点且值不一致时返回true
  return document.activeElement !== elm && elm.value !== checkVal;
}
```



#### isInputChanged

```javascript
function isInputChanged(elm, newVal){
  var value = elm.value;
  var modifiers = elm._vModifiers;
  // 比较数字s
  if((modifiers && modifiers.number) || elm.type === 'number'){
    return toNumber(value) !== toNumber(newVal);
  }
  // trim()
  if(modifiers && modifiers.trim){
    return value.trim() !== newVal.trim();
  }
  return value !== newVal;
}
```



------



### Style



```javascript
var style = {
  create: updateStyle,
  update: updateStyle
}
```



#### parseStyleText

```javascript
var parseStyleText = cached(function(cssText){
  var res = {};
  // 
  var listDelimiter = /;(?![^(]*\))/g;
  // 匹配':'后面1个或多个元素
  var propertyDelimiter = /:(.+)/;
  cssText.split(listDelimiter).forEach(function(item){
    if(item){
      var tmp = item.split(propertyDelimiter);
      tmp.length > 1 && (res[tmp[0].trim()] = tmp[1].trim());
    }
  });
  return res;
})
```



#### normalizeStyleData

```javascript
// 合并style
function normalizeStyleData(data){
  var style = normalizeStyleBinding(data.style);
  // 
  return data.staticStyle ? 
    extend(data.staticStyle, style) : 
    style
}
```



#### normalizeStyleBinding

```javascript
function normalizeStyleBinding(bindingStyle){
  if(Array.isArray(bindingStyle)){
    return toObject(bindingStyle);
  }
  if(typeof bindingStyle === 'string'){
    return parseStyleText(bindingStyle);
  }
  return bindingStyle;
}
```



#### getStyle

```javascript
function getStyle(vnode, checkChild){
  var res = {};
  var styleData;
  
  if(checkChild){
    var childNode = vnode;
    while(childNode.componentInstance){
      childNode = childNode.componentInstance._vnode;
      if(childNode.data && (styleData = normalizeStyleData(childNode.data))){
        extend(res, styleData);
      }
    }
  }
  
  if((styleData = normalizeStyleData(vnode.data))){
    extend(res, styleData);
  }
  
  var parentNode = vnode;
  while((parentNode = parentNode.parent)){
    if(parentNode.data && (styleData = normalizeStyleData(parentNode.data))){
      extend(res, styleData);
    }
  }
  return res;
}
```



#### setProp

```javascript
var cssVarRE = /^--/;
var importantRE = /\s*!important$/;
var setProp = function(el, name, val){
  if(cssVarRE.test(name)){
    el.style.setProperty(name, val);
  } else if(importantRE.test(val)){
    el.style.setProperty(name, val.replace(important, ''), 'important');
  } else{
    el.style[normalize(name)] = val;
  }
}
```



#### normalize

```javascript
var prefixes = ['Webkit','Moz','ms'];

var testEl;
var normalize = cached(function(prop){
  testEl = testEl || document.createElement('div');
  prop = camelize(prop);
  if(prop !== 'filter' && (prop in testEl.style)){
    return prop;
  }
  // 首字母大写
  var upper = prop.charAt(0).toUppercase() + prop.slice(1);
  for(var  i = 0; i < prefixes.length; i++){
    var prefixed = prefixes[i] + upper;
    if(prefixed in testEl.style){
      return prefixed;
    }
  }
})
```



#### updateStyle

```javascript
function updateStyle(oldVnode, vnode){
  var data = vnode.data;
  var oldData = oldVnode.data;
  if(!data.staticStyle && !data.style && !oldData.staticStyle && !oldData.style){
    return ;
  }
  
  var cur = name;
  var el = vnode.elm;
  var oldStaticStyle = oldVnode.data.staticStyle;
  var oldStyleBinding = oldVnode.data.style || {};
  
  var oldStyle = oldStaticStyle || oldStyleBinding;
  var style = normalizeStyleBinding(vnode.data.style) || {};
  
  vnode.data.style = style.__ob__ ?	extend({}, style) : style;
  var newStyle = getStyle(vnode, true);
  
  for(name in oldStyle){
    if(newStyle[name] == null){
      setProp(el, name, '');
    }
  }
  for(name in newStyle){
    cur = newStyle[name];
    if(cur !== oldStyle[name]){
      setProp(el, name, cur == null ? '' : cur);
    }
  }
}
```



---



### Class



#### addClass

```javascript
function addClass(el, cls){
  if(!cls || !(cls = cls.trim())){
    return;
  }
  // classList兼容IE10+ 还是部分兼容
  if(el.classList){
    if(cls.indexOf(' ') > -1){
      // 空白分割添加到类中
      cls.split(/\s+/).forEach(function(c){ return el.classList.add(c); });
    } else{
      el.classList.add(cls);
    }
  } else{
    var cur = ' ' + (el.getAttribute('class') || '') + ' ';
    if(cur.indexOf(' ' + cls + ' ') < 0){
      el.setAttribute('class', (cur + cls).trim());
    }
  }
}
```





#### removeClass

```javascript
// DOM节点与要删除的类
function removeClass(el, cls){
  // 这里已经对cls调用了trim()方法
  if(!cls || !cls = cls.trim()){
    return ;
  }
  // classList兼容IE10+
  // 如果支持 直接按空格分割字符串并调用remove方法删除类
  if(el.classList){
    if(cls.indexOf(' ') > -1){
      cls.split(/\s+/).forEach(function(c){ return el.classList.remove(c); });
    } else{
      el.classList.remove(cls);
    }
  } 
  // 对IE10以下做兼容 不知道为啥不用className
  else{
    // 修正类集合字符串 两边加空白
    // 'a b' => ' a b '
    var cur = ' ' + (el.getAttribute('class') || '') + ' ';
    // 修正目标类 'a' => ' a '
    var tar = ' ' + cls + ' ';
    // 进行替换 ' a b ' => ' b '
    while(cur.indexOf(tar) >= 0){
      cur = cur.replace(tar, ' ');
    }
    // 再调用trim()操作 ' b ' => 'b'
    el.setAttribute('class', cur.trim());
  }
}
```



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






















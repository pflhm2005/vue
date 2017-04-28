# event/domProps/style/class





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



------



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



------


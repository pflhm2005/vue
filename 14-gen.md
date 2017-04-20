# parse model





## gen



gen: 产生



### genComponentModel

```javascript
// for v-model
function genComponentModel(el, value, modifiers){
  var ref = modifiers || {};
  var number = ref.number;
  var trim = ref.trim;
  
  // v = b = '$$v'
  var baseValueExpression = '$$v';
  var valueExpression = baseValueExpression;
  // ref.trim存在
  // v = (typeof b === string ? b.trim() : b)
  if(trim){
    valueExpression = 
      '(typeof ' + baseValueExpression + ' === "string"' + 
      '? ' + baseValueExpression + '.trim()' + 
      ': ' + baseValueExpression + ')';
  }
  // ref.number存在
  // v = _n(v)
  if(number){
    valueExpression = '_n(' + valueExpression + ')';
  }
  var assignment = genAssignmentCode(value, valueExpression);
  
  // el.model = {value:(value),expression:"value",callback:function(b){a}}
  el.model = {
    value: ('(' + value + ')'),
    expression: ('"' + value + '"'),
    callback: ('function (' + baseValueExpression + ') {' + assignment + '}')
  };
}
```



### genAssignmentCode

```javascript
function genAssignmentCode(value, assignment){
  var modelRs = parseModel(value);
  if(modelRs.idx === null){
    // 括号不影响输出 纯粹美观
    // 'v = a'
    return (value + '=' + assignment)
  } else{
    // var $$exp = m.e,$$idx = m.i;
    // if(!Array.isArray($$exp)){ v = a; } 
    // else{ $$exp.splice($$idx, 1, a); }
    return 'var $$exp = ' + (modelRs.exp) + ', $$idx = ' + (modelRs.idx) + ';' + 
      'if (!Array.isArray($$exp)){' + 
      value + '=' + assignment + '}' + 
      'else{$$exp.splice($$idx, 1, ' + assignment + ')}'
  }
}
```



---



## parse



### parseModel

```javascript
// 
var len;
var str;
var chr;
var index$1;
var expressionPos;
var expressionEndPos;
function parseModel(val){
  str = val;
  len = str.length;
  index$1 = expressionPos = expressionEndPos = 0;
  
  if(val.indexOf('[') < 0 || val.lastIndexOf(']') < len  - 1){
    return {
      exp: val,
      idx: null
    }
  }
  while(!eof){
    chr = next();
    if(isStringStart(chr)){
      parseString(chr);
      // 0x5B =>91 => [
    } else if(chr === 0x5B){
      parseBracket(chr);
    }
  }
  
  return {
    exp: val.substring(0, expressionPos),
    idx: val.substring(expression + 1, expressionEndPos)
  }
}
```



### parseBracket

```javascript
function parseBracket(chr){
  var inBracket = 1;
  expressionPos = index$1;
  while(!eof){
    chr = next();
    if(isStringStart(chr)){
      parseString(chr);
      continue;
    }
    if(chr === 0x5B) { inBracket++; }
    if(chr === 0x5D) { inBracket--; }
    if(inBracket === 0){
      expressionEndPos = index$1;
      break;
    }
  }
}
```



#### parseString

```javascript
function parseString(chr){
  var stringQuote = chr;
  while(!eof()){
    chr = next();
    if(chr === stringQuote){
      break;
    }
  }
}
```



#### next

```javascript
// 对应的ASCII码
function next(){
  return str.charCodeAt(++index$1);
}
```



#### eof

```javascript
// 索引大于长度
function eof(){
  return index$1 >= len
}
```



#### isStringStart

```javascript
// 单双引号 => ' "
function isStringStart(chr){
  return chr === 0x22 || chr === 0x27;
}
```



---



## model





### model

```javascript
var warn$1;
var RANGE_TOKEN = '__r';
var CHECKBOX_RADIO_TOKEN = '__c';

function model(el, dir, _warn){
  warn$1 = _warn;
  var value = dir.value;
  var modifiers = dir.modifiers;
  var tag = el.tag;
  var type = el.attrsMap.type;
  
  // 动态类型是啥…
  var dynamicType = el.attrsMap['v-bind:type'] || el.attrsMap[':type'];
  if(tag === 'input' && dynamicType){
    warn$1(
      '<input :type="' + dynamicType + '" v-model="' + value + '">:\n' + 
      'v-model does not support dynamic input types. Use v-if branches instead.'
    );
  }
  // type = 'file'是只读 无法修改
  if(tag === 'input' && type === 'file'){
    warn$1(
      '<' + (el.tag) + ' v-model="' + value + '" type="file">:\n' + 
      'File inputs are read only. Use a v-on:change listener instead.'
    );
  }
  
  if(tag === 'select'){
    genSelect(el, value, modifiers);
  } else if(tag === 'input' && type === 'checkbox'){
    genCheckboxModel(el, value, modifiers);
  } else if(tag === 'input' && type === 'radio'){
    genRadioModel(el, value, modifiers);
  } else if(tag === 'input' || tag === 'textarea'){
    genDefaultModel(el, value, modifiers);
  } else if(!config.isReservedTag(tag)){
    genComponentModel(el, value, modifiers);
    return false;
  } else{
    warn$1(
      '<' + (el.tag) + ' v-model="' + value + '">: ' + 
      'v-model is not supported on this element type. ' + 
      'If you are working with contenteditable, it\'s recommended to ' + 
      'wrap a library dedicated for that purpose inside a custom component.'
    );
  }
  return true;
}
```



#### genCheckboxModel

```javascript
function genCheckboxModel(el, value, modifiers){
  var number = modifiers && modifiers.number;
  var valueBinding = getBindingAttr(el, 'value') || 'null';
  var trueValueBinding = getBindingAttr(el, 'true-value') || 'true';
  var falseValueBinding = getBindingAttr(el, 'false-value') || 'false';
  // Array.isArray(v) ? _i(v,vB) > -1 (tVB === 'true' ? ':("v")' : ':_q("v,tVB")')
  addProp(el, 'checked',
         'Array.isArray(' + value + ')' + 
          '?_i(' + value + ',' + valueBinding + ')>-1' + (
  			trueValueBinding === 'true' ? (':(' + value + ')') : (':_q(' + value + ',' + trueValueBinding + ')'))
         );
  addHandler(el, CHECKBOX_RADIO_TOKEN, 
             'var $$a=' + value + ',' + 
             '$$el=$event.target,' + 
             '$$c=$$el.checked?(' + trueValueBinding + '):(' + falseValueBinding + ');' + 
             'if(Array.isArray($$a)){' + 
             'var $$v=' + (number ? '_n(' + valueBinding + ')' : valueBinding) + ',' + 
             '$$i=_i($$a,$$v);' + 
             'if($$c){$$i<0&&(' + value + '=$$a.concat($$V))}' + 
             'else{$$i>-1&&(' + value + '=$$a.slice(0,$$i).concat($$a.slice($$i+1)))}' + 
             '}else{' + value + '=$$c}',
             null, true
            );
}
```



#### genRadioModel

```javascript
function genRadioModel(el, value, modifiers){
  var number = modifiers && modifiers.number;
  var valueBinding = getBindingAttr(el, 'value') || 'null';
  // _n(valueBinding)
  valueBinding = number ? ('_n(' + valueBinding + ')') : valueBinding;
  // _q(value, valueBinding)
  addProp(el, 'checked', ('_q(' + value + ',' + valueBinding + ')'));
  addHandler(el, CHECKBOX_RADIO_TOKEN, genAssignmentCode(value, valueBinding), null, true);
}
```



#### genSelect

```javascript
function genSelect(el, value, modifiers){
  var number = modifiers && modifiers.number;
  // Array.prototype.filter.call($event.target.options,function(o){return o.selected};)
  // .map(function(o){var val = '_value' in o ? o._value : o.value; return number ? _n(val) : val})
  var selectedVal = 'Array.prototype.filter' + 
      '.call($event.target.options,function(o){return o.selected})' + 
      '.map(function(o){var val = "_value" in o ? o._value : o.value;' + 
      'return ' + (number ? '_n(val)' : 'val') + '})';
  var assignment = '$event.target.multiple ? $$selectedVal : $$selectedVal[0]';
  var code = 'var $$selectedVal = ' + selectedVal + ';';
  code = code + ' ' + (genAssignmentCode(value, assignment));
  addHandler(el, 'change', code, null, true);
}
```



#### genDefaultModel

```javascript
function genDefaultModel(el, value, modifiers){
  var type = el.attrsMap.type;
  var ref = modifiers || {};
  var lazy = ref.lazy;
  var number = ref.number;
  var trim = ref.trim;
  var needCompositionGuard = !lazy && type !== 'range';
  //
  var event = lazy ? 'change' : type === 'range' ? RANGE_TOKEN : 'input';
  var valueExpression = '$event.target.value';
  if(trim){
    valueExpression = '$event.target.value.trim()';
  }
  if(number){
    valueExpression = '_n(' + valueExpression + ')';
  }
  var code = genAssignmentCode(value, valueExpression);
  if(needCompositionGuard){
    code = 'if($event.target.composition)return;' + code;
  }
  addProp(el, 'value', ('(' + value + ')'));
  addHandler(el, event, code, null, true);
  if(trim || number || type === 'number'){
    addHandler(el, 'blur', '$forceUpdate()');
  }
}
```



---



### Event



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



#### events

```javascript
// 两个一样的搞毛
var events = {
  create: updateDOMlisteners,
  update: updateDOMlisteners
}
```



---



### Props



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



#### domProps

```javascript
var domProps = {
  create: updateDOMProps,
  update: updateDOMProps
}
```



---



### Style



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














































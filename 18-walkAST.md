# walkAST







---



### optimize

```javascript
var isStaticKey;
var isPlatformReservedTag;

var genStaticKeysCached = cached(genStaticKey$1);

// 优化：将部分不会变的DOM节点移除虚拟树
function optimize(root, options){
  if(!root){ return }
  isStaticKey = genStaticKeysCached(options.staticKeys || '');
  isPlatformReservedTag = options.isReservedTag || no;
  // 1、标记所有非静态节点
  markStatic$1(root);
  // 2、标记静态节点
  markStaticRoots(root, false);
}
```



### genStaticKey$1

```javascript
function genStaticKey$1(keys){
  return makeMap(
    'type,tag,attrsList,attrsMap,plain,parent,children,attrs' + 
    (keys ? ',' +keys : '')
  );
}
```



### markStatic$1

```javascript
function markStatic$1(node){
  node.static = isStatic(node);
  if(node.type === 1){
    if(!isPlatformReservedTag(node.tag) && node.tag !== 'slot' node.attrsMap['inline-template'] == null){ return; }
    for(var i = 0; l = node.children.length; i < l; i++){
      var child = node.children[i];
      markStatic$1(child);
      if(!child.static){
        node.static = false;
      }
    }
  }
}
```



### markStaticRoots

```javascript
function markStaticRoots(node, isInFor){
  if(node.type === 1){
    if(node.static || node.once){
      node.staticInFor = isInFor;
    }
    if(node.static && node.children.length && !(node.children.length === 1 && node.children[0].type === 3)){
      node.staticRoot = true;
      return ;
    } else{
      node.staticRoot = false;
    }
    if(node.children){
      for(var i = 0; l = node.children.length; i < l; i++){
        markStaticRoots(node.children[i], isInFor || !!node.for);
      }
    }
    if(node.ifConditions){
      walkThroughConditionsBlocks(node.ifConditions, isInFor);
    }
  }
}
```



### walkThroughConditionsBlocks

```javascript
function walkThroughConditionsBlocks(conditionBlocks, isInFor){
  for(var i = 1, len = conditionBlocks.length; i < len; i++){
    markStaticRoots(conditionBlocks[i].block, isInFor);
  }
}
```



### isStatic

```javascript
function isStatic(node){
  if(node.type === 2){
    return false;
  }
  if(node.type === 3){
    return true;
  }
  return !!(node.pre || 
            (!node.hasBindings && 
             !node.if && !node.for && 
             !isBuiltInTag(node.tag) &&
             isPlatformReservedTag(node.tag) && 
             !isDirectChildOfTemplateFor(node) &&
             Object.keys(node).every(isStaticKey)
            ));
}
```



### isDirectChildOfTemplateFor

```javascript
function isDirectChildOfTemplateFor(node){
  while(node.parent){
    node = node.parent;
    if(node.tag !== 'template'){
      return false;
    }
    if(node.for){
      return true;
    }
  }
  return false;
}
```



---





## gen





### utils

```javascript
var fnExpRE = /^\s*([\w$_]+|\([^)]*?\))\s*=>|^function\s*\(/;
var simplePathRE = /^\s*[A-Za-z_$][\w$]*(?:\.[A-Za-z_$][\w$]*|\['.*?']|\[".*?"]|\[\d+]|\[[A-Za-z_$][\w$]*])*\s*$/;

// 键盘事件别名
var keyCodes = {
  esc: 27,
  tab: 9,
  enter: 13,
  space: 32,
  up: 38,
  left: 37,
  right: 39,
  down: 40,
  'delete': [8, 46]
};
```



### genGuard

```javascript
var genGuard = function(condition) { return ('if(' + condition + ')return null;'); };
```



### modifierCode

```javascript
// 事件小尾巴
var modifierCode = {
  stop: '$event.stopPropagation();',
  prevent: '$event.preventDefault();',
  self: genGuard('$event.target !== $event.currentTarget'),
  ctrl: genGuard('!$event.ctrlKey'),
  shift: genGuard('!$event.shiftKey'),
  alt: genGuard('!$event.altKey'),
  meta: genGuard('!$event.metaKey'),
  left: genGuard('"button" in $event && $event.button !== 0'),
  middle: genGuard('"button" in $event && $event.button !== 1'),
  right: genGuard('"button" in $event && $event.button !== 2')
};
```



### genHandlers

```javascript
function genHandlers(events, native, warn){
  var res = native ? 'nativeOn:{' : 'on:{';
  for(var name in events){
    var handler = events[name];
    // 右键点击没卵用
    if('development' !== 'production' && name === 'click' && handler && handler.modifiers && handler.modifiers.right){
      warn(
        'Use "contextmenu" instead of "click.right" since right clicks ' + 
        'do not actually fire "click" events.'
      );
    }
    res += '"' + name + '":' + (genHandler(name. handler)) + ',';
  }
  return res.slice(0, -1) + '}'
}
```



### genHandler

```javascript
function genHandler(name, handler){
  if(!handler) { return 'function(){}'; }
  if(Array.isArray(handler)){
    return ('[' + (handler.map(function(handler) { return genHandler(name, handler); }).join(',')) + ']');
  }
  
  var isMethodPath = simplePathRE.test(handler.value);
  var isFunctionExpression = fnExpRE.test(handler.value);
  
  if(!handler.modifiers){
    return isMethodPath || isFunctionExpression ? handler.value : ('function($event){' + (handler.value) + '}');
  } else{
    var code = '';
    var genModifierCode = '';
    var keys = [];
    for(var key in handler.modifiers){
      if(modifierCode[key]){
        genModifierCode += modifierCode[key];
        // 左右
        if(keyCodes[key]){
          keys.push(key);
        }
      } else{
        keys.push(key);
      }
    }
    if(keys.length){
      code += genKeyFilter(keys);
    }
    
    if(genModifierCode){
      code += genModifierCode;
    }
    var handlerCode = isMethodPath ? handler.value + '($event)' : isFunctionExpression ? 
        ('(' + (handler.value) + ')($event)') : handler.value;
    return ('function($event){' + code + handlerCode + '}');
  }
}
```



### genKeyFilter

```javascript
function genKeyFilter(keys){
  return ('if(!("button" in $event)&&' + (keys.map(genFilterCode).join('&&')) + ')return null;');
}
```



### genFilterCode

```javascript
function genFilterCode(key){
  var keyVal = parseInt(key, 10);
  if(keyVal){
    return ('$event.keyCode!==' + keyVal);
  }
  var alias = keyCodes[key];
  return ('_k($event.keyCode,' + (JSON.stringify(key)) + (alias ? ',' + JSON.stringify(alias) : '') + ')');
}
```



### bind$1

```javascript
function bind$1(el, dir){
  el.wrapData = function(code){
    return ('_b(' + code + ',"' + (el.tag) + '",' + (dir.value) + (dir.modifiers && dir.modifiers.prop ? ',true' : '') + ')');
  };
}
```



### baseDirectives

```javascript
var baseDirectives = {
  bind: bind$1,
  cloak: noop
}
```



---



## configurable

```javascript
var warn$3;
var transforms$1;
var dataGenFns;
var platformDirectives$1;
var isPlatformReservedTag$1;
var staticRenderFns;
var onceCount;
var currentOptions;
```



### generate

```javascript
function generate(ast, options){
  // 缓存之前的AST
  var prevStaticRenderFns = staticRenderFns;
  var currentStaticRenderFns = staticRenderFns = [];
  var prevOnceCount = onceCount;
  onceCount = 0;
  currentOptions = options;
  warn$3 = options.warn || baseWarn;
  transforms$1 = pluckModuleFunction(options.modules, 'transformCode');
  dataGenFns = pluckModuleFunction(options.modules, 'genData');
  platformDirectives$1 = options.directives || {};
  isPlatformReservedTag$1 = options.isReservedTag || no;
  var code = ast ? genElement(ast) : '_c("div")';
  staticRenderFns = prevStaticRenderFns;
  onceCount = prevOnceCount;
  
  return {
    render: ('with(this){return ' + code + '}'),
    staticRenderFns: currentStaticRenderFns
  }
}
```



### genElement

```javascript
function genElement(el){
  if(el.staticRoot && !el.staticProcessed){
    return genStatic(el);
  } else if(el.once && !el.onceProcessed){
    return genOnce(el);
  } else if(el.for && !el.forProcessed){
    return genFor(el);
  } else if(el.if && !el.ifProcessed){
    return genIf(el);
  } else if(el.tag === 'template' && !el.slotTarget){
    return genChildren(el) || 'void 0';
  } else if(el.tag === 'slot'){
    return genSlot(el);
  } else{
    // 组件
    var code;
    if(el.component){
      code = genComponent(el.component, el);
    } else{
      var data = el.plain ? undefined : genData(el);
      var children = el.inlineTemplate ? null : genChildren(el, true);
      code = '_c("' + (el.tag) + '"' + (data ? (',' + data) : '') + (children ? (',' + children) : '') + ')';
    }
    for(var i = 0; i < transforms$1.length; i++){
      code = transforms$1[i](el, code);
    }
    return code;
  }
}
```



### genStatic

```javascript
function genStatic(el){
  el.staticProcessed = true;
  staticRenderFns.push(('with(this){return ' + (genElement(el)) + '}'));
  return ('_m(' + (staticRenderFns.length - 1) + (el.staticInFor ? ',true' : '') + ')');
}
```



### genOnce

```javascript
function genOnce(el){
  el.onceProcessed = true;
  if(el.if && !el.ifProcessed){
    return genIf(el);
  } else if(el.staticInFor){
    var key = '';
    var parent = el.parent;
    while(parent){
      if(parent.for){
        key = parent.key;
        break;
      }
      parent = parent.parent;
    }
    if(!key){
      'development' !== 'production' && warn$3(
        'v-once can only be used inside v-for that is keyed. '
      );
      return genElement(el);
    }
    return ('_o(' + (genElement(el)) + ',' + (onceCount++) + (key ? (',' + key) : '') + ')');
  } else{
    return genStatic(el);
  }
}
```



### genIf

```javascript
function genIf(el){
  // 防止递归
  el.ifProcessed = true;
  return genIfConditions(el.ifConditions.slice());
}
```



### genIfConditions

```javascript
function genIfConditions(conditions){
  if(!conditions.length){
    return '_e()';
  }
  
  var condition = conditions.shift();
  if(condition.exp){
    return ('(' + (condition.exp) + ')?' + (genTernaryExp(condition.block)) + ':' + (genIfConditions(conditions)))
  } else{
    return ('' + (genTernaryExp(condition.block)));
  }
  
  function genTernaryExp(el){
    return el.once ? genOnce(el) : genElement(el);
  }
}
```


























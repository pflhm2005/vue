# directives





### directives

```javascript
var directives = {
  create: updateDirectives,
  update: updateDirectives,
  destroy: function unbindDirectives(vnode){
    updateDirectives(vnode, emptyNode);
  }
};
```



### updateDirectives

```javascript
function updateDirectives(oldVnode, vnode){
  if(oldVnode.data.directives || vnode.data.directives){
    _update(oldVnode, vnode);
  }
}
```



### _update

```javascript
function _update(oldVnode, vnode){
  var isCreate = oldVnode = emptyNode;
  var isDestroy = vnode = emptyNode;
  var oldDirs = normalizeDirectives$1(oldVnode.data.directives, oldVnode.context);
  var newDirs = normalizeDirectives$1(vnode.data.directives, vnode.context);
  
  var dirsWithInsert = [];
  var dirsWithPostpatch = [];
  
  var key, oldDir, dir;
  for(key in newDirs){
    oldDir = oldDirs[key];
    dir = newDirs[key];
    if(!oldDir){
      // 绑定新指令
      callHook$1(dir, 'bind', vnode, oldVnode);
      if(dir.def && dir.def.inserted){
        dirsWithInsert.push(dir);
      }
    } else{
      // 更新已存在的指令
      dir.oldValue = oldDir.value;
      callHook$1(dir, 'update', vnode, oldVnode);
      if(dir.def && dir.def.componentUpdated){
        dirsWithPostpatch.push(dir);
      }
    }
  }
  if(dirsWithinsert.length){
    var callInsert = function(){
      for(var i = 0; i < dirsWithInsert.length; i++){
        callHook$1(dirsWithInsert[i], 'inserted', vnode, oldVnode);
      }
    };
    if(isCreate){
      mergeVNodeHook(vnode.data.hook || (vnode.data.hook = {}), 'insert', callInsert);
    } else{
      callInsert();
    }
  }
  if(dirsWithPostpatch.length){
    mergeVNodeHook(vnode.data.hook || (vnode.data.hook = {}), 'postpatch', function(){
      for(var i = 0; i < dirsWithPostpatch.length; i++){
        callHook$1(dirsWithPostpatch[i], 'componentUpdated', vnode, oldVnode);
      }
    });
  }
  
  if(!isCreate){
    for(key in oldDirs){
      if(!newDirs[key]){
        callHook$1(oldDirs[key], 'unbind', oldVnode, oldVnode, isDestroy);
      }
    }
  }
}
```



### normalizeDirectives$1

```javascript
var emptyModifiers = Object.create(null);

function normalizeDirectives$1(dirs, vm){
  var res = Object.create(null);
  if(!dirs){
    return res;
  }
  var i, dir;
  for(i = 0; i < dirs.length; i++){
    dir = dirs[i];
    if(!dir.modifiers){
      dir.modifiers = emptyModifiers;
    }
    res[getRawDirName(dir)] = dir;
    dir.def = resolveAsset(vm.$options, 'directives', dir.name, true);
  }
  return res;
}
```



#### getRawDirName

```javascript
function getRawDirName(dir){
  return dir.rawName || ((dir.name) + '.' + (Object.keys(dir.modifiers || {}).join('.')))
}
```



#### callHook$1

```javascript
function callHook$1(dir, hook, vnode, oldVnode, isDestroy){
  var fn = dir.def && dir.def[hook];
  if(fn){
    fn(vnode.elm, dir, vnode, oldVnode, isDestroy);
  }
}
```



### baseModules

```javascript
var baseModules = [res, directives];
```



### updateAttrs

```javascript
function updateAttrs(oldVnode, vnode){
  if(!oldVnode.data.attrs && !vnode.data.attrs){
    return ;
  }
  var key, cur, old;
  var elm = vnode.elm;
  var oldAttrs = oldVnode.data.attrs || {};
  var attrs = vnode.data.attrs || {};
  // 克隆观测对象
  if(attrs.__ob__){
    attrs = vnode.data.attrs = extend({}, attrs);
  }
  
  for(key in attrs){
    cur = attrs[key];
    old = oldAttrs[key];
    if(old !== cur){
      setAttr(elm, key, cur);
    }
  }
  
  // IE9中设置type会重置type=radio的值
  if(isIE9 && attrs.value !== oldAttrs.value){
    setAttr(elm, 'value', attrs.value);
  }
  for(key in oldAttrs){
    if(attrs[key] == null){
      if(isXlink(key)){
        elm.removeAttributeNS(xlinkNS, getXlinkProp(key));
      } else if(!isEnumeratedAttr(key)){
        elm.removeAttribute(key);
      }
    }
  }
}
```



#### setAttr

```javascript
function setAttr(el, key, value){
  if(isBooleanAttr(key)){
    // 无值属性
    // 例如disabled
    if(isFalsyAttrValue(value)){
      el.removeAttribute(key);
    } else{
      el.setAttribute(key, key);
    }
  } else if(isEnumeratedAttr(key)){
    el.setAttribute(key, isFalsyAttrValue(value) || value === 'false' ? 'false' : 'true');
  } else if(isXlink(key)){
    if(isFalsyAttrValue(value)){
      el.removeAttributeNS(xlinkNS, getXlinkProp(key));
    } else{
      el.setAttributeNS(xlinkNS, key, value);
    }
  } else{
    if(isFalsyAttrValue(value)){
      el.removeAttribute(key);
    } else{
      el.setAttribute(key, value);
    }
  }
}
```



### attrs

```javascript
var attrs = {
  create: updateAttrs,
  update: updateAttrs
};
```



---



### updateClass

```javascript
function updateClass(oldVnode, vnode){
  var el = vnode.elm;
  var data = vnode.data;
  var oldData = oldVnode.data;
  if(!data.staticClass && !data.class && 
     (!oldData || (!oldData.staticClass && !oldData.class))){
    return ;
  }
  
  var cls = genClassForVnode(vnode);
  
  // transitionClass
  var transitionClass = el._transitionClass;
  if(transitionClass){
    cls = concat(cls, stringifyClass(transitionClass));
  }
  
  // 
  if(cls !== el._prevClass){
    el.setAttribute('class', cls);
    el._prevClass = cls;
  }
}
```



### klass

```javascript
var klass = {
  create: updateClass,
  update: updateClass
};
```



### validDivisionCharRE

```javascript
var validDivisionCharRe = /[\w).+\-_$]]/;
```



### parseFilters

```javascript
function parseFilters(exp){
  var inSingle = false;
  var inDouble = false;
  var inTemplateString = false;
  var inRegex = false;
  var curly = 0;
  var square = 0;
  var paren = 0;
  var lastFilterIndex = 0;
  var c, prev, i, expression, filters;
  
  for(i = 0; i < exp.length; i++){
    prev = c;
    c = exp.charCodeAt(i);
    // 特殊符号前没有转义符号代表非特殊
    if(inSingle){
      // 0x5C => 92 => \
      // 39 => '
      if(c === 0x27 && prev !== 0x5C) { inSingle = false; }
    } else if(inDouble){
      // 34 => "
      if(c === 0x22 && prev !== 0x5C) { inDouble = false; }
    } else if(inTemplateString){
      // 96 => `
      if(c === 0x60 && prev !== 0x5C) { inTemplateString = false; }
    } else if(inRegex){
      // 47 => /
      if(c === 0x2f && prev !== 0x5C) { inRegex = false; }
    } else if(
      // 124 => |
      c === 0x7C && 
      exp.charCodeAt(i + 1) !== 0x7C && 
      exp.charCodeAt(i - 1) !== 0x7C &&	
      !curly && !square && !paren
    ){
      if(expression === undefined){
        lastFilterIndex = i + 1;
        expression = exp.slice(0, i).trim();
      } else{
        pushFilter();
      }
    } else{
      // 数值对应符号参考上面的
      switch(c){
        case 0x22:	// "	
          inDouble = true;
          break;
        case 0x27:	// '
          inSingle = true;
          break;
        case 0x60:	// `
          inTemplateString = true;
          break;
        case 0x28:	// (
          paren++;
          break;
        case 0x29:	// )
          paren--;
          break;
        case 0x5B:	// [
          square++;
          break;
        case 0x5D:	// ]
          square--;
          break;
        case 0x7B:	// {
          curly++;
          break;
        case 0x7D:	// }
          curly--;
          break;
      }
      // /
      if(c === 0x2f){
        var j = i - 1;
        // undefined
        var p = (void 0);
        // 找到前面第一个非空格
        for(; j >= 0; j--){
          p = exp.charAt(j);
          if(p !== ' ') { break; }
        }
        if(!p || !validDivisionCharRE.test(p)){
          inRegex = true;
        }
      }
    }
  }
  
  if(expression === undefined){
    expression = exp.slice(0, i).trim();
  } else if(lastFilterIndex !== 0){
    pushFilter();
  }
  
  function pushFilter(){
    (filters || (filters = [])).push(exp.slice(lastFilterIndex, i).trim());
    lastFilterIndex = i + 1;
  }
  
  if(filters){
    for(i = 0; i < filters.length; i++){
      expression = wrapFilter(expression, filters[i]);
    }
  }
  
  return expression;
}
```



### wrapFilter

```javascript
function wrapFilter(exp, filter){
  var i = filter.indexOf('(');
  // 找不到左括号
  if(i < 0){
    // `_f("{filter}")({exp})`
    return ('_f("' + filter + '")(' + exp + ')');
  } else{
    var name = filter.slice(0, i);
    var args = filter.slice(i + 1);
    // 
    return ('_f("' + name + '")(' + exp + ',' + args);
  }
}
```



### baseWarn

```javascript
function baseWarn(msg){
  console.error(("[Vue compiler]: " + msg));
}
```



### pluckModuleFunction

```javascript{
function pluckModuleFunction(modules, key){
  
}
```


















































## makeMap

```javascript
var acceptValue = makeMap('input,textarea,option,select');

// 标签名、属性是否对应
var mustUseProp = function(tag, type, attr){
  return (
    (attr === 'value' && acceptValue(tag)) && type !== 'button' || 
    (attr === 'selected' && tag === 'option') || 
    (attr === 'checked' && tag === 'input') || 
    (attr === 'muted' && tag === 'video')
  )
};

// 枚举属性
var isEnumeratedAttr = makeMap('contenteditable,draggable,spellcheck');

// 布尔值的属性
var isBooleanAttr = makeMap(
  'allowfullscreen,async,autofocus,autoplay,checked,compact,controls,declare,' + 
  'default,defaultchecked,defaultmuted,defaultselected,defer,disabled,' + 
  'enabled,formnovalidate,hidden,indeterminate,inert,ismap,itemscope,loop,multiple,' + 
  'muted,nohref,noresize,noshade,novalidate,nowrap,open,pauseonexit,readonly,' + 
  'required,reversed,scoped,seamless,selected,sortable,translate,' + 
  'truespeed,typemustmatch,visible'
);

var xlinkNS = 'http://www.w3.org/1999/xlink';
```



---



## Xlink



### isXlink

```javascript
// 字符串形式为 => xlink:
var isXlink = function(name){
  return name.charAt(5) === ':' && name.slice(0,5) === 'xlink';
}
```



---



### getXlinkProp

```javascript
// 冒号后面的内容
var getXlinkProp = function(name){
  return isXlink(name) ? name.slice(6, name.length) : '';
}
```



---



### isFalsyAttrValue

```javascript
// val是null、undefined、false
var isFalsyAttrValue = function(val){
  return val == null || val == false;
}
```



---



## genClassForVnode

```javascript
// 给虚拟节点生成class
function genClassForVnode(vnode){
  var data = vnode.data;
  var parentNode = vnode;
  var childNode = vnode;
  while(childNode.componentInstance){
    childNode = childNode.componentInstance._vnode;
    if(childNode.data){
      data = mergeClassData(childNode.data, data);
    }
  }
  while((parentNode = parentNode.parent)){
    if(parentNode.data){
      data = mergeClassData(data, parentNode.data);
    }
  }
  return genClassFromData(data);
}
```



### mergeClassData

```javascript
// 返回一个对象
function mergeClassData(child, parent){
  return {
    staticClass: concat(child.staticClass, parent.staticClass),
    class: child.class ? [child.class, parent.class] : parent.class
  }
}
```



### genClassFromData

```javascript
function genClassFromData(data){
  var dynamicClass = data.class;
  var staticClass = data.staticClass;
  if(staticClass || dynamicClass){
    return concat(staticClass, stringifyClass(dynamicClass));
  }
  return '';
}
```



### concat

```javascript
// a,b => 'a b'
// a => a
// b => b
// !a,!b => ''
function concat(a, b){
  return a ? b ? (a + ' ' + b) : a : (b || '');
}
```



### stringifyClass

```javascript
function stringifyClass(value){
  var res = '';
  if(!value){
    return res;
  }
  // 字符串直接返回
  if(typeof value === 'string'){
    return value;
  }
  // 数组
  if(Array.isArray(value)){
    var stringified;
    for(var i = 0; l = value.length; i < l; i++){
      if(value[i]){
        // 递归解析
        if((stringified = stringifyClass(value[i]))){
          res += stringified + ' ';
        }
      }
    }
    // 去掉末尾的空格
    return res.slice(0,-1);
  }
  // 对象
  if(isObject(value)){
    for(var key in value){
      // 键拼接
      if(value[key]) { res += key + ' '; } 
    }
    return res.slice(0, -1);
  }
  return res;
}
```



---

## is



### namespaceMap

```javascript
var namespaceMap = {
  svg: 'http://www.w3.org/2000/svg',
  math: 'http://www/w3/org/1998/Math/MathML'
}
```



### isHTMLTag

```javascript
var isHTMLTag = makeMap(
  'html,body,base,head,link,meta,style,title,' + 
  'address,article,aside,footer,header,h1,h2,h3,h4,h5,h6,hgroup,nav,section,' + 
  'div,dd,dl,dt,figcaption,figure,hr,img,li,main,ol,p,pre,ul,' + 
  'a,b,abbr,bdi,bdo,br,cite,code,data,dfn,em,i,kbd,mark,q,rp,rt,rtc,ruby,' + 
  's,samp,small,span,strong,sub,sup,time,u,var,wbr,area,audio,map,track,video,' + 
  'embed,object,param,source,canvas,script,noscript,del,ins,' + 
  'caption,col,colgroup,table,thead,tbody,td,th,tr,' + 
  'button,datalist,fieldset,form,input,label,legend,meter,optgroup,option,' + 
  'output,progress,select,textarea,' + 
  'details,dialog,menu,menuitem,summary,' + 
  'content,element,shadow,template'
);
```



### isSVG

```javascript
var isSVG = makeMap(
  'svg,animate,circle,clippath,cursor,defs,desc,ellipse,filter,font-face,' + 
  'foreignObject,g,glyph,image,line,marker,mask,missing-glyph,path,pattern,' + 
  'polygon,polyline,rect,switch,symbol,text,textpath,tspan,use,view',true
);
```



### isPreTag

```javascript
var isPreTag = function(tag) { return tag === 'pre'; };
```



### isReservedTag

```javascript
// 内置tag
var isReservedTag = function(tag){
  return isHTMLTag(tag) || isSVG(tag);
};
```



### getTagNamespace

```javascript
function getTagNamespace(tag){
  if(isSVG(tag)){
    return 'svg';
  }
  //
  if(tag === 'math'){
    return 'math';
  }
}
```



### unknownElementCache

```javascript
var unknownElementCache = Object.create(null);
```



### isUnknownElement

```javascript
function isUnknownElement(tag){
  if(!inBrower){
    return true;
  }
  if(isReservedTag){
    return false;
  }
  tag = tag.toLowerCase();
  // 取缓存 
  if(unknownElementCache[tag] != null){
    return unknownElementCache[tag];
  }
  var el = document.createElement(tag);
  //  包含'-'
  if(tag.indexOf('-') > -1){
    return (unknownElementCache[tag] = (
      el.constructor === window.HTMLUnknownElement || 
      el.constructor === window.HTMLElement
    ));
  } else{
    return (unknownElementCache[tag] = /HTMLUnknownElement/.test(el.toString()));
  }
}
```



## query

```javascript
// 返回el选中的DOM节点
function query(el){
  if(typeof el === 'string'){
    var selected = document.querySelector(el);
    if(!selected){
      'development' !== 'production' && warn(
        'Cannot find element: ' + el
      );
      return document.createElement('div');
    }
    return selected;
  } else{
    return el;
  }
}
```



---



## nodeOps



```javascript
var nodeOps = Object.freeze({
  createElement: createElement$1,
  createElementNS: createElementNS,
  createTextNode: createTextNode,
  createComment: createComment,
  insertBefore: insertBefore,
  removeChild: removeChild,
  appendChild: appendChild,
  parentNode: parentNode,
  nextSibling: nextSibling,
  tagName: tagName,
  setTextContent: setTextContent,
  setAttribute: setAttribute
})
```



### createElement$1

```javascript
function createElement$1(tagName, vnode){
  var elm = document.createElement(tagName);
  if(tagName !== 'select'){
    return elm;
  }
  if(vnode.data && vnode.data.attrs && vnode.data.attrs.multiple !== undefined){
    elm.setAttribute('multiple','multiple');
  }
  return elm;
}
```



### createElementNS

```javascript
function createElementNS(namespace, tagName){
  return document.createElementNS(namespaceMap[namespace], tagName);
}
```



### createTextNode

```javascript
function createTextNode(text){
  return document.createTextNode(text);
}
```



### createComment

```javascript
function createComment(text){
  return document.createComment(text);
}
```



### insertBefore

```javascript
function insertBefore(parentNode, newNode, referenceNode){
  parentNode.insertBefore(newNode, referenceNode);
}
```



### removeChild

```javascript
function removeChild(node, child){
  node.removeChild(child);
}
```



### appendChild

```javascript
function appendChild(node, child){
  node.appendChild(child);
}
```



### parentNode

```javascript
function parentNode(node){
  return node.parentNode;
}
```



### nextSibling

```javascript
function nextSibling(node){
  return node.parentNode;
}
```



### tagName

```javascript
function tagName(node){
  return node.tagName;
}
```



### setTextContent

```javascript
function setTextContent(node, text){
  node.textContent = text;
}
```



### setAttribute

```javascript
function setAttribute(node, key, val){
  node.setAttribute(key, val);
}
```



## ref



```javascript
var ref = {
  create: function create(_, vnode){
    registerRef(vnode);
  },
  update: function update(oldVnode, vnode){
    if(oldVnode.data.ref !== vnode.data.ref){
      registerRef(oldVnode, true);
      registerRef(vnode);
    }
  },
  destroy: function destroy(vnode){
    registerRef(vnode, true);
  }
};
```



### registerRef

```javascript
function registerRef(vnode, isRemoval){
  var key = vnode.data.ref;
  if(!key) { return }
  var vm = vnode.context;
  var ref = vnode.componentInstance || vnode.elm;
  var refs = vm.$refs;
  if(isRemoval){
    if(Array.isArray(refs[key])){
      remove(refs[key], ref);
    } else if(refs[key] === ref){
      refs[key] = undefined;
    }
  } else{
    if(vnode.data.refInFor){
      if(Array.isArray(refs[key]) && refs[key].indexOf(ref) < 0){
        refs[key].push(ref);
      } else{
        refs[key] = [ref];
      }
    } else{
      refs[key] = ref;
    }
  }
}
```






































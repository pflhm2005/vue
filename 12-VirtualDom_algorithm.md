# Virtual DOM patching algorithm





```javascript
var emptyNode = new VNode('', {}, []);

var hooks = ['create', 'activate', 'update', 'remove', 'destroy'];

function isUndef(v){
  return v === undefined || v === null;
}

function isDef(v){
  return v !== undefined && v !== null;
}

function isTrue(v){
  return v === true;
}
```



### sameVnode

```javascript
function sameVnode(a, b){
  return (
    a.key === b.key && 
    a.tag === b.tag && 
    a.isComment === b.isComment && 
    isDef(a.data) === isDef(b.data) && 
    sameInputType(a, b)
  );
}
```



### sameInputType

```javascript
function sameInputType(a, b){
  if(a.tag !== 'input'){ return true }
  var i;
  // a.data.attrs.types
  var typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type;
  var typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type;
  return typeA === typeB;
}
```



### createKeyToOldIdx

```javascript
function createKeyToOldIdx(children, beginIdx, endIdx){
  var i, key;
  var map = {};
  for(i = beginIdx; i < endIdx; ++i){
    key = children[i].key;
    if(isDef(key)){ map[key] = i; }
  }
  return map;
}
```



### createPatchFunction

```javascript
function createPatchFunction(backend){
  var i, j;
  var cbs = {};
  
  var modules = backend.modules;
  var nodeOps = backend.nodeOps;
  
  for(var i = 0; i < hooks.length; ++i){
    cbs[hook[i]] = [];
    for(j = 0; j < modules.length; ++j){
      if(isDef(modules[j][hooks[i]])){
        cbs[hooks[i]].push(modules[j][hooks[i]]);
      }
    }
  }
  
  // emptyNodeAt
  function emptyNodeAt(elm){
    return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm);
  }
  
  // createRmcb
  function createRmcb(childElm, listeners){
    function remove$$1(){
      if(--remove$$1.listeners === 0){
        removeNode(childElm);
      }
    }
    remove$$1.listeners = listeners;
    return remove$$1;
  }
  
  // 获取父元素移除该DOM节点
  function removeNode(el){
    var parent = nodeOps.parentNode(el);
    if(isDef(parent)){
      nodeOps.removeChild(parent, el);
    }
  }
  
  var inPre = 0;
  
  // createElm
  function createElm(vnode, insertedVnodeQueue, parentElm, refElm, nested){
    vnode.isRootInsert = !nested;
    if(createComponent(vnode, insertedVnodeQueue, parentElm, refElm)){
      return ;
    }
    
    var data = vnode.data;
    var children = vnode.children;
    var tag = vnode.tag;
    if(isDef(tag)){
      if(data && data.pre){
        inPre++;
      }
      if(!inPre && !vnode.ns && 
         !(config.ignoredElements.length && config.ignoredElements.indexOf(tag) > -1) && 
         config.isUnknownElement(tag)
        ){
        warn(
          'Unknown custom element: <' + tag + '> - did you ' + 
          'register the component correctly? For recursive components, ' + 
          'make sure to provide the "name" option.',
          vnode.context
        );
      }
      vnode.elm = vnode.ns ? 
        nodeOps.createElementNS(vnode.ns, tag) : 
      	nodeOps.createElement(tag, vnode);
      setScope(vnode);
      
      createChildren(vnode, children, insertedVnodeQueue);
      if(isDef(data)){
        invokeCreateHooks(vnode, insertedVnodeQueue);
      }
      insert(parentElm, vnode.elm, refElm);
      
      if('development' !== 'production' && data && data.pre){
        inPre--;
      }
    } else if(isTrue(vnode.isComment)){
      vnode.elm = nodeOps.createComment(vnode.text);
      insert(parentElm, vnode.elm, refElm);
    } else{
      vnode.elm = nodeOps.createTextNode(vnode.text);
      insert(parentElm, vnode.elm, refElm);
    }
  }
  
  // createComponent
  function createComponent(vnode, insertedVnodeQueue, parentElm, refElm){
    var i = vnode.data;
    if(isDef(i)){
      var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
      if(isDef(i = i.hook) && isDef(i = i.init)){
        i(vnode, false, parentElm, refElm);
      }
      
      // 
      if(isDef(vnode.componentInstance)){
        initComponent(vnode, insertedVnodeQueue);
        if(isTrue(isReactivated)){
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
        }
        return true;
      }
    }
  }
  
  // initComponent
  function initComponent(vnode, insertedVnodeQueue){
    if(isDef(vnode.data.pendingInsert)){
      insertedVnodeQueue.push.apply(insertedVnodeQueue, vnode.data.pendingInsert);
    }
    vnode.elm = vnode.componentInstance.$el;
    if(isPatchable(vnode)){
      invokeCreateHooks(vnode, insertedVnodeQueue);
      setScope(vnode);
    } else{
      registerRef(vnode);
      insertedVnodeQueue.push(vnode);
    }
  }
  
  // reactivateComponent
  function reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm){
    var i;
    // 
    var innerNode = vnode;
    while(innerNode.componentInstance){
      innerNode = innerNode.componentInstance._vnode;
      if(isDef(i = innerNode.data) && isDef(i = i.transition)){
        for(i = 0; i < cbs.activate.length; ++i){
          cbs.activate[i](emptyNode, innerNode);
        }
        insertedVnodeQueue.push(innerNode);
        break;
      }
    }
    insert(parentElm, vnode.elm, refElm);
  }
  
  // insert
  function insert(parent, elm, ref){
    if(isDef(parent)){
      if(isDef(ref)){
        nodeOps.insertBefore(parent, elm, ref);
      } else{
        nodeOps.appendChild(parent, elm);
      }
    }
  }
  
  // createChildren
  function createChildren(vnode, children, insertedVnodeQueue){
    if(Array.isArray(children)){
      for(var i = 0; i < children.length; ++i){
        createElm(children[i], insertedVnodeQueue, vnode.elm, null, true);
      }
    } else if(isPrimitive(vnode.text)){
      nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(vnode.text));
    }
  }
  
  // isPatchable
  function isPatchable(vnode){
    while(vnode.componentInstance){
      vnode = vnode.componentInstance._vnode;
    }
    return isDef(vnode.tag);
  }
  
  // invokeCreateHooks
  function invokeCreateHooks(vnode, insertedVnodeQueue){
    for(var i$1 = 0; i$1 < cbs.create.length; ++i$1){
      cbs.create[i$1](emptyNode, vnode);
    }
    i = vnode.data.hook;
    if(isDef(i)){
      if(isDef(i.create)){ i.create(empty, vnode); }
      if(isDef(i.insert)){ insertedVnodeQueue.push(vnode); }
    }
  }
  
  // setScope
  function setScope(vnode){
    var i;
    var ancestor = vnode;
    while(ancestor){
      if(isDef(i = ancestor.context) && isDef(i = i.$options._scopeId)){
        nodeOps.setAttribute(vnode.elm, i, '');
      }
      ancestor = ancestor.parent;
    }
    
    // slot的内容也必须含有scopeId
    if(isDef(i = activeInstance) && i !== vnode.context && isDef(i = i.$options._scopeId)){
      nodeOps.setAttribute(vnode.elm, i, '');
    }
  }
  
  // addVnodes
  function addVnodes(parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue){
    for(; startIdx <= endIdx; ++startIdx){
      createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm);
    }
  }
  
  // invokeDestroyHook
  function invokeDestroyHook(vnode){
    var i, j;
    var data = vnode.data;
    if(isDef(data)){
      if(isDef(i = data.hook) && isDef(i = i.destroy)) { i(vnode); }
      for(i = 0; i < cbs.destroy.length; ++i){ cbs.destroy[i](vnode); }
    }
    if(isDef(i = vnode.children)){
      for(j = 0; j < vnode.children.length; ++j){
        invokeDestroyHook(vnode.children[j]);
      }
    }
  }
  
  // removeVnodes
  function removeVnodes(parentElm, vnodes, startIdx, endIdx){
    for(; startIdx <= endIdx; ++startIdx){
      var ch = vnodes[startIdx];
      if(isDef(ch)){
        if(isDef(ch.tag)){
          removeAndInvokeRemoveHook(ch);
          invokeDestroyHook(ch);
        } else{
          removeNode(ch.elm);
        }
      }
    }
  }
  
  // removeAndInvokeRemoveHook
  function removeAndInvokeRemoveHook(vnode, rm){
    if(isDef(rm) || isDef(vnode.data)){
      var listeners = cbs.remove.length + 1;
      if(isDef(rm)){
        rm.listeners += listeners;
      } else{
        rm = createRmcb(vnode.elm, listeners);
      }
      // 递归调用子组件
      if(isDef(i = vnode.componentInstance) && isDef(i = i._vnode) && isDef(i.data)){
        removeAndInvokeRemoveHook(i, rm);
      }
      for(i = 0; i < cbs.remove.length; ++i){
        cbs.remove[i](vnode, rm);
      }
      if(isDef(i = vnode.data.hook) && isDef(i = i.remove)){
        i(vnode, rm);
      } else{
        rm();
      }
    } else{
      removeNode(vnode.elm);
    }
  }
  
  // updateChildren
  function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly){
    var oldStartIdx = 0;
    var newStartIdx = 0;
    var oldEndIdx = oldCh.length - 1;
    var oldStartVnode = oldCh[0];
    var oldEndVnode = oldCh[oldEndIdx];
    var newEndIdx = newCh.length - 1;
    var newStartVnode = newCh[0];
    var newEndVnode = newCh[newEndIdx];
    var oldKeyToIdx, idxInOld, elmToMove, refElm;
    
    // 维持transition-group动画时的定位属性
    var canMove = !removeOnly;
    while(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx){
      if(isUndef(oldStartVnode)){
        oldStartVnode = oldCh[++oldStartIdx];
      } else if(isUndef(oldEndVnode)){
        oldEndVnode = oldCh[--oldEndIdx];
      } else if(sameVnode(oldStartVnode, newStartVnode)){
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      } else if(sameVnode(oldEndVnode, newEndVnode)){
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if(sameVnode(oldStartVnode, newEndVnode)){
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps, nextSibling(oldEndVnode, elm));
        oldStartVnode = oldCh[++oldStartIdx];
        newEndIdx = newCh[--newEndIdx];
      } else if(sameVnode(oldEndVnode, newStartVnode)){
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      } else{
        if(isUndef(oldKeyToIdx)){ oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); }
        idxInOld = isDef(newStartVnode.key) ? oldKeyToIdx[newStartVnode.key] : null;
      }
    }
  }
}
```


































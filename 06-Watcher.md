# Watcher



## 初始化变量

```javascript
var queue = [];
var has = {};
var circular = {};
var waiting = false;
var flushing = false;
// 用来做循环的
var index = 0;
```



### resetSchedulerState

```javascript
// 重置状态
function resetSchedulerState(){
  queue.length = 0;
  has = {};
  {
    circular = {};
  }
  waiting = flushing = false;
}
```



### flushSchedulerQueue

```javascript
// flush队列并遍历执行watchers
function flushSchedulerQueue(){
  flushing = true;
  var watcher, id, vm;
  
  // 先进行排序
  queue.sort(function(a,b){return a.id - b.id; });
  
  // 不缓存长度 执行中可能会有额外的watcher加入
  for(index = 0; index < queue.length; index++){
    watcher = queue[index];
    id = watcher.id;
    has[id] = null;
    watcher.run();
    
    // 开发者模式
    if('development' !== 'production' && has[id] != null){
      circular[id] = (circular[id] || 0) + 1;
      if(circular[id] > config._maxUpdateCount){
        warn(
          'You may have an infinite update loop' + (
          watcher.user 
          ? ('in watcher with expression "' + (watcher.expression) + '"')
          : 'in a component render function.'
          ),watcher.vm
        );
        break;
      }
    }
  }
  
  // 更新钩子调用前先清空任务队列
  var oldQueue = queue.slice();
  resetSchedulerState();
  
  // 调用钩子
  index = oldQueue.length;
  while(index--){
    watcher = oldQueue[index];
    vm = watcher.vm;
    if(vm._watcher === watcher && vm._isMounted){
      callHook(vm, 'updated');
    }
  }
  
  // 开发者模式钩子
  if(devtools && config.devtools){
    devtools.emit('flush');
  }
}
```



### queueWatcher

```javascript
// 将watcher推入队列
function queueWatcher(watcher){
  var id = watcher.id;
  if(has[id] == null){
    has[id] = true;
    // 重复ID仅在未flush时加入
    if(!flushing){
      queue.push(watcher);
    }
    // 如果处于flush状态
    else{
      var i = queue.length - 1;
      while(i >= 0 && queue[i].id > watcher.id){
        i--;
      }
      queue.splice(Math.max(i, index) + 1, 0, watcher);
    }
    
    // 如果不处于waiting状态 下一个microtask递归执行本函数
    if(!waiting){
      waiting = true;
      nextTick(flushSchedulerQueue);
    }
  }
}
```



---



## Watcher



### uid$2

```javascript
var uid$2 = 0;
```



### Watcher

```javascript
// construtor
var Watcher = function Watcher(vm, expOrFn, cb, options){
  this.vm = vm;
  vm._watchers.push(this);
  
  // 可选参数
  if(options){
    this.deep = !!options.deep;
    this.user = !!options.user;
    this.lazy = !!options.lazy;
    this.sync = !!options.sync;
  } else{
    this.deep = this.user = this.lazy = this.sync = false;
  }
  
  this.cb = cb;
  // batching/批量处理?
  this.id = ++uid$2;
  this.active = true;
  // lazy watchers???
  this.dirty = this.lazy;
  this.deps = [];
  this.newDeps = [];
  this.depIds = new _Set();
  this.newDepIds = new _Set();
  this.expression = expOrFn.toString();
  // 解析表达式for getter
  if(typeof expOrFn === 'function'){
    this.getter = expOrFn;
  } else{
    this.getter = parsePath(expOrFn);
    if(!this.getter){
      this.getter = function(){};
      'development' !== 'production' && warn(
        'Failed watching path: "' + expOrFn + '" ' + 
        'Watcher only accepts simple dot-delimited paths. ' + 
        'For full control, use a function instead.',
        vm
      );
    }
  }
  this.value = this.lazy ? undefined : this.get();
};
```



### Watcher.prototype



#### get

```javascript
// 评估getter 重新收集依赖
Watcher.prototype.get = function get(){
  pushTarget(this);
  var value;
  var vm = this.vm;
  if(this.user){
    try{
      value = this.getter.call(vm, vm);
    } catch(e){
      handleError(e, vm, ('getter for watcher "' + (this.expression) + '"'));
    }
  } else{
    value = this.getter.call(vm, vm);
  }
  
  // touch每个值 
  if(this.deep){
    traverse(value);
  }
  
  popTarget();
  this.cleanupDeps();
  return value;
}
```



#### addDep

```javascript
// 给指令添加一个依赖
Watcher.prototype.addDep = function addDep(dep){
  var id = dep.id;
  if(!this.newDepIds.has(id)){
    this.newDepIds.add(id);
    this.newDeps.push(dep);
    if(!this.depIds.has(id)){
      dep.addSub(this);
    }
  }
};
```



#### cleanupDeps

```javascript
// 依赖收集前的清空
Watcher.prototype.cleanupDeps = function cleanupDeps(){
  // 保存this引用
  var this$1 = this;
  var i = this.deps.length;
  while(i--){
    var dep = this$1.deps[i];
    if(!this$1.newDepIds.has(dep.id)){
      dep.removeSub(this$1);
    }
  }
  
  // depIds <= newDepIds
  var tmp = this.depIds;
  this.depIds = this.newDepIds;
  this.newDepIds = tmp;
  // 清空所有成员
  this.newDepIds.clear();
  tmp = this.deps;
  // deps <= newDeps
  this.deps = this.newDeps;
  this.newDeps = tmp;
  // 清空
  this.newDeps.length = 0;
}
```



#### update

```javascript
// 订阅者接口
// 依赖变动时会被调用
Watcher.prototype.update = function update(){
  if(this.lazy){
    this.dirty = true;
  } else if(this.sync){
    this.run();
  } else{
    queueWatcher(this);
  }
};
```



#### run

```javascript
// 任务接口
Watcher.prototype.run = function run(){
  if(this.active){
    // 获取新值
    var value = this.get();
    // 值改变、复杂数据类型、deep属性为true
    if(value !== this.value || isObject(value) || this.deep){
      // 设置新值
      var oldValue = this.value;
      this.value = value;
      if(this.user){
        try{
          this.cb.call(this.vm, value, oldValue);
        } catch(e){
          handleError(e, this.vm, ('callback for watcher "' + (this.expression) + '"'));
        }
      } else{
        this.cb.call(this.vm, value, oldValue);
      }
    }
  }
};
```



#### evaluate

```javascript
// 求值
// 只针对lazy属性的watcher
Watcher.prototype.evaluate = function evaluate(){
  this.value = this.get();
  this.dirty = false;
};
```



#### depend

```javascript
Watcher.prototype.depend = function depend(){
  var this$1 = this;
  var i = this.deps.length;
  while(i--){
    this$1.deps[i].depend();
  }
};
```



#### teardown

```javascript
// 拆解依赖
Watcher.prototype.teardown = function teardown(){
  var this$1 = this;
  if(this.active){
    // 从被观测列表中移除
    // 如果vm被摧毁则跳过
    if(!this.vm_isBeingDestroyed){
      remove(this.vm._watchers, this);
    }
    var i = this.deps.length;
    while(i--){
      this$1.deps[i].removeSub(this$1);
    }
    this.active = false;
  }
};
```



#### traverse

```javascript
// 递归traverse一个对象
// 保证对象中所有嵌套属性作为'deep'依赖被收集
// 只对复杂数据类型操作
var seenObjects = new _Set();

function traverse(val){
  // 清空依赖集合
  seenObjects.clear();
  _traverse(val, seenObject);
}

function _traverse(val, seen){
  var i, keys;
  var isA = Array.isArray(val);
  // 非数组、对象或不可扩展
  if((!isA && !isObject(val)) || !Object.isExtensible(val)){
    return ;
  }
  // 将id添加到空的Set对象中
  if(val.__ob__){
    var depId = val.__ob__.dep.id;
    // 如果已有对应ID 返回
    if(seen.has(depId)){
      return ;
    }
    seen.add(depId);
  }
  if(isA){
    i = val.length;
    while(i--){_traverse(val[i], seen); }
  } else{
    keys = Object.keys(val);
    i = keys.length;
    while(i--){_traverse(val[keys[i]], seen); }
  }
}
```
































# 不知道定什么标题







## strats



```javascript
var strats = config.optionMergeStrategies;

{
  strats.el = strats.propsData = function(parent, child, vm, key){
    if(!vm){
      warn(
        'option /"' + key + "\" can only be used during instance " + 
        'creation with the `new` keyword.'
      );
    }
    return defaultStart(parent, child);
  };
}
```



### mergeData



```javascript
// 两个对象 form => to
function mergeData(to,from){
  if(!from){return to}
  var key, toVal, fromVal;
  var keys = Object.keys(from);
  for(var i = 0; i < keys.length; i++){
    key = keys[i];
    toVal = to[key];
    fromVal = from[key];
    // hasOwnProperty.call(to,key)
    // 如果to中没有key键 动态添加属性
    if(!hasOwn(to,key)){
      set$1(to, key, fromVal);
    }
    // 如果值是对象 递归解析
    else if(isPlainObject(toVal) && isPlainObject(fromVal)){
      mergeData(toVal, fromVal);
    }
  }
  return to;
}
```



### strats.data



```javascript
strats.data = function(parentVal, childVal, vm){
  if(!vm){
    if(!childVal){
      return parentVal;
    }
    // 组件的data定义必须为函数
    // 否则所有组件共享一个对象
    if(typeof childVal !== 'function'){
      'development' !== 'production' && warn(
      	'The "data" option should be a function ' + 
        'that return a per-instance value in component ' + 
        'definitions.',vm);
      return parentVal;
    }
    // 未提供父元素值 
    if(!parentVal){
      return childVal;
    }
    
    // 两个都必须是函数
    return function mergeDataFn(){
      return function margeData(childVal.call(this),parentVal.call(this));
    }
  }
  
  // childVal => data:function(){return {...}}
  else if(parentVal || childVal){
    return function mergedInstanceDataFn(){
      var instanceData = typeof childVal === 'function' ? 
          childVal.call(vm) : childVal;
      var defaultData = typeof parentVal === 'function' ? 
          parentVal.call(vm) : "undefined";
      if(instanceData){
        return mergeData(instanceData,defaultData)
      }
      // 如果没有传props 默认data取父组件的
      else{
        return defaultData;
      }
    }
  }
};
```



### strats.watch



```javascript
//观察者
//观察者不应该被另一个重写，所以将他们合为数组
strats.watch = function(parentVal, childVal){
  //必须传满2个参数
  if(!childVal){return parentVal}
  if(!parentVal){return childVal}
  
  var ret = {};
  //ret现在是父对象
  extend(ret, parentVal);
  //遍历子对象键
  for(var key in childVal){
    //
    var parent = ret[key];
    var child = childVal[key];
    //父对象中如果有子对象属性 包装成对象
    if(parent && !Array.isArray(parent)){
      parent = [parent];
    }
    //拼接属性
    ret[key] = parent ? parent.concat(child) : [child];
  }
  return ret;
}
```



### strats.props/methods/computed



```javascript
strats.pros = 
strats.methods = 
strats.computed = 
function(parentVal, childVal){
  if(!childVal){ return parentVal }
  if(!parentVal){ return childVal }
  var ret = Object.create(null);
  extend(ret, parentVal);
  extend(ret, childVal);
  return ret;
}
```



### defaultStrat



```javascript
var defaultStrat = function(parentVal, childVal){
  return childVal === undefined ? parentVal :childVal;
}
```



### checkComponents



```javascript
//判断是否用内置指令或者标签作为组件ID
function checkComponents(options){
  for(var key in options.components){
    var lower = key.toLowerCase();
    if(isBuiltInTag(lower) || config.is ReservedTag(lower)){
      warn('Do not use built-in or reserved HTML elements as component id: ' + key);
    }
  }
}
```



## merge



### mergeHook



```javascript
// 返回数组形式的parentVal+childVal
function mergeHook(parentVal, childVal){
  return childVal ? (parentVal ? parentVal.concat(childVal) : Array.isArray(childVal) ? 
    childVal : [childVal]) : parentVal
}
```



### mergeAssets



```javascript
// 原型链 res => parentVal(null)
// 返回 ChildVal 链=> parentVal
function mergeAssets(parentVal, childVal){
  var res = Object.create(parentVal || null);
  return childVal ? extend(res, childVal) : res
}
```



### mergeOptions	



```javascript
//合并参数
function mergeOptions(parent, child, vm){
  //检查是否规范
  checkComponents(child);
  normalizeProps(child);
  normalizeDirectives(child);
  //不知道这是个啥
  var extendsFrom = child.extends;
  if(extendsFrom){
    //递归处理什么 暂时不动
    parent = typeof extendsFrom === 'function' ? 
    mergeOptions(parent,extendsFrom.options,vm) : 
    mergeOptions(parent,extendFrom,vm);
  }
  //混入
  if(child.mixins){
    for(var i = 0, l = child.mixins.length; i < l; i++){
      var mixin = child.mixins[i];
      //Vue$3代表vue的构造函数
      if(mixin.prototype instanceof Vue$3){
        mixin = mixin.options;
      }
      parent = mergeOptions(parent, mixin, vm);
    }
  }
  var option = {};
  var key;
  for(key in parent){
    mergeField(key);
  }
  for(key in child){
    if(!hasOwn(parent, key)){
      mergeField(key);
    }
  }
  
  function mergeField(key){
    var strat = strats[key] || defaultStrat;
    options[key] = strat(parent[key], child[key], vm, key);
  }
  return options
}
```





## normalize



### normalizeProps



```javascript
//确保options.props参数是规范的
function normalizeProps(options){
  var props = options.props;
  if(!props) {return }
  var res = {};
  var i, val, name;
  //props是数组
  if(Array.isArray(props)){
    i = props.length;
    while(i--){
      val = props[i];
      //值必须是字符串
      if(typeof val === 'string'){
        name = camelize(val);
        res[name] = {type: null};
      }
      else{
        warn('props must be string when using array syntax.');
      }
    }
  }
  //是对象
  else if(isPlainObject(props)){
    for(var key in props){
      val = props[key];
      name = camelize(key);
      //值不是对象转换为对象{type:val}
      res[name] = isPlainObject(val) ? val : {type: val};
    }
  }
  options.props = res;
}
```



### normalizeDirectives



```javascript
//原生函数转换为对象的值
function normalizeDirectives(options){
  var dirs = options.directives;
  if(dirs){
    for(var key in dirs){
      //dirs = {key: fn} => dirs = {key:{bind:fn, update:fn}}
      var def = dirs[key];
      if(typeof def === 'function'){
        dirs[key] = {bind: def, update: def};
      }
    }
  }
}
```



## prop



### resolveAsst



```javascript
//决议(resolve)一个asset
//子实例必须在assets的原型链上面
function resolveAsset(options, type, id, warnMissing){
  if(typeof id !== 'string'){
    return
  }
  var assets = options[type];
  //先查询本地属性
  if(hasOwn(assets, id)){return assets[id]}
  //驼峰处理
  var camelizedId = camelize(id);
  if(hasOwn(assets, camelizedId)){return assets[camelizedId]}
  //首字符大写处理
  var PascalCaseId = capitalize(camelizedId);
  if(hasOwn(assets, PascalCaseId)){return assets[PacalCaseId]}
  //遍历原型链
  var res = assets[id] || assets[camelizedId] || assets[PascalCaseId];
  if('development' !== 'production' && warnMissing && !res){
    warn('Failed to resolve' + type.slice(0,-1) + ': ' + id, options);
  }
  return res;
}
```



### validateProp



```javascript
function validateProp(key, propOptions, propsData, vm){
  var prop = propOptions[key];
  var absent = !hasOwn(propsData, key);
  var value = propsData[key];
  //处理props的布尔值
  if(isType(Boolean, prop.type)){
    if(absent && !hasOwn(prop, 'default')){
      value = false;
    }
    else if(!isType(String, prop.type) 
            && (value === '' || value === hyphenate(key))){
      value = true;
    }
  }
  if(value === undefined){
    value = getPropDefaultValue(vm, prop, key);
    //
    var prevShouldConvert = observerState.shouldConvert;
    observerState.shouldConvert = true;
    observe(value);
    observerState.shouldConvert = prevShouldConvert;
  }
  assertProp(prop, key, value, vm, absent);
  return value;
}
```



### getPropDefaultValue



```javascript
//获取默认的prop值
function getPropDefaultValue(vm, prop, key){
  //没有default直接返回undefined
  if(!hasOwn(prop, 'default')){
    return undefined;
  }
  var def = prop.default;
  //警告没有默认值提供给对象&数组
  if('development' !== 'production' && isObject(def)){
    warn('Invalid default value for prop "' + key + '": ' +
        'Props with type Object/Array must use a factory function ' + 
        'to return the default value.'), vm);
  }
  //
  if(vm && vm.$options.propsData && 
     vm.$options.propsData[key] === undefined && 
     vm._props[key] !== undefined){
    return vm._props[key];
  }
  //
  return typeof def === 'function' && 
    getType(prop.type) !== 'Function' ? def.call(vm) : def;
}
```



## assert



### assertProp



```javascript
//判断prop是否有效
function assertProp(prop, name, value, vm, absent){
  if(prop.required && absent){
    warn('Missing required prop: "' + name + '"', vm);
    return
  }
  if(value == null && !prop.required){
    return
  }
  var type = prop.type;
  var valid = !type || type === true;
  var expectedTypes = [];
  if(type){
    if(!Array.isArray(type)){
      type = [type];
    }
    for(var i = 0; i < type.length && !valud; i++){
      var assertedType = assertType(value, type[i]);
      expectedTypes.push(assertedType.expectedType || '');
      valid = assertedType.valid;
    }
  }
  if(!valid){
    warn('Invalid prop: type check failed for prop "' + name '".' + 
        ' Expected ' + expectedTypes.map(capitalize).join(',') + 
      ', got ' + Object.prototype.toString.call(value).slice(8,-1) + '.', vm);
    return ;
  }
  var validator = prop.validator;
  if(validator){
    if(!validator(value)){
      warn('Invalid prop: custom validator check failed for prop "' + name '".', vm);
    }
  }
}
```



### assertType



```javascript
//判断值类型
function assertType(value, type){
  var valid;
  var expectedType = getType(type);
  if(expectedType === 'String'){
    valid = typeof value === (expectedType = 'string');
  } else if(expectedType === 'Number'){
    valid = typeof value === (expectedType = 'number');
  } else if(expectedType === 'Boolean'){
    valid = typeof value === (expectedType = 'boolean');
  } else if(expectedType === 'Function'){
    valid = typeof value === (expectedType = 'function');
  } else if(expectedType === 'Object'){
    valid = isPlainObject(value);
  } else if(expectedType === 'Array'){
    valid = Array.isArray(value);
  } else{
    valid = value instanceof type;
  }
  return {
    valid: valid,
    expectedType: expectedType
  }
}
```



### getType



```javascript
//使用函数的toString方法判断是否是内置类型
//返回函数名
function getType(fn){
  var match = fn && fn.toString().match(/^\s*function (\w+)/);
  return match && match[1];
}
```



### isType



```javascript
function isType(type, fn){
  if(!Array.isArray(fn)){
    return getType(fn) === getType(type)
  }
  for(var i = 0, len = fn.length; i < len; i++){
    if(getType(fn[i]) === getType(Type)){
      return true;
    }
  }
  return false;
}
```



### handleError



```javascript
function handleError(err, vm, info){
  if(config.errorHandler){
    config.errorHandler.call(null, err, vm, info);
  } else {
    warn('Error in ' + info + ':', vm);
  }
  if(inBrowser && typeof console !== 'undefined'){
    console.error(err);
  } else {
    throw err;
  }
}
```



## Proxy



```javascript
var initProxy;
//所有内置原生对象
var allowedGlobals = makeMap(
'Infinity,undefined,NaN,isFinite,isNaN,parseFloat,parseInt,' + 
  'decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' + 
  'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Int1,' + 
  'require' //兼容webpack/Browserify
);
//这地方真奇怪 之前都是用'"' 这里偏偏用"\""转义一下
var warnNonPresent = function(target, key){
  warn('Property or method "' + key + '" is not defined on the instance but ' + 
      'referenced during render. Make sure to declare reactive data ' + 
      'properties in the data option.', target);
};
//是否支持Proxy 这TM是ES6的大家伙啊
var hasProxy = typeof Proxy !== 'undefined' && Proxy.toString().match(/native code/);
if(hasProxy){
  //内置特殊按键属性
  var isBuiltInModifier = makeMap('stop,prevent,self,ctrl,shift,alt,meta');
  config.keyCodes = new Proxy(condig.keyCodes, {
    set: function set(target, key, value){
      //判断是否内置修饰符
      if(isBuiltInModifier(key)){
        //这里加两个括号不知道为啥
        warn(('Avoid overwriting built-in modifier in config.keyCodes: .' + key));
        return false
      } else {
    	target[key] = value;
        return true;
      } 
    }
  });
}

var hasHandler = {
  has: function has(target, key){
    var has = key in target;
    var isAllowed = allowedGlobals(key) || key.charAt(0) === '_';
    if(!has && !isAllowed){
      warnNonPresent(target, key);
    }
    return has || !isAllowed;
  }
};

var getHandler = {
  get: function get(target, key){
    if(typeof key === 'string' && !(key in target)){
      warnNonPresent(target, key);
    }
    return target[key];
  }
};
//初始化
initProxy = function initProxy(vm){
  if(hasProxy){
    var options = vm.$options;
    var handlers = option.render && 
        options.render._withStripped ? getHandler : hasHandler;
    vm._renderProxy = new Proxy(vm, handlers);
  } else {
    vm._renderProxy = vm;
  }
};
```


































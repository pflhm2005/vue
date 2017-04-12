# 随便写写/一些方法





## RegExp



### camelize：驼峰命名

### capitalize：首字母大写

### hyphenate：反驼峰命名









---





## fn



### makeMap



```javascript
//功能函数 匹配指定字符串
//例： var isBuiltInTag = makeMap('slot,component',true)
//解析后 list = [slot,component] => map = {slot:true,component:true}
//返回 function(val){return map[val]}
//此函数若传入slot,component会返回true
function makeMap(str,expectsLowercase){
  //创建一个无原型对象
  var map = Object.create(null);
  var list = str.splite(",");
  for(Var i=0;i<list.length;i++){
    map[list[i]] = true;
  }
  return expectsLowercase ? 
    function(val){return map[val.toLowercase()];}
  :	function(val){return map[val];}
}
```



---



### cached



```javascript
//缓存函数生成器
//例：var capitalize = cached(function(str){return str.charAt(0).toUpperCase() + str.slice(1)});
//简化为 var cap = cached(fn);fn=function(str){return ...};
//此时 cap = function cachedFn(str){... fn(str);} 其中str为参数 fn为之前传入的回调函数
//调用 cap("sss") => fn("sss") => Sss 回调函数调用参数字符串实现首字母大写
function cached(fn){
  var cache = Object.create(null);
  return (function cachedFn(str){
    //尝试读取缓存
    var hit = cache[str];
    //返回缓存 或者调用fn(str)并返回缓存
    return hit || (cache[str] = fn(str));
  });
}
```



---



### bind$1



```javascript
//强绑函数上下文
//函数柯里化
//例：var fn_bind = bind$1(fn,ctx);
//此时 fn_bind = function(a){...};
//调用 fn_bind(...)时 根据传入的参数数量进行强绑
function bind$1(fn,ctx){
  function boundFn(a){
    var l = arguments.length;
    //如果未传参 => fn.call(ctx)
    //传入一个参数 => fn.call(ctx,a)
    //传入多个参数 => fn.apply(ctx,arguments)
    return l ? (l > 1 ? fn.apply(ctx,arguments) : fn.call(ctx,a)) :fn.call(ctx);
  }
  boundFn._length = fn.length;
  return boundFn;
}
```



---



### toArray



```javascript
// 类数组 => 数组
// list:目标类数组 start:开始位置 
function toArray(list,start){
  start = start || 0 ;
  var  i = list.length - start;
  var ret = new Array(i);
  while(i--){
    ret[i] = list[i + start];
  }
  return ret;
}
```



---



### extend



```javascript
//对象混入 复制键值对 重复的覆盖
function extentd(to,_from){
  for(var key in _form){
    to[key] = _form[key];
  }
  return to;
}
```



---



### parsePath



```javascript
//以点结尾的路径为错误格式
var bailRE = /[^\w.$]/;
function parsePath(path){
  if(bailRE).test(path){
    return
  }
  else{
    //分割路径 www.baidu.com => [www,baidu,com]
    var segments = path.split('.');
    return function(obj){
      for(var i=0;i<segments.length;i++){
        //传入的对象 obj={"www":{"baidu":{"com":1}}}才行 坑爹啊
        if(!obj){return}
        obj = obj[segments[i]];
      }
      return obj;
    }
  }
}
```






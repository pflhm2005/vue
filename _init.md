# 初始化







## initProvide

```javascript
function initProvide(vm){
  var provide = vm.$options.provide;
  if(provide){
    vm._provided = typeof provide === 'function' ? 
      provide.call(vm) : provide;
  }
}
```



---



## initInjections

```javascript
function initInjections(vm){
  var inject = vm.$options.inject;
  if(inject){
    var isArray = Array.isArray(inject);
    // inject是数组 => inject
    // inject不是数组 && 支持ES6 => Reflect.ownKeys(inject) => 返回inject自身属性(包括symbol)
    // inject不是数组 && 不支持ES6 => Object.keys(inject)
    var keys = isArray ? 
        inject : hasSymbol ? 
        Reflect.ownKeys(inject) : Object.keys(inject);
  }
}
```


































# HTML  Parse





## utils



```javascript
// vue实例注册工具方法
Vue$3.config.mustUseProp = mustUseProp;
Vue$3.config.isReservedTag = isReservedTag;
Vue$3.config.isReservedAttr = isReservedAttr;
Vue$3.config.getTagNamespace = getTagNamespace;
Vue$3.config.isUnknownElement = isUnknownElement;

// 注册运行时平台
extend(Vue$3.options.directives, platformDirectives);
extend(Vue$3.options.components, platformComponents);

// 打补丁
Vue$3.prototype.__patch__ = inBrowser ? patch : noop;

```



### $mount

```javascript
// 手动挂载
Vue$3.prototype.$mount = function(el, hydrating){
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating);
}
```



### devtools

```javascript
setTimeout(function(){
  if(config.devtools){
    if(devtools){
      devtools.emit('init', Vue$3);
    } else if('development' !== 'production' && isChrome){
      // info打印出的信息前面会有个小标记
      // chrome专用提示信息 开发者扩展工具
      console[console.info ? 'info' : 'log'](
        'Download the Vue Devtools extension for a better development experience:\n' + 
        'https://github.com/vuejs/vue-devtools'
      );
    }
  }
  if('development' !== 'production' && 
    config.productionTip !== false && 
    inBrowser && typeof console !== 'undefined'){
      console[console.info ? 'info' : 'log'](
        'You are running Vue in development mode.\n' + 
        'Make sure to turn on production mode when deploying for production.\n' + 
        'See more tips at https://vuejs.org/guide/deployment.html'
      );
    }
},0);
```



### shouldDecode

```javascript
// encode检测
function shouldDecode(content, encoded){
  var div = document.createElement('div');
  div.innerHTML = '<div a="' + content + '">';
  return div.innerHTML.indexOf(encoded) > 0;
}
```



### shouldDecodeNewlines

```javascript
var shouldDecodeNewlines = inBrowser ? shouldDecode('\n', '$#10;') : false;
```



### isUnaryTag

```javascript
// 自闭合标签
var isUnaryTag = makeMap(
  'area,base,br,col,embed,frame,hr,img,input,isindex,keygen,' + 
  'link,meta,param,source,track,wbr'
);
```



### canBeLeftOpenTag

```javascript
// 可以不写闭合的标签
var canBeLeftOpenTag = makeMap(
  'colgroup,dd,dt,li,options,p,td,tfoot,th,thead,tr,source'
);
```



### isNonPhrasingTag

```javascript
var isNonPhrasingTag = makeMap(
  'address,article,aside,base,blockquote,body,caption,col,colgroup,dd,' + 
  'details,dialog,div,dl,dt,fieldset,figcaption,figure,footer,form,' + 
  'h1,h2,h3,h4,h5,h6,head,header,hgroup,hr,html,legend,li,menuitem,meta,' + 
  'optgroup,option,param,rp,rt,source,style,summary,tbody,td,tfoot,th,thead,' + 
  'title,tr,track'
);
```



### decode

```javascript
var decoder;
function decode(html){
  decoder = decoder || document.createElement('div');
  decoder.innerHTML = html;
  return decoder.textContent;
}
```



---





## parseHTML





### RegExp

```javascript
// 正则
var singleAttrIdentifier = /([^\s"'<>/=]+)/;
var singleAttrAssign = /(?:=)/;
var singleAttrValues = [
  // 双引号属性
  /"([^"]*)"+/.source,
  // 单引号属性
  /'([^']*)'+/.source,
  // 无引号属性
  /([^\s"'=<>']+)/.source
];
var attribute = new RegExp(
  '^\\s*' + singleAttrIdentifier.source + 
  '(?:\\s*(' + singleAttrAssign.source + ')' + 
  '\\s*(?:' + singleAttrValues.join('|') + '))?'
);

var ncname = '[a-zA-Z_][\\w\\-\\.]*';
var qnameCapture = '((?:' + ncname + '\\:)?' + ncname + ')';
// 匹配标签的开启
var startTagOpen = new RegExp('^<' + qnameCapture);
// 匹配标签的闭合
var startTagClose = /^\s*(\/?)>/;
var endTag = new RegExp('^<\\/' + qnameCaptue + '[^>]*>');
```
































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





## main





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
var doctype = /^<DOCTYPE [^>]+>/i;
var comment = /^<!--/;
var conditionalComment = /^<!\[/;

var IS_REGEX_CAPTURING_BROKEN = false;
'x'.replace(/x(.)?/g, function(m, g){
  IS_REGEX_CAPTURING_BROKEN = g ==='';
});
```



### Special Elements

```javascript
var isPlainTextElement = makeMap('script,style,textarea', true);
var reCache = {};

var decodingMap = {
  '&lt;': '<',
  '&gt;': '>',
  '&quot;': '"',
  '&amp;': '&',
  '&#10;': '\n'
};
var encodeAttr = /&(?:lt|gt|quot|amp);/g;
var encodeAttrWithNewLines = /&(?:lt|gt|quot|amp|#10);/g;
```



### decodeAttr

```javascript
function decodeAttr(value, shouldDecodeNewLines){
  var re = shouldDecodeNewLines ? encodedAttrWithNewLines : encodedAttr;
  return value.replace(re, function(match){ return decodingMap[match]; });
}
```



---



## parseHTML

```javascript
function parseHTML(html, options){
  var stack = [];
  var expectHTML = options.expectHTML;
  var isUnaryTag$$1 = options.isUnaryTag || no;
  var canBeLeftOpenTag$$1 = options.canBeLeftOpenTag || no;
  var index = 0;
  var last, lastTag;
  while(html){
    last = html;
    // 确保不是script,style标签中
    if(!lastTag || !isPlainTextElement(lastTag)){
      var textEnd = html.indexOf('<');
      if(textEnd === 0){  
        // 注释
        if(comment.test(html)){
          var commentEnd = html.indexOf('-->');
          if(commmentEnd >= 0){
            advance(commentEnd + 3);
            continue;
          }
        }
        
        // 向下兼容注释段
        // 例如：<![if !IE]>
        if(conditionalComment.test(html)){
          var conditionalEnd = html.indexOf(']>');
          if(conditionalEnd >= 0){
            advance(conditionalEnd + 2);
            continue;
          }
        }
        
        // Doctype
        // 例如：<DOCTYPE ...>
        var doctypeMatch = html.match(doctype);
        if(doctypeMatch){
          advance(doctypeMatch[0].length);
          continue;
        }
        
        // 结束标签
        var endTagMatch = html.match(endTag);
        if(endTagMatch){
          var curIndex = index;
          advance(endTagMatch[0].length);
          parseEndTag(endTagMatch[1], curIndex, index);
          continue;
        }
        
        // 开始标签
        var startTagMatch = parseStartTag();
        if(startTagMatch){
          handleStartTag(startTagMatch);
          continue;
        }
      }
      
      // 相当于undefined
      // 1.防止被重写 2.减少字符数
      var text = (void 0),
          rest$1 = (void 0),
          next = (void 0);
      if(textEnd >= 0){
        rest$1 = html.slice(textEnd);
        while(!endTag.test(rest$1)) && !startTagOpen.test(rest$1) && !comment.test(rest$1) && !conditionalComment.test(rest$1){
          //  非标签的'<'处理为文本
          next = rest$1.indexOf('<',1);
          if(next < 0){ break; }
          textEnd += next;
          rest$1 = html.slice(textEnd);
        }
        text = html.subString(0, textEnd);
        advance(textEnd);
      }
      if(textEnd > 0){
        text = html;
        html = '';
      }
      if(options.chars && text){
        options.chars(text);
      }
    } else{
      var stackedTag = lastTag.toLowerCase();
      var reStackedTag = reCache[stackedTag] || (reCache[stackedTag] = new RegExp('([\\s\\S]*?)(</' + stackedTag + '[^>]*>)','i'));
      var endTagLength = 0;
      var rest = html.replace(reStackedTag, function(all, text, endTag){
        endTagLength = endTag.length;
        if(!isPlainTextElement(stackedTag) && stackedTag !== 'noscript'){
          text = text.replace(/<!--([\s\S]*?)-->/g, '$1').replace(/<!\[CDATA\[([\s\S]*?)]]>/g, '$1');
        }
        if(options.chars){
          options.chars(text);
        }
        return '';
      });
      index += html.length - rest.length;
      html = rest;
      parseEndTag(stackedTag, index - endTagLength, index);
    }
    if(html === last){
      options.char && options.chars(html);
      if('development' !== 'production' && !stack.length && options.warn){
        options.warn('Mal-formatted tag at end of template: "' + html + '"');
      }
      break;
    }
  }
  
  // 收尾
  parseEndTag();
  
  function advance(n){
    index += n;
    html = html.substring(n);
  }
  
  function parseStartTag(){
    var start = html.match(startTagOpen);
    if(start){
      var match = {
        tagName: start[1],
        attrs: [],
        start: index
      };
      advance(start[0].length);
      var end, attr;
      while(!(end = html.match(startTagClose)) && (attr = html.match(attribute))){
        advance(attr[0].length);
        match.attrs.push(attr);
      }
      if(end){
        match.unarySlash = end[1];
        advance(end[0].length);
        match.end = index;
        return match;
      }
    }
  }
  
  function handleStartTag(match){
    var tagName = match.tagName;
    var unarySlash = match.unarySlash;
    
    if(expectHTML){
      if(lastTag === 'p' && isNonPhrasingTag(tagName)){
        parseEndTag(lastTag);
      }
      if(canBeLeftOpenTag$$1(tagName) && lastTag === tagName){
        parseEndTag(tagName);
      }
    }
    var unary = isUnaryTag$$1(tagName) || tagName === 'html' && lastTag === 'head' || !!unarySlash;
    
    var l = match.attrs.length;
    var attrs = new Array(l);
    for(var i = 0; i < l; i++){
      var args = match.attrs[i];
      if(IS_REGEX_CAPTURING_BROKEN && args[0].indexOf('""') === -1){
        if(args[3] === ''){ delete args[3]; }
        if(args[4] === ''){ delete args[4]; }
        if(args[5] === ''){ delete args[5]; }
      }
      var value = args[3] || args[4] || args[5] || '';
      attrs[i] = {
        name: args[i],
        value: decodeAttr(value, options.shouldDecodeNewlines)
      };
    }
    if(!unary){
      stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs });
      lastTag = tagName;
    }
    if(options.start){
      options.start(tagName, attrs, unary, match.start, match.end);
    }
  }
  
  function parseEndTag(tagName, start, end){
    var pos, lowerCasedTagName;
    if(start == null) { start = index; }
    if(end == null) { end = index; }
    
    if(tagName){
      lowerCasedTagName = tagName.toLowerCase();
    }
    
    // 寻找最近的同类型标签
    if(tagName){
      for(pos = stack.length - 1; pos > 0; pos--){
        if(stack[pos].lowerCasedTag === lowerCasedTagName){
          break;
        }
      }
    } else{
      pos = 0;
    }
    
    if(pos >= 0){
      // 关闭所有元素
      for(var i = stack.length - 1; i >= pos; i--){
        if('development' !== 'production' && (i > pos || !tagName) && options.warn){
          options.warn(
            ('tag <' +(stack[i].tag) + '> has no matching end tag.')
          );
        }
        if(options.end){
          options.end(stack[i].tag, start, end);
        }
      }
      //
      stack.length = pos;
      lastTag = pos && stack[pos - 1].tag;
    } else if(lowerCasedTagName === 'br'){
      if(options.start){
        options.start(tagName, [], true, start, end);
      }
    } else if(lowerCasedTagName === 'p'){
      if(options.start){
        options.start(tagName, [], false, start, end);
      }
      if(options.end){
        options.end(tagName, start, end);
      }
    } 
  }
}
```



---



## parseText

```javascript
var defaultTagRE = /\{\{((?:.|\n)+?)\}\}/g;
// 匹配正则符号与[]\/
var regexEscapeRE = /[-.*+?^${}()|[\]\/\\]/g;

var buildRegex = cached(function(delimiters){
  var open = delimiters[0].replace(regexEscapeRE, '\\$&');
  var close = delimiters[1].replace(regexEscapeRE, '\\$&');
  return new RegExp(open + '((?:.|\\n)+?)' + close, 'g');
});

function parseText(text, delimiters){
  var tagRE = delimiters ? buildRegex(delimiters) : defaultTagRE;
  if(!tagRE.test(text)){
    return;
  }
  var tokens = [];
  var lastIndex = tagRE.lastIndex = 0;
  var match, index;
  while((match = tagRE.exec(text))){
    index = match.index;
    if(index > lastIndex){
      tokens.push(JSON.stringify(text.slice(lastIndex, index)));
    }
    var exp = parseFilters(match[1].trim());
    tokens.push(('_s(' + exp + ')'));
    lastIndex = index + match[0].length;
  }
  if(lastIndex < text.length){
    tokens.push(JSON.stringify(text.slice(lastIndex)));
  }
  return tokens.join('+');
}
```



---



## parse

```javascript
var onRE = /^@|^v-on:/;
var dirRE = /^v-|^@|^:/;
var forAliasRE = /(.*?)\s+(?:in|of)\s+(.*)/;
// ({[*]})
var forIteratorRE = /\((\{[^}]*\}|[^,]*),([^,]*)(?:,([^,]*))?\)/;

var argRE = /:(.*)$/;
// 
var bindRE = /^:|^v-bind:/;
var modifierRE = /\.[^.]+/g;

var decodeHTMLCached = cached(decode);

// 状态设置
var warn$2;
var delimiters;
var transforms;
var preTransforms;
var postTransforms;
var platformIsPreTag;
var platformMustUseProp;
var platformGetTagNamespace;

// 将HTML转化为抽象语法树
function parse(template, options){
  warn$2 = options.warn || baseWarn;
  // no => return false
  platformGetTagNamespace = options.getTagNamespace || no;
  platformMustUseProp = options.mustUseProp || no;
  platformIsPreTag = options.isPreTag || no;
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode');
  transforms = pluckModuleFunction(options.modules, 'transformNode');
  postTransform = pluckModuleFunction(options.modules, 'postTransformNode');
  delimiters = options.delimiters;
  
  var stack = [];
  var preserveWhitespace = options.preserveWhitespace !== false;
  var root;
  var currentParent;
  var inVPre = false;
  var inPre = false;
  var warned = false;
  
  function warnOnce(msg){
    if(!warned){
      warned = true;
      warn$2(msg);
    }
  }
  
  function endPre(element){
    if(element.pre){
      inVPre = false;
    }
    if(platformIsPreTag(element.tag)){
      inPre = false;
    }
  }
  
  parseHTML(template, {
    warn: warn$2,
    expectHTML: options.expectHTML,
    isUnaryTag: options.isUnaryTag,
    canBeLeftOpenTag: options.canBeLeftOpenTag,
    shouldDecodeNewlines: options.shouldDecodeNewlines,
    start: function start(tag, attrs, unary){
      var ns = (currentParent && currentParent.ns) || platformGetTagNamespace(tag);
      
      // IE svg BUG
      if(isIE && ns === 'svg'){
        attrs = guardIESVGBug(attrs);
      }
      
      var element = {
        type: 1,
        tag: tag,
        attrsList: attrs,
        attrsMap: makeAttrsMap(attrs),
        parent: currentParent,
        children: []
      };
      if(ns){
        element.ns = ns;
      }
      
      if(isForbiddenTag(element) && !isServerRendering()){
        element.forbidden = true;
        'development' !== 'production' && warn$2(
          'Templates should only be responsible for mapping the state to the ' + 
          'UI. Avoid placing tags with side-effects in your templates, such as ' + 
          '<' + tag + '>' + ', as they will not be parsed.'
        );
      }
      
      for (var i = 0; i < preTransforms.length; i++){
        preTransform[i](element, options);
      }
      
      if(!inVPre){
        processPre(element);
        if(element.pre){
          inVPre = true;
        }
      }
      
      if(platformIsPreTag(element.tag)){
        inPre = true;
      }
      if(inVPre){
        processRawAttrs(element);
      } else{
        processFor(element);
        processIf(element);
        processOnce(element);
        processKey(element);
        
        // 
        element.plain = !element.key && !attrs.length;
        
        processRef(element);
        processSlot(element);
        processComponent(element);
        for(var i$1 = 0; i$1 < transform.length; i$1++){
          transform[i$1](element, options);
        }
        processAttrs(element);
      }
      
      function checkRootConstraints(el){
        if(el.tag === 'slot' || el.tag === 'template'){
          warnOnce(
            'Cannot use <' + (el.tag) + '> as component root element because it may ' + 
            'contain multiple nodes.'
          );
        }
        if(el.attrMap.hasOwnProperty('v-for')){
          warnOnce(
            'Cannot use v-for on stateful component root element because ' + 
            'it renders multiple elements.'
          );
        }
      }
      // 树管理
      if(!root){
        root = element;
        checkRootConstraints(root);
      } else if(!stack.length){
		// 允许根元素存在v-if,v-else,v-else-if
        if(root.if && (element.elseif || element.else)){
          checkRootConstraints(element);
          addIfCondition(root, {
            exp: element.elseif,
            block: element
          });
        } else{
          warnOnce(
            'Component template should contain exactly one root element. ' + 
            'If you are using v-if on multiple elements, ' + 
            'use v-else-if to chain them instead.'
          );
        }
      }
      if(currentParent && !element.forbidden){
        if(element.elseif || element.else){
          processIfConditions(element, currentParent);
        } else if(element.slotScope){
          currentParent.plain = false;
          var name = element.slotTarget;
          (currentParent.scopedSlots || (currentParent.scopeSlots = {}))[name] = element;
        } else{
          currentParent.children.push(element);
          element.parent = currentParent;
        }
      }
      if(!unary){
        currentParent = element;
        stack.push(element);
      } else{
        endPre(element);
      }
      for(var i$2 = 0; i$2 < postTransforms.length; i$2++){
        postTransforms[i$2](element, options);
      }
    },
    
    end: function end(){
      // 移除尾部空格
      var element = stack[stack.length - 1];
      var lastNode = element.children[element.children.length - 1];
      if(lastNode && lastNode.type === 3 && lastNode.text === ' ' && !inPre){
        element.children.pop();
      }
      // pop
      stack.length -= 1;
      currentParent = stack[stack.length - 1];
      endPre(element);
    },
    
    chars: function chars(text){
      if(!currentParent){
        if(text === template){
          warnOnce(
            'Component template requires a root element, rather than just text.'
          );
        } else if((text = text.trim())){
          warnOnce(
            ('text "' + text + '" outside root element will be ignored.')
          );
        }
        return ;
      }
      
      if(isIE && currentParent.tag === 'textarea' && currentParent.attrsMap.placeholder === text){
        return ;
      }
      var children = currentParent.children;
      text = inPre || text.trim() ? 
        (isTextTag(currentParent) ? text : decodeHTMLCached(text)) : (preserveWhitespace && children.length ? ' ' : '');
      if(text){
        var expression;
        if(!inVPre && text !== ' ' && (expression = parseText(text, delimiters))){
          children.push({
            type: 2,
            expression: expression,
            text: text
          })
        } else if(text !== ' ' || !children.length || children[children.length - 1].text !== ' '){
          children.push({
            type: 3,
            text: text
          });
        }
      }
    }
  });
  
  // 这是大函数的返回
  return root;
}
```



### processPre

```javascript
function processPre(el){
  if(getAndRemoveAttr(el, 'v-pre') != null){
    el.pre = true;
  }
}
```



### processRawAttrs

```javascript
function processRawAttrs(el){
  var l = el.attrsList.length;
  if(l){
    var attrs = el.attrs = new Array(l);
    for(var i = 0; i < l; i++){
      attrs[i] = {
        name: el.attrsList[i].name,
        value: JSON.stringify(el.attrsList[i].value)
      };
    }
  } else if(!el.pre){
    el.plain = true;
  }
}
```



### processKey

```javascript
function processKey(el){
  var exp = getBindingAttr(el, 'key');
  if(exp){
    if('development' !== 'production' && el.tag === 'template'){
      warn$2('<template> cannot be keyed. Place the key on real elements instead.');
    }
    el.key = exp;
  }
}
```



### processRef

```javascript
function processRef(el){
  var ref = getBindingAttr(el, 'ref');
  if(ref){
    el.ref = ref;
    el.refInFor = checkInFor(el);
  }
}
```



### processFor

```javascript
function processFor(el){
  var exp;
  if((exp = getAndRemoveAttr(el, 'v-for'))){
    var inMatch = exp.match(forAliasRE);
    if(!inMatch){
      'development' !== 'production' && warn$2(
        ('Invalid v-for expression: ' + exp)
      );
      return ;
    }
    el.for = inMatch[2].trim();
    var alias = inMatch[1].trim();
    var iteratorMatch = alias.match(forIteratorRE);
    if(iteratorMatch){
      el.alias = iteratorMatch[1].trim();
      el.iterator1 = iteratorMatch[2].trim();
      if(iteratorMatch[3]){
        el.iterator2 = iteratorMatch[3].trim();
      }
    } else{
      el.alias = alias;
    }
  }
}
```



### processIf

```javascript
function processIf(el){
  var exp = getAndRemoveAttr(el, 'v-if');
  if(exp){
    el.if = exp;
    addIfConditon(el, {
      exp: exp,
      block: el
    });
  } else{
    if(getAndRemoveAttr(el, 'v-else') != null){
      el.else = true;
    }
    var elseif = getAndRemoveAttr(el, 'v-else-if');
    if(elseif){
      el.elseif = elseif;
    }
  }
}
```



### processIfConditions

```javascript
function processIfConditions(el, parent){
  var prev = findPrevElement(parent.children);
  if(prev && prev.if){
    addIfCondition(prev, {
      exp: el.elseif,
      block: el
    });
  } else{
    warn$2(
      'v-' + (el.elseif ? ('else-if="' + el.elseif + '"') : 'else') + ' ' + 
      'used on element <' + (el.tag) + '> without corresponding v-if.'
    );
  }
}
```



### findPrevElement

```javascript
function findPrevElement(children){
  var i = children.length;
  while(i--){
    if(children[i].type === 1){
      return children[i];
    } else{
      if('development' !== 'production' && children[i].text !== ' '){
        warn$2(
          'text "' + (children[i].text.trim()) + '" between v-if and v-else(-if)' + 
          'will be ignored.'
        );
      }
      children.pop();
    }
  }
}
```



### addIfCondition

```javascript
function addIfCondition(el, condition){
  if(!el.ifConditions){
    el.ifConditions = [];
  }
  el.ifConditions.push(condition);
}
```



### processOnce

```javascript
function processOnce(el){
  var once$$1 = getAndRemoveAttr(el, 'v-once');
  if(once$$1 != null){
    el.once = true;
  }
}
```



### processSlot

```javascript
function processSlot(el){
  if(el.tag === 'slot'){
    el.slotName = getBindingAttr(el, 'name');
    if('development' !== 'production' && el.key){
      warn$2(
        '"key" does not work on <slot> because slots are abstract outlets ' + 
        'and can possibly expand into multiple elements. ' + 
        'Use the key on a wrapping element instead.'
      );
    }
  } else{
    var slotTarget = getBindingAttr(el, 'slot');
    if(slotTarget){
      el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget;
    }
    if(el.tag === 'template'){
      el.slotScope = getAndRemoveAttr(el, 'scope');
    }
  }
}
```



### processComponent

```javascript
function processComponent(el){
  var binding;
  if((binding = getBindingAttr(el, 'is'))){
    el.component = binding;
  }
  if(getAndRemoveAttr(el, 'inline-template') != null){
    el.inlineTemplate = true;
  }
}
```



### processAttrs

```javascript
function processAttrs(el){
  var list = el.attrsList;
  var i, l, name, rawName, value, modifiers, isProp;
  for(i = 0, l = list.length; i < l; i++){
    name = rawName = list[i].name;
    value = list[i].value;
    if(dirRE.test(name)){
      // 标记
      el.hasBindings = true;
      // 修改
      modifiers = parseModifiers(name);
      if(modifiers){
        name = name.replace(modifierRE, '');
      }
      // v-bind
      if(bindRE.test(name)){
        name = name.replace(bindRE, '');
        value = parseFilters(value);
        isProp = false;
        if(modifiers){
          if(modifiers.prop){
            isProp = true;
            name = camelize(name);
            if( name === 'innerHtml' ){ name = 'innerHTML'; }
          }
          if(modifiers.camel){
            name = camelize(name);
          }
          if(modifiers.sync){
            addHandler(
              el,('update:' + (camelize(name))), genAssignmentCode(value, '$event')
            );
          }
        }
        if(isProp || platformMustUseProp(el.tag, el.attrsMap.type, name)){
          addProp(el, name, value);
        } else{
          addAttr(el, name, value);
        }
      }
      // v-on
      else if(onRE.test(name)){
        name = name.replace(onRE, '');
        addHandler(el, name, value, modifiers, false, warn$2);
      }
      // 普通指令
      else{
        name = name.replace(dirRE, '');
        var argMatch = name.match(argRE);
        var arg = argMatch && argMatch[1];
        if(arg){
          name = name.slice(0, -(arg.length + 1));
        }
        addDirective(el, name, rawName, value, arg, modifiers);
        if('development' !== 'production' && name === 'model'){
          checkForAliasModel(el, value);
        }
      }
    } else{
      // 文字属性
      var expression = parseText(value, delimiters);
      if(expression){
        warn$2(
          name + '="' + value + '": ' + 
          'Interpolation inside attributes has been removed. ' + 
          'Use v-bind or the colon shorthand instead. For example, ' + 
          'instead of <div id="{{ val }}">, use <div :id="val">.'
        );
      }
      addAttr(el, name, JSON.stringify(value));
    }
  }
}
```



### checkInFor

```javascript
function checkInFor(el){
  var parent = el;
  while(parent){
    if(parent.for !== undefined){
      return true;
    }
    parent = parent.parent;
  }
  return false;
}
```



### parseModifiers

```javascript
function parseModifiers(name){
  var match = name.match(modifierRE);
  if(match){
    var ret = {};
    match.forEach(function(m){ ret[m.slice(1)] = true; });
    return ret;
  }
}
```



### makeAttrsMap

```javascript
function makeAttrsMap(attrs){
  var map = {};
  for(var i = 0; l = attrs.length; i < l; i++){
    if('development' !== 'production' && map[attrs[i].name] && !isIE && !isEdge){
      warn$2('duplicate attribute: ' + attrs[i].name);
    }
    map[attrs[i].name] = attrs[i].value;
  }
  return map;
}
```



### isTextTag

```javascript
function isTextTag(el){
  return el.tag === 'script' || el.tag === 'style';
}
```



### isForbiddenTag

```javascript
function isForbiddenTag(el){
  return (
    el.tag === 'style' || (el.tag === 'script' && (!el.attrsMap.type || el.attrsMap.type === 'text/javascript'))
  );
}
```



### guardIESVGBug

```javascript
var ieNSBug = /^xmlns:NS\d+/;
var ieNSPrefix = /^NS\d+:/;

function guardIESVGBug(attrs){
  var res = [];
  for(var i = 0; i < attrs.length; i++){
    var attr = attrs[i];
    if(!ieNSBug.test(attr.name)){
      attr.name = attr.name.replace(ieNSPrefix, '');
      res.push(attr);
    }
  }
  return res;
}
```



### checkForAliasModel

```javascript
function checkForAliasModel(el, value){
  var _el = el;
  while(_el){
    if(_el.for && _el.alias === value){
      warn$2(
        '<' + (el.tag) + ' v-model="' + value + '">: ' + 
        'You are binding v-model directly to a v-for iteration alias. ' + 
        'This will not be able to modify the v-for source array because ' + 
        'writing to the alias is like modifying a function local variable. ' + 
        'Consider using an array of objects and use v-model on an object property instead.'
      );
    }
    _el = _el.parent;
  }
}
```




















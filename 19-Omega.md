# END







## utils





### prohibitedKeywordRE

```javascript
// 禁用的关键词 不包含typeof等运算符
var prohibitedKeywordRE = new RegExp('\\b' + (
  'do,if,for,let,new,try,var,case,else,with,await,break,catch,class,const,' + 
  'super,throw,while,yield,delete,export,import,return,switch,default,' + 
  'extends,finally,continue,debugger,function,arguments'
).split(',').join('\\b|\\b') + '\\b');
```



### unaryOperatorsRE

```javascript
var unaryOperatorsRE = new RexExp('\\b' + (
  'delete,typeof,void'
).split(',').join('\\s*\\([^\\)]*\\)|\\b') + '\\s*\\([^\\)]*\\');
```



### identRE

```javascript
var identRE = /[A-Za-z_$][\w$]*/;
```



### stripStringRE

```javascript
var stripStringRE = /'(?:[^'\\]|\\.)*'|"(?:[^"\\]|\\.)*"|`(?:[^`\\]|\\.)*\$\{|\}(?:[^`\\]|\\.)*`|`(?:[^`\\]|\\.)*`/g;
```





---





## check





### detectErrors

```javascript
function detectErrors(ast){
  var errors = [];
  if(ast){
    checkNode(ast, errors);
  }
  return errors;
}
```



### checkNode

```javascript
function checkNode(node, errors){
  if(node.type === 1){
    for(var name in node.attrsMap){
      if(dirRE.test(name)){
        var value = node.attrsMap[name];
        if(value){
          if(name === 'v-for'){
            checkFor(node, ('v-for="' + value + '"'), errors);
          } else if(onRE.test(name)){
            checkEvent(value, (name + '="' + value + '"'), errors);
          } else{
            checkExpression(value, (name + '="' + value + '"'), errors);
          }
        }
      }
    }
    if(node.children){
      for(var i = 0; i < node.children.length; i++){
        checkNode(node.children[i], errors);
      }
    }
  } else if(node.type === 2){
    checkExpression(node.expression, node.text, errors);
  }
}
```



### checkEvent

```javascript
function checkEvent(exp, text, errors){
  var stipped = exp.replace(stripStringRE, '');
  var keywordMatch = stipped.match(unaryOperatorsRE);
  if(keywordMatch && stipped.charAt(keywordMatch.index - 1) !== '$'){
    errors.push(
      'avoid using Javascript unary operator as property name: ' + 
      '"' + (keywordMatch[0]) + '" in expression ' + (text.trim())
    );
  }
  checkExpression(exp, text, errors);
}
```



### checkFor

```javascript
function checkFor(node, text, errors){
  checkExpression(node.for || '', text, errors);
  checkIdentifier(node.alias, 'v-for alias', text, errors);
  checkIdentifier(node.iterator1, 'v-for iterator', text, errors);
  checkIdentifier(node.iterator2, 'v-for iterator', text, errors);
}
```



### checkIdentifier

```javascript
function checkIdentifier(ident, type, text, errors){
  if(typeof ident === 'string' && !identRE.test(ident)){
    errors.push(('invalid' + type + ' "' + ident + '" in expression: ' + (text.trim())));
  }
}
```



### checkExpression

```javascript
function checkExpression(exp, text, errors){
  try{
    new Function(('return ' + exp));
  } catch(e){
    var keywordMatch = exp.replace(stripStringRE, '').match(prohibitedKeywordRE);
    if(keywordMatch){
      errors.push(
        'avoid using Javascript keyword as property name: ' + 
        '"' + (keywordMatch[0]) + '" in expression ' + (text.trim())
      );
    } else{
      errors.push(('invalid expression: ' + (text.trim())));
    }
  }
}
```





---





## compile







### baseCompile

```javascript
function baseCompile(template, options){
  var ast = parse(template.trim(), options);
  optimize(ast, options);
  var code = generate(ast, options);
  return {
    ast: ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}
```



### makeFunction

```javascript
function makeFunction(code, errors){
  try{
    return new Function(code);
  } catch(err){
    errors.push({err: err, code: code });
    return noop;
  }
}
```



### createCompiler

```javascript
function createCompiler(baseOptions){
  var functionCompileCache = Object.create(null);
  
  // compile
  function compile(template, options){
    var finalOptions = Object.create(baseOptions);
    var errors = [];
    var tips = [];
    finalOptions.warn = function(msg, tip$$1){
      (tip$$1 ? tips : errors).push(msg);
    };
    
    if(options){
      if(options.modules){
        finalOptions.modules = (baseOptions.modules || []).concat(options.modules);
      }
      if(options.directives){
        finalOptions.directives = extend(
          Object.create(baseOptions.directives),
          options.directives
        );
      }
      
      for(var key in options){
        if(key !== 'modules' && key !== 'directives'){
          finalOptions[key] = options[key];
        }
      }
    }
    var compiled = baseCompile(template, finalOptions);
    errors.push.apply(errors, detectErrors(compiled.ast));
    compiled.errors = errors;
    compiled.tips = tips;
    return compiled;
  }
  
  // compileToFunctions
  function compileToFunctions(template, options, vm){
    options = options || {};

    try{
      new Function('return 1');
    } catch(e){
      if(e.toString().match(/unsafe-eval|CSP/)){
        warn(
          'It seems you are using the standalone build of Vue.js in an ' + 
          'environment with Content Security Policy that prohibits unsafe-eval. ' + 
          'The template compiler cannot work in this environment . Consider ' + 
          'relaxing the policy to allow unsafe-eval or pre-compiling your ' + 
          'templates into render functions.'
        );
      }
    }

    var key = options.delimiters ? String(options.delimiters) + template : template;
    if(functionCompileCache[key]){
      return functionCompileCache[key];
    }
	
    // compile
    var compiled = compile(template, options);
    
    // errors,tips信息打印
    if(compiled.errors && compiled.errors.length){
      warn(
        'Error compiling template:\n\n' + template + '\n\n' + 
        compiled.errors.map(function(e){ return ('- ' + e); }).join('\n') + '\n',
        vm
      );
    }
    if(compiled.tips && compiled.tips.length){
      compiled.tips.forEach(function(msg){ return tip(msg, vm); });
    }
    
    var res = {};
    var fnGenErrors = [];
    res.render = makeFunction(compiled.render, fnGenErrors);
    var l = compiled.staticRenderFns.length;
    res.staticRenderFns = new Array(l);
    for(var i = 0; i < l; i++){
      res.staticRenderFns[i] = makeFunction(compiled.staticRenderFns[i], fnGenErrors);
    }
    
    // 编译器问题检测
    if((!compiled.errors || !compiled.errors.length) && fnGenErrors.length){
      warn(
        'Failed to generate render function:\n\n' + 
        fnGenErrors.map(function(ref){
          var err = ref.err;
          var code = ref.code;
          return ((err.toString()) + ' in\n\n' + code +'\n');
        }).join('\n'),
        vm
      );
    }
    return (functionCompileCache[key] = res);
  }
  return {
    compile: compile,
    compileToFunctions: compileToFunctions
  }
}
```



### transformNode

```javascript
function transformNode(el, options){
  var warn = options.warn || baseWarn;
  var staticClass = getAndRemoveAttr(el, 'class');
  if('development' !== 'production' && staticClass){
    var expression = parseText(staticClass, options.delimiters);
    if(expression){
      warn(
        'class="' + staticClass + '": ' + 
        'Interpolation inside attributes has been removed. ' + 
        'Use v-bind or the colon shorthand instead. For example, ' + 
        'instead of <div class="{{ val }}">, use <div :class="val">.'
      );
    }
  }
  if(staticClass){
    el.staticClass = JSON.stringify(staticClass);
  }
  var classBinding = getBindingAttr(el, 'class', false);
  if(classBinding){
    el.classBinding = classBinding;
  }
}
```



### genData$1

```javascript
function genData$1(el){
  var data = '';
  if(el.staticClass){
    data += 'staticClass:' + (el.staticClass) + ',';
  }
  if(el.classBinding){
    data += 'class:' + (el.classBinding) + ',';
  }
  return data;
}
```





### klass$1



```javascript
var klass$1 = {
  staticKeys: ['staticClass'],
  transformNode: transformNode,
  genData: genData$1
}
```





---





### transformNode$1

```javascript
function transformNode$1(el, options){
  var warn = options.warn || baseWarn;
  var staticStyle = getAndRemoveAttr(el, 'style');
  if(staticStyle){
    var expression = parseText(staticStyle, options.delimiters);
    if(expression){
      warn(
        'style="' + staticStyle + '": ' + 
        'Interpolation inside attribute has been removed. ' + 
        'Use v-bind or the colon shorthand instead. For example, ' + 
        'instead of <div style="{{ val }}">, use <div :style="val">.'
      );
    }
    el.staticStyle = JSON.stringify(parseStyleText(staticStyle));
  }
  var styleBinding = getBindingAttr(el, 'style', false);
  if(styleBinding){
    el.styleBinding = styleBinding;
  }
}
```



### genData$2

```javascript
function genData$2(el){
  var data = '';
  if(el.staticStyle){
    data += 'staticStyle:' + (el.staticStyle) + ',';
  }
  if(el.styleBinding){
    data += 'style:(' + (el.styleBinding) + '),';
  }
  return data;
}
```



### style$1



```javascript
var style$1 = {
  staticKeys: ['staticStyle'],
  transformNode: transformNode$1,
  genData: genData$2
};
```



### module$1 

```javascript
var modules$1 = [
  klass$1,
  style$1
];
```





## /* */



### text

```javascript
function text(el, dir){
  if(dir.value){
    addProp(el, 'textContent', ('_s(' + (dir.value) + ')'));
  }
}
```



### html

```javascript
function html(el, dir){
  if(dir.value){
    addProp(el, 'innerHTML', ('_s(' + (dir.value) + ')'));
  }
}
```



### directives$1

```javascript
var directives$1 = {
  // 第5900行的函数
  model: model,
  text: text,
  html: html
};
```





---





## baseOptions





```javascript
function baseOptions = {
  expectHTML: true,
  modules: modules$1,
  directives: directives$1,
  isPreTag: isPreTag,
  isUnaryTag: isUnaryTag,
  mustUseProp: mustUseProp,
  canBeLeftOpenTag: canBeLeftOpenTag,
  isReservedTag: isReservedTag,
  getTagNamespace: getTagNamespace,
  staticKeys: genStaticKeys(modules$1)
}
```



### ref$1

```javascript
var ref$1 = createCompiler(baseOptions);
```



### compileToFunctions

```javascript
var compileToFunctions = ref$1.compileToFunctions;
```



### idToTemplate

```javascript
var idToTemplate = cached(function(id){
  var el = query(id);
  return el && el.innerHTML;
});
```



---





## mount





```javascript
// 手动挂载mount
var mount = Vue$3.prototype.$mount;
Vue$3.prototype.$mount = function(el, hydrating){
  el = el && query(el);
  
  if(el === document.body || el === document.documentElement){
    'development' !== 'production' && warn(
      'Do not mount Vue to <html> or <body> - mount to normal elements instead.'
    );
    return this;
  }
  
  var options = this.$options;
  
  if(!options.render){
    var template = options.template;
    if(template){
      if(typeof template === 'string'){
        if(template.charAt(0) === '#'){
          template = idToTemplate(template);
          if('development' !== 'production' && !template){
            warn(
              ('Template element not found or is empty: ' + (options.template)),
              this
            );
          }
        }
      } else if(template.nodeType){
        template = template.innerHTML;
      } else{
        warn('invalid template option:' + template, this);
        return this;
      }
    } else if(el){
      template = getOuterHTML(el);
    }
    if(template){
      if('development' !== 'production' && config.performance && mark){
        mark('compile');
      }
      
      var ref = compileToFunctions(template, {
        shouldDecodeNewlines: shouldDecodeNewlines,
        delimiters: options.delimiters
      }, this);
      var render = ref.render;
      var staticRenderFns = ref.staticRenderFns;
      options.render = render;
      options.staticRenderFns = staticRenderFns;
      
      if('development' !== 'production' && config.performance && mark){
        mark('compile end');
        measure(((this._name) + ' compile'), 'compile', 'compile end');
      }
    }
  }
  return mount.call(this, el, hydrating);
};
```



### getOuterHTML

```javascript
function getOuterHTML(el){
  if(el.outerHTML){
    return el.outerHTML;
  } else{
    var container = document.createElement('div');
    container.appendChild(el.cloneNode(true));
    return container.innerHTML;
  }
}
```





## /* */





```javascript
Vue$3.compile = compileToFunctions;
reutrn Vue$3;
```










# Babel从入门到插件开发

最近的技术项目里大量用到了需要修改源文件代码的需求，也就理所当然的用到了Babel及其插件开发。这一系列专题我们介绍下Babel相关的知识及使用。

对于刚开始接触代码编译转换的同学，单纯的介绍Babel相关的概念只是会当时都能看懂，但是到了自己去实现一个需求的时候就又会变得不知所措，所以我们再介绍中穿插一些例子。

大概分为以下几块：

0、Babel基础介绍  
1、使用npm上好用的Babel插件提升开发效率  
2、使用Babel做代码转换使用到的模块及执行流程  
3、示例：类中插入方法、类方法中插入代码  
4、Babel插件开发介绍  
5、示例：通过Babel实现打包构建优化 -- 组件模块按需打包  


## 0.Babel基础介绍

用到的名词：  
- AST：Abstract Syntax Tree, 抽象语法树  
- DI: Dependency Injection, 依赖注入  

我们在实际的开发过程中，经常有需要修改js源代码的需求，比如一下几种情形：
- ES6/7转化为浏览器可支持的ES5甚至ES3代码；  
- JSX代码转化为js代码（原来是Facebook团队支持在浏览器中执行转换，现在转到在babel插件中维护）；  
- 部分js新的特性动态注入（用的比较多的就是babel-plugin-transform-runtime）；  
- 一些便利性特性支持，比如：React If/Else/For/Switch等标签支持；  

于是，我们就需要一款支持动态修改js源代码的模块，babel则是用的最多的一个。

### Babel的解析引擎
Babel使用的引擎是babylon，babylon并非由babel团队自己开发的，而是fork的acorn项目，不过acorn引擎只提供基本的解析ast的能力，遍历还需要配套的acorn-travesal, 替换节点需要使用acorn-，而这些开发，在Babel的插件体系开发下，变得一体化了。

### 如何使用
使用方式有很多种：
- webpack中作为js(x)文件的loader使用；
- 单独在Node代码中引入使用；
- 命令行中使用：
package.json中配置：
"scripts": {
    "build": "rimraf lib && babel src --out-dir lib"
}

命令中执行：npm run build。

通常，如果我们在项目根目录下配置一个.babelrc文件，其配置规则会被babel引入并使用。

## 1、使用npm上好用的Babel插件提升开发效率
在使用webpack做打包工具的时候，我们队js(x)文件使用的loader通常就是babel-loader，babel只是提供了最基础的代码编译能力，主要用到的一些代码转换则是通过插件的方式实现的。在loader中配置插件有两种方式：presets及plugins，这里要注意presets配置的也是插件，只是优先级比较高，而且他的执行顺序是从左到右的，而plugins的优先级顺序则是从右到左的。我们经常用到的插件会包括：ES6/7转ES5代码的babel-plugin-es2015，React jsx代码转换的babel-plugin-react，对新的js标准特性有不同支持程度的babel-plugin-stage-0等（不同阶段js标准特性的制定是不一样的，babel插件支持程度也就不一样，0表示完全支持），将浏览器里export语法转换为common规范exports/module.exports的babel-plugin-add-module-exports，根据运行时动态插入polyfill的babel-plugin-transform-runtime（绝不建议使用babel-polyfill，一股脑将所有polyfill插入，打的包会很大），对Generator进行编译的babel-plugin-transform-regenerator等。想了解更多的配置可以参见这篇文章：如何写好.babelrc？Babel的presets和plugins配置解析（https://excaliburhan.com/post/babel-preset-and-plugins.html?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io）

如果你是基于完全组件化（标签式）的开发模式的话，如果能提供常用的控制流标签如：If/ElseIf/Else/For/Switch/Case等给我们的话，那么我们的开发效率则会大大提升。在这里我要推荐一款实现了这些标签的babel插件：jsx-control-statement，建议在你的项目中加入这个插件并用起来，不用再艰难的书写三元运算符，会大大提升你的开发效率。

## 2、使用Babel做代码转换使用到的模块及执行流程
Babel将源码转换AST之后，通过遍历AST树（其实就是一个js对象），对树做一些修改，然后再将AST转成code，即成源码。

将js源码转换为AST用到的模块叫：babylon，对树进行遍历并做修改用到的模块叫：babel-traverse，将修改后的AST再生成js代码用到的模块则是：babel-generator。而babel-core模块则是将三者结合使得对外提供的API做了一个简化，使用babel-core只需要执行以下的简单代码即可：

```js
import { transform } from 'babel-core';
var result = babel.transform("code();", options);
result.code;
result.map;
result.ast;
```

我们在Node中使用的时候一般都是使用的三步转换的方式，方便做更多的配置及操作。所以整个的难点主要就在对AST的操作上，为了能对AST做一些操作后进而能对js代码做到修改，babel对js代码语法提供了各种类型，比如：箭头函数类型ArrowFunctionExpression，for循环里的continue语句类型：ContinueStatement等等，我们主要就是根据这些不同的语法类型来对AST做操作（生成/替换/增加/删除节点），具体有哪些类型全部在：babel-types(https://www.npmjs.com/package/babel-types)。

其实整个大的操作流程还是比较简单的，我们直接上例子好了。

## 3、示例

### Babel使用案例0：往类中插入方法
比如我们有这样的需求：我们有一个jsx代码模板，该模板中有一个类似与下面的组件类：
```js
class MyComponent extends React.Component {
    constructor(props, context) {
        super(props, context);
    }

    // 其他代码
}
```  

我们会需要根据当前的DSL生成对应的render方法并插入进`MyComponent`组件类中，该如何实现呢？  

上面已经讲到，我们对代码的操作其实是通过`对代码生成的AST操作生成一个新的AST`来完成的，而对AST的操作则是通过`babel-traverse`这个库来实现的。  

该库通过简单的hooks函数的方式，给我们提供了在遍历AST时可以操作当前被遍历到的节点的相关操作，要获取并修改（增删改查）当前节点，我们需要知道AST都有哪些节点类型，而所有的节点类型都存放于`babel-types`这个库中。我们先看完整的实现代码，然后再分析：

```js
// 先引入相关的模块
const babylon = require('babylon');
const Traverse = require('babel-traverse').default;
const generator = require('babel-generator').default;
const Types = require('babel-types');
const babel = require('babel-core');

// === helpers ===

// 将js代码编译成AST
 function parse2AST(code) {
    return babylon.parse(code, {
        sourceType: 'module',
        plugins: [
            'asyncFunctions',
            'classConstructorCall',
            'jsx',
            'flow',
            'trailingFunctionCommas',
            'doExpressions',
            'objectRestSpread',
            'decorators',
            'classProperties',
            'exportExtensions',
            'exponentiationOperator',
            'asyncGenerators',
            'functionBind',
            'functionSent'
        ]
    });
}

// 直接将一小段js通过babel.template生成对应的AST
function getTemplateAst(tpl, opts = {}) {
    let ast = babel.template(tpl, opts)({});

    if (Array.isArray(ast)) {
        return ast;
    } else {
        return [ast];
    }
}

/**
 *  检测传入参数是否已在插入代码中定义
 */
checkParams = function(argv, newAst) {
    let params = [];
    const vals = getAstVals(newAst);
    if (argv && argv.length !== 0) {
        for (let i = 0; i < argv.length; i++) {
            if (vals.indexOf(argv[i]) === -1) {
                params.push(Types.identifier(argv[i]));
            } else {
                throw TypeError('参数名' + argv[i] + '已在插入代码中定义，请更名');
            }
        }
    }
    return params;
}

const code = `
    class MyComponent extends React.Component {
        constructor(props, context) {
            super(props, context);
        }

        // 其他代码
    }
`;

const insert = [
    {
        // name为方法名
        name: 'render',
        // body为方法体
        body: `
            return (
                <div>我是render方法的返回内容</div>
            );
        `,
        // 方法参数
        argv: null,
        // 如果原来的Class有同名方法则强制覆盖
        isCover: true
    }
];

const ast = parse2AST(code);

Traverse(ast, {
    // ClassBody表示当前类本身节点
    ClassBody(path) {
        if (!Array.isArray(insert)) {
            throw TypeError('插入字段类型必须为数组');
        }

        for (let key in insert) {
            const methodObj = insert[key],
                name = methodObj.name,
                argv = methodObj.argv,
                body = methodObj.body,
                isCover = methodObj.isCover;

            if (typeof name !== 'string') {
                throw TypeError('方法名必须为字符串');
            }

            const newAst = getTemplateAst(body, {
                sourceType: "script"
            });
            
            const params = checkParams(argv, newAst);
            
            // 通过Types.ClassMethodAPI，生成方法AST
            const property = Types.ClassMethod('method', Types.identifier(name), params, Types.BlockStatement(newAst));

            // 插入进AST
            path.node.body.push(property);
        }
    }
});

console.log(generator(ast).code);
```

其中，最核心的地方就是下面的这一行代码：
```js
const property = Types.ClassMethod('method', Types.identifier(name), params, Types.BlockStatement(newAst));
```

确定好我们要进行怎么样的操作（比如要往一个类中插入一个方法），休闲要确定是怎样的钩子名（这里是ClassBody），然后通过要插入的代码生成对应的AST，生成AST可以通过Babel.Types的相关方法一点点生成，但是这里有个比较方便的API：babel.template，然后通过path的相关操作将新生成的AST插入即可。

### 穿插：AST树的创建方法

一些AST树的创建方法，有：
1、使用babel-types定义的创建方法创建
比如创建一个var a = 1;
```javascript
types.VariableDeclaration(
     'var',
     [
        types.VariableDeclarator(
                types.Identifier('a'), 
                types.NumericLiteral(1)
        )
     ]
)
```

如果使用这样创建一个ast节点，肯定要累死了，可以：
- 使用replaceWithSourceString方法创建替换
- 使用template方法来创建AST结点
- template方法其实也是babel体系中的一部分，它允许使用一些模板来创建ast节点

比如上面的var a = 1可以使用：
```js
var gen = babel.template(`var NAME = VALUE;`);
 
var ast = gen({
    NAME: t.Identifier('a'), 
    VALUE: t.NumberLiteral(1)
});
```
也可以简单写：
```js
var gen = babel.template(`var a = 1;`);
 
var ast = gen({});
```

### Babel使用案例1：往类的方法中插入代码

这个案例会更复杂一点，大家可以先试着去实现下，明天再讲解具体实现。

往方法中要插入代码，我们先找下类中方法的babel-types值是什么，查阅文档：https://www.npmjs.com/package/babel-types,可以发现是叫：ClassMethod。于是就可以像下面这样实现：

```js
const injectCode = [{
    name: 'constructor',
    code: insertCodeNext,
}];

const ast = parse2AST(originCode);
Traverse(ast, {
    ClassMethod(path) {
        if (!Array.isArray(injectCode)) {
            throw TypeError('插入字段类型必须为数组');
        }

        // 获取当前方法的名字
        const methodName = path.get('body').container.key.name;

        for (let key in injectCode) {
            const inject = injectCode[key],
                name = inject.name,
                code = inject.code,
                pos = inject.pos;

            if (methodName === name) {
                const newAst = getTemplateAst(code, {
                    sourceType: "script"
                });

                if (pos === 'prev') {
                    Array.prototype.unshift.apply(path.node.body.body, newAst);
                } else {
                    Array.prototype.push.apply(path.node.body.body, newAst);
                }
            }
        }
    }
});

console.log(generator(ast).code);
```

其实跟往Class中插入method一样的道理。

## 4、Babel插件开发介绍

Babel的插件就是一个带有babel参数的函数，该函数返回类似于babel-traverse的配置对象，即下面的格式：

```js
module.exports = function(babel) {
    var t = babel.types;

    return {
        visitor: {
            ImportDeclaration(path, ref) {
                var opts = ref.opts; // 配置的参数
            }
        }
    };
};
```
在babel插件的时候，配置的参数就会存放在ref参数里，见上面的代码所所示。具体可以参见babel插件手册：https://github.com/thejameskyle/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md。

下面我们看一个具体的示例。

## 5、示例：通过Babel实现打包构建优化 -- 组件模块按需打包

### 需求
比如，我们有一个UI组件库，在入口文件中会把所有的组件放在这里，并export出对外服务，大概类似于如下的代码：

```js
export Button from './lib/button/index.js';
export Input from './lib/input/index.js';
// ......
```

那么我们在使用的时候就可以如下引用：
```js
import {Button} from 'ant'
```

这样就有一个问题，就是比如我们只是用了一个Button组件，这样引用就会导致会把所有的组件打包进来，导致整个js文件会非常大。我们能不能把代码动态实时的编译成如下的代码来解决这个问题？

```js
import Button from 'ant/lib/button';
```

我们可以写个babel插件来实现这样的需求。
```js
// 入口文件
var extend = require('extend');
var astExec = require('./ast-transform');

// 一些个变量预设
var NEXT_MODULE_NAME = 'ant';
var NEXT_LIB_NAME = 'lib';
var MEXT_LIB_NAME = 'lib';

module.exports = function(babel) {
    var t = babel.types;

    return {
        visitor: {
            ImportDeclaration: function ImportDeclaration(path, _ref) {
                var opts = _ref.opts;
                var next = opts.next || {};

                var nextJsName = next.nextJsName || NEXT_MODULE_NAME;
                var nextCssName = next.nextCssName || NEXT_MODULE_NAME;
                var nextDir = next.dir || NEXT_LIB_NAME;
                var nextHasStyle = next.hasStyle;

                var node = path.node;

                var baseOptions = {
                    node: node,
                    path: path,
                    t: t,
                    jsBase: '',
                    cssBase: '',
                    hasStyle: false
                };

                if (!node) {
                    return;
                }

                var jsBase;
                var cssBase;

                if (node.source.value === nextJsName) {
                    jsBase = nextJsName + '/' + nextDir + '/';
                    cssBase = nextCssName + '/' + nextDir + '/';

                    astExec(extend(baseOptions, {
                        jsBase: jsBase,
                        cssBase: cssBase,
                        hasStyle: nextHasStyle
                    }));
                }
            }
        }
    };
};
```

这里将部分的功能单独放到了一个ast-transform文件中，代码如下：
```js
function transformName(name) {
    if (!name)
        return '';
    return name.replace(/[A-Z]/g, function(ch, index) {
        if (index === 0)
            return ch.toLowerCase();
        return '-' + ch.toLowerCase();
    });
}

module.exports = function astExec(options) {
    var node = options.node; // 当前节点
    var path = options.path; // path辅助处理变量
    var t = options.t; // babel-types
    var jsBase = options.jsBase;
    var cssBase = options.cssBase;
    var hasStyle = options.hasStyle;

    node.specifiers.forEach(specifier => {
        if (t.isImportSpecifier(specifier)) {
            var comName = specifier.imported.name;
            var lcomName = transformName(comName);
            var libName = jsBase + lcomName;
            var libCssName = cssBase + lcomName + '/index.scss';

            // AST节点操作
            path.insertAfter(t.importDeclaration([t.ImportDefaultSpecifier(t.identifier(comName))], t.stringLiteral(libName)));

            if (hasStyle) {
                path.insertAfter(t.importDeclaration([], t.stringLiteral(libCssName)));
            }
        }
    });

    // 把原来的代码删除掉
    path.remove();
};
```

这样我们在用的时候就可以像下面这样使用：
在`.babelrc`文件中像下面这样配置即可：
```js
{
  "presets": [...], // babel-preset-react等
  "plugins" :[
    [
      'armor-fusion',
      {
          next: {
              jsName: 'ant', //js库名，默认值:ant
              cssName: 'ant', //css库名，当如果其他的主题包时，可以换成别的主题包名,默认值:ant
              dir: 'lib', //目录名,一般不需要设置，默认值:lib
              hasStyle: true //会编译出scss引用，不加则默认不会编译
          }
      }
    ]
  ]
}
```

大家可以把上面比较实用的插件功能整理下放到自己的github上，也许能给你的面试加分也说不定哦。

## 硬广
欢迎想探讨前端一切问题的同学加入《前端专题攻破》小密圈，一起学习进步，这里会有更多的开发技巧、黑魔法、项目实践、知识点探究及开发效率提升实践等专题分享！相关链接：
[前端专题攻破](http://t.xiaomiquan.com/NVJq7YR)
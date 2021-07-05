
计算机语言分为编译型语言和解释型语言，编译型表示将代码进行‘翻译’后进行运行，解释型则是一边运行一边‘翻译’。编译型代表有：C、C++、Java 等、解释型代表有：Javascript、Python 等。
> 计算机底层并不认识我们所使用的这些‘高级语言’，它只能识别 010101 之类的二进制。所以我们的代码需要层层翻译后计算机才能识别。

虽然 Javascript 是解释型语言，但是我们也是绕不开编译，今天我们能够在开发中使用 ES6+、TS、SASS、ESLint 等等，都离不开
‘编译’。

常用的 JavaScript Parser：

- esprima

- uglifyJS2

- traceur

- acorn

- espree

- @babel/parser

那么 JS 的编译是如何进行的呢？
![image.png](https://i.loli.net/2021/05/28/OltH3B6GL9gQ8Xz.png)

以 babel 执行 es6 转 es5 为例：
![image.png](https://i.loli.net/2021/05/28/Ldm9lojW18yKZnw.png)

以下我们将以 Babel 为例进行进一步的探索;

> 不同 Parser 生成的 AST 会有一定不同

## 词法分析

在进行编译时，编译器会逐一读取字符流，分词就是将字符进行识别标记的过程；

```js
var foo = 1;
```

上面这行为例，读取字符流及分词的步骤如下：

1. v
2. a
3. r
4. 空格
   > 识别为变量声明，生成 Token `Keyword(var)`
5. f
6. o
7. o
8. 空格
   > 识别为变量声明的标识符，生成 Token `Identifier(foo)`
9. =
   > 识别为变量声明的付值操作符，生成 Token `Punctuator(=)`
10. 空格
11. 1
12. ;
    > 识别为标识符分配的值，生成 Token `NumericLiteral(1)`
    > 识别为语句的结束符号，生成 Token `Punctuator(;)`

分词的产出物就是一个类似下面的扁平的数组：

```js
[
  { type: { ... }, value: "var", start: 0, end: 2, loc: { ... } },
  { type: { ... }, value: "foo", start: 4, end: 6, loc: { ... } },
  { type: { ... }, value: "=", start: 8, end: 9, loc: { ... } },
  ...
]
```

得到这些词后，就可以进入下一步语法分析了；

> 使用 [AST VISUALIZER](https://resources.jointjs.com/demos/javascript-ast) 可以更加直观感受到分词的效果。

## 语法分析

以 Babel 为例，其会根据上一步生成的 Token 转为 AST 的表述结构；（你可以通过 [AST Explorer](https://astexplorer.net/)实时生成 AST）

```js
{
  "type": "File",
  "start": 0,
  "end": 12,
  "loc": {
    "start": {
      "line": 1,
      "column": 0
    },
    "end": {
      "line": 1,
      "column": 12
    }
  },
  "errors": [],
  "program": {
    "type": "Program",
    "start": 0,
    "end": 12,
    "loc": {
      "start": {
        "line": 1,
        "column": 0
      },
      "end": {
        "line": 1,
        "column": 12
      }
    },
    "sourceType": "module",
    "interpreter": null,
    "body": [
      {
        "type": "VariableDeclaration",
        "start": 0,
        "end": 12,
        "loc": {
          "start": {
            "line": 1,
            "column": 0
          },
          "end": {
            "line": 1,
            "column": 12
          }
        },
        "declarations": [
          {
            "type": "VariableDeclarator",
            "start": 4,
            "end": 11,
            "loc": {
              "start": {
                "line": 1,
                "column": 4
              },
              "end": {
                "line": 1,
                "column": 11
              }
            },
            "id": {
              "type": "Identifier",
              "start": 4,
              "end": 7,
              "loc": {
                "start": {
                  "line": 1,
                  "column": 4
                },
                "end": {
                  "line": 1,
                  "column": 7
                },
                "identifierName": "foo"
              },
              "name": "foo"
            },
            "init": {
              "type": "NumericLiteral",
              "start": 10,
              "end": 11,
              "loc": {
                "start": {
                  "line": 1,
                  "column": 10
                },
                "end": {
                  "line": 1,
                  "column": 11
                }
              },
              "extra": {
                "rawValue": 1,
                "raw": "1"
              },
              "value": 1
            }
          }
        ],
        "kind": "var"
      }
    ],
    "directives": []
  },
  "comments": []
}
```

上面数据中，`Program` 是我们的程序本体，其中的 `body` 就是我们的代码了。body 是个数组，其中只有一个对象，说明是只有一条执行语句。
我们通过该对象内后代节点的 type 属性来说明下每个节点：

- VariableDeclaration：变量声明，它的同级属性 `kind` 表明它是个 `var` 声明，`declarations` 内描述了具体的声明信息，它允许存在多个，如：`var a,b;`；
- VariableDeclarator：变量声明符，用来标记每个声明的标识符及初始值信息，分别对应同级属性 `id` 和 `init`；
- Identifier：标识符，它用来给变量进行命名，即上例中的 `foo`；
- NumericLiteral：数字字面量，它是上例中的 `1`；

除此之外，AST 节点还包括 loc、comment、extra 等描述信息；
<br/>

下面是常用的节点类型含义对照表（完整版：[babel-types](https://babeljs.io/docs/en/babel-types)）：

| 类型名称 | 中文译名 | 描述 |
|----|----|----|
| Program | 程序主体 | 整段代码的主体 |
| VariableDeclaration | 变量声明 | 声明变量，比如 let const var |
| FunctionDeclaration | 函数声明 | 声明函数，比如 function |
| ExpressionStatement | 表达式语句| 通常为调用一个函数，比如 console.log(1) |
| BlockStatement | 块语句 | 包裹在 {} 内的语句，比如 if (true) { console.log(1) } |
| BreakStatement | 中断语句 | 通常指 break |
| ContinueStatement | 持续语句 | 通常指 continue |
| ReturnStatement | 返回语句 | 通常指 return |
| SwitchStatement | Switch 语句| 通常指 switch |
| IfStatement | If 控制流语句| 通常指 if (true) {} else {} |
| Identifier | 标识符 | 标识，比如声明变量语句中 const a = 1 中的 a |
| ArrayExpression | 数组表达式 | 通常指一个数组，比如 [1, 2, 3] |
| StringLiteral | 字符型字面量 | 通常指字符串类型的字面量，比如 const a = '1' 中的 '1' |
| NumericLiteral | 数字型字面量 | 通常指数字类型的字面量，比如 const a = 1 中的 1 |
| ImportDeclaration | 引入声明 | 声明引入，比如 import |


## 使用场景

AST 使用场景通常在一些打包工具、代码检查、语法高亮等等，我们接下来依然用 Babel 来做一个简单的 Plugin 来进行介绍；

让我们快速编写一个可用的插件来展示一下它是如何工作的。下面是我们的源代码：

```js
foo === bar;
```

其 AST 形式如下：

```js
{
  type: "BinaryExpression",
  operator: "===",
  left: {
    type: "Identifier",
    name: "foo"
  },
  right: {
    type: "Identifier",
    name: "bar"
  }
}
```

我们 Plugin 大致如下：

```js
export default function(babel) {
  // plugin contents
  const { type: t } = babel;

  return {
    visitor: {
      // visitor contents
      BinaryExpression(path) {
        if (path.node.operator !== "===") {
          return;
        }
        path.node.left = t.identifier("sebmck");
        path.node.right = t.identifier("dork");
      }
    }
  };
}
// 上面插件编译后的结果为：
sebmck === dork;
```

这个插件里，type 是 babel 提供可以用来创建各种 AST 节点的对象，而 return 的 {visitor: ...} 按照官方说明是基于访问者模式（visitor）而设计的，
其定义是用来访问每个 AST 节点的。
所以 visitor 的属性大多和 AST Node 的 Type 相同。比如上例中的 `BinaryExpression`，表明在遍历 AST Tree 时，遇到 Type 为
`BinaryExpression` 的节点，则执行 `BinaryExpression(path) {}` 方法，你可以在这里对该节点进行任意操作；
这里返回的 Path 是表示两个节点之间连接的对象。它的基本属性如下：

- 属性

    - node 当前节点
    - parent 父节点
    - parentPath 父 path
    - scope 作用域
    - context 上下文
    - ...

- 方法
    - get 当前节点
    - findParent 向父节点搜寻节点
    - getSibling 获取兄弟节点
    - replaceWith 用 AST 节点替换该节点
    - replaceWithMultiple 用多个 AST 节点替换该节点
    - insertBefore 在节点前插入节点
    - insertAfter 在节点后插入节点
    - remove 删除节点
    - ...

我们通过 path 取到 node，最后通过 type 提供的方法创建一个新的标识符以修改源码：

```js
path.node.left = t.identifier("sebmck");
```

### 实际案例

根据上面插件介绍，我们实现一个 Antd 按需加载的 Plugin；
假设源码如下：

```js
import { Button } from "antd";
```

转为 AST 如下：

```js
{
  "type": "ImportDeclaration",
  "start": 0,
  "end": 30,
  "loc": {
    "start": {
      "line": 1,
      "column": 0
    },
    "end": {
      "line": 1,
      "column": 30
    }
  },
  "specifiers": [
    {
      "type": "ImportSpecifier",
      "start": 9,
      "end": 15,
      "loc": {
        "start": {
          "line": 1,
          "column": 9
        },
        "end": {
          "line": 1,
          "column": 15
        }
      },
      "imported": {
        "type": "Identifier",
        "start": 9,
        "end": 15,
        "loc": {
          "start": {
            "line": 1,
            "column": 9
          },
          "end": {
            "line": 1,
            "column": 15
          },
          "identifierName": "Button"
        },
        "name": "Button"
      },
      "importKind": null,
      "local": {
        "type": "Identifier",
        "start": 9,
        "end": 15,
        "loc": {
          "start": {
            "line": 1,
            "column": 9
          },
          "end": {
            "line": 1,
            "column": 15
          },
          "identifierName": "Button"
        },
        "name": "Button"
      }
    }
  ],
  "importKind": "value",
  "source": {
    "type": "StringLiteral",
    "start": 23,
    "end": 29,
    "loc": {
      "start": {
        "line": 1,
        "column": 23
      },
      "end": {
        "line": 1,
        "column": 29
      }
    },
    "extra": {
      "rawValue": "antd",
      "raw": "'antd'"
    },
    "value": "antd"
  },
  "assertions": []
}
```

编写插件：

```js
module.exports = function testPlugin(babel) {
  // plugin contents
  const { types: t } = babel;

  return {
    visitor: {
      // visitor contents
      ImportDeclaration(path) {
        if (path.node?.source?.value === "antd") {
          const specifiers = path.node.specifiers;

          const declarations = specifiers.map(specifier => {
            const newNode = t.importDeclaration(
              [t.ImportDefaultSpecifier(specifier.local)],
              t.stringLiteral("antd/lib/button.js")
            );
            return newNode;
          });

          path.replaceWithMultiple(declarations);
        }
      }
    }
  };
};
// 编译结果：
import Button from "antd/lib/button.js";
console.log(Button);
```


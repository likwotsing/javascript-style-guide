# 1介绍

本文档完整地定义了谷歌的JavaScript编程语言源代码的编码标准。当且仅当JavaScript源文件遵循本文的规范时，才将其称为Google Style。

就像其它编程风格指南一样，这些问题不仅涉及到格式化的美学问题，还涉及到其它类型的约定或编码标准。然而，本文档主要关注的是我们普遍遵循的硬性规则，并避免给出无法明确执行的建议（无论是通过人工还是使用工具）。

## 1.1术语说明

在本文档，除非特别说明：

1. 术语注释总是指实现注释。我们不使用"documentation comments"短语，而是使用人和机器都可以识别的公共术语"JSDoc"的`/** ···*/`注释。
2. 本风格指南使用短语must、must not、should、should not和may时遵循了RFC2119术语。术语*prefer*和*avoid*分别对应*should*和*should not*。命令式和声明式语句是说明性的，并且与*must*对应。

其它术语说明会偶尔在文档中出现。

## 1.2指南说明

文档中的示例代码是不规范的。也就是说，当示例代码使用的是Google Style，它们不是显示代码的唯一方式。示例中所显示的可选格式不能作为规则强制执行。

# 2源文件的基本信息

## 2.1文件名称

文件名称必须小写，可以包括下划线和连接号，但不能有额外的标点符号。遵循项目使用的规则。文件的扩展名必须为`.js`。

## 2.2文件编码：UTF-8

源文件使用UTF-8编码。

## 2.3特殊字符
### 2.3.1空白字符

除了行终止符，文件中唯一可以出现的是ASCII水平空白字符，也就是说：
1. 在字符串里所有其它空白字符都转义
2. Tab制表符不用于缩进

### 2.3.2特殊转义序列
对任何字符，若有特殊的转义序列`\', \" \\, \b, \f, \n, \r, \t, \v`，那就不要使用对应的数字转义（例如：`\x0a, \u000a, \u{a}`）。不要使用遗留的八进制转义。

### 2.3.3非ASCII字符

对于remaining的非ASCII字符（此处应该指的是超过65535的字符），可以使用实际的Unicode字符（例如：`∞`），也可以使用对应的十六进制或转义Unicode（例如：`\u221e`），哪种能够使代码更容易阅读和理解就使用哪种。

> 建议：使用Unicode转义，或者有时使用实际的Unicode字符时，注释是非常有用的

例子 | 说明
-- | --
const units = 'μs'; | best：在没有注释的情况下代码非常干净
const units = '\u03bcs'; // 'μs' | Allowed，但是没有理由这么做
const units = '\u03bcs'; // Greek letter mu, 's' | Allowed，但是容易出错
const units = '\u03bcs'; | Poor，阅读者不知道啥意思
return '\ufeff' + content; // byte order mark | Good，对非打印字符使用转义，也有必要的注释

> 建议：永远不要担心程序不能正确处理非ASCII字符而降低代码的可阅读性。如果有这种情况，那程序必须重构了。


# 3源文件结构

一个源文件包括以下几部分：
1. 许可证或版权说明，如果有的话
2. `@fileoverview` JSDoc，如果有的话
3. goog.module 声明
4. goog.require 声明
5. 文件的实现部分
  
在不同的section之间确保**只有一行空白**，除非是文件的实现部分，可能会有1或2行空白。

## 3.1许可证或版权信息，如果有的话

如果文件有许可证或版权说明，那就应该出现在这里

## 3.2 `@fileoverview`JSDoc，如果有的话

查看7.5文件注释的规则

## 3.3goog.module声明

所有文件必须在单独一行声明一个goog.module名称：改行包括了goog.module声明并且不能被其他wrapped，除非超过了限制的80列。

例子
```
goog.module('search.urlHistory.UrlHistoryService');
```

### 3.3.1层级

永远不要把其他模块命名空间的直接子元素命名为模块的命名空间。

不合法的：
```
goog.module('foo.bar');   // 'foo.bar.qux'可能会好点儿
goog.module('foo.bar.baz')
```
文件夹的层级反应了命名空间的层次结构，深层次嵌套的子目录是高层级的子目录。这意味着从它们在同一个文件夹开始，父命名空间的所有者必然知道所有的子命名空间。

### 3.3.2goog.setTestOnly

在调用`goog.setTestOnly()`后，可以选择性地配置`goog.module`声明。

### 3.3.3goog.module.declareLegancyNamespace

在调用`goog.module.declareLegacyNamespace()`后，可以选择性地配置`goog.module`。

例子：
```
goog.module('my.test.helpers');
goog.module.declareLegacyNamespace();
goog.setTestOnly();
```
`goog.module.declareLegacyNamespace`的存在简化了传统的基于对象层次结构的命名空间的转换，但同时也带来了一些命名限制。由于子模块命名必须在父命名空间之后创建，名称不能是`goog.mogule`的其他任何子名称或父名称（例如，`goog.module('parent')`和`goog.module('parent.child')`不能同时存在，`goog.module('parent')`和`goog.module('parent.child.grandchild')`也不能）。

### 3.3.4 ES6 Modules

不要使用ES6模块（例如`export`和`import`关键字），因为这些语义还没有完全标准化。等这些规则标准化后就可以使用了。

## 3.4 `goog.require`声明

在`goog.require`声明后就会完成导入，在模块声明后会立即重组。每个`goog.require`会被分配一个常量别名，或者拆解后分配多个常量别名。这些别名时唯一能够识别`required`的依赖，无论是在代码中还是在类型注释中：除了`goog.require`外都不使用完全限定名。别名应该匹配导入模块的最后一个使用点分隔符的部分，如果需要消除歧义，也可使包括更多的部分（使用适当的外层包裹，以便别名仍然能正确识别其类型），或者显著提高了可阅读性。`goog.require`不能出现在文件的其他任何地方。

如果一个模块的导入只是为了使用其
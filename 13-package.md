# 编写R包

我们不介绍官方的编写方法，因为它需要太多的人力。我们的观点是能自动化的全都自动化（凡是需要记忆的都是将来出错的隐患），本章基于一个范例R包**rmimi**：<https://github.com/yihui/rmini>。尽管我们不推崇官方的办法，但还是需要说明一下，官方手册是“[Writing R Extensions](http://cran.r-project.org/doc/manuals/R-exts.html)”，它涵盖了所有编写R包的细节和规则，忍者可能需要时不时参考一下，但通常不需要通读。继续阅读之前，请用GIT将**rmini**包克隆到本地：

```bash
git clone git://github.com/yihui/rmini.git
```

以下内容用<http://cos.name/2011/05/write-r-packages-like-a-ninja/>填充。

## 工具链

Windows装Rtools

## R包结构

一个最简单的包结构如下（括号中为相应解释）：

    pkg (包的名字，请使用一个有意义的名字，不要照抄这里的pkg三个字母)
    |
    |--DESCRIPTION (描述文件，包括包名、版本号、标题、描述、依赖关系等)
    |--R (函数源文件)
       |--function1.R
       |--function2.R
       |--...
    |--man (帮助文档)
       |--function1.Rd
       |--function2.Rd
       |--...
    |--...

DESCRIPTION文件描述一个包的信息（<https://github.com/yihui/rmini/blob/master/DESCRIPTION>），包括：

- 包的名字
- 版本（介绍语义版本命名法，主要.次要.补丁：<http://semver.org/>，让版本号变得有意义，除非你是Knuth，用pi做版本号）
- 日期
- 标题
- 描述（详细说明）
- 作者（可以多人）
- 维护者（一个人，可以不同于作者，必须要有邮箱）
- 依赖关系
  - Depends 加载这个包会依赖加载进来的包
  - Imports 只是导入命名空间，不直接加载（被导入的包中的函数对用户不直接可见）
  - Suggests 推荐安装的包，通常不涉及到本包的核心功能，但如果有这些包的话，本包会更强大
- 许可证（发布到CRAN的包必须用开源许可证，不限于GPL）
- 网址
- Bug报告地址
- R源文件列表（指定用哪些R代码来创建本包）


## 重要的命令

`R CMD build`

`R CMD INSTALL`

## roxygen

Roxygen注释可以通过**roxygen2**包翻译为官方Rd文档文件，注释以一个或多个井号开头，如`#'`或`##'`。主要的用法参见<https://github.com/yihui/rmini/blob/master/R/roxygen.R>，说明如下：

第一段为标题（对应Rd中的`\title{}`），第二段为描述（对应`\description{}`），接下来的是细节描述（`\details{}`）；然后可以用`@`开头的标签来写一些细节文档，例如`@param`写函数参数的说明，`@return`说明本函数的返回值，`@author`写作者，`@examples`提供示例代码。一些高级标签下面介绍。用roxygen注释写好文档之后在R里面使用**roxygen2**包翻译这些注释为Rd文档：

```r
library(roxygen2)
roxygenize('rmini') # 保证rmini文件夹在当前目录下
```

现在检查man文件夹，里面多了一些`*.Rd`文件，就是官方的Rd文档文件。

用roxygen而不直接写Rd的最大好处在于文档和源代码在同一个地方，开发程序的时候只需要在同一个文件内操作即可：举头望文档，低头思函数。实际上这是文学化编程（Literate Programming）的思想。

## 其它子目录

data文件夹放R数据，扩展名为`rda`，通常可以用`save()`函数生成，例如


```r
iris3 = iris
save(iris3, file = "iris3.rda")
```


然后把`iris3.rda`文件放到`data`文件夹底下。对每一个数据，都必须有相应的Rd文档，它可以通过roxygen生成，参见<https://github.com/yihui/rmini/blob/master/R/data.R>。其中关键点有：

- `@docType`必须为`data`
- 必须有`@name`，因为roxygen不能从底下的R代码中推导出这份文档的名字（对普通函数文档来说，可以从赋值符号的左边推导出来）
- R代码不能为空（否则roxygen会跳过这段文档），通常可以用`NULL`填充
- 其它标签可选，例如`@format`说明这份数据的格式，`@source`说明它的来源

demo文件夹里可以放一些演示，这些演示文件将来可以用`demo()`函数来调用。一个演示文件就是一个R代码文件，注意所有的演示名称都必须写入一个`00Index`文件，里面同时也要写演示的标题，参见<https://github.com/yihui/rmini/tree/master/demo>。**rmini**包中有一个演示叫`mini_fun.R`，那么安装好这个包之后我们可以以这样的方式观看这个演示：


```r
demo("mini_fun", package = "rmini")
```


inst文件夹下的所有文件都会被原封不动复制到安装包的路径下，这个文件夹下可以放任意文件，但有一个例外是`doc`，它用来放R包的手册（Vignette），后文详述。

## 命名空间

命名空间（NAMESPACE）是R包管理包内对象的一个途径，它可以控制哪些R对象是对用户可见的，哪些对象是从别的包导入（import），哪些对象从本包导出（export）。为什么要有这么个玩意儿存在？主要是为了更好管理你的一堆对象。写R包时，有时候可能会遇到某些函数只是为了另外的函数的代码更短而从中抽象、独立出来的，这些小函数仅仅供你自己使用，对用户没什么帮助，他们不需要看见这些函数，这样你就可以在包的根目录下创建一个NAMESPACE文件，里面写上`export(函数名)`来导出那些需要对用户可见的函数。自R 2.14.0开始，命名空间是R包的强制组成部分，所有的包必须有命名空间，如果没有的话，R会自动创建。

前面我们也提到DESCRIPTION文件中有Imports一栏，这里设置的包通常是你只需要其部分功能的包，例如我只想在我的包中使用**foo**包中的`bar()`函数，那么Imports中就需要填`foo`，而NAMESPACE中则需要写`importFrom(foo, bar)`，在自己的包的源代码中则可以直接调用`bar()`函数，R会从NAMESPACE看出这个`bar()`对象是从哪里来的。

roxygen注释对这一类命名空间有一系列标签，如一个函数的文档中若标记了`#' @export`，那么这个函数将来就会出现在命名空间文件中（被导出），若写了`#' @importFrom foo bar`，那么**foo**包的`bar`对象也会被写在命名空间中。这些内容参见官方手册的1.6节和roxygen2的`?export`帮助。

仍然以**rmini**包为例（<https://github.com/yihui/rmini/blob/master/R/roxygen.R>），对`split_filename()`函数我们使用了`@importFrom`，它将**tools**包中的两个函数`file_ext()`和`file_path_sans_ext()`导入到**rmini**包，这样我们就可以在包内明目张胆使用这两个函数了，而不必`library(tools)`再用；对于懒人来说，可以用`@import`导入一个包中所有可见对象，但我们不提倡这种铺张浪费的导入方式，而是用什么函数就导入什么函数。

最后，我们可以看见包中的`add_one()`函数没有被导出，但在**rmini**包的内部它是可以被随意调用的，而用户`library(rmini)`之后看不到它，此时我们可以用暗黑的三冒号访问它，如`rmini:::add_one`，但通常我们也不推荐这种方式，因为一个对象不导出通常是有其理由的，对本作者而言，这些未导出的对象可能有被更名甚至删除的危险，所以写包的时候尽量不要依赖别人未导出的对象。

## S3泛型函数

S3泛型函数的核心思想是基于对象的类去匹配函数，示例参见<https://github.com/yihui/rmini/blob/master/R/S3.R>。S3函数可以用`UseMethod()`去定义，然后函数加`.类名`就是具体的子函数，例如`hello()`这个函数有两个子函数`hello.default()`和`hello.character()`，分别对应它的默认方法以及对字符对象应用的方法。


```r
library(rmini)
hello(1)
```

```
## hello, numeric
```

```r
hello("a")
```

```
## Hi! I love characters!
```

```r
hello(structure(1, class = "world"))
```

```
## hello, world
```


对S3子函数，roxygen中可以用`@S3method 函数名 类名`来声明这是一个S3泛型函数，而不是一个普通的`*.*`函数，这一点非常重要，它告诉R一个形如`foo.bar`的函数到底该如何调用。同时这也引出一个编程规范的问题：如果不是S3函数，尽量不要在函数名中用点，比如可以以下划线代替（`foo_bar`），可惜R内部就存在这种混乱，如`package.skeleton()`等函数都只是普通函数。声明了`@S3method`的函数将来会在NAMESPACE文件中添加一项`S3method()`，R依据这个命名空间文件来决定带点的函数究竟是什么样的函数。

最后说一句，S3的意思是第3代S语言，S4是第4代，这里不介绍S4。

## 嵌入其它语言

R可以与其它语言沟通，常见的如C和Fortran，这里举一个C的例子（<https://github.com/yihui/rmini/tree/master/src>），其它语言的源代码都放在`src`文件夹底下。`reverse.c`是一个小白例子，它将一个数值向量中的元素倒序过来（R函数`rev()`可以干这事儿），这个c文件将来在`R CMD INSTALL`过程中会被编译成一个动态链接库，供R调用。

R函数`reverse()`（<https://github.com/yihui/rmini/blob/master/R/C.R>）中我们使用`.C()`调用前面提到的C函数。注意这里在调用之前我们必须告诉R加载编译好的动态链接库，所以我们使用`@useDynLib`标签，它会在NAMESPACE文件中生成相应的`useDynLib()`命令，当R包加载的时候，动态链接库也会被加载。

## 手册

R有自己独特的手册编写方法，手册源文档可以是一个Sweave文档（`*.Rnw`），放在`inst/doc/`目录下，Sweave是R代码和LaTeX的混合体。**rmini**包提供了一个示例（<https://github.com/yihui/rmini/tree/master/inst/doc>），R代码部分用Sweave语法，其它部分都是普通的LaTeX语法。

为了编制索引方便，R手册需要在注释中声明`% \VignetteIndexEntry{文档标题}`，这个标题将来会在R包的帮助文档中出现，例如打开`help.start()`，点到该包的帮助文档，里面会出现手册列表，手册的标题就会出现在那个列表里。

理论上来说，手册是最有用的学习资源，因为它就像一篇论文（有些R包的手册就是真的论文），相比起单个函数的帮助页面来说，手册的内容可以更丰富，加上Sweave的帮助，更是丰富了内容，同时也保证了代码的可执行性（因为手册都是动态编译出来的，里面的代码每次都要重复执行）。不过可惜，带有手册的包只是少数，Sweave本身也有各种缺陷，更多高级暗黑魔法参见<http://yihui.name/knitr/demo/vignette/>。


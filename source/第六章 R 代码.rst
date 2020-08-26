第六章 R 代码
===================

使用程序包的第一个原则是所有 R 代码都放在 ``R/`` 中。在本章中，您将了解 ``R/`` 目录、\
我对将函数组织到文件中的建议，以及一些有关良好风格的提示。您还将了解脚本中的函数和程序包中的函数之间的一些重要区别。


6.1 R 代码工作流
--------------------

使用程序包的第一个实际的优点是，很容易重新加载您的代码。可以运行 ``devtools::load_all()``，\
或者在 RStudio 中按下 \ **Ctrl/Cmd + Shift + L**\ ，同时保存所有打开的文件，以节省按键次数。

利用这个快捷键可建立一个流畅的开发流程。

1. 编辑一个 R 文件。
2. 按 Ctrl/Cmd + Shift + L。
3. 在控制台中浏览代码。
4. 修改代码，重复上面的过程。

祝贺您！您已经学到了第一个程序包的开发流程！即使您从本书中没有学到任何其他的东西，也已经了解了编辑和重新加载 R 代码的一个有用的工作流程。


6.2 组织您的函数
--------------------

\ *removed in deference to material in https://style.tidyverse.org; see tidyverse/style/#121*\ 


6.3 代码风格
-------------------

\ *removed in deference to material in https://style.tidyverse.org; see tidyverse/style/#122*\ 

TL;DR = "Use the \ `styler package <https://styler.r-lib.org/>`__\ ".


6.4 顶层代码
--------------------

到目前为止，您可能一直在编写\ **脚本**\ ，使用 ``source()`` 加载保存在文件中的 R 代码。脚本和程序包中的代码有两个主要区别：

- 在脚本中，代码在加载时运行。在程序包中，代码在编译时运行。这意味着您的程序包代码应该只创建对象，其中绝大多数是函数。
- 程序包中的函数将被用于您没有想象过的情况。这意味着您的函数需要小心处理它们与外界之间的交互。

接下来的两节将讨论这些重要的差异。


6.4.1 加载代码
....................

当您用 ``source()`` 加载脚本时，每一行代码都会执行，且执行的结果可以立刻使用。对程序包来说，情况有所不同，它的加载过程分为两步。\
当包在编译时（例如通过 CRAN），``R/`` 目录下所有的代码都会被执行，结果会被保存下来。\
当使用 ``library()`` 或 ``require()`` 加载一个程序包时，这些保存的结果就可以供您使用了。如果用这个方式来加载脚本的话，代码看起来是这样的：

.. code-block:: R

    # Load a script into a new environment and save it
    env <- new.env(parent = emptyenv())
    source("my-script.R", local = env)
    save(envir = env, "my-script.Rdata")

    # Later, in another R session
    load("my-script.Rdata")

以 ``x <- Sys.time()`` 为例，如果您把它放入一个脚本中，``x`` 会告诉您脚本是什么时候被执行 ``source()`` 的。\
但是如果您把相同的代码放入程序包中，``x`` 会告诉您程序包是什么时候被\ *编译* \ 的。

这意味着您不应该在程序包的顶层运行代码：程序包的代码只能创建对象，大部分是函数。例如，假设你的 foo 程序包包含这样的代码：

.. code-block:: R

    library(ggplot2)

    show_mtcars <- function() {
        qplot(mpg, wt, data = mtcars)
    }

如果某人试图这样使用它：

.. code-block:: R

    library(foo)
    show_mtcars()

该代码不会工作，因为 ggplot2 的 ``qplot()`` 函数不可用：``library(foo)`` 不会执行 ``library(ggplot2)``。\
程序包的顶层代码只会在程序包被编译的时候执行，而不是加载的时候。

为了解决这个问题，您可能会做如下修改：

.. code-block:: R

    show_mtcars <- function() {
        library(ggplot2)
        qplot(mpg, wt, data = mtcars)
    }

一会儿您将会看到，这同样是有问题的。需要在 ``DESCRIPTION`` 中描述您的代码所需要的程序包，\
您将在 \ `package dependencies <https://r-pkgs.org/description.html#dependencies>`__\  学到这一内容。


6.4.2 R 运行环境
.....................

脚本和程序包的另一个巨大区别是：别人会使用您的程序包，并且会在一个您从未想到的环境中使用它。这意味着你需要注意 R 的运行环境，\
这不仅包括那些可用的函数和对象，也包括所有的全局设置。如果用 ``library()`` 加载了一个包，或者用 ``options()`` 修改了一个全局设置，\
或者利用 ``setwd()`` 修改了工作目录，那么您已经修改了 R 的运行环境。如果有\ *其他*\ 函数的行为在运行您的函数前后发生了改变，\
那么您就已经修改了 R 的运行环境。修改 R 的运行环境是不好的，因为这会使得代码很难理解。

有些修改全局设置的函数不应该被使用，因为有更好的替代方法：

- \ **不要使用 **\ ``library()``\ ** 或者 **\ ``require()``。这些函数修改了搜索路径，影响了全局环境下可用的函数。更好的方式是用 ``DESCRIPTION`` 来指定您的程序包的需求，这将在下一章说明。这种方式也保证了您的程序包被安装时，它需要的程序包也会被安装。
- \ **不要使用 **\ ``source()`` 从文件加载代码。``source()`` 会将代码执行的结果添加到当前环境，因此会修改当前环境。您可以使用工具 ``devtools::load_all()``，它会自动加载 ``R/`` 目录下所有的文件。如果您要用 ``source()`` 来建立数据集，请使用 ``data/`` 目录，这将在 \ `datasets <https://r-pkgs.org/data.html#data>`__\  中讲到。

还有其他一些函数需要谨慎使用。如果你要使用它们，请确保使用 ``on.exit()`` 在退出的时候清理干净。

- 如果你修改全局的 ``options()`` 或图形的 ``par()``，先保存好旧的设置，然后在你用完之后恢复到原来的值：

.. code-block:: R

    old <- options(stringsAsFactors = FALSE)
    on.exit(options(old), add = TRUE)

- 不要修改工作目录。如果必须修改它，确保在您完成工作后改回去：

.. code-block:: R

    old <- setwd(tempdir())
    on.exit(setwd(old), add = TRUE)

- 创建图像和输出到控制台是另外两种影响 R 全局环境的方式。通常你无法避免这些（因为它们很重要！），但好的做法是把它们封装成\ **只能**\ 产生输出的独立的函数。这也使得其他人更容易将你的工作用于新的用途。例如，如果你将数据准备和绘图分成两个函数，其他人可以使用你的数据准备工作（通常是最难的部分！）来创建新的可视化结果。

另一方面，您应该避免依赖用户的运行环境，因为这些环境可能和你的不同。例如，函数 ``read.csv()`` 是危险的，\
因为 ``stringsAsFactors`` 参数的值是来自全局的 ``stringsAsFactors`` 参数。如果您希望它是 ``TRUE``（默认值），但用户如果把它设为 ``FALSE``，那您的代码就可能会出错。


6.4.3 何时需要副作用
..........................

偶尔，程序包确实需要一些副作用。最常见的情况是，您的程序包需要与外部系统进行交互——当程序包加载时，您可能需要做一些初始化设置。\
为此，您可以使用两个特殊函数：``.onLoad()`` 和 ``.onAttach()``。当程序包加载和附加时，这两个函数会被调用。\
在 \ `Namespaces <https://r-pkgs.org/namespace.html#namespace>`__\  中您会了解到这两者的区别。\
目前您应该总是使用 ``.onLoad()``，除非明确指出应该使用 ``.onAttach()``。

``.onLoad()`` 和 ``.onAttach()`` 的常见用法包括以下这些。

- 在程序包加载时显示一些有用的信息。这可以使得程序包的使用条件明确，或者显示一些有用的提示。启动信息是一个您应该使用 ``.onAttach()`` 而不是 ``.onLoad()`` 的地方。要显示启动消息，请总是使用 ``packageStartupMessage()`` 而不是 ``message()``（这可以让 ``suppressPackageStartupMessages()`` 函数来选择是否显示包的启动消息）。

.. code-block:: R

    .onAttach <- function(libname, pkgname) {
        packageStartupMessage("Welcome to my package")
    }

- 用 ``options()`` 来为您的程序包设置自定义选项。为避免和其他程序包的冲突，要确保选项名使用您的程序包名作为前缀。还要注意不要覆盖用户已设置的选项。

我在 devtools 中使用下面的代码来建立选项：

.. code-block:: R

    .onLoad <- function(libname, pkgname) {
        op <- options()
        op.devtools <- list(
            devtools.path = "~/R-dev",
            devtools.install.args = "",
            devtools.name = "Your name goes here",
            devtools.desc.author = "First Last <first.last@example.com> [aut, cre]",
            devtools.desc.license = "What license is it under?",
            devtools.desc.suggests = NULL,
            devtools.desc = list()
        )
        toset <- !(names(op.devtools) %in% names(op))
        if(any(toset)) options(op.devtools[toset])

        invisible()
    }

然后 devtools 函数可以使用比如 ``getOption("devtools.name")`` 来获得程序包作者的名字，或者判断一个默认值是否已经被设置。

- 把 R 连接到另一种编程语言。例如，如果你使用 rJava 来跟一个 ``.jar`` 文件交互，你需要调用 ``rJava::jpackage()``。要想在 R 中使用 Rcpp 模块来引用 C++ 类，可以调用 ``Rcpp::loadRcppModules()``。
- 使用 ``tools::vignetteEngine()``，注册一个 vignette 生成引擎。

正如您在上面的例子中看到的，``.onLoad()`` 和 ``.onAttach()`` 函数带有两个参数：``libname`` 和 ``pkgname``。\
但它们很少使用（当需要使用 ``library.dynam()`` 来加载已编译的代码时，它们才会被用到）。它们给出了程序包安装的路径（也就是库），以及程序包的名称。

如果您使用了 ``.onLoad()``，请考虑使用 ``.onUnload()`` 来清理任何副作用。按照惯例，``.onLoad()`` 以及相关函数通常保存在一个叫 ``zzz.R`` 的文件中。\
（注意，``.First.lib()`` 和 ``.Last.lib()`` 是 ``.onLoad()`` 和 ``.onUnload()`` 的老版本，不应该继续使用了。）


6.4.4 S4 类、泛型和方法
.............................

另一种类型的副作用是定义 S4 类、方法和泛型。R 包会捕捉这些副作用，以便当包被加载的时候可以重现它们，\
但它们需要按照正确的顺序调用。例如，在定义一个方法之前，你必须定义泛型和类。这要求 R 文件按照指定的顺序加载。\
这一顺序是由 ``DESCRIPTION`` 文件中的 ``Collate`` 字段来控制的。在 \ `docimenting S4 <https://r-pkgs.org/man.html#man-s4>`__\  中有详尽的描述。


6.5 CRAN 注记
--------------------

（每章的最后都会给出提交程序包到 CRAN 的一些提示。如果不打算提交你的程序包到 CRAN，可以忽略这些内容！）

如果打算提交您的程序包到 CRAN，您在 ``.R`` 文件中就只能使用 ASCII 字符。\
但您仍然可以在字符串中包含 Unicode 字符，这需要使用特殊的 Unicode 转义格式（例如 ``"\u1234"``）。最简单的做法是使用 ``stringi::stri_escape_unicode()``：

.. code-block:: R

    x <- "This is a bullet •"
    y <- "This is a bullet \u2022"
    identical(x, y)
    #> [1] TRUE

    cat(stringi::stri_escape_unicode(x))
    #> This is a bullet \u2022

您可以将 ``.R`` 文件传入 ``tools::ShowNonASCIIfile()`` 以检测包含非 ASCII 字符的所有行：

.. code-block:: R

    library(purrr)

    walk(list.files("R", full.names = TRUE),
        tools::showNonASCIIfile)

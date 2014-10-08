Kaleidoscope: 教程介绍和词法分析器
-------------------------------------------------

原文地址：http://llvm.org/docs/tutorial/LangImpl1.html


教程介绍
^^^^

欢迎来到“使用LLVM编写语言”教程。本教程过实现一个简单的语言，让你感受其中的乐趣和轻松。我们将从一个简单的框架开始，让你可以扩展到其它语言。你也可以用教程中的代码测试llvm中的各种特性。

我们的目的是逐步构建我们的编程语言，解释其中每一步的构建过程。我们将大量涉及到语言设计和LLVM使用技巧，每一部分代码我们都会展示出来，并详细解释它，而不是给你成吨的代码细节让你自行挖掘其中的奥秘。

在开始教程前，我要强调的是这个教程是关于编译器和LLVM的具体实现，*不包含*教授现代化和理论化的软件工程原理。这意味着在实践中我们使用了一些并不科学但是简单的方法来介绍我们的做法。比如，内存泄露，大量使用全局变量，不好的设计模式如访客模式等……但是，它非常简单。如果你将来想对这些代码做改进和复用，修复这些缺陷应该不难。

我把这些章节简介放在这里，若你对相关内容已经有所了解，可以跳过他们。教程的结构是这样的：

* 第一章：介绍Kailedoscope语言和它的词法分析器——这部分展示了我们要做的基本功能。为了最大限度的可理解性和可玩性，我们选择将用C++实现所有的代码，而不是用现成的词法分析器和语法分析器。当然，LLVM可以很好地工作在这些工具上，选择你喜欢的就好。
* 第二章：实现解析器和AST——在词法分析器已经完成基础上，我们可以讨论解析器和AST（抽象语法树）的构造。这个教程解释了递归向下法解析流程和运算符优先级分析。第一章和第二章没有关于LLVM具体的使用，甚至和LLVM没有一点联系。
* 第三章：LLVM IR（中间码）生成。AST已经就绪后，我要向你展示生成LLVM IR是多么简单。
* 第四章：添加JIT（即时解释器）和优化支持——有相当多的人对把LLVM作为JIT感兴趣。我们将深入JIT的使用并向你展示如何用三行代码引入JIT的支持。LLVM能在很多方面大展身手，但是只有JIT才能最“性感”地展示其强大之处。
* 第五章：扩展语言之控制流——在语言完善过程中，我将向你展示如何添加控制流操作（if/then/else和for循环）。我们将抓住这个机会来讨论SSA构造和控制流。
* 第六章：扩展语言之用户定义运算符——相当简单但是有趣的一章，关于如何扩展语言实现用户自定义自己的一元和二元运算符。这是建立一个语言库过程中显著的里程碑。
* 第七章：扩展语言之可变地变量——这章节讨论如何用赋值语句添加用户自定义地本地变量。有趣的是，构造SSA在LLVM是相当简单的，但是LLVM并不要求你的前端来构造SSA结构。
* 第八章：总结和花絮——我们会在这一章唠叨一系列扩展语言的潜在的其它方法，包括一个“特别主题”比如添加垃圾回收支持、异常、调试，支持“意面栈（spaghetti stack）”，和一堆其它的技巧和窍门。

整个教程中，我们写的代码不包括注释空行将不到700行。用如此少量的代码，我们将建立一个合理的编译器来编译一个不简单的语言，包括手写的词法分析器，解析器，抽象语法树和JIT的支持。也许其它的教程有一个很有趣的“hello world”实例，我认为这个教程的生命力在于感受LLVM的强大，和明白如果你对语言和编译器设计有兴趣的话，研究LLVM是十分必要的。

还有一点提醒：我们希望你亲手扩展并把玩语言。疯狂地去玩转代码，编译器不是一个可怕地生物——其实与语言打交道也可以很有趣。


基本语言
^^^^

我们设计一个玩具语言，我们称之为“Kailedoscope”（由“美丽，形式和观点”派生出来的）。Kailedoscope是一种程序语言，它允许您定义功能，使用条件，数学语句等，在教程中，我们将扩展Kailedoscope支持的if / then/ else结构，for循环中，用户定义的运算符，支持JIT的一个简单的命令行窗口等。

我们希望让事情变得简单，所以Kailedoscope中唯一的数据类型是一个64位浮点型（float）（在C语言中又称“Double”）。因此，所有的值都隐含dboule的精度，在语言中也就不需要类型声明。这使得语言有一个非常简单的语法。比如，下面这个简单的例子是计算斐波那契数的：

.. code-block:: python

    # 计算第x个数字
	def fib(x)
	  if x < 3 then
	    1
	  else
	    fib(x-1)+fib(x-2)

	# 计算第40个数字
	fib(40)

同时我们也允许万花筒调用标准库函数（LLVM的JIT做到这个简直轻而易举）。这意味着您可以使用“extern”关键字，你使用之前定义的函数（这在相互递归的函数中很有用）。例如：

.. code-block:: c

    extern sin(arg);
	extern cos(arg);
	extern atan2(arg1 arg2);

	atan2(sin(.4), cos(42))

在第6章还有一个更有趣的例子，我们编写一个显示Mandelbrot集放大程序。 

那现在，让我们深入Kailedoscope语言的实现吧！

词法分析
^^^^

实现语言的第一步，就是让语言具备从文本文件中读取程序代码，理解自己应该去做什么。传统方法的实现方法是“词法分析”（又名，“扫描器”）将输入分解为一串tokens，词法分析器返回每一个token包括token代码和一些可能的元数据（如数值）。首先，我们定义以下token：


.. code-block:: C++

    // The lexer returns tokens [0-255] if it is an unknown character, otherwise one
	// of these for known things.
	enum Token {
	  tok_eof = -1,

	  // commands
	  tok_def = -2, tok_extern = -3,

	  // primary
	  tok_identifier = -4, tok_number = -5,
	};

	static std::string IdentifierStr;  // Filled in if tok_identifier
	static double NumVal;              // Filled in if tok_number

我们的词法分析器返回的每一个token可能是一个枚举值或者是一个字符比如“+”，实际上这返回的是字符的ASCII码，如果当前的token是一个标识符（identifier），全局变量`IdentifierStr`就会保存标识符的名字，如果当前标识符是数字字符（如1），`NumVal`将负责保存它的值。注意，我们保存变量时用了全局变量，这并不是一个最好的选择，只是为了让它写起来足够简单。


词法分析器实际实现的功能是`gettok`函数。该函数被调用来从标准输入返回下一个token。它的定义开始为：

.. code-block:: C++

    /// gettok - Return the next token from standard input.
	static int gettok() {
	  static int LastChar = ' ';

	  // Skip any whitespace.
	  while (isspace(LastChar))
	    LastChar = getchar();

`gettok`通过调用C语言的`getchar()`来读取标准输入流的字符，它读取字符后会将其保存在`LastChar`并剔除出输入流。首先要做的是忽略token之间的空白符。这个可以用下面的循环实现。

接着`gettok`要做的是识别标识符和保留字符如“def”。Kaleidoscope通过以下的简单循环实现：

.. code-block:: C++

    if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
	  IdentifierStr = LastChar;
	  while (isalnum((LastChar = getchar())))
	    IdentifierStr += LastChar;

	  if (IdentifierStr == "def") return tok_def;
	  if (IdentifierStr == "extern") return tok_extern;
	  return tok_identifier;
	}

当分析到标识符时，Kaleidoscope会设置全局变量`IdentifierStr`，在这个匹配中，还会检测是否出现了关键字。同样，匹配数字时，我们也这样：

.. code-block:: C++

    if (isdigit(LastChar) || LastChar == '.') {   // Number: [0-9.]+
	  std::string NumStr;
	  do {
	    NumStr += LastChar;
	    LastChar = getchar();
	  } while (isdigit(LastChar) || LastChar == '.');

	  NumVal = strtod(NumStr.c_str(), 0);
	  return tok_number;
	}

这是处理输入的一段漂亮、简洁的代码。当读到数字时，我们使用了C中的`strtod`函数转化为数字，并储存在`NumVal`中。注意，这里可能会出现一些错误，当错误地读入“1.13.45.67”，这将被当作“1.23”处理。你可以自行去修复这个bug。下一步，我们处理注释：

.. code-block:: C++

    if (LastChar == '#') {
	  // Comment until end of line.
	  do LastChar = getchar();
	  while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

	  if (LastChar != EOF)
	    return gettok();
	}

我们跳过注释直接来到行的末尾，然后返回下一个token。最后，如果输入不匹配上述情况，则可能时一个操作符“+”或者文件结束。这些都由下面代码处理：

.. code-block:: C++

      // Check for end of file.  Don't eat the EOF.
	  if (LastChar == EOF)
	    return tok_eof;

	  // Otherwise, just return the character as its ascii value.
	  int ThisChar = LastChar;
	  LastChar = getchar();
	  return ThisChar;
	}

至此，我们有了一个完整的词法分析器（完整的词法分析代码在下一章列出）。下一步，我们将构建一个抽象语法树和一个简单解析器。我们还包括一个驱动程序，以便您将词法分析器和语法分析器结合在一起。
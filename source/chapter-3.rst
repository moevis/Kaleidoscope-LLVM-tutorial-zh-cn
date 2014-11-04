Kaleidoscope: LLVM中间代码（IR）生成
----------------------------

:原文: `Kaleidoscope: Code generation to LLVM IR
`__

__http://llvm.org/docs/tutorial/LangImpl3.html: http://llvm.org/docs/tutorial/LangImpl3.html


介绍
""

欢迎来到“用LLVM实现语言”教程的第三章。这一章将介绍你如何从AST（第二章我们已经建好的）转换到LLVM中间码。你将接触到一点LLVM的工作原理，你会发现使用LLVM相当容易——生成LLVM中间码要比建立词法分析器和解析器容易得多。

*注意*：这部分的代码要求LLVM版本大于等于2.2。2.1以及更早的版本不兼容这部分代码。还有，你引入的库文件应当和你使用的LLVM版本相同：如果你在使用官方的release版，你可以在`llvm.org releases page`__ 上找到相应的库文件。

__llvm.org releases page: http://llvm.org/releases/

中间码生成配置
"""""""

为了生成中间码，我们需要代码上一些简单的配置。首先我们要在每一个AST类中添加虚函数（codegen）：

.. code-block:: C++

	/// ExprAST - Base class for all expression nodes.
	class ExprAST {
	public:
	  virtual ~ExprAST() {}
	  virtual Value *Codegen() = 0;
	};

	/// NumberExprAST - Expression class for numeric literals like "1.0".
	class NumberExprAST : public ExprAST {
	  double Val;
	public:
	  NumberExprAST(double val) : Val(val) {}
	  virtual Value *Codegen();
	};

\ ``Codegen()``运行后会生成中间码以及其它运行时需要的信息，这些信息以LLVM value对象形式返回。"Value"类用来表示LLVM中的“静态单赋值寄存器（Static Single Assignment register）"或者“SSA value”。SSA值的特点是，它在经过相关指令计算出，并不能被改变（除非程序从头来过）。换句话说，SSA是个常量。你想了解SSA更多的话，请阅读 `静态单赋值`__ --一旦你了解它，你会发现这相当简单。

.. __静态单赋值: http://

值得说的是，除了添加\ ``Codegen()``\ 虚函数外，使用访客模式也是一种很好的方法。重申一遍，这个教程并不是停留在使用优雅的方法实践软件工程上；对于我们的目的来说，使用虚函数是最简单的方法。

第二件事情要注意的是，我们在解析器中使用的“Error”方法将用来报告错误（比如，使用了一个未声明的变量。=）

.. code-block:: C++

	Value *ErrorV(const char *Str) { Error(Str); return 0; }

	static Module *TheModule;
	static IRBuilder<> Builder(getGlobalContext());
	static std::map<std::string, Value*> NamedValues;

静态变量会在整个代码生成阶段中被中使用，\ ``TheModule``\ 便是一个用来储存这些函数以及全局变量的LLVM结构体。在某种程度上，这就是LLVM中间码所构造的顶层结构。

\ ``Builder``\ 是一个辅助对象，用来为生成LLVM指令提供方便。它是\ ``IRBuilder``\ 类的实例，用来标记当前位置以插入新的指令。

\ ``NameValues``\ 键值表保存了当前的代码范围内定义的值，和记录并表示这些值的LLVM对象（换句话说，这就是当前代码的符号表）。在这种形式下，唯一可以参考的是函数参数（In this form of Kaleidoscope, the only things that can be referenced are function parameters. ）。因此，当生成函数体代码时，函数参数会被记录到这个表里去。

当这些搭建完毕后，我们离为每句表达式生成代码更近了一步。我们还需要做的是配置好\ ``Builder``\ ，但现在，假设我们已经将\ ``Builder``\ 已经配置完毕，开始用它来生成代码。

表达式代码生成
"""""""

从表达式生成LLVM代码相当直接：4种表达式节点总共不到45行带注释的代码。我们依次将这四种节点列出来：

.. code-block:: C++

	Value *NumberExprAST::Codegen() {
	  return ConstantFP::get(getGlobalContext(), APFloat(Val));
	}

在LLVM中间码里，数字常量用\ ``ConstantFP``\ 类来表示，它将数字储存在内部的\ ``APFloat``\ 中（\ ``APFloat``\ 可以存储任意精度的浮点数）。这段代码主要用来创建和返回一个\ ``ConstantFP``\ 。注意在LLVM中间码中，所有的常量都是唯一并共享的。
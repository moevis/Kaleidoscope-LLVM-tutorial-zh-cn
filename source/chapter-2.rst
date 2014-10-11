Kaleidoscope: 实现解析器和抽象语法树
-------------------------

:原文: `Kaleidoscope: Implementing a Parser and AST`__
__ http://llvm.org/docs/tutorial/LangImpl2.html

介绍
^^

欢迎来到第二章教程，这一章介绍如何使用词法分析器来建立一个Kaleidoscope语言解析器。一旦我们完成了解析器，我们可以定义抽象语法树（AST）。

我们的解析器将使用递归下降法和运算符优先级分析来解析Kaleidoscope语言（后者用来解析运算符表达式，前者将负责其他部分所有的解析）。在我们开始解析之前，我们先来讨论一下我们这一步的目标：生成抽象语法树。

抽象语法树(AST)
""""""""""

一段程序的抽象语法树很容易在接下来的阶段编译器（比如：代码生成阶段）翻译成机器码。我们通常喜欢用一种对象来构建语言，毫无疑问，抽象语法树是最贴近我们要求的模型。在Kaleidoscope中，我们有表达式，原型，函数对象，我们先从生成表达式的AST先：


.. code-block:: C++

    /// ExprAST - Base class for all expression nodes.
	class ExprAST {
	public:
	  virtual ~ExprAST() {}
	};

	/// NumberExprAST - Expression class for numeric literals like "1.0".
	class NumberExprAST : public ExprAST {
	  double Val;
	public:
	  NumberExprAST(double val) : Val(val) {}
	};

上面的代码展示了语法树节点的基类和由此继承出的数字节点子类``NumberExprAST``。需要注意的是，``NumberExprAST``会将解析出的文本转化为数字并存储起来，以便今后的编译器处理中可以获得这个节点存储的值。

现在我们只生成了AST，还没有对他们的访问方法。不过我们可以轻易地添加成员函数来实现这些访问方法，以下是Kaleidoscope中其他的AST节点的定义：

.. code-block:: C++

	/// VariableExprAST - Expression class for referencing a variable, like "a".
	class VariableExprAST : public ExprAST {
	  std::string Name;
	public:
	  VariableExprAST(const std::string &name) : Name(name) {}
	};

	/// BinaryExprAST - Expression class for a binary operator.
	class BinaryExprAST : public ExprAST {
	  char Op;
	  ExprAST *LHS, *RHS;
	public:
	  BinaryExprAST(char op, ExprAST *lhs, ExprAST *rhs)
	    : Op(op), LHS(lhs), RHS(rhs) {}
	};

	/// CallExprAST - Expression class for function calls.
	class CallExprAST : public ExprAST {
	  std::string Callee;
	  std::vector<ExprAST*> Args;
	public:
	  CallExprAST(const std::string &callee, std::vector<ExprAST*> &args)
	    : Callee(callee), Args(args) {}
	};

这些代码相当直观：变量节点记录变量名，二元运算符节点记录运算符（如'+'），函数调用节点记录调用的函数名和函数参数列表。这样的结构相当不错，我们的AST结构记录的信息是与语法是无关的，没有涉及到运算符的优先级和词法结构。

在我们基本的语言里，以上定义了我们表达式节点所有的AST节点类型。因为我们的语言中没有定义条件控制语法（如if/else），所以这并没有做到图灵完备；我们将在后面慢慢修复它。我们现在需要做两件事情，一件是实现在Kaleidoscope中调用函数，另一件是记录函数体的本身。

.. code-block:: C++

	/// PrototypeAST - This class represents the "prototype" for a function,
	/// which captures its name, and its argument names (thus implicitly the number
	/// of arguments the function takes).
	class PrototypeAST {
	  std::string Name;
	  std::vector<std::string> Args;
	public:
	  PrototypeAST(const std::string &name, const std::vector<std::string> &args)
	    : Name(name), Args(args) {}
	};

	/// FunctionAST - This class represents a function definition itself.
	class FunctionAST {
	  PrototypeAST *Proto;
	  ExprAST *Body;
	public:
	  FunctionAST(PrototypeAST *proto, ExprAST *body)
	    : Proto(proto), Body(body) {}
	};

在Kaleidoscope中，函数调用需要带有传入的参数。因为目前所有的变量都当做浮点类型，我们并不需要记录参数类型。在现代计算机语言语言中，``ExprAST``类应当有一个记录类型的变量。

建立了这些类型后，我们可以开始着手解析这些表达式和函数体了。
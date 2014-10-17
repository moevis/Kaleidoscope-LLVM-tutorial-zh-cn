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

解析基础
""""

现在我们要开始建立抽象语法树了。我们先试着解析``x+y``这样的表达式，这可以由这样的调用产生。

.. code-block:: C++

    ExprAST *X = new VariableExprAST("x");
	ExprAST *Y = new VariableExprAST("y");
	ExprAST *Result = new BinaryExprAST('+', X, Y);

为了达到上面的目的，我们现在还需要以下的辅助函数

.. code-block:: C++
	/// CurTok/getNextToken - Provide a simple token buffer.  CurTok is the current
	/// token the parser is looking at.  getNextToken reads another token from the
	/// lexer and updates CurTok with its results.
	static int CurTok;
	static int getNextToken() {
	  return CurTok = gettok();
	}

以上实现了一个简单的token缓存，这使得我们可以向前读取下一个token，每一个解析器的函数将默认``CurTok``是当前正在被解析的token。

.. code-block:: C++

    /// Error* - These are little helper functions for error handling.
	ExprAST *Error(const char *Str) { fprintf(stderr, "Error: %s\n", Str);return 0;}
	PrototypeAST *ErrorP(const char *Str) { Error(Str); return 0; }
	FunctionAST *ErrorF(const char *Str) { Error(Str); return 0; }

错误处理函数将用来处理简单的错误。我们的解析器的错误恢复并不是最好的，也不是特别的方便，但对于我们的教程来说已经够了。这些程序可以让我们更容易地处理不同的返回类型程序的错误，在一般情况下一般返回``NULL``。

具备好了这些基础的辅助函数，我们可以实现我们的第一个语法：解析数字。

基本表达式解析
"""""""

我们先从数字开始，因为它们是最容易处理的。首先，我们先定义一个处理数字的函数：

.. code-block:: C++
	/// numberexpr ::= number
	static ExprAST *ParseNumberExpr() {
	  ExprAST *Result = new NumberExprAST(NumVal);
	  getNextToken(); // consume the number
	  return Result;
	}

这一部分代码很简单：若当前的token是一个指向数字的``tok_number``，则调用``ParseNumberExpr``，它会读取当前数值，创建``NumberExprAST``节点，然后读取下一token，以便接下来的解析，最后，返回结果。

这其中还有一些有趣的东西。最重要的一点是，这些解析节点的代码会将所有与之相关的token都读取掉，同时在返回结果前会再次调用``getNextToken``来清除掉当前的token，得到下一个token（通常这个token不属于当前节点）。这在递归下降解析器中是一个普遍的做法。下面给出一个例子可以更好地理解，这个例子是关于解析一对括号的：

.. code-block:: C++

	/// parenexpr ::= '(' expression ')'
	static ExprAST *ParseParenExpr() {
	  getNextToken();  // eat (.
	  ExprAST *V = ParseExpression();
	  if (!V) return 0;

	  if (CurTok != ')')
	    return Error("expected ')'");
	  getNextToken();  // eat ).
	  return V;
	}

这个函数演示了几个关于解析器的有趣的方面：

* 异常检测：当被调用时，这个函数会默认当前的token是``(``，但是当结束表达式解析后，有可能末尾的token就不是``)``。比如，如果用户错将``(4)``打成了``(4 *``，解析器就会检测到这个错误，为了提醒有错误发生，我们的解析器将返回NULL。
* 递归式解析：这段函数中调用了``ParseExpression``（我们将很快看到``ParseExpression``同样会调用``ParseParenExpr）。这种方式相当强大，因为它允许我们处理嵌套的语法，同时也保持了每一个过程都是相当简洁。注意，括号并不会成为抽象语法树的组成部分，它的作用是将表达式组合起来引导引导解析器正确地处理它们。当建立好了抽象语法树后，它们便可以被抛弃了。

下一步我们来写变量的解析器：

.. code-block:: C++

	/// identifierexpr
	///   ::= identifier
	///   ::= identifier '(' expression* ')'
	static ExprAST *ParseIdentifierExpr() {
	  std::string IdName = IdentifierStr;

	  getNextToken();  // eat identifier.

	  if (CurTok != '(') // Simple variable ref.
	    return new VariableExprAST(IdName);

	  // Call.
	  getNextToken();  // eat (
	  std::vector<ExprAST*> Args;
	  if (CurTok != ')') {
	    while (1) {
	      ExprAST *Arg = ParseExpression();
	      if (!Arg) return 0;
	      Args.push_back(Arg);

	      if (CurTok == ')') break;

	      if (CurTok != ',')
	        return Error("Expected ')' or ',' in argument list");
	      getNextToken();
	    }
	  }

	  // Eat the ')'.
	  getNextToken();

	  return new CallExprAST(IdName, Args);
	}

这段解析代码和其它的很类似。若当前token为``tok_identifier``时，该函数被调用。同样具有递归的解析思想，和同样的错误处理方法。有趣的一点是，这里还用到了一个前置判断（look-ahead）来决定当前的identifier是一个函数调用，还是一个变量。判断的方法是读取下一个token，若下一个token**不是**``(``，则这是函数调用这时候返回``VariableExprAST``，否则是使用变量，返回``CallExprAST。

现在我们所有的简单表达式解析器代码已经就位，我们可以定义一个辅助函数来包装并调用它们。我们把目前我们完成的简单的表达式取名为**基本表达式**（primary expressions），到后面你就会更加理解这个名字了。以下就是基本表达式解析器：

.. code-block:: C++

    /// primary
	///   ::= identifierexpr
	///   ::= numberexpr
	///   ::= parenexpr
	static ExprAST *ParsePrimary() {
	  switch (CurTok) {
	  default: return Error("unknown token when expecting an expression");
	  case tok_identifier: return ParseIdentifierExpr();
	  case tok_number:     return ParseNumberExpr();
	  case '(':            return ParseParenExpr();
	  }
	}

通过基本表达式解析器，我们可以明白为什么我们要使用``CurTok``了，这里用了前置判断来选择并调用解析器。

现在基本的表达式解析器已经完成了，我们下一步开始处理二元表达式，这会有一点复杂。

二元表达式解析
"""""""

二元表达式的解析过程相对复杂，因为二元表达式会有二义性。比如，当出现``x+y*z``，解析器可以选择``(x+y)*z``或者``x+(y*z)``两种解析顺序。在数学定义中，我们期望后一种解析方式，因为``*``比``+``有更高的优先级。

面对优先级问题，我们可用的处理方法有很多，不过论最优雅最高效的还是要数远算符优先级分析法（Operator-Precedence Parsing）。这种解析方法借助运算符优先级来选择解析顺序，所以，起初需要一个一个优先级表格：

.. code-block:: C++

	/// BinopPrecedence - This holds the precedence for each binary operator that is
	/// defined.
	static std::map<char, int> BinopPrecedence;

	/// GetTokPrecedence - Get the precedence of the pending binary operator token.
	static int GetTokPrecedence() {
	  if (!isascii(CurTok))
	    return -1;

	  // Make sure it's a declared binop.
	  int TokPrec = BinopPrecedence[CurTok];
	  if (TokPrec <= 0) return -1;
	  return TokPrec;
	}

	int main() {
	  // Install standard binary operators.
	  // 1 is lowest precedence.
	  BinopPrecedence['<'] = 10;
	  BinopPrecedence['+'] = 20;
	  BinopPrecedence['-'] = 20;
	  BinopPrecedence['*'] = 40;  // highest.
	  ...
	}

现在我们可以开始着手解析二元表达式了，最核心的思想方法是将可能出现二义性的表达式分解成多个部分。想一下，比如表达式``a+b+(c+d)*e*f+g``。解析器将这个字符串看做一串由二元运算符分隔的基本表达式。因此，它将先解析第一个基本表达式``a``，接着将解析到成对出现的[+, b] [+, (c+d)] [*, e] [*, f]和 [+, g]。因为括号也是基础表达式，不用担心解析器会对``(c+d)``出现困惑。

开始解析第一步，表达式是由第一个基础表达式和之后的一连串[运算符, 基础表达式]组成。

.. code-block:: C++
 
	/// expression
	///   ::= primary binoprhs
	///
	static ExprAST *ParseExpression() {
	  ExprAST *LHS = ParsePrimary();
	  if (!LHS) return 0;

	  return ParseBinOpRHS(0, LHS);
	}

``ParseBinOpRHS``是为我们解析*运算符-表达式*对的函数。它记录优先级和已解析部分的指针。

优先级数值被传入``ParseBinOpRHS``，凡是比这个优先级值低的运算符都不能被使用。比如如果当前的解析的是[+, x]，且目前传入的优先级值为40，那么函数就不会消耗任何token（因为"+"优先级值仅20）。因此我们函数应该这样写：

.. code-block:: C++

    /// binoprhs
	///   ::= ('+' primary)*
	static ExprAST *ParseBinOpRHS(int ExprPrec, ExprAST *LHS) {
	  // If this is a binop, find its precedence.
	  while (1) {
	    int TokPrec = GetTokPrecedence();

	    // If this is a binop that binds at least as tightly as the current binop,
	    // consume it, otherwise we are done.
	    if (TokPrec < ExprPrec)
	      return LHS;


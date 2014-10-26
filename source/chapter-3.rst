Kaleidoscope: LLVM中间代码（IR）生成
----------------------------

介绍
""

欢迎来到“用LLVM实现语言”教程的第三章。这一章将介绍你如何从AST（第二章我们已经建好的）转换到LLVM中间码。你将接触到一点LLVM的工作原理，你会发现使用LLVM相当容易——生成LLVM中间码要比建立词法分析器和解析器容易得多。

*注意*：这部分的代码要求LLVM版本大于等于2.2。2.1以及更早的版本不兼容这部分代码。还有，你引入的库文件应当和你使用的LLVM版本相同：如果你在使用官方的release版，你可以在`llvm.org releases page`_ 上找到相应的库文件。

.. _llvm.org releases page: http://llvm.org/releases/
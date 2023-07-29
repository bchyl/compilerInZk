测试集覆盖
必要性
一方面语言功能完备性，从最小可测单元尽量覆盖测试所有可列举的语言要素；
另一方面提前排雷复杂的业务场景，如erc20、vote、swap等复杂合约，计算及存储功能较为复杂，无法确定是单点问题还是多点组合问题。
测试场景分类
从高级语言或AST对应的要素看，即类型及相关的操作类型来确定case类型。ir、asm均可依此确定pattern。
Declare
Declare & init
Assign  var
  Literal
Operator access
             length
             slice
             push
             pop
  类型种类：
  - 标量u32，
  - 向量address，
  - 聚合如string、array、map，
  - 及两种或以上类型的组合嵌套。
  表达式种类：出现在程序不同要素中，如：
  - 声明、
  - 左值赋值、
  - 右值运算、
  - 函数调用的实参、
  - 形参、
  - 返回值。
  操作数需同时考虑立即数与寄存器，当为寄存器时，来源：
  - 函数参数
  - 局部变量声明
  - 新的临时计算值（SSA指令而来）
  （下表不一而足，逐一补充）
expr
u32
string
address
array
struct
N-d array
Dynamic array(u32)
N-d Dynamic array(u32)
Dynamic array(string)
N-d Dynamic array(string)
...
聚合与向量、标量类型组合，嵌套等
decl
声明，无初始化
u32 a;










decl&init with imm
声明，以立即数初始化
u32 a = 1;










Decl & init with var
声明，变量初始化(参数、指令临时结果)
u32 a = 1;
u32 b = a;










assign with imm
赋值，右值立即数
u32 a;
a = 1;











assign with var
赋值，右值变量
u32 a = 1;
u32 b = 2;
b = a;










Assign with expr
赋值，右值指令临时结果
u32 a = 1;
u32 b = 2;
b = a + b;










Operand with expr
运算符的操作数(运算符种类穷举)
u32 a = 1;
u32 b = 2;
u32 c = 3;
c = a + b; // anchor a,b










Actual param
实参(立即数或变量)
u32 a = 1;
foo(a);










Formal param
形式参数，变量
foo(u32 a) {}










return
返回值(立即数或变量)
u32 foo {
return 1;
}










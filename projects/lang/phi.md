
# phi节点概述
LLVM IR设计基于栈的SSA形式，但无法解决分支、循环控制流，故IR中引入phi伪指令机制区分。

但通常后端目标架构并无phi指令，因此后端需要对SSA形式进行析构，并消除phi节点转换为其他指令。

示例如下:

假设如下的条件分支语句：
```
void foo(u32 a) {
u32 b = 6, c = 8;
if(a = 1) {
    b = 2;
else {
    b = 3;
}
c = b;
}
```
其SSA形式的伪代码逻辑为：
```
void foo(u32 a) {
u32 b = 6, c = 8;
if(a = 1) {
    u32 b1 = 2;
else {
    u32 b2 = 3;
}
c = phi(b1,b2);
}
```
后端消除phi节点(消边)后，伪代码逻辑为：
```
void foo(u32 a) {
u32 b = 6, c = 8;
if(a = 1) {
    u32 b1 = 2;
    c = b1;
else {
    u32 b2 = 3;
    c = b2;
}
c = phi(b1,b2);
}
```
# 后端消除phi
## IR场景
为分支、循环等结构达到生成IR符合SSA格式，生成的phi IR中，至少2个以上val-label对，基本语法如下：
```<result> = phi <ty> [<val0>, <label0>], [<val1>, <label1>] …```
含义为result值为：执行label0 BB则为val0，执行label1 BB则为val1
其中pattern场景：
- val可为imm或vreg
- 一般分支控制流场景为label0、label1为该phi指令前的不同的两个BB，phi指令当前为的label2，即三个均为不同的BB，label0 != label1 != label2
- 循环结构中，phi节点BB label2可能与phi中第二个BB label1为同一个BB，即label0 != label1 = label2
- 同一个BB中包含多条phi
- 同一个函数中包含多条phi
## 后端处理phi伪指令
由于olaVM ISA中并不不包含该指令，因此需要依据控制流关系将其转换成mov指令到每一个前导BB中，并最终消除该伪指令。
### 指令选择
该阶段ir经解析后，分成三部分：
- opcode继续沿用phi伪指令
- operand[0]为结果
- operand val为i64类型的imm或vreg
- Operand bb为block类型
如场景：
```
label0:
    br label label2
label1:
    br label label2  
label2:
    %indvar = phi i64 [ 0, %label0 ], [ %nextindvar, %label1 ]
```
经指令选择后label2在phi指令选择为：
```phi [i64_vreg %indvar，i64_imm 0, vreg %nextindvar, block %label0, block %label1 ]```
### phi消除
- 函数为单位，遍历所有指令，找到phi指令并加入待处理列表中
- 按序处理列表中phi，依据上述选择后的指令跳跃找到block，依次在所在BB中最后一条指令前插入一条mov指令，源为block对应value，目标为operand[0]
- 删除phi伪指令
如上述场景此时机器码转换为：
```
label0:
    mov %indvar 0 ;movri
    jmp label2
 label1:
    mov %indvar %nextindvar ;movrr
    jmp label2
 label2:
     phi [i64_vreg %indvar，i64_imm 0, vreg %nextindvar, block %label0, block %label1 ]
```
注：该处理为一般场景，针对上述ir后三种场景，需要微调。
如为循环，插入mov指令与phi节点为同一个BB，则插入mov指令位置为当前phi节点位置的上一条位置。
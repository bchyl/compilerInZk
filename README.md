# compilerInZk
Compiler Architecture in ZKVM

# Architecture flow

```mermaid
graph TB

advance-program-language -- Lexer ---> Tokens 
Tokens -- Parser ---> AST
AST -- Sema ---> Sema-AST
Sema-AST -- HirGen --->  HIR
HIR -- HirOpt --->  opt-HIR
opt-HIR -- LirGen --->  LIR
LIR -- Opt --->  Opt-LiR
Opt-LiR -- CodeGen ---> TargetAsm
TargetAsm -- CodeGenOpt ---> TargetOptAsm
TargetOptAsm -- Assembler ---> TargetByteCode
TargetByteCode -- Executor ---> Trace
```

## advance-program-language

## compiler front-end

### lexer for tokens

### parser for AST

### sema for rich-AST

## compiler middle-end

### hir and generate

### lir and generate

### ir opt

## compiler back-end

### ISA
abstract model of a computer

- data type
- instructions set
- register
- addressing(fundamental)
- storage model
- break and exception(fundamental) 
- input/output model(fundamental)

### backend code generate

- frame lowering
- instruction selection
- slot eliminate
- pro/epi-logue
- inst schdule
- register allocation
- asm print
- ...

## assembler 
- fixup 
- relocation
- encoding

## executor
- link, load and lib
- generate trace
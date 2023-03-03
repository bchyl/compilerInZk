# High-level language library solution implementation for lang&compiler

Functional Requirements

## ola high-level language implementation libs

- 1. determine the number of library functions corresponding to each type of u32/u64/u128 

- 2. use the conventional field type operation + builtin + rust / python syntax to represent the prompts of a total of 3 different types of language features, the implementation of each library function  

- 3. library function encapsulation and call: encapsulation is syntactic sugar form operator form or direct intrinsic function definition, call the library function using import/use 

## Compiler front-end

- 1. parsing, prompts codeString pass-through

- 2. In addition to the regular ir, you need to attach the context information of the prompts in AST/Sema, including location, variable use symbol table, virtual memory alloca mapping relationship

- 3. When the library function function call occurs, ir need to expand the import/use library function definition

- 4. irGen contains the json file interface output of the above information of prompts

## Compiler back-end

- 1. parse ir json, process insts and prompts contents respectively

- 2. prompts codeString pass-through, contextual information location, symbol table + virtual memory mapping specific slot number

- 3. CodeGen interface output of json file containing the above information of prompts
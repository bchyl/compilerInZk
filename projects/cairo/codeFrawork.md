# cairo1.0 code

```
-------------------------------frontEnd------------------------------------------------   cairo high-level program
|    cairo-lang-syntax   <-  cairo-lang-syntax-codegen                                |
|          ↓                                                                          |
|    cairo-lang-parser                                                                |
|           ↓                                                                         |
|    cairo-lang-diagnostics                                                           |  <-  cairo-lang-compiler
|           ↓                                                                         |
|    cairo-lang-defs    cairo-lang-proc-macros                                        |
|           ↓                                                                         |
|    cairo-lang-semantic                                                              |
|           ↓                                                                         |
|    cairo-lang-lowering                                                              |
|           ↓                                                                         |
-------------------------------midEnd--------------------------------------------------   sierra IR
|    cairo-lang-sierra-generator  <=  cairo-lang-sierra   <-   LALRPOP-parser         |
|           ↓                                                                         |
|        cairo-lang-sierra-gas   cairo-lang-sierra-ap-change  cairo-lang-eq-solver    |
|           ↓                                                                         |
-------------------------------backEnd-------------------------------------------------   cairo asm code
|    cairo-lang-sierra-to-casm   <-    cairo-lang-casm                                |  <-  compile SierraProgram to CairoProgram
|           ↓                                                                         |
|    cairo-lang-runner                                                                |  <-   SierraCasmRunner
---------------------------------------------------------------------------------------   cairo byte code

```
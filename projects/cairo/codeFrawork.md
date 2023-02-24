# cairo1.0 code

```
-------------------------------frontEnd------------------------------------------------   cairo high-level program
|    cairo-lang-syntax   <-  cairo-lang-syntax-codegen                                |
|           ↓                                                                         |
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

# code transform flow

take libfuncCall for example:

```
   highLevelprogramBinOp a_u32 - b_u32
-> ir {funcStmt ... libFuncU32SubStmt}
-> asm {funcInsts ... libFuncU32SubTpl}
-> bytecodeTrace {funcBytesTrace ... libfuncU64AddBytesAssertTrace}

```

libFuncU32SubTpl logic is:

```
fn build_small_uint_overflowing_sub(
    builder: CompiledInvocationBuilder<'_>,
    limit: BigInt,
) -> Result<CompiledInvocation, InvocationError> {
    let failure_handle_statement_id = get_non_fallthrough_statement_id(&builder);
    let [range_check, a, b] = builder.try_get_single_cells()?;
    let mut casm_builder = CasmBuilder::default();
    add_input_variables! {casm_builder,
        buffer(0) range_check;
        deref a;
        deref b;
    };
    casm_build_extend! {casm_builder,
            let orig_range_check = range_check;
            tempvar no_overflow;
            tempvar a_minus_b = a - b;
            const u128_limit = (BigInt::from(u128::MAX) + 1) as BigInt;
            const limit = limit;
            hint TestLessThan {lhs: a_minus_b, rhs: limit} into {dst: no_overflow};
            jump NoOverflow if no_overflow != 0;
            // Underflow:
            // Here we know that 0 - (limit - 1) <= a - b < 0.
            tempvar fixed_a_minus_b = a_minus_b + u128_limit;
            assert fixed_a_minus_b = *(range_check++);
            tempvar wrapping_a_minus_b = a_minus_b + limit;
            jump Target;
        NoOverflow:
            assert a_minus_b = *(range_check++);
    };
    Ok(builder.build_from_casm_builder(
        casm_builder,
        [
            ("Fallthrough", &[&[range_check], &[a_minus_b]], None),
            ("Target", &[&[range_check], &[wrapping_a_minus_b]], Some(failure_handle_statement_id)),
        ],
        CostValidationInfo {
            range_check_info: Some((orig_range_check, range_check)),
            extra_costs: None,
        },
    ))
}
```
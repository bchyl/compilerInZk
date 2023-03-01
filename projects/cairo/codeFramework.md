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
# program case
## fib felt case

- high level program

```
fn fib(a: felt, b: felt, n: felt) -> felt {
    match n {
        0 => a,
        _ => fib(b, a + b, n - 1),
    }
}
```

- ir
```
type felt = felt;
type NonZero<felt> = NonZero<felt>;

libfunc revoke_ap_tracking = revoke_ap_tracking;
libfunc dup<felt> = dup<felt>;
libfunc felt_is_zero = felt_is_zero;
libfunc branch_align = branch_align;
libfunc drop<felt> = drop<felt>;
libfunc store_temp<felt> = store_temp<felt>;
libfunc jump = jump;
libfunc drop<NonZero<felt>> = drop<NonZero<felt>>;
libfunc felt_add = felt_add;
libfunc felt_const<1> = felt_const<1>;
libfunc felt_sub = felt_sub;
libfunc function_call<user@fib::fib::fib> = function_call<user@fib::fib::fib>;
libfunc rename<felt> = rename<felt>;

revoke_ap_tracking() -> ();
dup<felt>([2]) -> ([2], [4]);
felt_is_zero([4]) { fallthrough() 8([3]) };
branch_align() -> ();
drop<felt>([1]) -> ();
drop<felt>([2]) -> ();
store_temp<felt>([0]) -> ([5]);
jump() { 19() };
branch_align() -> ();
drop<NonZero<felt>>([3]) -> ();
dup<felt>([1]) -> ([1], [7]);
felt_add([0], [7]) -> ([6]);
felt_const<1>() -> ([8]);
felt_sub([2], [8]) -> ([9]);
store_temp<felt>([1]) -> ([11]);
store_temp<felt>([6]) -> ([12]);
store_temp<felt>([9]) -> ([13]);
function_call<user@fib::fib::fib>([11], [12], [13]) -> ([10]);
rename<felt>([10]) -> ([5]);
rename<felt>([5]) -> ([14]);
return([14]);

fib::fib::fib@0([0]: felt, [1]: felt, [2]: felt) -> (felt);
```

- asm
```
jmp rel 5 if [fp + -3] != 0;
[ap + 0] = [fp + -5], ap++;
jmp rel 8;
[ap + 0] = [fp + -4], ap++;
[ap + 0] = [fp + -5] + [fp + -4], ap++;
[fp + -3] = [ap + 0] + 1, ap++;
call rel -9;
ret;
```
## fib box felt case

- high level program
```
fn fib(a: Box::<felt>, b: Box::<felt>, n: Box::<felt>) -> Box::<felt> {
    let unboxed_n = unbox::<felt>(n);
    if unboxed_n == 0 {
        a
    } else {
        fib(
            b,
            into_box::<felt>(unbox::<felt>(a) + unbox::<felt>(b)),
            into_box::<felt>(unboxed_n - 1),
        )
    }
}
```

- ir
```
type felt = felt;
type Box<felt> = Box<felt>;
type NonZero<felt> = NonZero<felt>;

libfunc revoke_ap_tracking = revoke_ap_tracking;
libfunc unbox<felt> = unbox<felt>;
libfunc store_temp<felt> = store_temp<felt>;
libfunc dup<felt> = dup<felt>;
libfunc felt_is_zero = felt_is_zero;
libfunc branch_align = branch_align;
libfunc drop<Box<felt>> = drop<Box<felt>>;
libfunc drop<felt> = drop<felt>;
libfunc store_temp<Box<felt>> = store_temp<Box<felt>>;
libfunc jump = jump;
libfunc drop<NonZero<felt>> = drop<NonZero<felt>>;
libfunc dup<Box<felt>> = dup<Box<felt>>;
libfunc felt_add = felt_add;
libfunc into_box<felt> = into_box<felt>;
libfunc felt_const<1> = felt_const<1>;
libfunc felt_sub = felt_sub;
libfunc function_call<user@fib_box::fib_box::fib> = function_call<user@fib_box::fib_box::fib>;
libfunc rename<Box<felt>> = rename<Box<felt>>;

revoke_ap_tracking() -> ();
unbox<felt>([2]) -> ([3]);
store_temp<felt>([3]) -> ([3]);
dup<felt>([3]) -> ([3], [5]);
felt_is_zero([5]) { fallthrough() 10([4]) };
branch_align() -> ();
drop<Box<felt>>([1]) -> ();
drop<felt>([3]) -> ();
store_temp<Box<felt>>([0]) -> ([6]);
jump() { 29() };
branch_align() -> ();
drop<NonZero<felt>>([4]) -> ();
unbox<felt>([0]) -> ([7]);
dup<Box<felt>>([1]) -> ([1], [9]);
unbox<felt>([9]) -> ([8]);
store_temp<felt>([7]) -> ([7]);
store_temp<felt>([8]) -> ([8]);
felt_add([7], [8]) -> ([10]);
store_temp<felt>([10]) -> ([10]);
into_box<felt>([10]) -> ([11]);
felt_const<1>() -> ([12]);
felt_sub([3], [12]) -> ([13]);
store_temp<felt>([13]) -> ([13]);
into_box<felt>([13]) -> ([14]);
store_temp<Box<felt>>([1]) -> ([16]);
store_temp<Box<felt>>([11]) -> ([17]);
store_temp<Box<felt>>([14]) -> ([18]);
function_call<user@fib_box::fib_box::fib>([16], [17], [18]) -> ([15]);
rename<Box<felt>>([15]) -> ([6]);
rename<Box<felt>>([6]) -> ([19]);
return([19]);

fib_box::fib_box::fib@0([0]: Box<felt>, [1]: Box<felt>, [2]: Box<felt>) -> (Box<felt>);
```

- asm
```
[ap + 0] = [[fp + -3] + 0], ap++;
jmp rel 5 if [ap + -1] != 0;
[ap + 0] = [fp + -5], ap++;
jmp rel 14;
[ap + 0] = [[fp + -5] + 0], ap++;
[ap + 0] = [[fp + -4] + 0], ap++;
[ap + 0] = [ap + -2] + [ap + -1], ap++;
%{
if '__boxed_segment' not in globals():
    __boxed_segment = segments.add()
memory[ap + 0] = __boxed_segment
__boxed_segment += 1
%}
[ap + -1] = [[ap + 0] + 0], ap++;
[ap + -5] = [ap + 0] + 1, ap++;
%{
if '__boxed_segment' not in globals():
    __boxed_segment = segments.add()
memory[ap + 0] = __boxed_segment
__boxed_segment += 1
%}
[ap + -1] = [[ap + 0] + 0], ap++;
[ap + 0] = [fp + -4], ap++;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = [ap + -3], ap++;
call rel -16;
ret;
```

## fib u128 case

- high level program

```
fn fib(a: u128, b: u128, n: u128) -> u128 implicits(RangeCheck) {
    if n == 0_u128 {
        a
    } else {
        fib(b, a + b, n - 1_u128)
    }
}
```

- ir

```
type u128 = u128;
type Unit = Struct<ut@Tuple>;
type core::bool = Enum<ut@core::bool, Unit, Unit>;
type RangeCheck = RangeCheck;
type felt = felt;
type Array<felt> = Array<felt>;
type core::PanicResult::<core::integer::u128> = Enum<ut@core::PanicResult::<core::integer::u128>, u128, Array<felt>>;
type core::result::Result::<core::integer::u128, core::integer::u128> = Enum<ut@core::result::Result::<core::integer::u128, core::integer::u128>, u128, u128>;

libfunc revoke_ap_tracking = revoke_ap_tracking;
libfunc u128_const<0> = u128_const<0>;
libfunc dup<u128> = dup<u128>;
libfunc u128_eq = u128_eq;
libfunc branch_align = branch_align;
libfunc struct_construct<Unit> = struct_construct<Unit>;
libfunc enum_init<core::bool, 0> = enum_init<core::bool, 0>;
libfunc store_temp<core::bool> = store_temp<core::bool>;
libfunc jump = jump;
libfunc enum_init<core::bool, 1> = enum_init<core::bool, 1>;
libfunc enum_match<core::bool> = enum_match<core::bool>;
libfunc drop<Unit> = drop<Unit>;
libfunc store_temp<RangeCheck> = store_temp<RangeCheck>;
libfunc store_temp<u128> = store_temp<u128>;
libfunc function_call<user@core::integer::U128Add::add> = function_call<user@core::integer::U128Add::add>;
libfunc enum_match<core::PanicResult::<core::integer::u128>> = enum_match<core::PanicResult::<core::integer::u128>>;
libfunc drop<u128> = drop<u128>;
libfunc enum_init<core::PanicResult::<core::integer::u128>, 1> = enum_init<core::PanicResult::<core::integer::u128>, 1>;
libfunc store_temp<core::PanicResult::<core::integer::u128>> = store_temp<core::PanicResult::<core::integer::u128>>;
libfunc u128_const<1> = u128_const<1>;
libfunc function_call<user@core::integer::U128Sub::sub> = function_call<user@core::integer::U128Sub::sub>;
libfunc function_call<user@fib_u128::fib_u128::fib> = function_call<user@fib_u128::fib_u128::fib>;
libfunc enum_init<core::PanicResult::<core::integer::u128>, 0> = enum_init<core::PanicResult::<core::integer::u128>, 0>;
libfunc u128_overflowing_add = u128_overflowing_add;
libfunc enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 0> = enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 0>;
libfunc store_temp<core::result::Result::<core::integer::u128, core::integer::u128>> = store_temp<core::result::Result::<core::integer::u128, core::integer::u128>>;
libfunc enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 1> = enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 1>;
libfunc felt_const<39878429859757942499084499860145094553463> = felt_const<39878429859757942499084499860145094553463>;
libfunc rename<core::result::Result::<core::integer::u128, core::integer::u128>> = rename<core::result::Result::<core::integer::u128, core::integer::u128>>;
libfunc store_temp<felt> = store_temp<felt>;
libfunc function_call<user@core::result::ResultTraitImpl::<core::integer::u128, core::integer::u128>::expect> = function_call<user@core::result::ResultTraitImpl::<core::integer::u128, core::integer::u128>::expect>;
libfunc u128_overflowing_sub = u128_overflowing_sub;
libfunc felt_const<39878429859763533771555484554338820190071> = felt_const<39878429859763533771555484554338820190071>;
libfunc enum_match<core::result::Result::<core::integer::u128, core::integer::u128>> = enum_match<core::result::Result::<core::integer::u128, core::integer::u128>>;
libfunc drop<felt> = drop<felt>;
libfunc array_new<felt> = array_new<felt>;
libfunc array_append<felt> = array_append<felt>;

revoke_ap_tracking() -> ();
u128_const<0>() -> ([4]);
dup<u128>([3]) -> ([3], [5]);
u128_eq([5], [4]) { fallthrough() 9() };
branch_align() -> ();
struct_construct<Unit>() -> ([6]);
enum_init<core::bool, 0>([6]) -> ([7]);
store_temp<core::bool>([7]) -> ([8]);
jump() { 13() };
branch_align() -> ();
struct_construct<Unit>() -> ([9]);
enum_init<core::bool, 1>([9]) -> ([10]);
store_temp<core::bool>([10]) -> ([8]);
enum_match<core::bool>([8]) { fallthrough([11]) 65([12]) };
branch_align() -> ();
drop<Unit>([11]) -> ();
store_temp<RangeCheck>([0]) -> ([15]);
store_temp<u128>([1]) -> ([16]);
dup<u128>([2]) -> ([2], [17]);
store_temp<u128>([17]) -> ([17]);
function_call<user@core::integer::U128Add::add>([15], [16], [17]) -> ([13], [14]);
enum_match<core::PanicResult::<core::integer::u128>>([14]) { fallthrough([18]) 25([19]) };
branch_align() -> ();
store_temp<u128>([18]) -> ([20]);
jump() { 32() };
branch_align() -> ();
drop<u128>([2]) -> ();
drop<u128>([3]) -> ();
enum_init<core::PanicResult::<core::integer::u128>, 1>([19]) -> ([21]);
store_temp<RangeCheck>([13]) -> ([22]);
store_temp<core::PanicResult::<core::integer::u128>>([21]) -> ([23]);
return([22], [23]);
u128_const<1>() -> ([24]);
store_temp<RangeCheck>([13]) -> ([27]);
store_temp<u128>([3]) -> ([28]);
store_temp<u128>([24]) -> ([29]);
function_call<user@core::integer::U128Sub::sub>([27], [28], [29]) -> ([25], [26]);
enum_match<core::PanicResult::<core::integer::u128>>([26]) { fallthrough([30]) 41([31]) };
branch_align() -> ();
store_temp<u128>([30]) -> ([32]);
jump() { 48() };
branch_align() -> ();
drop<u128>([20]) -> ();
drop<u128>([2]) -> ();
enum_init<core::PanicResult::<core::integer::u128>, 1>([31]) -> ([33]);
store_temp<RangeCheck>([25]) -> ([34]);
store_temp<core::PanicResult::<core::integer::u128>>([33]) -> ([35]);
return([34], [35]);
store_temp<RangeCheck>([25]) -> ([38]);
store_temp<u128>([2]) -> ([39]);
store_temp<u128>([20]) -> ([40]);
store_temp<u128>([32]) -> ([41]);
function_call<user@fib_u128::fib_u128::fib>([38], [39], [40], [41]) -> ([36], [37]);
enum_match<core::PanicResult::<core::integer::u128>>([37]) { fallthrough([42]) 57([43]) };
branch_align() -> ();
store_temp<u128>([42]) -> ([44]);
jump() { 62() };
branch_align() -> ();
enum_init<core::PanicResult::<core::integer::u128>, 1>([43]) -> ([45]);
store_temp<RangeCheck>([36]) -> ([46]);
store_temp<core::PanicResult::<core::integer::u128>>([45]) -> ([47]);
return([46], [47]);
store_temp<RangeCheck>([36]) -> ([48]);
store_temp<u128>([44]) -> ([49]);
jump() { 71() };
branch_align() -> ();
drop<Unit>([12]) -> ();
drop<u128>([2]) -> ();
drop<u128>([3]) -> ();
store_temp<RangeCheck>([0]) -> ([48]);
store_temp<u128>([1]) -> ([49]);
enum_init<core::PanicResult::<core::integer::u128>, 0>([49]) -> ([50]);
store_temp<RangeCheck>([48]) -> ([51]);
store_temp<core::PanicResult::<core::integer::u128>>([50]) -> ([52]);
return([51], [52]);
u128_overflowing_add([0], [1], [2]) { fallthrough([3], [4]) 81([5], [6]) };
branch_align() -> ();
enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 0>([4]) -> ([7]);
store_temp<RangeCheck>([3]) -> ([8]);
store_temp<core::result::Result::<core::integer::u128, core::integer::u128>>([7]) -> ([9]);
jump() { 85() };
branch_align() -> ();
enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 1>([6]) -> ([10]);
store_temp<RangeCheck>([5]) -> ([8]);
store_temp<core::result::Result::<core::integer::u128, core::integer::u128>>([10]) -> ([9]);
felt_const<39878429859757942499084499860145094553463>() -> ([11]);
rename<core::result::Result::<core::integer::u128, core::integer::u128>>([9]) -> ([13]);
store_temp<felt>([11]) -> ([14]);
function_call<user@core::result::ResultTraitImpl::<core::integer::u128, core::integer::u128>::expect>([13], [14]) -> ([12]);
enum_match<core::PanicResult::<core::integer::u128>>([12]) { fallthrough([15]) 93([16]) };
branch_align() -> ();
store_temp<u128>([15]) -> ([17]);
jump() { 98() };
branch_align() -> ();
enum_init<core::PanicResult::<core::integer::u128>, 1>([16]) -> ([18]);
store_temp<RangeCheck>([8]) -> ([19]);
store_temp<core::PanicResult::<core::integer::u128>>([18]) -> ([20]);
return([19], [20]);
enum_init<core::PanicResult::<core::integer::u128>, 0>([17]) -> ([21]);
store_temp<RangeCheck>([8]) -> ([22]);
store_temp<core::PanicResult::<core::integer::u128>>([21]) -> ([23]);
return([22], [23]);
u128_overflowing_sub([0], [1], [2]) { fallthrough([3], [4]) 108([5], [6]) };
branch_align() -> ();
enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 0>([4]) -> ([7]);
store_temp<RangeCheck>([3]) -> ([8]);
store_temp<core::result::Result::<core::integer::u128, core::integer::u128>>([7]) -> ([9]);
jump() { 112() };
branch_align() -> ();
enum_init<core::result::Result::<core::integer::u128, core::integer::u128>, 1>([6]) -> ([10]);
store_temp<RangeCheck>([5]) -> ([8]);
store_temp<core::result::Result::<core::integer::u128, core::integer::u128>>([10]) -> ([9]);
felt_const<39878429859763533771555484554338820190071>() -> ([11]);
rename<core::result::Result::<core::integer::u128, core::integer::u128>>([9]) -> ([13]);
store_temp<felt>([11]) -> ([14]);
function_call<user@core::result::ResultTraitImpl::<core::integer::u128, core::integer::u128>::expect>([13], [14]) -> ([12]);
enum_match<core::PanicResult::<core::integer::u128>>([12]) { fallthrough([15]) 120([16]) };
branch_align() -> ();
store_temp<u128>([15]) -> ([17]);
jump() { 125() };
branch_align() -> ();
enum_init<core::PanicResult::<core::integer::u128>, 1>([16]) -> ([18]);
store_temp<RangeCheck>([8]) -> ([19]);
store_temp<core::PanicResult::<core::integer::u128>>([18]) -> ([20]);
return([19], [20]);
enum_init<core::PanicResult::<core::integer::u128>, 0>([17]) -> ([21]);
store_temp<RangeCheck>([8]) -> ([22]);
store_temp<core::PanicResult::<core::integer::u128>>([21]) -> ([23]);
return([22], [23]);
enum_match<core::result::Result::<core::integer::u128, core::integer::u128>>([0]) { fallthrough([2]) 134([3]) };
branch_align() -> ();
drop<felt>([1]) -> ();
store_temp<u128>([2]) -> ([4]);
jump() { 143() };
branch_align() -> ();
drop<u128>([3]) -> ();
array_new<felt>() -> ([5]);
array_append<felt>([5], [1]) -> ([6]);
struct_construct<Unit>() -> ([7]);
drop<Unit>([7]) -> ();
enum_init<core::PanicResult::<core::integer::u128>, 1>([6]) -> ([8]);
store_temp<core::PanicResult::<core::integer::u128>>([8]) -> ([9]);
return([9]);
enum_init<core::PanicResult::<core::integer::u128>, 0>([4]) -> ([10]);
store_temp<core::PanicResult::<core::integer::u128>>([10]) -> ([11]);
return([11]);

fib_u128::fib_u128::fib@0([0]: RangeCheck, [1]: u128, [2]: u128, [3]: u128) -> (RangeCheck, core::PanicResult::<core::integer::u128>);
core::integer::U128Add::add@75([0]: RangeCheck, [1]: u128, [2]: u128) -> (RangeCheck, core::PanicResult::<core::integer::u128>);
core::integer::U128Sub::sub@102([0]: RangeCheck, [1]: u128, [2]: u128) -> (RangeCheck, core::PanicResult::<core::integer::u128>);
core::result::ResultTraitImpl::<core::integer::u128, core::integer::u128>::expect@129([0]: core::result::Result::<core::integer::u128, core::integer::u128>, [1]: felt) -> (core::PanicResult::<core::integer::u128>);
```

- asm
```
[fp + -3] = [ap + 0] + 0, ap++;
jmp rel 4 if [ap + -1] != 0;
jmp rel 6;
[ap + 0] = 0, ap++;
jmp rel 4;
[ap + 0] = 1, ap++;
jmp rel 56 if [ap + -1] != 0;
[ap + 0] = [fp + -6], ap++;
[ap + 0] = [fp + -5], ap++;
[ap + 0] = [fp + -4], ap++;
call rel 60;
jmp rel 5 if [ap + -3] != 0;
[ap + 0] = [ap + -2], ap++;
jmp rel 8;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = [ap + -4], ap++;
ret;
[ap + 0] = [ap + -5], ap++;
[ap + 0] = [fp + -3], ap++;
[ap + 0] = 1, ap++;
call rel 90;
jmp rel 5 if [ap + -3] != 0;
[ap + 0] = [ap + -2], ap++;
jmp rel 8;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = [ap + -4], ap++;
ret;
[ap + 0] = [ap + -5], ap++;
[ap + 0] = [fp + -4], ap++;
[ap + 0] = [ap + -27], ap++;
[ap + 0] = [ap + -4], ap++;
call rel -51;
jmp rel 5 if [ap + -3] != 0;
[ap + 0] = [ap + -2], ap++;
jmp rel 8;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -4], ap++;
[ap + 0] = [ap + -4], ap++;
ret;
[ap + 0] = [ap + -5], ap++;
[ap + 0] = [ap + -2], ap++;
jmp rel 4;
[ap + 0] = [fp + -6], ap++;
[ap + 0] = [fp + -5], ap++;
[ap + 0] = [ap + -2], ap++;
[ap + 0] = 0, ap++;
[ap + 0] = [ap + -3], ap++;
[ap + 0] = 0, ap++;
ret;
[ap + 1] = [fp + -4] + [fp + -3], ap++;
%{ memory[ap + -1] = memory[ap + 0] < 340282366920938463463374607431768211456 %}
jmp rel 7 if [ap + -1] != 0, ap++;
[ap + -1] = [ap + 0] + 340282366920938463463374607431768211456, ap++;
[ap + -1] = [[fp + -5] + 0];
jmp rel 12;
[ap + -1] = [[fp + -5] + 0];
ap += 1;
[ap + 0] = [fp + -5] + 1, ap++;
[ap + 0] = 0, ap++;
[ap + 0] = [ap + -4], ap++;
jmp rel 7;
[ap + 0] = [fp + -5] + 1, ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -3], ap++;
[ap + 0] = 39878429859757942499084499860145094553463, ap++;
call rel 69;
jmp rel 5 if [ap + -3] != 0;
[ap + 0] = [ap + -2], ap++;
jmp rel 10;
ap += 1;
[ap + 0] = [ap + -11], ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -5], ap++;
[ap + 0] = [ap + -5], ap++;
ret;
[ap + 0] = [ap + -11], ap++;
[ap + 0] = 0, ap++;
[ap + 0] = [ap + -3], ap++;
[ap + 0] = 0, ap++;
ret;
[fp + -4] = [ap + 1] + [fp + -3], ap++;
%{ memory[ap + -1] = memory[ap + 0] < 340282366920938463463374607431768211456 %}
jmp rel 7 if [ap + -1] != 0, ap++;
[ap + 0] = [ap + -1] + 340282366920938463463374607431768211456, ap++;
[ap + -1] = [[fp + -5] + 0];
jmp rel 12;
[ap + -1] = [[fp + -5] + 0];
ap += 1;
[ap + 0] = [fp + -5] + 1, ap++;
[ap + 0] = 0, ap++;
[ap + 0] = [ap + -4], ap++;
jmp rel 7;
[ap + 0] = [fp + -5] + 1, ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -3], ap++;
[ap + 0] = 39878429859763533771555484554338820190071, ap++;
call rel 22;
jmp rel 5 if [ap + -3] != 0;
[ap + 0] = [ap + -2], ap++;
jmp rel 10;
ap += 1;
[ap + 0] = [ap + -11], ap++;
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -5], ap++;
[ap + 0] = [ap + -5], ap++;
ret;
[ap + 0] = [ap + -11], ap++;
[ap + 0] = 0, ap++;
[ap + 0] = [ap + -3], ap++;
[ap + 0] = 0, ap++;
ret;
jmp rel 5 if [fp + -5] != 0;
[ap + 0] = [fp + -4], ap++;
jmp rel 11;
%{ memory[ap + 0] = segments.add() %}
ap += 1;
[fp + -3] = [[ap + -1] + 0];
[ap + 0] = 1, ap++;
[ap + 0] = [ap + -2], ap++;
[ap + 0] = [ap + -3] + 1, ap++;
ret;
[ap + 0] = 0, ap++;
[ap + 0] = [ap + -2], ap++;
[ap + 0] = 0, ap++;
ret;
```

# hints 
## high level program expr
## token && ast
## ir

irGen transform from semaAST to irProgram

note Function and Libfunc

```
pub struct Program {
    /// Declarations for all the used types.
    pub type_declarations: Vec<TypeDeclaration>,
    /// Declarations for all the used library functions.
    pub libfunc_declarations: Vec<LibfuncDeclaration>,
    /// The code of the program.
    pub statements: Vec<Statement>,
    /// Descriptions of the functions - signatures and entry points.
    pub funcs: Vec<Function>,
}
```

### starkNet encoded ir
`sierra_to_felts` convert `sierraProgram` to `ContractClass` serializedSierraProgram
```
    let mut sierra_program = db
        .get_sierra_program_for_functions(
            chain!(&external_functions, &constructor_functions).cloned().collect(),
        )
        .to_option()
        .with_context(|| "Compilation failed without any diagnostics.")?;

    if compiler_config.replace_ids {
        sierra_program = Arc::new(replace_sierra_ids_in_program(db, &sierra_program));
    }
    let replacer = CanonicalReplacer::from_program(&sierra_program);
    let sierra_program = replacer.apply(&sierra_program);

    let entry_points_by_type = ContractEntryPoints {
        external: get_entry_points(db, &external_functions, &replacer)?,
        l1_handler: vec![],
        /// TODO(orizi): Validate there is at most one constructor.
        constructor: get_entry_points(db, &constructor_functions, &replacer)?,
    };
    let contract_class = ContractClass {
        sierra_program: sierra_to_felts(
            sierra_version::VersionId::current_version_id(),
            &sierra_program,
        )?,
```

```
/// Represents a contract in the Starknet network.
#[derive(Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct ContractClass {
    pub sierra_program: Vec<BigIntAsHex>,
V    pub sierra_program_debug_info: Option<cairo_lang_sierra::debug_info::DebugInfo>,
    pub contract_class_version: String,
    pub entry_points_by_type: ContractEntryPoints,
    pub abi: Option<Contract>,
}
```

```
// Serializes a Sierra program into a vector of felts.
pub fn sierra_to_felts(
    sierra_version: VersionId,
    program: &Program,
) -> Result<Vec<BigIntAsHex>, FeltSerdeError> {
    let mut serialized = vec![];
    sierra_version.serialize(&mut serialized)?;
    program.serialize(&mut serialized)?;
    Ok(serialized)
}
```

## asm

asmCodeGen transform from irProgram(Program) to AsmProgram(CairoProgram)

### asm struct: insts + hints

```
// asm program struct
pub struct CairoProgram {
    pub instructions: Vec<Instruction>,
    ...
}

// inst: inst + hints
pub struct Instruction {
    pub body: InstructionBody,
    pub inc_ap: bool,
    pub hints: Vec<Hint>,
}

// inst ISA enum
pub enum InstructionBody {
    AddAp(AddApInstruction),
    AssertEq(AssertEqInstruction),
    Call(CallInstruction),
    Jnz(JnzInstruction),
    Jump(JumpInstruction),
    Ret(RetInstruction),
}

// hints enum
pub enum Hint {
    AllocSegment {
        dst: CellRef,
    },
    ...
    SquareRoot {
        value: ResOperand,
        dst: CellRef,
    },
    ...
}
```

### asm print(human readable)
- hints printAsm
```
impl Display for Hint {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        match self {
            Hint::AllocSegment { dst } => write!(f, "memory{dst} = segments.add()"),
            Hint::SquareRoot { value, dst } => {
                write!(f, "(memory{dst}) = sqrt({})", ResOperandFormatter(value))
            }
            ...
        }
    }
}
```
- general printAsm

InstructionBody fit to ISA

### asm(encoded)

encoded serialize format
```
pub struct CasmContractClass {
    #[serde(serialize_with = "serialize_big_uint", deserialize_with = "deserialize_big_uint")]
    pub prime: BigUint,
    pub compiler_version: String,
    pub bytecode: Vec<BigIntAsHex>,
    pub hints: Vec<(usize, Vec<String>)>,
    pub entry_points_by_type: CasmContractEntryPoints,
}
```

encoded insts+hints

```
        for instruction in cairo_program.instructions {
            if !instruction.hints.is_empty() {
                hints.push((
                    bytecode.len(),
                    instruction.hints.iter().map(|hint| hint.to_string()).collect(),
                ))
            }
            bytecode.extend(instruction.assemble().encode().iter().map(|big_int| {
                let (_q, reminder) = big_int.magnitude().div_rem(&prime);

                BigIntAsHex {
                    value: if big_int.is_negative() { &prime - reminder } else { reminder },
                }
            }))
        }
```

case 
see [hello_starknet.casm](https://github.com/starkware-libs/cairo/blob/main/crates/cairo-lang-starknet/test_data/hello_starknet.casm)
### 

### compile ir to asm

for stmt case `Statement::Invocation` use `compile_invocation` to extend  `instructions`  vec.

testcase `sierra_to_casm` flow: 

`sierra_code -> ProgramParser -> compileIrToAsm -> CairoProgram`.

## trace

vm running ir program

Runner struct contains three parts: `ir`, `asm` and map for `register libFunc`

```
pub struct SierraCasmRunner {
    /// The sierra program.
    sierra_program: cairo_lang_sierra::program::Program,
    /// Metadata for the Sierra program.
    metadata: Metadata,
    /// Program registry for the Sierra program.
    sierra_program_registry: ProgramRegistry<CoreType, CoreLibfunc>,
    /// The casm program matching the Sierra code.
    casm_program: CairoProgram,
}
```

## db to ir

```
let db = &mut RootDatabase::builder().detect_corelib().build()?;
...
let main_crate_ids = setup_project(db, Path::new(&args.path))?;
...
let sierra_program = db
    .get_sierra_program(main_crate_ids)
...
```

```
pub fn get_sierra_program(
    db: &dyn SierraGenGroup,
    requested_crate_ids: Vec<CrateId>,
) -> Maybe<Arc<cairo_lang_sierra::program::Program>>
```

## ir to asm

instancing runner means that compile ir to casm with register-libFunc

```
    pub fn new(
        sierra_program: cairo_lang_sierra::program::Program,
        calc_gas: bool,
    ) -> Result<Self, RunnerError> {
        let metadata = create_metadata(&sierra_program, calc_gas)?;
        let sierra_program_registry =
            ProgramRegistry::<CoreType, CoreLibfunc>::new(&sierra_program)?;
        let casm_program =
            cairo_lang_sierra_to_casm::compiler::compile(&sierra_program, &metadata, calc_gas)?;
        Ok(Self { sierra_program, metadata, sierra_program_registry, casm_program })
    }
```

## run function 
```
let result = runner
    .run_function("::main", &[], args.available_gas)
    .with_context(|| "Failed to run the function.")?;

    pub fn run_function(
        &self,
        name_suffix: &str,
        args: &[Felt],
        available_gas: Option<usize>,
    ) -> Result<RunResult, RunnerError>
```

`find_function` -> `create_entry_code` for (entry,builtin) -> `create_code_footer` -> 
    `run_function`on (instructions,builtins,additional_initialization,RunFunctionContext)
```
    pub fn run_function(
        &self,
        name_suffix: &str,
        args: &[Felt],
        available_gas: Option<usize>,
    ) -> Result<RunResult, RunnerError> {
        let func = self.find_function(name_suffix)?;
        let initial_gas = self.get_initial_gas(func, available_gas)?;
        let (entry_code, builtins) = self.create_entry_code(func, args, initial_gas)?;
        let footer = self.create_code_footer();
        let (cells, ap) = casm_run::run_function(
            chain!(entry_code.iter(), self.casm_program.instructions.iter(), footer.iter()),
            builtins,
            |context| {
                let vm = context.vm;
                // Create the builtin cost segment, with dummy values.
                let builtin_cost_segment = vm.add_memory_segment();
                for token_type in CostTokenType::iter_precost() {
                    vm.insert_value(
                        &(builtin_cost_segment + (token_type.offset_in_builtin_costs() as usize)),
                        Felt::from(DUMMY_BUILTIN_GAS_COST),
                    )?;
                }
                // Put a pointer to the builtin cost segment at the end of the program (after the
                // additional `ret` statement).
                vm.insert_value(&(vm.get_pc() + context.data_len), builtin_cost_segment)?;
                Ok(())
            },
        )?;
    ...
```
### `run_function` on insts and hints
```
pub fn run_function<'a, Instructions: Iterator<Item = &'a Instruction> + Clone>(
    instructions: Instructions,
    builtins: Vec<String>,
    additional_initialization: fn(
        context: RunFunctionContext<'_>,
    ) -> Result<(), Box<VirtualMachineError>>,
) -> Result<(Vec<Option<Felt>>, usize), Box<VirtualMachineError>> 
```

1. `inst.assemble().encode())` 

2. instance `iCairoHintProcessor`

```
let mut hint_processor = CairoHintProcessor::new(instructions);
```

3. fill cairoVM program and instance CairoRunner
```
pub struct Program {
    pub builtins: Vec<String>,
    pub prime: String,
    pub data: Vec<MaybeRelocatable>,
    pub constants: HashMap<String, Felt>,
    pub main: Option<usize>,
    //start and end labels will only be used in proof-mode
    pub start: Option<usize>,
    pub end: Option<usize>,
    pub hints: HashMap<usize, Vec<HintParams>>,
    pub reference_manager: ReferenceManager,
    pub identifiers: HashMap<String, Identifier>,
    pub error_message_attributes: Vec<Attribute>,
    pub instruction_locations: Option<HashMap<usize, InstructionLocation>>,
}

    let mut runner = CairoRunner::new(&program, "all", false)
        .map_err(VirtualMachineError::from)
        .map_err(Box::new)?;
```

4. run program in cairo-vm
```
    let mut vm = VirtualMachine::new(true);

    let end = runner.initialize(&mut vm).map_err(VirtualMachineError::from).map_err(Box::new)?;

    additional_initialization(RunFunctionContext { vm: &mut vm, data_len })?;
    
    // hints two traits
    runner.run_until_pc(end, &mut vm, &mut hint_processor)?;
    runner.end_run(true, false, &mut vm, &mut hint_processor).map_err(Box::new)?;
    runner.relocate(&mut vm).map_err(VirtualMachineError::from).map_err(Box::new)?;
    Ok((runner.relocated_memory, runner.relocated_trace.unwrap().last().unwrap().ap))
```

- need impl `compile_hint` trait at `get_hint_data_dictionary` in `run_until_pc`

- need impl `execute_hint` trait at `step_hint` of `vm.step` in `run_until_pc`

#### compile_hint 

string -> hints

```
    fn compile_hint(
        &self,
        hint_code: &str,
        _ap_tracking_data: &ApTracking,
        _reference_ids: &HashMap<String, usize>,
        _references: &HashMap<usize, HintReference>,
    ) -> Result<Box<dyn Any>, VirtualMachineError> {
        Ok(Box::new(self.string_to_hint[hint_code].clone()))
    }
```

#### execute_hint

pattern match to interpretation hints enum

take `SquareRoot` for example, it call rustlib `sqrt` to exec logic, and fill vm memory cell with the result.

```
   fn execute_hint(
        &mut self,
        vm: &mut VirtualMachine,
        exec_scopes: &mut ExecutionScopes,
        hint_data: &Box<dyn Any>,
        _constants: &HashMap<String, Felt>,
    ) -> Result<(), HintError> {
        let hint = hint_data.downcast_ref::<Hint>().unwrap();
        match hint {
            ...
             Hint::SquareRoot { value, dst } => {
                let val = get_val(vm, value)?.to_biguint();
                insert_value_to_cellref!(vm, dst, Felt::from(val.sqrt()))?;
            }
            ...
        }
        ...
    }
```

### cairo-vm hint design

see [Hint Processor](https://github.com/lambdaclass/cairo-rs/tree/main/docs/hint_processor)

compile hint data content:
- hint code string
- ariable name map
- variable references map
- ap tracking

execute hint content:
- exec_scopes_proxy
- vm_proxy
- MemoryProxy
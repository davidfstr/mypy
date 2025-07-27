# TypeForm Implementation Explainer

This document explains the implementation of PEP 747's TypeForm feature in Mypy, aimed at providing context for AI agents working on this feature during code review.

## What is TypeForm?

TypeForm is a special form introduced by PEP 747 that enables type-safe annotation of functions that accept "type form objects" - runtime representations of types that result from evaluating type expressions.

### Problem Solved

Before TypeForm, functions that operated on runtime type objects (like `int | str`, `list[int]`, etc.) had to be annotated with overly broad types like `object`. This made type checking impossible and reduced type safety. TypeForm solves this by providing a way to precisely annotate what kinds of type objects a function accepts.

### Key Use Cases

- **Runtime type checkers** (beartype, typeguard, trycast)
- **Data validation libraries** (pydantic, cattrs)
- **Type introspection utilities** (typing.get_origin, typing.get_args)
- **Metaprogramming frameworks** that work with types at runtime

## Core Concepts

### TypeForm vs Type[T]

- `Type[T]` - Only accepts class objects (instances of `builtins.type`)
- `TypeForm[T]` - Accepts any type form object representing type `T`, including:
  - Union types (`int | str`)
  - Generic types (`list[int]`)
  - Special forms (`Any`, `Literal['x']`)
  - String annotations (`"int | str"`)

### Type Form Recognition

Type expressions can be recognized as TypeForm values in specific contexts:

```python
# Assignment context
typx: TypeForm[int | str] = int | str  # OK

# Function argument context
def check_type(value: object, typx: TypeForm[T]) -> TypeIs[T]: ...
check_type(42, int | str)  # OK

# Return context
def get_type() -> TypeForm[int]:
    return int  # OK
```

## Implementation Architecture

### Core Type Representation

TypeForm is implemented by extending the existing `TypeType` class with an `is_type_form` flag:

- `TypeType(item=int, is_type_form=False)` → `Type[int]`
- `TypeType(item=int, is_type_form=True)` → `TypeForm[int]`

This design allows reuse of existing type manipulation logic while distinguishing between the two concepts.

### Two-Pass Analysis

The implementation uses a two-pass approach due to architectural constraints:

#### Pass 1: SemanticAnalyzer
- Attempts to parse expressions as type expressions at specific syntactic locations
- Stores successful parse results in `Expression.as_type` attribute
- Only certain expression types support this: `IndexExpr`, `NameExpr`, `OpExpr`, `StrExpr`

#### Pass 2: TypeChecker
- When an expression is in a TypeForm context, checks if it was successfully parsed as a type
- If yes, assigns `TypeForm[parsed_type]` instead of normal expression type
- Falls back to normal type inference if parsing failed

### Supported Expression Types

Only certain AST node types can potentially be type expressions:

- **`NameExpr`** - Simple names like `int`, `str`
- **`MemberExpr`** - Qualified names like `typing.Union`
- **`IndexExpr`** - Generic types like `list[int]`
- **`OpExpr`** - Union expressions like `int | str`
- **`StrExpr`** - String annotations like `"int | str"`

### String Annotation Handling

String annotations require special handling because they need to be parsed as Python code:

- Full parsing happens during SemanticAnalyzer pass when symbol tables are available
- TypeChecker pass cannot parse string annotations due to lack of local symbol context
- Error code `maybe-unrecognized-str-typeform` warns when string usage might be intended as TypeForm

## Key Implementation Components

### New AST Node: TypeFormExpr

Represents explicit `TypeForm(...)` constructor calls:
```python
x = TypeForm(int | str)  # Creates TypeFormExpr node
```

### Enhanced Type Visitors

All type manipulation visitors updated to handle `is_type_form` flag:
- `is_subtype()` - TypeForm covariance rules
- `join_types()` - Union logic for TypeForm types
- `meet_types()` - Intersection logic for TypeForm types
- Type copying/transformation visitors

### Error Handling

New error code `MAYBE_UNRECOGNIZED_STR_TYPEFORM` for cases where string annotations cannot be recognized as TypeForm in certain contexts.

## Architectural Challenges & Trade-offs

### Limited Recognition Contexts

**Challenge**: Type form literals only recognized in specific syntactic locations (assignments, function calls, returns), not everywhere.

**Reason**: TypeChecker cannot parse type expressions itself; only SemanticAnalyzer can. Checking every possible location would require excessive parsing attempts.

**Trade-off**: Some valid TypeForm usage may not be recognized, requiring explicit `TypeForm(...)` wrapper.

### Memory Overhead

**Challenge**: Adding `as_type` attribute to Expression nodes increases memory usage.

**Solution**: Only specific expression subtypes that can be type expressions get the attribute, minimizing impact.

### Performance Concerns

**Challenge**: Many unnecessary calls to `try_parse_as_type_expression()` during semantic analysis.

**Mitigation**: Optimizations added to quickly reject expressions that clearly aren't types (e.g., most OpExpr that aren't `|`, IndexExpr with non-type bases).

## Type Relationships

### Subtyping Rules

- `TypeForm[B]` is subtype of `TypeForm[A]` if `B` is subtype of `A` (covariant)
- `Type[B]` is subtype of `TypeForm[A]` if `B` is subtype of `A`
- `TypeForm` has all attributes/methods of `object`

### Join/Meet Behavior

TypeForm types participate in union/intersection operations:
- `TypeForm[int] | TypeForm[str]` → `TypeForm[int | str]`
- `Type[int] | TypeForm[str]` → `TypeForm[int | str]`

### Normalization

Unlike `Type[X | Y]` which normalizes to `Type[X] | Type[Y]`, TypeForm preserves unions:
- `TypeForm[X | Y]` stays as `TypeForm[X | Y]`

## Feature Enablement

TypeForm is an **opt-in feature** requiring:
```bash
mypy --enable-incomplete-feature=TypeForm
```

This allows the feature to be tested and refined before becoming standard.

## Integration Points

### Built-in Functions

Special handling for:
- `isinstance(obj, typx)` where `typx: TypeForm[T]` narrows to `Type[T]`
- Type introspection functions like `typing.get_origin()`

### Testing

Comprehensive test suite in `test-data/unit/check-typeform.test` covers:
- Basic TypeForm usage patterns
- String annotation handling
- Error cases and edge conditions
- Integration with other typing features

## Current Limitations

1. **Incomplete syntactic coverage** - Not all valid TypeForm usage locations are recognized
2. **String annotation restrictions** - Some string type expressions cannot be parsed in certain contexts
3. **Performance overhead** - Additional parsing attempts during semantic analysis
4. **Memory usage** - Extra attributes on Expression nodes

## Future Improvements

Based on code review feedback, potential enhancements include:

1. **Enhanced TypeChecker parsing** - Enable type expression parsing during TypeChecker pass to eliminate two-pass requirement
2. **Broader recognition** - Recognize TypeForm literals in more syntactic contexts
3. **Performance optimization** - Reduce unnecessary parsing attempts
4. **Better error messages** - More specific guidance for unrecognized TypeForm usage

## References

- **PEP 747**: https://peps.python.org/pep-0747/
- **Implementation PR**: https://github.com/python/mypy/pull/18690
- **Test suite**: `test-data/unit/check-typeform.test`
- **Error documentation**: `docs/source/error_code_list.rst` (maybe-unrecognized-str-typeform section)

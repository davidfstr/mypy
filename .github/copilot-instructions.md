# Mypy Development Guide for AI Agents

Mypy is a static type checker for Python, featuring a multi-pass semantic analyzer, incremental daemon mode, and an optimizing compiler (mypyc).

## Architecture Overview

### Core Components
- **`mypy/main.py`**: Entry point and CLI handling
- **`mypy/build.py`**: Build orchestration, dependency resolution, and incremental analysis
- **`mypy/semanal*.py`**: Multi-pass semantic analyzer (handles forward refs, import cycles)
- **`mypy/checker.py`**: Type checking engine
- **`mypy/dmypy_server.py`**: Daemon mode for fast incremental checking
- **`mypyc/`**: Python-to-C compiler using type annotations

### Key Data Structures
- **`MypyFile`**: AST root for a module
- **`TypeInfo`**: Class/type metadata with MRO, symbol tables
- **`SymbolTable`**: Name resolution mappings
- **`Type` hierarchy**: `mypy/types.py` - runtime type representations

## Development Workflow

### Setting Up
```bash
# Install in editable mode with test dependencies
pip install -r test-requirements.txt
pip install -e .
```

### Testing Patterns
```bash
# Run specific test by name
python runtests.py testNewSyntaxBasics
# or: pytest -n0 -k testNewSyntaxBasics

# Run specific test file
python runtests.py check-dataclasses.test

# Self-check mypy's own code
python runtests.py self

# Run linting
python runtests.py lint
```

### Test File Format (`test-data/unit/check-*.test`)
```
[case testMyFeature]
# flags: --python-version 3.10
x: int = "hello"  # E: Incompatible types in assignment
y: str
y = 5  # E: Incompatible types in assignment (expression has type "int", variable has type "str")

[builtins fixtures/dict.pyi]  # Use specific stubs
```

## Key Conventions

### Error Handling
- Use `mypy.errors.Errors` for collecting type check errors
- Error codes in `mypy/errorcodes.py` (e.g., `codes.ASSIGNMENT`)
- Messages via `MessageBuilder` in `mypy/messages.py`

### Plugin System
- Plugins in `mypy/plugins/` extend semantic analysis/type checking
- Hook into specific full names via `get_method_hook()`, `get_attribute_hook()`
- Must be idempotent (called multiple times per target)
- Use `add_plugin_dependency()` for incremental safety

### Semantic Analysis Phases
1. **Pass 1** (`semanal_pass1.py`): Basic symbol table population
2. **Main passes** (`semanal.py`): Type resolution, forward reference handling
3. **Final pass**: Type checking (`checker.py`)

### Incremental Analysis
- Fine-grained dependency tracking in `mypy/server/`
- File system watching via `fswatcher.py`
- Metadata stored in `.mypy_cache/` directories
- Use `build.BuildManager` for coordinating rebuilds

## Common Tasks

### Adding New Type Checking Logic
1. Add error code to `errorcodes.py`
2. Implement check in `checker.py` or `checkexpr.py`
3. Add message template to `messages.py`
4. Write tests in `test-data/unit/check-*.test`

### Extending the AST
1. Add node types to `mypy/nodes.py`
2. Update visitors in `mypy/visitor.py` and `mypy/traverser.py`
3. Handle in semantic analyzer (`semanal.py`)
4. Add type checking logic (`checker.py`)

### Plugin Development
- Subclass `mypy.plugin.Plugin`
- Override `get_*_hook` methods for specific hooks
- Use `anal_type()` for type analysis, handle `None` returns with deferrals
- Store plugin state in `TypeInfo.metadata[plugin_name]`

## Performance Considerations
- Mypy itself is compiled with mypyc for 4x speedup
- Use `dmypy` daemon mode for sub-second incremental checks
- Cache symbol tables and type information aggressively
- Avoid expensive operations in hot paths (semantic analysis loops)

## Debugging Strategies

### Development Complexity Notes
- **Take detailed notes**: Mypy's complexity requires written planning before implementation
- Individual steps can be difficult; maintain a written list of goals as an anchor
- **Static analysis limitations**: Code behavior prediction often fails; prefer dynamic analysis

### Essential Debugging Techniques
- **Conditional breakpoints on error reporting**:
  ```python
  # Set breakpoint on mypy.errors.Errors.add_error_info()
  # with condition like: 'substring' in message
  ```
- **Stack trace analysis**: When breakpoint hits, examine call stack to trace error origin
- **Dynamic analysis preferred**: Use debugger over static code reading for complex flows
- **Error message tracking**: Follow specific error messages from source to reporting

### Key Debugging Targets
- `mypy/errors.py:Errors.add_error_info()` - All error reporting flows through here
- `mypy/checker.py` - Type checking decision points
- `mypy/semanal.py` - Symbol resolution and forward reference handling
- `mypy/build.py` - Module dependency and build orchestration

## Integration Points
- **Typeshed**: Standard library stubs in `mypy/typeshed/`
- **Stubgen**: Automatic stub generation (`mypy/stubgen.py`)
- **mypyc**: Compiler integration in `mypyc/` subdirectory
- **IDE support**: Language server protocol via daemon mode

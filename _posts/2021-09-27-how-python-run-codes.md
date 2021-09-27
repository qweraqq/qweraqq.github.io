---
layout: post
title: "How Python run codes"
nav_order: 4
date: 2021-09-27 00:00:00 +0800
author: xiangxiang
categories: python
tags: [python cpython]
---
[CPython internals](https://realpython.com/products/cpython-internals-book/)学习笔记

* auto-gen TOC:
{:toc}
# How Python run codes
- Python runtime (such as CPython) compiles your code when it runs **for the first time**
- It's compiled into a low-level intermediary language called **bytecode**
- This bytecode is stored in `.pyc` files and cached for execution
- The evaluation loop runs codes

## 0 The big picture
```text
File Input  -----------+                           +-------------------------+                        
(python <file>)        |                           |                         |
                       |                           |                         |
                       |                      AST  |          CFG            |  Bytecode
IO Stream   -----------|--> Reader --> Parser ---> | Compiler ----> Assembler| ---------> Execution
(cat <file> | python)  |                           |                         |           (Event Loop)
                       |                           |                         |
                       |                           |    Compilation part     |           
String Input-----------+                           +-------------------------+
(python -c <str>)                                   
```

- The Python Language Specification: The first step to creating a compiler is to define the language `Grammar/python.gram` -> `make regen-pegen`
- Input and configuration: Read Python text from various sources
- Lexing and Parsing With Syntax Trees: Convert into a structure (AST) that the compiler can use
- CPython compiler(Compiler and Assembler): turn the AST into instructions the CPU can understand
- The evaluation loop: Execution of code

## 1 The Python Language Specification
- Human readable: `Doc/reference` directory contains rst explanations of the features in the Python language, the same as [https://docs.python.org/3/reference/](https://docs.python.org/3/reference/)
- Machine-readable: the grammer file `Grammar/python.gram`
- [PEP 5 -- Guidelines for Language Evolution](https://www.python.org/dev/peps/pep-0005/)
### 1.1 The Grammer File `Grammar/python.gram`
- Parsing expression grammar (PEG) specification
- Backus-Naur Form (BNF) specification `Grammar/Grammar` removed from 3.10
### 1.2 The Parser Generator
- If you make changes to ghe grammer file, then you must regenerate the parser
- `make regen-pegen`
### 1.3 Tokens
- `Grammar/Tokens`
- The unique types found as leaf nodes in a parse tree
- Each token also has a name and a generated unique ID
- [tokenize](https://docs.python.org/3/library/tokenize.html) The `tokenize` module is written in pure Python and is located in `Lib/tokenize.py`

## 2 Input and configuration
To execute any Python code, the interpreter needs three elements in place:
1. A module to execute
2. A state to hold information such as variables
3. A configuration, such as which options are enabled [PEP 587](https://www.python.org/dev/peps/pep-0587/)
With these three components, the interpreter can execute code and provide an output
```text   
         +-----> Configuration -----+
         |                          |
         |                          |    
Input ---|-----> State         -----|--> Runtime --> Output
         |                          |
         |                          |
         +-----> Modules       -----+
```
### 2.1 Configuration State
- Preinitialization Configuration: [PyPreConfig](https://github.com/python/cpython/blob/v3.9.0/Include/cpython/initconfig.h#L125)
- Runtime Configuration: [PyConfig](https://github.com/python/cpython/blob/v3.9.0/Include/cpython/initconfig.h#L425)
### 2.2 Build Configuration
- Build configuration properties are compile-time values used to select additional modules to be linked into the binary
- See build configuration by running `python -m sysconfig`
### 2.3 Code inputs
- Local files and packages
- I/O streams, such as stdin or a memory pipe
- Strings

## 3 Lexing and Parsing with Syntax Trees
### 3.1 Concrete Syntax Trees (CST)
- A non-contextual tree representation of tokens and symbols
- The CST is created from a `tokenizer` and a `parser`

```python
import symbol
import token
import parser

def lex(expression):
    symbols = {v: k for k, v in symbol.__dict__.items()
               if isinstance(v, int)}
    tokens = {v: k for k, v in token.__dict__.items()
              if isinstance(v, int)}
    lexicon = {**symbols, **tokens}
    st = parser.expr(expression)
    st_list = parser.st2list(st)

    def replace(l: list):
        r = []
        for i in l:
            if isinstance(i, list):
                r.append(replace(i))
            else:
                if i in lexicon:
                    r.append(lexicon[i])
                else:
                    r.append(i)
        return r
    return replace(st_list)
```
### 3.2 Abstract Syntax Trees (AST)
- A contextual tree representation of Python’s grammar and statements
- [ast library](https://docs.python.org/3/library/ast.html)

```python
import ast
ast.dump(ast.parse('[ord(c) for line in file for c in line]', mode='eval'))
```

## 4 Compiler
-This compilation task is split into two components:
1. Compiler: Traverse the AST and create a control flow graph(CFG), which represents the logical sequence for execution.
2. Assembler: Convert the nodes in the CFG to sequential, executable statements known as bytecode.
- built-in function `compile()`: We can call the `compiler()`in Python, it returns a code object
- [dis library](https://docs.python.org/3/library/dis.html) module, which disassembles the bytecode instructions

```python
import dis
co = compile("b+1", "test.py", mode="eval")
dis.dis(co.co_code)
```

### 4.1 Instantiating a Compiler
- Before the compiler starts, a global compiler state is created. 
- The compiler state (compiler type) structure contains properties used by the compiler, such as compiler flags, the stack, and the PyArena. It also contains links to other data structures, like the symbol table
### 4.2 Compiler flags
- There are two types of flags to toggle the features inside the compiler, `future flags` and `compiler flags`. These flags can be set in two places:
1. The configuration state, which contains environment variables and command-line flags
2. Inside the source code of the module through the use of `__future__` statements
### 4.3 Symbol Tables
- The purpose of the symbol table is to provide a list of namespaces, globals, and locals for the compiler to use for referencing and resolving scopes
- [symtable library](https://docs.python.org/3/library/symtable.html)
```python
import symtable
table = symtable.symtable("def some_func(): pass", "string", "exec")
table.get_symbols()
```
### 4.4 Core Compilation Process (PyAST_CompileObject)
- [PyAST_CompileObject()](https://github.com/python/cpython/blob/v3.9.0/Python/compile.c#L318)
### 4.5 Assembly
- Once these compilation stages have completed, the compiler has a list of frame blocks, each containing a list of instructions and a pointer to the next block. 
- The assembler performs a depth-first search (DFS) of the basic frame blocks and merges the instructions into a single bytecode sequence


## 5 The evaluation loop: Execution of code
- stack frame-based system 
- `Python/ceval.c`: The core evaluation loop implementation 
- `Python/ceval-gil.h`: GIL definination and control algorithm

```text
           interpreter
                |
                |
                |
    thread0             thread1      ...
(thread_state0)    (thread_state1)   ...
        |                  | 
        |                  |
        |                  |
frame object[s]     frame object[s]


  FRAME 0 --- code object                 
    | 
    |
    | fd_back_ptr 
    | (previous)
    |
    +---FRAME 1 --- code object
           |
           |
           | fd_back_ptr
           | (previous)
           |
           +---FRAME 2 --- code object 
                  |
                  |
thread_state------+


+----------------------------------+
|           Frame Object           |
+----------------------------------+
|              +----------------   |
|  Builtins    |  Code Object  |   |
|              +---------------+   |
|  Globals     |               |   |
|              |  Instructions |   |
|  Locals      |               |   |
|              |  Names        |   |
|  Values      |               |   |
|              |  Constants    |   |
|              +---------------+   |
+----------------------------------+
```

- The evaluation loop will take a **code object** and convert it into a series of **frame objects**
- **value stack**: where variables are created, modified, and used by the bytecode
- **evaluation loop**: evaluate and execute a code object fetched from either the marshaled `.pyc` or the compiler
- The interpreter has at least on **thread**
- Each thread has a **thread state**
- Frame objects are executed in a stack, called the **frame stack**

### 5.1 Thread State
- `Include/cpython/pystate.h`

```c++
// Include/pystate.h
/* struct _ts is defined in cpython/pystate.h */
typedef struct _ts PyThreadState;

// Include/cpython/pystate.h
// The PyThreadState typedef is in Include/pystate.h.
struct _ts {
    /* Unique thread state id. */
    uint64_t id;

    /* The frame object type is a PyObject*/
    PyObject *context;
    uint64_t context_ver;

    /* A linked list to the other thread states */
    struct _ts *prev;
    struct _ts *next;

    /* The interpreter state it was spawned by */
    PyInterpreterState *interp;

    /* The currently executing frame */
    /* Borrowed reference to the current frame (it can be NULL) */
    PyFrameObject *frame;

    /* The current recursion depth */
    int recursion_depth;

    char overflowed; /* The stack has overflowed. Allow 50 more calls
                        to handle the runtime error. */
    char recursion_critical; /* The current calls must not cause
                                a stack overflow. */
    int stackcheck_counter;

    /* Optional tracing functions */
    /* 'tracing' keeps track of the execution depth when tracing/profiling.
       This is to prevent the actual trace/profile code from being recorded in
       the trace/profile. */
    int tracing;
    int use_tracing;
    Py_tracefunc c_profilefunc;
    Py_tracefunc c_tracefunc;
    PyObject *c_profileobj;
    PyObject *c_traceobj;

    /* The exception currently being raised */
    PyObject *curexc_type;
    PyObject *curexc_value;
    PyObject *curexc_traceback;

    /* Any async exception currently being handled */
    /* The exception currently being handled, if no coroutines/generators
     * are present. Always last element on the stack referred to be exc_info. 
     */
    _PyErr_StackItem exc_state;

    /* A stack of exceptions raised when multiple exceptions have been raised (within an except block, for example) */
    /* Pointer to the top of the stack of the exceptions currently
     * being handled */
    _PyErr_StackItem *exc_info;

    PyObject *dict;  /* Stores per-thread state */

    /* A GIL counter */
    int gilstate_counter;

    PyObject *async_exc; /* Asynchronous exception to raise */
    unsigned long thread_id; /* Thread id where this tstate was created */

    int trash_delete_nesting;
    PyObject *trash_delete_later;

    /* Called when a thread state is deleted normally, but not when it is destroyed after fork(). */
    void (*on_delete)(void *);
    void *on_delete_data;

    int coroutine_origin_tracking_depth;

    PyObject *async_gen_firstiter;
    PyObject *async_gen_finalizer;
};
```

### 5.3 Frame Object
- `Include/frameobject.h` and `Objects/frameobject.c`
- Compiled code objects are inserted into frame objects
- Frame objects are a Python type, can be referenced from both C and Python
- Frame objects also contain other runtime data required for executing (local variables, global variables, and built-in modules)

### 5.4 Frame Execution
- The public API, `PyEval_EvalFrameEx()`, calls the interpreter’s configured frame evaluation function in the `eval_frame` property. 
- Frame evaluation was made pluggable in Python 3.7 with [PEP 523](https://www.python.org/dev/peps/pep-0523/)
- Default frame execution [_PyEval_EvalFrameDefault() ceval.c#L829](https://github.com/python/cpython/blob/v3.9.0/Python/ceval.c#L829)
- We can use [traceback](https://docs.python.org/3/library/traceback.html) to trace frame execution

### 5.5 The Value Stack
- Inside the core evaluation loop, a value stack is created. 
- **This stackis a list of pointers to PyObject instances**
- These could be values like variables, references to functions (which are objects inPython), or any other Python object.
- Bytecode instructions in the evaluation loop will take input from the value stack.
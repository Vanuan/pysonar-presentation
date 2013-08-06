# Automatic analysis of Python programs

---

## Formalized problem

  Deduce all the objects with methods and their arguments of the specific class
  that might be created/invoked during the run of a given entry point.

---

## Brainstorming

  - record files
    - the simplest
    - doesn't cover all the branches
  - documentation (manual check)
    - requires updates
    - a lot of manual work
  - single entry-point
    - a much more work involved
    - not clear if it would simplify the task
  - mock everything
    - doesn't cover all branches
  - automatic static analysis
    - hard to implement
    - great potential

---

## Basic idea - abstract interpretation

  - "Interpret" invocation of main and save all the objects created with their method calls.
  - Visit all the branches (sacrificing precision)
  - Each statement has its state and modifies the state for the next statement
  - The result of each expression is a set of types (objects/values).
  - "Call" is a special expression that "binds" a set of types to each argument of the callable and interprets callable's body.

---

### Abstract Syntax Tree (AST)

  A built-in Python 'ast' module provides an access to interpreter's internal parser.

---

### An example

ast.parse(

    class A(object):
        def method(self, arg):
            pass
    
    def func():
        a = A()
        a.method('string')
    
    func()

) =>
 
    Module{
      ClassDef 'A' {
        FunctionDef 'method' {
          Pass
        }
      }
      FunctionDef 'func' {
        Assign[ Name('a') = Call(Name('A')()) ]
        Expr[ Call(Attribute[ Name('a').'method' ](Str('string'),)) ]
      }
      Expr[ Call(Name('func')()) ]
    }

---

### Implementation

* [mini-pysonar](http://yinwang0.wordpress.com/2013/06/21/pysonar-slides/)
* Two functions: infer and invoke

---

### Example

    def f(): pass
    f()

    
 1. infer Module.body
     1. infer FunctionDef:
         1. save FunctionDef node (with previous env) to env {"f": Closure(node, env, stk)}
         2. infer next statements:
             1. infer Call:
                 1. infer f -> Closure
                 2. infer arguments and bind them to formal parameters (via env)
                 2. invoke Closure
                     1. infer FunctionDef.body: same as Module.body

---

### Why we thought we would succeed

POC was already done by the original author of mini-pysonar
 
- Basic stuff
  - FunctionDef
  - Name
  - Assign
  - If
  - Call
  - primitive types (Str, Num)

---

### Why we thought we would succeed

It seemed that it would just require a trivial implementation of the semantics for more syntax constructs

* Exceptions (TryExcept, TryExceptFinally, ExceptHandler)
* Loop (For, While)
* Modules (Import, ImportFrom)
* Objects (attributes, methods support (self))
* Inheritance
* Built-in types support (dict, list, tuple)


---

### Why we failed

  - it's turned out that the semantics of syntax is fairly complex
  - statement context is what is the most difficult to infer
  - mini-pysonar was not designed for test-ability
  - poor performance due to bad data structures used
  - difficult source code comprehension due to multiple recursion

---
### Current state

182 entry points

* 149 with (some) information collected
* 33 without information
  * 13 due to non-trivial code (tuple, dict.update, __call__, subscript, etc)
  * 14 correctly
  * 6 failed with "maximum recursion depth exceeded"

Running time: 3 minutes (with some modules skipped)

---
### What's next

  - Fix bugs and edge cases
  - Write (more) tests
  - Detect infinite loops
  - Improve performance
    "Improve __hash__ and __eq__ methods. Skip some modules: logger.py"
  - Implement global/local scope (if needed)
  - Implement built-in types (dict, tuple, list, set, etc) and functions (e.g. map, eval, exec) - hard
  - ast.Subscript

---
### ???


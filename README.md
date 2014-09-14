#Lint.jl

[![Lint](http://pkg.julialang.org/badges/Lint_0.3.svg)](http://pkg.julialang.org/?pkg=Lint&ver=0.3)
[![Build Status](https://travis-ci.org/tonyhffong/Lint.jl.svg?branch=master)](https://travis-ci.org/tonyhffong/Lint.jl)
[![Coverage Status](https://img.shields.io/coveralls/tonyhffong/Lint.jl.svg)](https://coveralls.io/r/tonyhffong/Lint.jl)

## Introduction

Lint.jl is a tool to hunt for imperfections and dodgy structures that could be
improved.

## Installation
```julia
Pkg.add( "Lint" )
```

## Usage
```julia
using Lint
lintfile( "your_.jl_file" )
```
It'd follow any `include` statements.

The output is of the following form:
```
         filename.jl [       function name] Line CODE  Explanation
```
`Line` is the line number relative to the start of the function.
`CODE` gives an indication of severity, and is one of `INFO`, `WARN`, `ERROR`, or `FATAL`.

To simplify life, there is a convenience function for packages:
```julia
using Lint
lintpkg( "MyPackage" )
```

If your package always lints clean, you may want to keep it that way in a test:
```julia
using Test
using Lint
@test isempty(lintpkg( "MyPackage", returnMsgs=true))
```

## What it can find?
* simple deadcode detection (e.g if true/false)
* simple premature-return deadcode detection
* Bitwise `&`, `|` being used in a Bool context. Suggest `&&` and `||`
* Declared but unused variable. Overruled by adding the no-op statement `lintpragma( "Ignore Unused [variable name]" )`
  just before the warning line.
* Using an undefined variable
* Duplicate key as in `[:a=>1, :b=>2, :a=>3]`
* Mixed types in uniform dictionary `[:a=>1, :b=>""]` (ERROR)
* Uniform types in mixed dictionary `{:a=>1, :b=>2}` (performance INFO)
* Exporting non-existing symbols (not fully done yet)
* Exporting the same symbol more than once
* Name overlap between a variable and a lambda argument
* Implicitly-local (local keyword not provided, but neither is global)
  variable assignment that looks like it should be a global variable
  binding, if the name is long enough
* Assignment in an if-predicate, as a potential confusion with `==`
* warn `length()` being used as Bool, suggest `!isempty()`
* Consecutively similar expressions block and that its last part looks different from the rest (work-in-progress)
* Out-of-scope local variable name being reused again inside the same code block. (legal but frowned upon)
* Function arguments being Container on abstract e.g. f(x::Array{Number,1}). Suggest f{T<:Number}(x::Array{T,1})
* Concatenation of strings using `+`. It also catches common string functions, e.g. `string(...) + replace( ... )`
* Iteration over an apparent dictionary using only one variable instead of (k,v) tuple
* Incorrect ADT usage in function definition, e.g. `f{Int}( x::Int )`, `f{T<:Int64}( x::T )`, `f{Int<:Real}( x::Int)`
* Suspicious range literals e.g. `10:1` when it probably should be `10:-1:1`
* Understandable but actually non-existent constructors e.g. String(), Symbol()
* Redefining mathematical constants, such as `e = 2.718`
* Illegal `using` statement inside a function definition
* Repeated function arguments e.g. `f(x,y,x)`
* Wrong usage of ellipsis in function arguments e.g. `f(x, y... , z...)`
* Wrong usage of default value in function arguments e.g. `f(x=1, y)`
* Named arguments without default value e.g. `f(x; y, q=1)`
* Non-leaf type in a container's eltype in an argument e.g. `f( x::Array{Number,1} )`
* Code extending deprecated functions, as defined in deprecated.jl (Base)
* Mispelled constructor function name (when it calls new() inside)
* Constructor forgetting to return the constructed object
* Rudimentary type instability warnings e.g. `a = 1` then followed by `a = 2.0`. Overruled using a no-op statement
  `lintpragma( "Ignore unstable type variable [variable name]" )` just before the warning.
* Incompatible type assertion and assignment e.g. `a::Int = 1.0`
* Incompatible tuple assignment sizes e.g. `(a,b) = (1,2,3)`
* Flatten behavior of nested vcat e.g. `[[1,2],[3,4]]`
* Loop over a single number. e.g. `for i=1 end`
* More indices than dimensions in an array lookup
* Look up a dictionary with the wrong key type, if the key's type can be inferred.

## lintpragma: steering Lint-time behavior
You can insert lintpragma to suppress or generate messages. At runtime, lintpragma is a no-op.
However, at lint-time, these pragmas steer the lint behavior.

Lint message suppression (do not include the square brackets)
* `lintpragma( "Ignore unused [variable name]" )`
* `lintpragma( "Ignore unstable type variable [variable name]" )`
* `lintpragma( "Ignore deprecated [function name]" )`
* `lintpragma( "Ignore undefined module [module name]" )`. Useful to support Julia packages across different Julia releases.

Lint message generation (do not include the square brackets)
* `lintpragma( "Info type [expression]")`. Generate the best guess type of the expression during lint-time.
* `lintpragma( "Info me [any text]")`. An alternative to-do.
* `lintpragma( "Warn me [any text]")`. Remind yourself this code isn't done yet.
* `lintpragma( "Error me [any text]")`. Remind yourself this code is wrong.

## Current false positives
* Because macros can generate new symbols on the fly. Lint will have a hard time dealing
with that. To help Lint and to reduce noise, module designers can add a
`lint_helper` function to their module.

## Module specific lint helper(WIP)
Key info about adding a `lint_helper` function in your module
* You don't need to export this function. Lint will find it.
* It must return true if an expression is part of your module's
  enabling expression (most likely a macrocall). Return false otherwise
  so that Lint can give other modules a go at it. Note that
  if you always returning true in your code you will break Lint.
* `lint_helper` takes two argument, an `Expr` instance and a context.
  - if you find an issue in your expression, call `Lint.msg( ctx, level, "explanation" )`
  - level is 0: INFO, 1:WARN, 2:ERROR, 4:FATAL
* typical structure looks like this
```julia
function lint_helper( ex::Expr, ctx )
    if ex.head == :macrocall
        if ex.args[1] == symbol("@fancy_macro1")
            # your own checking code
            return true
        elseif ex.args[1]== symbol("@fancy_macro2")
            # more checking code
            return true
        end
    end
    return false
end
```

## Advanced lint helper interface information
* the context argument `ctx` has a field called `data` typed `Dict{Symbol,Any}`.
 Access it using `ctx.callstack[end].data`.
 Use it to store anything, although you should be careful of colliding
 name space with other modules' `lint_helper`. It is particularly useful
 for storing current lint context, such as when a certain macro is only allowed
 inside another macro.
* If your macro generates new local variables, call this:
```julia
ctx.callstack[end].localvars[end][ varsymbol ] = ctx.line
```
* If your macro generates new free variables (not bound to a block scope), call this:
```julia
ctx.callstack[end].localvars[1][ varsymbol ] = ctx.line
```
* If your macro generates new functions,
```julia
push!( ctx.callstack[end].functions, funcsymbol )
```
* If your macro generates new types,
```julia
push!( ctx.callstack[end].types, roottypesymbol )
```
You just need to put in the root symbol for a parametric type, for example
`:A` for `A{T<:Any}`.

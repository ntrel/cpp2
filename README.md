Disclaimer:
* These docs are unofficial and may be inaccurate or incomplete.
* At the time of writing, Cpp2 is an *unstable experimental language*, see:
  https://github.com/hsutter/cppfront#cppfront

Note: Some examples are snipped/adapted from:
https://github.com/hsutter/cppfront/tree/main/regression-tests

Note: Examples here use C++23 `std::println` instead of `std::cout`.
If you don't have it, you can use this 1/2 parameter definition:

```c++
std: namespace = {
    println: (a) = std::cout << a << "\n";
    println: (a, b) = std::cout << a << b << "\n";
}
```

# Contents

* [Declarations](#declarations)
* [Variables](#variables)
* [Modules](#modules)
* [Types](#types)
* [Statements](#statements)
* [Functions](#functions)
* [Expressions](#expressions)
* [User-Defined Types](#user-defined-types)
* [Templates](#templates)
* [Aliases](#aliases)


# Declarations

These are of the form:

 * *declaration*:
   + *identifier* `:` *type*? `=` *initializer*

*type* can be omitted for type inference (though not at global scope).

```c++
    x: int = 42;
    y := x;
```
A global declaration can be used before the line declaring it.

## Mixing Cpp1 Declarations

Cpp1 declarations can be mixed in the same file.

```c++
// Cpp2
x := 42;

// Cpp1
int main() {
    return x;
}
```
A Cpp2 declaration cannot use Cpp1 declaration format internally:

```c++
f := () = {
    int x; // error
}
```

Note: `cppfront` has a `-p` switch to only allow pure Cpp2.


# Variables

## Uninitialized Variables

Use of an uninitialized variable is statically detected.
Both branches of an `if` statement must
initialize a variable, or neither.
```c++
    x: int;
    y := x; // error, x is uninitialized
    if f() { x = 1; } // error, x must also be initialized in else branch
```


# Modules

Cpp2 files have the file extensions `.cpp2` and `.h2`.

## Imports

C++23 will support:

```c++
import std;
```
This will be implicitly done in Cpp2. For now common `std` headers are imported.


# Types

See also: [User-Defined Types](#user-defined-types).

## Arrays

Use:
* `std::array` for fixed-size arrays.
* `std::vector` for dynamic arrays.
* `std::span` to reference consecutive elements from either.


## Pointers

A pointer to `T` has type `*T`. Pointer arithmetic is illegal.

### Postfix Pointer Operators

Address of and dereference operators are postfix:
```c++
    x: int = 42;
    p: *int = x&;
    y := p*;
```
This makes `p->` obsolete - use `p*.` instead.

To distinguish these from binary `&` and `*`, use preceeding whitespace.


### `new<T>`

`new<T>` gives `unique_ptr` by default:

```c++
    p: std::unique_ptr<int> = new<int>;
    q: std::shared_ptr<int> = shared.new<int>;
```
Note: `gc.new<T>` will allocate from a garbage collected arena.

There is no `delete` operator. Raw pointers cannot own memory.

### Null Dereferences

Initialization or assignment from null is an error:

```c++
    q: *int = nullptr; // error
```
Instead of using null for `*T`, use `std::optional<*T>`.

By default, `cppfront` also detects a runtime null dereference.
For example when dereferencing a pointer created in Cpp1 code.

```c++
int *ptr;

f: () -> int = ptr*;
```
Calling `f` above produces:

    Null safety violation: dynamic null dereference attempt detected


# Expressions

## Postfix Operators

Besides the [pointer operators](#postfix-pointer-operators),
Cpp2 also only uses postfix instead of prefix form for:
* `++`
* `--`
* `~`

Unlike Cpp1, the immediate result of postfix increment/decrement is the new value.

```c++
    i := 0;
    [[assert: i++ == 1]]
```

## String Interpolation

A bracketed expression with a trailing `$` inside a string will
evaluate the expression, convert it to string and insert it into the
string.

```c++
    a := 2;
    b: std::optional<int> = 2;
    s: std::string = "a^2 + b = (a * a + b.value())$\n";
    [[assert: s == "a^2 + b = 6\n"]]
```

Note: `$` means 'capture' and is also used in [closures](#function-literals)
and [postconditions](#contracts):
<https://github.com/hsutter/cppfront/wiki/Design-note%3A-Capture>

## Bounds Checks

By default, `cppfront` does runtime bound checks when indexing:
```c++
    v: std::vector = (1, 2);
    i := v[-1]; // aborts program

    s: std::string = ("hi");
    i = s[2]; // aborts program
```

## `as`

* *asExpression*:
  + *expression* `as` *type*

`x as T` attempts:
* type conversion (if the type of `x` implicitly converts to `T`)
* customized conversion (using `operator as<T>`), useful for `std::optional`,
  `std::variant` etc.
* construction of `T(x)`
* dynamic casting

An exception is thrown if the expression is well-formed but the conversion is invalid.

```c++
    c := 'A';
    i: int = c as int;
    [[assert: i == 65]]

    v := std::any(5);
    i = v as int;

    s := "hi" as std::string;
    [[assert: s.length() == 2]]
```

## `is`

* *isExpression*:
  + *type* `is` (*type* | *template*)
  + *expression* `is` (*type* | *expression* | *template*)

### Type Tests

Test a type matches another type - `T is Target` is `true` when `T` is the
same type as `Target`.

Test a template predicate with a type - `T is target` attempts `target<T>`
when the result is convertible to `bool`.

### Expression Tests

Test type of an expression - `x is T` attempts:
* `true` when the type of `x` is `T`
* `x.operator is<T>()`

```c++
    i := 5;
    [[assert: i is int]]
    [[assert: !(i is long)]]

    v := std::any(i);
    [[assert: v is int]] // `v.operator is<int>()`
```

Test expression is a certain value - `x is v` attempts:
* `x.operator is(v)`
* `x == v`
* `x as V == v` where `V` is the type of `v`
* `v(x)` if the result is `bool`

```c++
    i := 5;
    [[assert: i is 5]]
    v := std::any(i);
    [[assert: v is 5]]
```

The last lowering allows to test a value by calling a predicate function:
```c++
pred: (x: int) -> bool = x < 20;

test_int: (i: int) = {
    if i is (pred) {
        std::println("(i)$ is less than 20");
    }
}

main: () = 
{
    test_int(5);
    test_int(15);
    test_int(25);
}
```
Note that `pred` is not a type identifier so it must be parenthesized.

Test a template predicate with a compile-time value - `x is Template` tests `Template<(x)>`
if the result is convertible to `bool`.

## `inspect`

* *inspectExpression*:
  + `inspect` `constexpr`? *expression* (`->` *type*)? `{` *alternative** `}`
* *alternative*:
  + `is` (*type* | *expression*) `=` *statement*
  + `as` *type* `=` *statement*

```c++
v : std::any = 12;

main: () = {
    s: std::string;
    s = inspect v -> std::string {
        is 5 = "five";
        is int = "some other integer";
        is _ = "not an integer";
    };
    std::println(s);
}
```
An inspect expression must have an `is _` case.


# Statements

A condition expression does not require parentheses in Cpp2, though when
a statement immediately follows a condition, a *blockStatement* is required.

## `if`

* *ifStatement*:
  + `if` `constexpr`? *expression* *blockStatement* *elseClause*?
* *elseClause*:
  + `else` *blockStatement*
  + `else` *ifStatement*

```c++
    if c1 {
        ...
    } else if c2 {
        ...
    } else {
        ...
    }
```

## Assertions

```c++
    x := 1
    [[assert: x == 1]]
```

## Parameterized Statement

* *parameterizedStatement*:
  + *parameterList* *statement*

A parameterized statement declares one or more variables that are
defined only for the scope of *statement*.

```c++
(tmp := some_complex_expression) func(tmp, tmp);
// tmp no longer in scope
```

## `while`

* *whileStatement*:
  + `while` *expression* *nextClause*? *blockStatement*
* *nextClause*:
  + `next` *expression*

If `next` is present, its expression will be evaluated at the
end of each loop iteration.

```c++
    (copy i := 0) while i < 3 next i++ {
        std::println(i);
    }
```

## `do`

* *doWhileStatement*:
  + `do` *blockStatement* `while` *expression* *nextClause*? `;`

```c++
    (copy i := 0) do {
        std::println(i);
    } while i < 2 next i++;
```

## `for`

* *forStatement*:
  + `for` *expression* *nextClause*? `do` `(` *parameter* `)` *statement*

The first *expression* must be a range.
*[parameter](#functions)* is initialized from each element of the
range. The parameter type is inferred. 
*parameter* can have `inout` *[parameterStorage](#parameter-passing)*.

```c++
    vec: std::vector<int> = (1, 2, 3);

    for vec do (inout e)
        e++;

    [[assert: vec[0] == 2]]
    for vec do (e)
        std::println(e);
```
<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-intro-for-with-counter-include-last.cpp2>


## Labelled `break` and `continue`

The target of these statements can be a labelled loop.

```c++
    outer: while true {
        j := 0;
        while j < 3 next j++ {
            if done() {
                break outer;
            }
        }
    }
```

# Functions

## Function Types

 * *functionType*:
   + *parameterList* *returnSpec*
 * *parameterList*:
   + `(` *parameter** `)`
 * *parameter*:
   + *[parameterStorage](#parameter-passing)*? *type*.
   + *[parameterStorage](#parameter-passing)*? *identifier* `:` *type*.
 * *returnSpec*:
   + `->` (`forward` | `move`)? *type*
   + `->` *parameterList*

E.g. `(int, float) -> bool`.

## Function Declarations

 * *functionDeclaration*:
   + *identifier*? `:` *parameterList* *returnSpec*? (`;` | [*contracts*](#contracts)? `=` *functionInitializer*)

Function declarations extend the [declaration form](#declarations).
Each parameter must have an identifier.

If *returnSpec* is missing, the function returns `void`.
The return type can be inferred from the initializer by using `-> _`.

See also [Template Functions](#template-functions).


## functionInitializer

 * *functionInitializer*:
   + (*expression* `;` | *statement*)

A function is initialized from a statement or an expression.

```c++
d: (i: int) = { std::println(i); }
e: (i: int) = std::println(i); // same
```

If the function has *returnSpec*, the expression form implies a `return` statement.

```c++
f: (i: int) -> int = return i;
g: (i: int) -> int = i; // same
```

## Returning Multiple Values

When a return parameter is declared, it must be named. 
Each parameter must be initialized in the function body.

```c++
f: () -> (i: int, s: std::string) = {
    i = 10;
    s = "hi";
}

main: () =
{
    t := f();
    [[assert: t.i == 5]]
    [[assert: t.s == "hi"]]
}
```

## `main`

* *mainFunction*:
  + `main` `:` `(` `args`? `)` (`->` `int`)? `=` *functionInitializer*

If `args` is declared, it is a `std::vector<std::string_view>` containing
each command-line argument to the program.


## Uniform Call Syntax

If a method doesn't exist when using method call syntax, and there is a
function whose first parameter can take the type of the 'object'
expression, then that function is called instead.

```c++
main: () -> int = {
    // call C functions
    myfile := fopen("xyzzy", "w");
    myfile.fprintf("Hello %d!", 2); // fprintf(myfile, "Hello %d!", 2)
    myfile.fclose(); // fclose(myfile)
}
```

## Parameter Passing

* `in` - default, read-only. 
  Will pass by reference when more efficient, otherwise pass by value.
* `inout` - pass by mutable reference.
* `out` - must be written to.
  Can accept an uninitialized argument, otherwise destroys the argument.
  The first assignment constructs the parameter.
  Used for [constructors](#operator).
* `move` - argument can be moved from. Used for [destructors](#operator).
* `copy` - argument can be copied from.
* `forward` - accepts lvalue or rvalue, pass by reference.

```c++
e: (i: int) = i++; // error, `i` is read-only
f: (inout i: int) = i++; // mutate argument

g: (out i: int) = {
    v := i; // error, `i` used before initialization
    // error, `i` was not initialized
}
```
Functions can return by reference:
```c++
first: (forward v: std::vector<int>) -> forward int = v[0];

main: () -> int = {
    v : std::vector = (1,2,3);
    first(v) = 4;
}
```
<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-parameter-passing.cpp2>

A variable can also be explictly moved. The move constructor of `z` will destroy `x`:

```c++
    x: std::string = "hi";
    z := (move x);
    [[assert: z == "hi"]]
    [[assert: x == ""]]
```

## Contracts

```c++
vec: std::vector<int> = ();

insert_at: (where: int, val: int)
    [[pre:  0 <= where && where <= vec.ssize()]]
    [[post: vec.ssize() == vec.ssize()$ + 1]]
= {
    vec.insert( vec.begin()+where, val );
}
```
The postcondition compares the vector size at the end of the function call with
an expression that captures the vector size at the start of the function call.

<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-postexpression-with-capture.cpp2>

## Function Literals

A function literal is declared like a named function, but omitting the leading identifier.
Variables can be captured:

```c++
    s: std::string = "Got: ";
    f := :(x) = { std::println(s$, x); };
    f(5);
    f("str");
```
* `s$` means capture `s` by value.
* `s&$*` can be used to dereference the captured address of `s`.


## Template Functions

A [template](#templates) function declaration can have template parameters:

 * *functionTemplate*:
   + *identifier*? `:` *templateParameterList*? *parameterList* *returnSpec*? *requiresClause*?

E.g. `size:<T> () -> _ = sizeof(T);`

When a function parameter type is `_`, this implies a template with a
corresponding type parameter.

A template function parameter can also be just `identifier`.

```c++
f: (x: _) = x;
g: (x) = x; // same
```


# User-Defined Types

`type` declares a user-defined type with data members and member functions.
When the first parameter is `this`, it is an instance method.
```c++
myclass : type = {
    data: int = 42;
    more: std::string = std::to_string(42);

    // method
    print: (this) = {
        std::println("data: (data)$, more: (more)$");
    }

    // non-const method
    inc: (inout this) = data++;
}

main: () = {
    x: myclass = ();
    x.print();
    x.inc();
    x.print();
}
```
Data members are `private` by default, whereas methods are `public`.
Member declarations can be prefixed with `private` or `public`.


## `operator=`

Official docs: <https://github.com/hsutter/cppfront/wiki/Cpp2:-operator=,-this-&-that>.

`operator=` with an `out this` first parameter is called for construction.
When only one subsequent parameter is declared, assignment will also
call this function.

```c++
    operator=: (out this, i: int) = {
        this.data = i;
    }
...
    x: myclass = 99;
    x = 1;
```
With only one parameter `move this`, it is called to destroy the object:

```c++
    operator=: (move this) = {
        std::println("destroying (data)$ and (more)$");
    }
```
Objects are destroyed on last use, not end of scope.


## Inheritance

```c++
base: type = {
    operator=: (out this, i: int) = {}
}

derived: type = {
    this: base = (5); // declare parent class & construct with `base(5)`
}
```

## Type Templates

 * *typeTemplate*:
   + *identifier*? `:` *templateParameterList*? `type` *requiresClause*?


# Templates

 * *templateParameterList*:
   + `<` *templateParameters* `>`
 * *templateParameter*
   + *identifier* (`:` `type`)?
   + *identifier* `:` *type*

The first parameter form accepts a type.

The second parameter form accepts a value.
To use a non-type identifier as a template parameter,
enclose it in parentheses:
```c++
f: <i: int> () -> _ = i;
constexpr int n = 5;
...
    std::println(f<(n)>());
```
Note: This helps the parser to unambiguously parse an expression.
An identifier argument would otherwise be parsed as a type, which would only
be resolved as a value after semantic analysis.

## Constraints

 * *requiresClause*:
   + `requires` *constExpression*

For now use a constraint instead of an inline concept:
```c++
f: <T> (_: T) requires std::regular<T> = { }
```

# Aliases

 * *alias*:
   + *identifier* `:` *templateParameterList*? `type`? `==` *aliasInitializer*
   + *identifier* `:` `namespace` `==` *aliasInitializer*

```c++
main: () = {
    v := 5;
    v2 :== v; // variable alias
    //v2++; // error
    v++;
    [[assert: v == v2]]

    myfunc :== main; // function alias
    view: type == std::string_view;
    N4: namespace == std::literals;
}
```

<link rel="stylesheet" href="style.css">

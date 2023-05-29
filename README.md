Disclaimer: These docs are unofficial and may be inaccurate or incomplete.

Note: Some examples are snipped/adapted from:
https://github.com/hsutter/cppfront

Note: Examples here use C++23 `std::println` instead of `cout`.
If you don't have it, you can use this 1/2 parameter definition:

```c++
std: namespace = {
    println: (a) = std::cout << a << "\n";
    println: (a, b) = std::cout << a << b << "\n";
}
```
You will also need to manually `#include <cassert>` for `assert`.


# Contents

* [Declarations](#declarations)
* [Variables](#variables)
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
   + *identifier* `:` [*type*] `=` *initializer*

*type* can be omitted for type inference (though not at global scope).

```c++
    x: int = 42;
    y := x;
```
A global declaration can be used before the line declaring it.

## C++1

C++1 declarations can be mixed in the same file.

```c++
x := 42;

// C++1
int main() {
    return x;
}
```
A C++2 declaration must not use C++1 declarations internally.

Note: `cppfront` has a `-p` switch to only allow pure C++2.


# Variables

## Uninitialized Variables

Use of an uninitialized variable is statically detected.
Both branches of an `if` statement must
initialize a variable, or neither.
```c++
    x: int;
    y := x; // error
    if f() { x = 1; } // error
```


# Imports

C++23 will support:

```c++
import std;
```
This is implicitly done.


# Types

See also: [User-Defined Types](#user-defined-types).

## Arrays

Use:
* `std::array` for fixed-size arrays.
* `std::vector` for dynamic arrays.
* `std::span` to reference consecutive elements from either.


## Pointers

A pointer to T has type `*T`. Pointer arithmetic is illegal.

### Postfix Operators

Address of and dereference operators are postfix:
```c++
    x: int = 42;
    p: *int = x&;
    y := p*;
```
This makes `p->` obsolete - use `p*.` instead.

To distinguish binary `&` and `*`, use preceeding whitespace.


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
By default, `cppfront` also detects a runtime null dereference.
For example when dereferencing a pointer created in C++1 code.

Instead of using null, use `std::optional<*T>`.


# Expressions

## String Interpolation

A bracketed expression with a trailing `$` inside a string will
evaluate the expression, convert it to string and insert it into the
string.
```c++
    a := 2;
    b: std::optional<int> = 2;
    s: std::string = "a^2 + b = (a * a + b.value())$\n";
    assert(s == "a^2 + b = 6\n");
```

Note: `$` means 'capture' and is also used in [closures](#function-literals)
and [postconditions](#contracts):
<https://github.com/hsutter/cppfront/wiki/Design-note%3A-Capture>

## Bounds Checks

By default, `cppfront` detects out-of-bounds indexing:
```c++
    v: std::vector = (1, 2);
    i := v[2]; // aborts program
```

## `as`
<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-inspect-values.cpp2>

## `is`

Test type of expression:
```c++
    assert(5 is int);
    assert(!(5 is long));
```

Test value by calling a predicate function:
```c++
less_than: (value) -> _ =
    :(x) -> _ = x < value$; // capture parameter

test_int: (i: int) = {
    if i is (less_than(20)) { // calls `less_than(20)(i)`
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

## `inspect`

```c++
    i := 15;

    std::println(inspect i -> std::string {
        is (less_than(10)) = "i less than 10";
        is (in(11,20)) = "i is between 11 and 20";
        is _ = "i is out of our interest";
    });
```
<https://github.com/hsutter/cppfront/blob/main/regression-tests/pure2-inspect-expression-in-generic-function-multiple-types.cpp2>


# Statements

## `if`

* *ifStatement*:
  + `if` [`constexpr`] *expression* *blockStatement* [`else` *blockStatement*]


## `while`

* *whileStatement*:
  + [*parameterList*] `while` *expression* [`next` *expression*] *blockStatement*

If `next` is present, its expression will be evaluated at the
end of each loop iteration.

```c++
(copy i := 0) while i < 3 next i++ {
    std::println(i);
}
```

## `for`

* *forStatement*:
  + [*parameterList*] `for` *expression* [`next` *expression*] `do` `(` *parameter* `)` *statement*

The first *expression* must be a range.
*[parameter](#functions)* is initialized from each element of the
range. The parameter type is inferred. 
*parameter* can have `inout` *[parameterStorage](#parameter-passing)*.

```c++
vec: std::vector<int> = (1, 2, 3);
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

 * *functionType*:
   + `(` [*parameterTypes*] `)` `->` *returnType*

*parameterTypes* is a comma-separated list of types, which can be empty.
E.g. `(int, float) -> bool`.

 * *functionDeclaration*:
   + [*identifier*] `:` *parameterList* [`->` *returnSpec*] [[*contracts*](#contracts)]
 * *parameterList*:
   + `(` *parameter** `)`

Function declarations extend the [declaration form](#declarations).
Each parameter must have an identifier:

 * *parameter*:
   + [*[parameterStorage](#parameter-passing)*] *identifier* `:` *type*. 

If `-> returnSpec` is missing, the function returns nothing (like Cpp1 `void`). 
The return type can be inferred from the initializer by using `-> _`.

See also [Template Functions](#template-functions).


## functionInitializer

 * *functionInitializer*:
   + (*expression* `;` | *statement*)

A function is initialized from a statement or an expression.
If the function has *returnSpec*, the expression form implies a `return` statement.

```c++
d: (i: int) = { std::println(i); }
e: (i: int) = std::println(i); // same
```
```c++
f: (i: int) -> int = { return i; }
g: (i: int) -> int = i; // same
```

## returnSpec

 * *returnSpec*:
   + [*returnStorage*] (*returnType* | *parameterList*)

When a return parameter is declared, it must be named. 
Each parameter must be initialized in the function body.

```c++
f: () -> (i: int, s: std::string) = {
    i = 10;
    s = "hi";
}

int main() {
    auto [a,b] = f(); // C++1 structured binding, no equivalent yet
    assert(a == 10);
    assert(b == "hi");
}
```

## `main`

* *mainFunction*:
  + `main` `:` `(` [`args`] `)` [`->` `int`] = *functionInitializer*

If `args` is declared, it is a `std::vector<string_view>`.


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
    assert(z == "hi");
    assert(x == "");
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

A literal is declared like a named function, omitting the leading identifier.
A literal can capture variables:

```c++
y: std::string = "\n";
callback := :(x) = { std::print(x, y&$*); };
```
`y&$*` means dereference the captured address of `y`.


## Template Functions

A template function declaration can have template parameters:

 * *functionTemplate*:
   + [*identifier*] `:` [*templateParameterList*] *parameterList* [`->` *returnSpec*] [`requires` *constExpression*]

E.g. `size:<T> () = sizeof(T);`

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
        std::println("    data: (data)$, more: (more)$");
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


## Type Templates

 * *typeTemplate*:
   + [*identifier*] `:` [*templateParameterList*] `type` [`requires` *constExpression*]


# Templates

 * *templateParameterList*:
   + `<` *templateParameters* `>`
 * *templateParameter*
   + *identifier* [`:` `type`]
   + *identifier* `:` *type*

For now use `requires` instead of concepts:
```c++
f: <T> (_: T) requires std::regular<T> = { }
```

# Aliases

 * *alias*:
   + *identifier* `:` [*templateParameterList*] [`type`] `==` *aliasInitializer*
   + *identifier* `:` `namespace` `==` *aliasInitializer*

```c++
main: () = {
    v := 5;
    v2 :== v; // variable alias
    //v2++; // error
    v++;
    assert(v == v2);

    myfunc :== main; // function alias
    view: type == std::string_view;
    N4: namespace == std::literals;
}
```

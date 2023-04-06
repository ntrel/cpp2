Disclaimer: These docs are unofficial, may be out of date, may be inaccurate.
Note: Examples here use C++23 `std::print` and `println` instead of `cout`.

# Contents

* [Declarations](#declarations)
* [Variables](#variables)
* [Types](#types)
* [Statements](#statements)
* [Functions](#functions)
* [Expressions](#expressions)
* [Inherited C++ features](#inherited-c-features)


# Declarations

These are of the form `identifier: Type = initializer`. `Type` can be
omitted for type inference.
```c++
    x: int = 42;
    y := x;
```
A global declaration can be used before the line declaring it.

## C++

C++ declarations can be mixed in the same file.

```c++
x := 42;

// C++
int main() {
    return x;
}
```

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


# Types

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
Note: `cppfront` has a `-n` switch to detect pointer dereferences.


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

## `as`
<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-inspect-values.cpp2>

## `is`

```c++
test_generic: ( x ) = {
    msg: std::string = typeid(x).name();
    msg += " is int? ";
    print( msg, x is int );
}
```
Assuming `less_than` and `in` are defined as `constexpr`:
```c++
    if i is (less_than(20)) {
        println("less than 20");
    }

    if i is (in(10,30)) {
        println("i is between 10 and 30");
    }
```

## `inspect`

```c++
    i := 15;

    println(inspect i -> std::string {
        is (less_than(10)) = "i less than 10";
        is (in(11,20)) = "i is between 11 and 20";
        is _ = "i is out of our interest";
    });
```
<https://github.com/hsutter/cppfront/blob/main/regression-tests/pure2-inspect-expression-in-generic-function-multiple-types.cpp2>


# Statements

## `for`

* `for` *expression* `do` *functionLiteral*
* `for` *expression* `next` *expression* `do` *functionLiteral*

```c++
vec: std::vector<int> = (1, 2, 3);
for vec do :(i) =
    println(i);
```
<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-intro-for-with-counter-include-last.cpp2>

# Functions

A function has type `(ParameterTypes) -> Type`. `ParameterTypes` is a
comma-separated list, which can be empty.
`Type` is the return type.

Function declarations follow the [declaration form](#declarations),
except each parameter must have an identifier using the form
`identifier: Type`.

A function is initialized from a statement or an expression.
For the latter, `return` is implied.

```c++
f: (i: int) -> int = return i;
g: (i: int) -> int = i; // same
```
See also [Template Functions](#template-functions).


## Returning a Tuple

A function returning a tuple with named members must initialize
each of those members.
```c++
f: () -> (i: int, s: std::string) = {
    i = 10;
    s = "hi";
}

main: () -> int = {
    auto [a,b] = f(); // C++ structured binding
    assert(a == 10);
    assert(b == "hi");
}
```

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

* in - default
* out - also used for construction
* inout
* move
* copy
* forward

<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-parameter-passing.cpp2>

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
callback := :(x) = { print(x, y&$*); };
```
`y&$*` means dereference the captured address of `y`.


## Template Functions

When a function parameter type is `_`, this implies a template with a
corresponding type parameter.

A template function parameter can also be just `identifier`.

## Operator Overloading

<https://github.com/hsutter/cppfront/wiki/Cpp2:-operator=,-this-&-that>



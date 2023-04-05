Note: Examples here use C++23 `std::print` and `println` instead of `cout`.

# Declarations

These are of the form `identifer: Type = initializer`. `Type` can be
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


# Expressions

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

## Bounds Checks

# Constructs

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

<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-parameter-passing.cpp2>

## Contracts

<https://github.com/hsutter/cppfront/blob/main/regression-tests/mixed-postexpression-with-capture.cpp2>

## Function Literals

## Template Functions

When the parameter type is `_`, this implies a template function with
inferred parameter type.

### `is`

```c++
test_generic: ( x: _ ) = {
    msg: std::string = typeid(x).name();
    msg += " is int? ";
    print( msg, x is int );
}
```

### `inspect`

<https://github.com/hsutter/cppfront/blob/main/regression-tests/pure2-inspect-expression-in-generic-function-multiple-types.cpp2>

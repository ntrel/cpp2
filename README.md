# Declarations

These are of the form `identifer: Type = initializer`. `Type` can be
omitted for type inference.
```c++
    x: int = 42;
    y := x;
```

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

## Uninitialized variables

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

### `new<T>` gives `unique_ptr`

```c++
    p: *int = new<int>;
```

### Initialization to null

Initialization to null is an error:
```c++
    q: *int = nullptr; // error
```

### Postfix operators

Address of and dereference operators are postfix:
```c++
    x: int = 42;
    p: *int = x&;
    y := p*;
```

# Functions

A function has type `(ParameterTypes) -> Type`. `ParameterTypes` is a
comma-separated list, which can be empty.
`Type` is the return type.

Function declarations follow the [declaration form](#declarations),
except each parameter must have an identifier using the form
`identifier: Type`.

A function is initialized from a statement, or an expression.

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

If a method doesn't exist with method call syntax, but there is a
function whose first parameter takes the type of the 'object' expression,
then that function is called instead.
```c++
main: () -> int = {
    // call C functions
    myfile := fopen("xyzzy", "w");
    myfile.fprintf("Hello %d!", 2); // fprintf(myfile, "Hello %d!", 2)
    myfile.fclose();
}
```


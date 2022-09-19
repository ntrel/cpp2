# Declarations

These are of the form `identifer: Type = initializer`. `Type` can be
omitted for type inference.
```c++
    x: int = 42;
    y := x;
```

## C++

C++ declarations can be mixed in the same file. `cppfront` has a `-p`
switch to only allow pure C++2.

```c++
x := 42;

// C++
int main() {
    return x;
}
```

# Expressions

## Uninitialized variables

This is statically detected. Both branches of an `if` statement must
initialize a variable, or neither.
```c++
    x: int;
    y := x; // error
    if f() { x = 1; } // error
```

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

A function has type `Type -> Type`, where the first `Type` must be a
tuple, which can be empty. That tuple declares parameter types.
The second `Type` declares the return type.

Function declarations follow the [declaration form](#declarations),
except each parameter must have an identifier using the form
`identifier: Type`.

## Returning a Tuple

A function returning a tuple with named members must initialize
each of those members.
```c++
f: () -> (i: int, s: std::string) = {
    i = 10;
    s = "hi";
}

main: () -> int = {
    auto [a,b] = f();
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
    myfile := fopen("xyzzy", "w");
    myfile.fprintf("Hello with UFCS!"); // fprintf(myfile, "Hello with UFCS!")
    myfile.fclose();
}
```


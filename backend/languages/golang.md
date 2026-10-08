## Terminology

1. Compile Time --> Compiler reads the actual code and turns it into a .exe file. Our program is not running yet. 
2. Runtime --> Program is actually executing. 

Catching thing's at compile time is better, because it happens on our machine and not on the user's. 


## Key Points
### 1. Golang is statically typed

**Definition**: Each variable has a *type* that is known and checked at compile time (**before the program runs**). 

```go
var age int = 30
age = 31 // perfectly okay
age = "thirty" // COMPILE ERROR: cannot use "thirty" (untyped string constant) as int value
```

That compile error appears when we run `go build` and compile the program. 

> NOTE: `:=` is inference. The compiler basically infers the type. 


### 2. Golang is strongly typed

**Definition**: Golang will refuse to convert one type into another (will not perform any implicit conversions). 

```go
s := "5"
n := 3
result := s + n  // COMPILE ERROR: invalid operation: mismatched types string and int
```

### 3. Golang is compiled.

`go build` will straight away produce the machine code. Golang doesn't use stuff like interpreters or JIT, making it very fast. 


## Key Concepts

### Package

A folder that contains a collection of `go` files. 


### Modules

A collection of these packages is called a module. Whenever we are initializing a new `go` project, we are essentially initializing a new module. 


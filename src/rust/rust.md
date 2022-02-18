# Rust
['The Rust Programming Lanauge'](https://doc.rust-lang.org/book/title-page.html) is an excellent introduction. This section includes slightly extra detail in a different order. 

# Basics
We start with the most basic variables and functions.

## Variable and Constant
Variables are introduced with the 'let' keyword. They are immutable by default (though they can be made mutable):
```rust
// Declare a variable of type signed 16-bit.
// The value 10 is permanantly bound to the name 'count'
let count: i16 = 10;
// Error: "cannot assign twice to immutable variable `count`"
count = 12;
```
Constants must have a type specification and are always immutable:
```rust
// A_BYTE has type unsigned 8-bit
const A_BYTE: u8 = 8;
// Error: cannot assign to this expression (because it's constant)
A_BYTE = 7
```

## Functions
The classic entry point 'main' and a simple addition function are declared as follows:
```rust
fn main() {
    println!("Hello, world!");

    another_function();
}

fn another_function() {
    println!("Another function.");
}

```
We make use of the 'println!' macro. It formats a string using '{}' as a placeholder.
```rust
fn main() {
    let x = add(5, 3);

    println!("The value of x is: {}", x);
}

fn add(x: i32, y: i32) -> i32 {
    return x + y;
}

```
A quick look at the last line of a function. A function's return value is the tail expression on the last line:
```rust,noplayground
fn foo() -> i32 {
    // This is an expression at the end of a function:
    3 + 4 // Tail expression
}

fn foobad() -> i32 {
    // But this is a statement. Statements return the unit type '()' which indicates the absence of any value.
    3 + 4; // Error, you cannot end with a statement - it returns nothing
}

fn foobar() -> i32 {
    // 'return' generates a value from a statement. So '3 + 4' is same as 'return 3 + 4;'
    // It can be used for early exit before the tail expression
    if (return_early)
        return 2 + 3;
    
    3 + 4 // or 'return 3 + 4';
}
```

## Variable scope
Scope in Rust is similar to other languages. A variable is 'valid' when it's in scope and 'not valid' when it's out of scope. 
```rust
fn main() {
    // 'name' is not valid.

    { // Start of scope: 'name' is not valid (not yet declared).
 
        name: String = String::from("Kitt Peak"); // name is now 'valid'.
 
    } // End of scope: name is not valid anymore

    // Error: cannot find value 'name' in this scope
    println!("{}", name); 
}
```
That's easy enough. But now let's see how the heap allocator is utilised as scope changes:
```rust
fn main() {
    // 'name' does not exist on the heap

    {
        name: String = String::from("Kitt's Peak");
        // Rust has asked Allocator to reserve memory on the heap for 'name'

    } //Rust calls name.drop(). This returns the reserved memory to the Allocator where it can be reused. 

    // 'name' does not exist on the heap
}
```
# Ownership
We already know Rust talks about a value being bound to a variable. In reverse, Rust talks says a variable OWNS the memory location containing the value. The memory location may be within the heap or the stack. Rust insists that only one variable can ever own that memory. 

## Reassignment
TODO: organize under here

## Ownership on the stack
When a new scope is entered, the compiler creates a new stack frame. On the other hand, when the scope is left, the compiler destroys the frame. If within that scope Rust can figure out the size of a value at compile time, it can pre-allocate stack space in which it can place that value. It does this because operations (e.g. copying values between locations or obliterating stack frames) are very fast. In Rust, values of known size have 'simple types' such as i8, u32, bool or char. By contrast, variables of unknown size are kept on the slower but more flexible heap.

Here's an example:
```rust,noplaygroud
fn main() {// Compiler sees new scope and type of known size. Compiler allocates stack frame and reserves slot for a u8.
        
        // 'age' is the owner of the slot
        let age: u8 = 100; // Value moved into slot.
    
}// 'age' exits scope and compiler obliterates stack frame
```
Notice, the owner of the slot is called 'age'. When the stack frame is destroyed then 'age' goes out of scope. 

What happens if one variable copies the value of another? Here's an example:
```rust
fn main() { // main stack frame has slots for both 'age' and 'mosha'.
    let age: u8 = 100;
    // Value of 'age' is copied to 'mosha' slot on stack
    let mosha: u8 = age;
    println!("{}", mosha);
    // Only 'mosha' slot is updated
    let mosha = mosha + 1;
    println!("{}", mosha);
    println!("{}", age);
}
```
This shows the Rust compiler has created two stack locations: one for 'age' and one for 'mosha'. Think of the basic rule, a value can have only one owner. Simple types keep to this rule by copying values so although mosha and age have the same value, they store these values in different locations which they separately own.

### Function parameter
The function call site is almost the same as the above. In this case there is a new stack frame (foo). The value of 'age' will be copied into the slot for 'mosha'. 
```rust
fn main() { //main stack frame created with slot for 'age', but this time no slot for 'mosha'.
    let age = 100;
    // foo call site creates new frame. Value in 'age' slot copied to 'mosha' slot in foo stack
    foo(age);
    println!("{}", age)
}

fn foo(mosha: u8) { // foo stack frame created with it's own slot for 'mosha'.
    let mosha = mosha + 1;
    println!("{}", mosha);
}
```
Once again, there is only one owner for each value.

### Function return value
This is no different to a function parameter. Once gain, when using simple types, the compiler reserves space within the call sites stack frame for the return value. This way, when the callee's stack is obliterated, the return value is still accessible. Hence the return value is ready to be copied into a slot owned by a capture variable:
```rust
fn main() {
    let age = 100;
    println!("{}", age);
    // Copy value from return slot to 'older' slot
    let older = foo(age);
    println!("{}", older);
}

fn foo(mosha: u8) -> u8 { // foo stack frame
    // 'mosha' contains copy of call site argument
    mosha + 1
}// Return value copied to call site slot. Frame obliterated.
```

## Local scope
We can assume local scope results in a new stack frame. But, FYI what actually happens is described below:
```rust
fn main() { // main stack frame created. Space reserved for 'age' and 'mosha' values.
    // Record value at location for 'age'.
    let age: u8 = 100;
    { // In reality, NO new stack frame is created.
        // Value moved between main stack frame locations for 'age' and 'mosha'.
        let mosha: u8 = age;
        let mosha = mosha + 1;
        println!("{}", mosha);
    } // Compiler will not read mosha after this point. But it will call drop if required.
    // Age's stack location was always unchanged
    println!("{}", age);
} // main stack frame obliterated.
```

## Variables in memory
Every time a function is entered, the program stack is extended with uninitialized memory. The compiler is able to calculate the size of t
A variable owns a slot within the local stack frame

## Allocators in Rust
See [Building an allocator for a Rust OS](https://os.phil-opp.com/allocator-designs/)

An Allocator is an inteface to heap memory. When your code creates an object, it will ask the allocator to reserve unused memory for your use. When you're finished your code returns the memory to the allocator where it can be reused. Rust is different from C or C++ where it is literally the code YOU write that requests and returns memory to the allocator. Instead Rust  will request memory from the allocator on your behalf when your code creates an object. It will also return memory to the allocator when Rust decides that memory is no longer required.


# Quick peek at the current memory model (ver 1.58 as of this writing)
Bear in mind Rust (quite deliberately) has no stable internal calling convention or ABI defined https://github.com/rust-lang/rfcs/issues/600

We're using [the wonderful](https://godbolt.org/), rustc 1.58 with compiler optimizations turned off '--codegen opt-level=0'. Keep reminding yourself that optimizations can start inlining function calls and other wonders.

The point here is that the 'stack model' we've been using is only an approximation...only a quick and convenient way to get to grips with things. Usually, the real code will make use of inlining, jump instructions and registers for greater efficiency. 

Here's the function:
```rust
pub fn aging(age: u16) -> u16 {
    let mosha: u16 = age + 1;
    mosha
}

pub fn main() {
    let age: u16 = 65535;
    let result: u16 = aging(age);
    println!("{}", result);
}
```

The main() call site looks like this:
```x86asm
example::main:
  sub rsp, 104
  ; argument is placed in edi register
  mov edi, 65535
  call qword ptr [rip + example::aging@GOTPCREL]
  ; result is in ax register and moved into local variable 'result'
  mov word ptr [rsp + 30], ax
```

And here's the assembly so we can see what happens on the stack (again, this is unoptimized):
```x86asm
; Call site has pushed 8 byte return address on stack eg:
example::aging:
; So push 8 byte dummy to align with 16 byte boundary. Essentially 'sub rsp 8'
  push rax
  ;Copy 'age' into ax register and increment
  mov ax, di
  add ax, 1
  
  ;Checks if the increment caused an overlow
  mov word ptr [rsp + 6], ax ; temporarily hold the result on stack
  setb al
  test al, 1
  jne .LBB0_2 ; "attempt to add with overflow"
  mov ax, word ptr [rsp + 6] ; No overflow, so restore the result

  ; Delete the 8 byte dummy data leaving return address on stack
  pop rcx
  ret
.LBB0_2:
  lea rdi, [rip + str.0]
  lea rdx, [rip + .L__unnamed_1]
  mov rax, qword ptr [rip + core::panicking::panic@GOTPCREL]
  mov esi, 28
  call rax
  ud2
```

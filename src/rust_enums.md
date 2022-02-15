# Enums
Step by step introcution to enums in Rust.

## Why?
You want to model different types of single selection 'pick list' for users of your API. Here are some examples:
* Your 'reminder' API allows client code to select a single day of week {Monday, Tuesday, Wednesday,...}.
* Your 'reminder' API also allows client code to select a single month {January, February, March,...}
* Your GUI framework API signals client code when user events have occurred {Paste, MouseClick, KeyPress, Draw,...}
* Some calls in your API return one of {Success, Failure} to calling code. This is the basis for Rust's Result enum.
* Some calls in your API return one of {Something, Nothing} to avoid transmitting null values. This is the basis for Rust's Option enum.
* Your 'logging' API allows a call site to specify a trace level from one of {DEBUG, INFO, ERROR, WARN}
* Your drawing API sets background colours using one of a list {Red, Blue, Green, White, Black,...};

It's worth noting that none of the above are 'universal' values, external constraints or limits of your system. These are modeled by constants (e.g. Pi, maximum number of file handles, maximum size of integer, platform...). There is no list and no selection involved, just a value that is fixed while your software runs. However, what is constant for an enum is the number of items in the list itself.

### Type Safety
In the above examples, each pick list is distinct. A colour is not a day is not a month. In other words, they are different types of list. Consider the issue with the following snippet:
```rust,noplayground
// Reminder API has two methods
fn setDay(day: &str);
fn setMonth(month: &str);

// This is bound to happen...
x: &str = "Monday";
y: &str = "January";
// Sometime later...
setDay(y);
setMonth(x);
```
The problem is that the compiler cannot distinguish between each list and its items. Fortunately, each Rust enum is a distinct type of list. For example:
```rust,noplayground
enum DayOfWeek {
    // list items will go here
}

enum MonthOfYear {
    // list items will go here
}
```
We can now declare a variable of either type and we can define a value of either type. But the compiler will prevent a DayOfWeek value (e.g. "Monday") being assigned to a MonthOfYear variable.

## Enum variants
Each item in the list of possible values is called a *variant*. Variants can have different structures so let's start with the simplest, fieldless variants.

### Fieldless variant
Let's expand our DayOfWeek type to include some actual items:
```rust
enum DayOfWeek {
    Monday,
    Tuesday,
}
```
'Monday' is an enum variant as is 'Tuesday'. The variants shown here are called 'fieldless'. That's because, in contrast to other variant types (e.g. structure or tuple), the item has no fields (members).

How do we actually use an enum? We define (declare and initialize) as follows:
```rust
enum DayOfWeek {
    Monday,
    Tuesday,
    Wednesday,
}
// Declare a variable of type DayOfWeek and initialize it with a value:
let d1: DayOfWeek = DayOfWeek::Monday;

// Use enum type to delcare a function parameter:
fn log(d: DayOfWeek) {
    // Match a supplied argument against each possible variant:
    match d {
        DayOfWeek::Monday => println!("First day of week"),
        DayOfWeek::Tuesday => println!("Second day of week"),
        DayOfWeek::Wednesday => println!("Third day of week"),
    }
}

log(d1);
```

### Tuple variant
TODO: explain why a list item can contain values.

This variant allows an item to have unnamed fields. The item is defined using a tuple constructor:
```rust
enum GUIEvent {
    KeyPress(key: String),
}
```

### Struct variant
Item can have named fields which are defined using a struct constructor:
```rust
enum GUIEvent {
    MouseClick {x: i32, y: i32},
}
```

### Discriminant
Each variant is implicitly assigned an integer value ('isize') under the covers. This integer is used by the Rust compiler to ascertain which variant an enum instance holds. We can import a function called 'discriminant' which reveals that internal value:
```rust
use std::mem;

enum DayOfWeek {
        Monday,
        Tuesday,
    }
    println!("Monday variant was implicitly assigned {:?}", 
            mem::discriminant(&DayOfWeek::Monday));
    println!("Tuesday variant was implicitly assigned {:?}", 
            mem::discriminant(&DayOfWeek::Tuesday));
```
TODO: You can also cast to i32 'as i32'.

### Explicit discriminant
If your enum contains only fieldless variants you can explicitly set the value of the discriminant:
```rust
use std::mem;

enum DayOfWeek {
        Monday = 2,
        Tuesday,
        Wednesday = 0x1ABC,
        Thursday,
    }
    
    println!("Monday variant explicitly assigned {:?}", mem::discriminant(&DayOfWeek::Monday));
    println!("Tuesday variant implicitly assigned {:?}", mem::discriminant(&DayOfWeek::Tuesday));
    println!("Wednesday variant explicitly assigned  {:#X?}", mem::discriminant(&DayOfWeek::Wednesday));
    println!("Thursday variant implicitly assigned  {:#X?}", mem::discriminant(&DayOfWeek::Thursday));
```
Notice how implicit discriminants are always one more than the last enum item.

Though you may not use explicit descriminants that often they can be used to assign values used as bit flags (see Rust's bitmask) or perhaps the hex value of a small RGB colour swatch.
They can also be assigned when mapping between enums in different libraries e.g. nix wraps libc:
```rust,noplayground
pub enum Errno {
        UnknownErrno    = 0,
        EPERM           = libc::EPERM,
        ENOENT          = libc::ENOENT,
        ESRCH           = libc::ESRCH,
```

Finally, remember that once an enum contains more 'complex' items such as structs or tuples you cannot assign explicit discriminants.

# Defining (declare and initialize) a variable of type enum.
```rust
enum DayOfWeek {
    Monday,
}
let d1: DayOfWeek = DayOfWeek::Monday;
// WARNING: mdbook doesn't like this - but it should work?
// Bring enum type into scope so you don't have to keep initializing variables with 'DayOfWeek::Monday'
use DayOfWeek::Monday;
let d2: DayOfWeek = Monday;
```
### Scoping enum items
The following didn't run within mdbook, but it works elsewhere.

In the above examples you have to explicitly declare the enum type when initializing a variable. 
```rust,noplayground
// Enum item must be fully declared by initializer
let d: DayOfWeek = DayOfWeek::Monday;
```
To avoid the extra typing on the RHS you can bring enum items into scope with the 'use' statement:
```rust,noplayground
// scope single enum item
use DayOfWeek::Monday;
let d: DayOfWeek = Monday;

// scope multiple items
use DayOfWeek::{Monday, Tuesday};
let d: DayOfWeek = Tuesday;

// scope all enum items
use DayOfWeek::*;
let d: DayOfWeek = Saturday;
```
### Prelude enums
The Rust prelude is a list of things Rust automatically imports into every Rust program. It includes enums called Option and Result. That's why you wont need a 'use' statement to import them and you can immediately write statements such as:
```rust,noplayground
let r: Result<str, i32> = Ok("yay");
let o: Option<str> = Some("some value");
```
No need to worry about the unusual syntax for now. All you need to appreciate is that Rust has automatically imported some enum types and so you can use their variants without 'use' statements.

## Formally...
An enumerated type is a *nominal, heterogeneous, disjoint union*. 

Most languages use a mix of nominal and structural type systems. A nominal type system will always consider two types different if they differ by name (but might have the same structure). In Rust enums are nominal because Rust only compares two enums by type and, for example, will not allow the initialization of 'a' by values of type 'B' even though they have the same structure.
```rust
enum A {
        X(i32)
    }
    enum B {
        X(i32)
    }

    let a: A = B::X(32);
```
An enum is a heterogenous union because it allows variants of different types. It is disjoint because only one variant can be assigned to a variable.

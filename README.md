# rust-recipes-cheatsheet
Cheatsheet of rust recipes from a Rust Programming Recipes course


## Stack vs Heap

- `Stack`: Just the vars that exist only during a function
- `Heap`: Memory that's generally available. Need to free it before ending program.

## Strings

- `String`: Dynamic string heap type. *Acts as a pointer. Use when I need owned string data (passing to other threads, building at runtime)*
- `str`: immutable seq of bytes of dynamic/unknown length in memory. Commonly ref by `&str`, e.g. a ptr to a slice of data that can be anywhere. *Use when I only need a view of a string.*
- `&'static str`: e.g. "foo" string literal, hardcoded into executable, loaded into mem when program runs.

- Trait `AsRef<str>` : if S (generic var) is already a string, then it returns pointer to itself, but if it's already a pointer to str, then it doesn't do anything (no op). Has method `.as_ref()` *Saves me from implementing fns for both strings and slices*
- `format!("a {}", n)`: returns string, concatenates

## Return Handling

### Result: Ok vs Error

    pub enum Result<T,E> {
    	Ok(T),
    	Err(E),
    }

### ? Shorthand

*? applies to a `Result` value, and if it was an `Ok`, it **unwraps** and gives the inner value (Directly uses the into() method), or it returns the Err type from the current* function.* 

    // Unwraps a result, or propagates an error (returns the `into` of an Error)
    // ? needs the From trait implementation for the error type you are converting
    Ok(serde_json::from_str(&std::fs::read_to_string(fname)?)?)

### Use custom errors for new structs

*To handle proper error handling*

    use serde_derive::*;
    
    #[derive(Debug)]
    pub enum TransactionError {
        LoadError(std::io::Error),
        ParseError(serde_json::Error),
    }
    
    impl From<std::io::Error> for TransactionError {
        // we have to impl this required method
                                    // Self: the type we are working with right now, in this case its TransactionError
        fn from(e: std::io::Error) -> Self {
            TransactionError::LoadError(e)
        }
    }
    
    #[derive(Deserialize, Serialize, Debug)]
    pub struct Transaction { ... }

### Option: Some vs None

    pub enum Option<T> {
    	Some(T),
    	None,
    }

### Result <> Option

    // Result into Option
    let t = get_result.ok()?;
    
    // Option into Result
    // ok_or converts None into Error
    // ? gets it into the Err type we want for our function
    let t = get_option.ok_or("An Error message if the Option is None")?;

### Refactor error handling

1. Put all error traits & implementations in `error.rs`; Put all functions in `lib.rs`
2. Create a lib in cargo.toml, point [main.rs](http://main.rs) to library crate  `extern crate lib_name`

    # Allows us to have both a library and an application
    [lib]     # library name, path, etc
    [[bin]]   # binary target name, path etc. Can have multiple binaries!

3. Inside `[lib.rs](http://lib.rs)` import error.rs with `mod error`  `use error::...`

**Failure Trait:** *Better failure handling, without having to create my own error type*

[failure](https://rust-lang-nursery.github.io/failure/)

- When using it to write a library: create a new Error type
- When using it to write an application: have all functions that can fail return failure::Error

## Iterators

Important iterating traits: 

- `Iterator`: Requires Item type, `.next()`
- `IntoIterator`: Key trait that allows for loop to work, for things that doesn't naturally iterate over themselves
- `a_vec.into_iter()`: turns a vector into an iterator

## Generics

*Generic Type, Functions & Trait imps*

    // TRAITS
    pub struct Thing<T> { foo: T, ...}...
    impl <T> Thing<T> {...}
    impl <T> TraitA for Thing<T> {...}
    
    // Limit generics to certain Traits & provide addn traits it needs
    impl <T:TraitA + TraitB> ...
    // Or
    impl <T> ... where T:SomeTrait {...}
    
    // 1. Refactor the multi-traits by creating a new Trait
    pub trait CombiTrait:TraitA + TraitB {..// any fns ..}
    // 2. Impl CombiTrait for all T that has traitA/B, otherwise get error
    impl <T:TraitA + TraitB> CombiTrait for T {}

    // FUNCTIONS
    // It makes the type available for the function
    fn print_c<I:TraitA>(foo:I) {

## References

    &i32        // a reference
    &'a i32     // a reference with an explicit lifetime
    &'a mut i32 // a mutable reference with an explicit lifetime

## Lifetimes

*Think of lifetimes as a constraint, bounding output to the same lifetime as its input(s)*

- By default, lifetimes exist only within the scope of a function
- Only things relating to references need a lifetime, e.g. struct containing a ref, or strings
- `'static` ensures a lifetime for the entirety of the program

    // FUNCTION output lifetime binds to input variables: 
    // 1. We assign the lifetime 'a' to variables s, t 
    // 2. We constrain the output lifetime to 'a' as well
    // 3. Meaning the output now has to live as long as the smaller of s or t
    fn foo<'a>(s: &'a str; t: &'a &str) -> &'a str {...}
    
    // STRUCTs containing refs, require lifetimes, 
    // To ensure any reference to Foo doesn't outlive reference to x (internal value)
    struct Foo<'a> {	x: &'a i32, }
    impl<'a> Foo<'a> { ... }

    // ELISIONS: i.e. inferences & allowed shorthands
    fn print(s: &str); // elided
    fn print<'a>(s: &'a str); // expanded
    
    // If ONE input lifetime, elided or not, that lifetime is assigned to all elided lifetimes in the return values of that function.
    fn substr(s: &str, until: u32) -> &str; // elided
    fn substr<'a>(s: &'a str, until: u32) -> &'a str; // expanded
    // fn frob(s: &str, t: &str) -> &str; // ILLEGAL, two inputs

## Closures

- `FnOnce`: consumes variables it captures. Moves vars into the closure when its defined. FnOnce can only be called once on the same variables
- `FnMut`: can change env bc it mutably borrows values. *recommended for working with Vecs*
- `Fn`: borrow values from env immutably

    // FnOnce:
    pub fn foo<F>(&mut self, f: F)
    	where F: FnOnce(&mut String), {...}
    
    // FnMut: The function itself is going to change over time
    pub fn edit_all<F>(&mut self, mut f: F)
    	where F: FnMut(&mut String), {...}

## Important Traits

### From vs Into

*Two sides of the same coin, returns the same Type the trait is impl on. `Into` is free after impl `From`*

    let my_string = String::from(my_str);
    let num: Number = int.into();

### Number Comparators

- `PartialOrd`: Allows the > , < , = operators in T
- `std::ops::AddAssign`: Allows arithmetic operations in T
- `Copy`: Allows copy for primitives, rather than just move

## Important Crates

- `Failure`: makes custom error creation easier
- `Itertools`: interleave(), interlace() *for strings* , combines iterators together in woven way
-
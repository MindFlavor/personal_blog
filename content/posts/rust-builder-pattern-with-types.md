---
title: Rust builder pattern with types
date: 2018-07-26
draft: false
description: An implementation of the builder pattern in Rust.
tags: [ "rust", "patterns" ]
cover:
  image: /images/rust-builder-pattern-with-types/cover.jpg
  alternate: something good
---

## Intro

Rust has a very rich type system. It also has move semantics. Using these two features together you can build your APIs using the builder pattern. Let's see an actual example on why you would want to use the builder pattern and how to implement is (well, I'm showing you *a* way to implement it, not necessarily the best one).

### Example

Suppose we have a function that takes a lot of parameters: some of them are mandatory, others are optional. For example:

```rust
fn cook_pasta(
    pasta_type: String,
    pasta_name: Option<String>,
    pasta_length: u64,
    altitude: u64,
    water_type: Option<String>,
) {
    // your code here
}
```

This method has two optional parameters, and three mandatory ones. Using it is ugly: 

```rust
cook_pasta("Penne".to_owned(), None, 100, 300, Some("Salty".to_owned()));
```

The problem here is we have to explicitly tell the method to ignore an optional field (`pasta_name`) and also explicitly specify the other optional field (`water_type`). Being an Option forces us to wrap the optional field with `Some()`. Also positional fields are hard to interpret. For example we have two `u64` fields: without looking at the method signature is really hard to tell what's what. Is 100 referring to the `pasta_length` or to the `altitude`? 
This hampers readability as we have to jump back and forth in our code just to figure out what is happening. 

With the builder pattern we want to achieve something like this: 

```rust
cook_pasta()
    .with_pasta_type("Penne".to_owned())
    .with_pasta_length(100)
    .with_altitude(300)
    .with_water_type("Salty".to_owned())
    .execute();
```
 
This syntax is really better: we know with a glance what is `pasta_length` (100) and what is `altitude` (300). Also we do not need to wrap the optional fields in `Some()` nor we need to use `None` for optional ones we do not want to pass to the function. 
Readability is way better at the expense of a extra `execute()` method call. But how to achieve it?

## Builder struct

The trick here is to have a `builder` object that we continually move around, adding fields in the process. The `builder` object will own all the method fields. Something like this:

```rust
#[derive(Debug, Clone, Default)]
struct CookPastaBuilder {
    pub pasta_type: String,
    pub pasta_name: Option<String>,
    pub pasta_length: u64,
    pub altitude: u64,
    pub water_type: Option<String>,
}
```

Now out `cook_pasta()` function (we'll call it `cook_pasta2()` to differentiate if from the previous version) just creates a default instance of that structure. 

```rust
fn cook_pasta2() -> CookPastaBuilder {
    CookPastaBuilder::default()
}
```

Our `cook_pasta` code will then be run using the `CookPastaBuilder` defined above:

```rust
impl CookPastaBuilder {
    fn execute(&self) {
        // your code here
    }
}
```

As it is now you can use it like this:

```rust
let mut cpb = cook_pasta2();
cpb.water_type = Some("Penne".to_owned());
cpb.pasta_length = 100;
cpb.altitude = 300;
cpb.water_type = Some("Salty".to_owned());

cpb.execute();
```

Not quite what we want but better than the original method. This solution has two problems: firstly you have no way to enforce the mandatory fields. If you forget to set a mandatory field you will notice it at runtime (bad, bad, bad!). Secondly, the syntax is cumbersome. We'll address the second issue first.

### Moving around

Instead of giving access to inner fields we can expose them with *moving* methods. For example:

```rust
impl CookPastaBuilder {
    fn with_pasta_type(self, pasta_type: String) -> CookPastaBuilder {
        CookPastaBuilder {
            pasta_type,
            pasta_name: self.pasta_name,
            pasta_length: self.pasta_length,
            altitude: self.altitude,
            water_type: self.water_type,
        }
    }
```

Notice three things:

1. We are *consuming* the `self` object. This means we can no longer call methods on it (`self` in the method definition).
2. We construct a new `CookPastaBuilder` copying all the fields from the previous one *except* the field we want to assign (in the example `pasta_type`).
3. We pass the newly generated `CookPastaBuilder` to the caller (it becomes the new owner). This way we can chain these calls as they were on the same object. In reality we are changing `CookPastaBuilder`s at each call.

Also we can get rid of the `Some()` wraps putting it inside the setter function:

```rust
    fn with_water_type(self, water_type: String) -> CookPastaBuilder {
        CookPastaBuilder {
            pasta_type: self.pasta_type,
            pasta_name: self.pasta_name,
            pasta_length: self.pasta_length,
            altitude: self.altitude,
            water_type: Some(water_type),
        }
```    

Now we can do this:

```rust
cook_pasta2()
        .with_pasta_type("Penne".to_owned())
        .with_pasta_length(100)
        .with_altitude(300)
        .with_water_type("Salty".to_owned())
        .execute();
```

Nice! 
But we still have to address the mandatory field check. For example this compiles fine: 

```rust
cook_pasta2().execute();
```

But it's not fine at all (for instance, cooking pasta without actually having the pasta in the first place will not go well).

### Types all around

Suppose now we have two `builder`s instead of one. `CookPastaBuilderWithoutPastaType` and `CookPastaBuilderWithPastaType` (yes, I know, they suck but please bear with me). 

If we define the `execute` method on the `WithPasta` variant only we can make sure no one will be able to call it on the `WithoutPasta` variant. At compile time. Which is good. 

So our logical flow will be: 

1. Call of `cook_pasta2()` will generate a `CookPastaBuilderWithoutPastaType`.
2. Calling `with_pasta_type(..)` will consume the `CookPastaBuilderWithoutPastaType` and return a `CookPastaBuilderWithPastaType`.
3. Now calling `execute()` will work because `CookPastaBuilderWitPastaType` implements the method.

If we were to call `execute()` without calling `with_pasta_type(..)` first we would get a compiler error. 

But, hey, we have just handled one mandatory field. We have three of them! Sure you can come up with something like `CookPastaBuilderWithoutPastaTypeWithoutAltitudeWithoutPastaLength` and *all the permutations* (in this case 12) like `CookPastaBuilderWithPastaTypeWithoutAltitudeWithPastaLength` and `CookPastaBuilderWithoutPastaTypeWithoutAltitudeWithPastaLength`... but there is a better way of doing so in less keystrokes.


### Generics?

We can use generics to achieve a better compromise. We have three mandatory fields:

1. `pasta_type`
2. `pasta_length`
3. `latitude`

We can simulate the *presence* or *absence* of a value with a boolean type. Something like:

```rust
#[derive(Debug, Default)]
pub struct Yes;
#[derive(Debug, Default)]
pub struct No;
```

Our new class becomes:

```rust
struct CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET>
```

These generic types can either be `No` or `Yes` indicating the fact the specific field has been assigned. For example setting the `pasta_type` field:

```rust
    fn with_pasta_type(
        self,
        pasta_type: String,
    ) -> CookPastaBuilder<Yes, PASTA_LENGTH_SET, ALTITUDE_SET> {
        CookPastaBuilder {
            pasta_type,
            pasta_name: self.pasta_name,
            pasta_length: self.pasta_length,
            altitude: self.altitude,
            water_type: self.water_type,
        }
    }
```

This function will return a new `CookPastaBuilder` instance with the `PASTA_NAME_SET` set on `Yes` so we know the field has been set. Notice that we are returning `PASTA_LENGHT_SET` and `ALTITUDE_SET` in this case because we do not want to change the underlying type for those fields (they can either be set or not and we won't change that here). 

Doing so for every mandatory field means we end up with a type like this: 

```rust
CookPastaBuilder<Yes, Yes, Yes>
```

This gives us the compile-time guarantee that our caller has set all the mandatory fields. All we need to do is to constrain the execution of the `execute()` function to this specific type: 

```rust
impl CookPastaBuilder<Yes, Yes, Yes> {
    fn execute(&self) {
        // your code here
    }
}
```

## Result

From the client perspective our API is still the same, this invocation works: 

```rust
cook_pasta2()
    .with_pasta_type("Penne".to_owned())
    .with_pasta_length(100)
    .with_altitude(300)
    .with_water_type("Salty".to_owned())
    .execute();
```

But this does not because we forgot to set the mandatory `altitude` parameter:

```rust
cook_pasta2()
    .with_pasta_type("Penne".to_owned())
    .with_pasta_length(100)
    .with_water_type("Salty".to_owned())
    .execute();
```

The error will be like this one:

![](/images/rust-builder-pattern-with-types/00.png)

The `no method named execute found for type CookPastaBuilder<Yes, Yes, No> in the current scope` is not very helpful but at least you catch the error before runtime. At least the generic type names will help you determine what is missing:

![](/images/rust-builder-pattern-with-types/01.png)

That is if you add the necessary `WHERE`s to the struct declaration:

```rust
#[derive(Debug, Clone, Default)]
pub struct CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET>
where
    PASTA_TYPE_SET: ToAssign,
    PASTA_LENGTH_SET: ToAssign,
    ALTITUDE_SET: ToAssign,
{
    pasta_type_set: PhantomData<PASTA_TYPE_SET>,
    pasta_length_set: PhantomData<PASTA_LENGTH_SET>,
    altitude_set: PhantomData<ALTITUDE_SET>,

    pasta_type: String,
    pasta_name: Option<String>,
    pasta_length: u64,
    altitude: u64,
    water_type: Option<String>,
}
```

### Last bit: the Phantoms

This is what we wanted: a builder pattern with elegant syntax, compile time checks. But we are using a *fake* generic, that is a generic not actively used in our code. Rust complains about this so in order to make him happy we add three new fields, one for each generic, of the type `PhantomData`. Do not worry, it won't appear at runtime so there is no cost in adding it (besides the nuisance of having them at all).

The final code if this one. Let me know what you think about it!

```rust
use std::fmt::Debug;
use std::marker::PhantomData;

#[derive(Debug, Default)]
pub struct Yes;
#[derive(Debug, Default)]
pub struct No;

pub trait ToAssign: Debug {}
pub trait Assigned: ToAssign {}
pub trait NotAssigned: ToAssign {}

impl ToAssign for Yes {}
impl ToAssign for No {}

impl Assigned for Yes {}
impl NotAssigned for No {}

pub fn cook_pasta(
    pasta_type: String,
    pasta_name: Option<String>,
    pasta_length: u64,
    altitude: u64,
    water_type: Option<String>,
) {
    // your code here
    println!(
        "cooking pasta! -> {:?}, {:?}, {:?}, {:?}, {:?}",
        pasta_type, pasta_name, pasta_length, altitude, water_type
    );
}

#[derive(Debug, Clone, Default)]
pub struct CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET>
where
    PASTA_TYPE_SET: ToAssign,
    PASTA_LENGTH_SET: ToAssign,
    ALTITUDE_SET: ToAssign,
{
    pasta_type_set: PhantomData<PASTA_TYPE_SET>,
    pasta_length_set: PhantomData<PASTA_LENGTH_SET>,
    altitude_set: PhantomData<ALTITUDE_SET>,

    pasta_type: String,
    pasta_name: Option<String>,
    pasta_length: u64,
    altitude: u64,
    water_type: Option<String>,
}

impl<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET>
    CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET>
where
    PASTA_TYPE_SET: ToAssign,
    PASTA_LENGTH_SET: ToAssign,
    ALTITUDE_SET: ToAssign,
{
    pub fn with_pasta_type(
        self,
        pasta_type: String,
    ) -> CookPastaBuilder<Yes, PASTA_LENGTH_SET, ALTITUDE_SET> {
        CookPastaBuilder {
            pasta_type_set: PhantomData {},
            pasta_length_set: PhantomData {},
            altitude_set: PhantomData {},
            pasta_type,
            pasta_name: self.pasta_name,
            pasta_length: self.pasta_length,
            altitude: self.altitude,
            water_type: self.water_type,
        }
    }

    pub fn with_pasta_name(
        self,
        pasta_name: String,
    ) -> CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET> {
        CookPastaBuilder {
            pasta_type_set: PhantomData {},
            pasta_length_set: PhantomData {},
            altitude_set: PhantomData {},
            pasta_type: self.pasta_type,
            pasta_name: Some(pasta_name),
            pasta_length: self.pasta_length,
            altitude: self.altitude,
            water_type: self.water_type,
        }
    }

    pub fn with_pasta_length(
        self,
        pasta_length: u64,
    ) -> CookPastaBuilder<PASTA_TYPE_SET, Yes, ALTITUDE_SET> {
        CookPastaBuilder {
            pasta_type_set: PhantomData {},
            pasta_length_set: PhantomData {},
            altitude_set: PhantomData {},
            pasta_type: self.pasta_type,
            pasta_name: self.pasta_name,
            pasta_length,
            altitude: self.altitude,
            water_type: self.water_type,
        }
    }

    pub fn with_altitude(
        self,
        altitude: u64,
    ) -> CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, Yes> {
        CookPastaBuilder {
            pasta_type_set: PhantomData {},
            pasta_length_set: PhantomData {},
            altitude_set: PhantomData {},
            pasta_type: self.pasta_type,
            pasta_name: self.pasta_name,
            pasta_length: self.pasta_length,
            altitude,
            water_type: self.water_type,
        }
    }

    pub fn with_water_type(
        self,
        water_type: String,
    ) -> CookPastaBuilder<PASTA_TYPE_SET, PASTA_LENGTH_SET, ALTITUDE_SET> {
        CookPastaBuilder {
            pasta_type_set: PhantomData {},
            pasta_length_set: PhantomData {},
            altitude_set: PhantomData {},
            pasta_type: self.pasta_type,
            pasta_name: self.pasta_name,
            pasta_length: self.pasta_length,
            altitude: self.altitude,
            water_type: Some(water_type),
        }
    }
}

impl CookPastaBuilder<Yes, Yes, Yes> {
    pub fn execute(&self) {
        // your code here
        println!("cooking pasta! -> {:?}", self);
    }
}

pub fn cook_pasta2() -> CookPastaBuilder<No, No, No> {
    CookPastaBuilder::default()
}

fn main() {
    cook_pasta("Penne".to_owned(), None, 100, 300, Some("Salty".to_owned()));

    cook_pasta2()
        .with_pasta_type("Penne".to_owned())
        .with_pasta_length(100)
        .with_water_type("Salty".to_owned())
        .with_altitude(300)
        .execute();
}
```

---

Happy Coding

**Francesco Cogno**


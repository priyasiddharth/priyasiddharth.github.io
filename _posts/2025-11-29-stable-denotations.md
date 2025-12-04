---
layout: post
title: Stable Denotations
date: 2025-12-04 21:01:00
description: Stable denotations implementation in Rust
tags: Rust, Monads, Effects
categories: PL
thumbnail: assets/img/9.jpg
---

# Stable Denotations: Effects as Interactions

An executable version can be found [here](https://github.com/priyasiddharth/stable-denotations).

* Table of contents
{:toc}

## Introduction

This post explores **stable denotations** and **extensible effects** in Rust, following the [okmij blog](https://okmij.org/ftp/Computation/having-effect.html).
Our goal is to write an interpreter for a simple (functional) programming language in an extensible way, i.e., as new features are added existing definitions need not be modified.
We will do this through different experiments and (re)discover Effects and Monads.
After this tutorial, we recommend reading the original to reinforce the concepts and also learn how to incorporate first-class function in the language, which is not covered here.

Our journey will cover
the following:

| Interpreter | Description |
|-------------|-------------|
| I-0         | Traditional definitional interpreter for integers and increment (`Expr`, `Dom`, `eval`). |
| I-1         | Extended interpreter adding booleans, equality, and conditionals by redefining syntax and eval. |
| I-2         | Tagless-final style core language (`EBasic`) with abstract domain and reusable terms like `ttinc`. |
| I-2+        | Compositional extension adding conditionals (`ECond`) without changing existing definitions. |
| I-3         | State-threading interpretation using explicit state transformers (`DomIntState`). |
| I-4         | Effects-as-interactions with request/handler model over computations `Comp<ReqT>`. |
| I-4s        | I-4 extended with a sequencing operator (`seq`) built from `bind` for effectful composition. |
| I-4s+       | I-4s with conditionals reintroduced over the computation monad. |

Let's begin.

## Part 0: The Traditional Approach - Definitional Interpreters

Before we explore tagless-final style, let's see the traditional approach to denotational semantics and understand why it's problematic.

We'll build a **definitional interpreter**: a compositional mapping from expressions to their meanings. This is how denotational semantics has traditionally been formalized.

### Step 1: A Simple Language interprter (I-0)

We start with the simplest language: integer literals, increment, and application.
We refer to an interpreter for this language as I-0.

In Rust, we define the language as an enum `Expr`:


```rust
// Traditional definitional interpreter approach
// Language 1: Just integers and increment

#[derive(Debug)]
enum Expr {
    EInt(i64),
    EInc,
    EApp(Box<Expr>, Box<Expr>),
}

// The semantic domain: what expressions mean
enum Dom {
    DError,
    DInt(i64),
    DBool(bool),
    DFun(Box<dyn FnOnce(Dom) -> Dom>),
}

impl std::fmt::Debug for Dom {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Dom::DError => write!(f, "DError"),
            Dom::DInt(n) => write!(f, "DInt({})", n),
            Dom::DBool(b) => write!(f, "DBool({})", b),
            Dom::DFun(_) => write!(f, "DFun(<function>)"),
        }
    }
}

// The interpreter: compositional mapping from Expr to Dom
fn eval(expr: Expr) -> Dom {
    match expr {
        Expr::EInt(x) => Dom::DInt(x),
        Expr::EInc => Dom::DFun(Box::new(|x| match x {
            Dom::DInt(n) => Dom::DInt(n + 1),
            _ => Dom::DError,
        })),
        Expr::EApp(e1, e2) => {
            let v1 = eval(*e1);
            let v2 = eval(*e2);
            match v1 {
                Dom::DFun(f) => f(v2),
                _ => Dom::DError,
            }
        }
    }
}

// Sample expression: inc (inc 2)
let tinc = Expr::EApp(
    Box::new(Expr::EInc),
    Box::new(Expr::EApp(
        Box::new(Expr::EInc),
        Box::new(Expr::EInt(2))
    ))
);

// Evaluate it
let result = eval(tinc);
println!("Traditional interpreter: inc (inc 2) ->* {:?}", result);
```

    Traditional interpreter: inc (inc 2) ->* DInt(4)


### Step 2: Extending the Language (I-1)

Now we want to add booleans, equality, and conditionals.
This interpreter is called I-1.

In Rust, `enum` and `functions` are not extensible.
Thus, we copy and redefine `Expr` to `ExprExtended` and `eval` to `eval_extended`.


```rust
// Language 2: Extended with conditionals and equality
// We MUST redefine everything!

#[derive(Debug)]
enum ExprExtended {
    // Old constructors (copied!)
    EInt(i64),
    EInc,
    EApp(Box<ExprExtended>, Box<ExprExtended>),
    // New constructors
    EEq,
    EIf(Box<ExprExtended>, Box<ExprExtended>, Box<ExprExtended>),
}

// We MUST rewrite the entire interpreter!
fn eval_extended(expr: ExprExtended) -> Dom {
    match expr {
        // Old cases (copied verbatim!)
        ExprExtended::EInt(x) => Dom::DInt(x),
        ExprExtended::EInc => Dom::DFun(Box::new(|x| match x {
            Dom::DInt(n) => Dom::DInt(n + 1),
            _ => Dom::DError,
        })),
        ExprExtended::EApp(e1, e2) => {
            let v1 = eval_extended(*e1);
            let v2 = eval_extended(*e2);
            match v1 {
                Dom::DFun(f) => f(v2),
                _ => Dom::DError,
            }
        }
        // New cases
        ExprExtended::EEq => Dom::DFun(Box::new(|x| match x {
            Dom::DInt(n1) => Dom::DFun(Box::new(move |y| match y {
                Dom::DInt(n2) => Dom::DBool(n1 == n2),
                _ => Dom::DError,
            })),
            _ => Dom::DError,
        })),
        ExprExtended::EIf(e, et, ef) => {
            let cond = eval_extended(*e);
            match cond {
                Dom::DBool(true) => eval_extended(*et),
                Dom::DBool(false) => eval_extended(*ef),
                _ => Dom::DError,
            }
        }
    }
}

// We MUST rewrite tinc even though it's identical!
// This time we write it inline wherever needed since ExprExtended is a different type

// Sample conditional: if 3 == inc (inc 2) then 10 else inc (inc (inc 2))
let tif = ExprExtended::EIf(
    Box::new(ExprExtended::EApp(
        Box::new(ExprExtended::EApp(
            Box::new(ExprExtended::EEq),
            Box::new(ExprExtended::EInt(3))
        )),
        // inc (inc 2) - written inline
        Box::new(ExprExtended::EApp(
            Box::new(ExprExtended::EInc),
            Box::new(ExprExtended::EApp(
                Box::new(ExprExtended::EInc),
                Box::new(ExprExtended::EInt(2))
            ))
        ))
    )),
    Box::new(ExprExtended::EInt(10)),
    // inc (inc (inc 2)) - different expression
    Box::new(ExprExtended::EApp(
        Box::new(ExprExtended::EInc),
        Box::new(ExprExtended::EApp(
            Box::new(ExprExtended::EInc),
            Box::new(ExprExtended::EApp(
                Box::new(ExprExtended::EInc),
                Box::new(ExprExtended::EInt(2))
            ))
        ))
    ))
);

let result = eval_extended(tif);
println!("Extended interpreter: if 3 == 4 then 10 else 5 ->* {:?}", result);
```

    Extended interpreter: if 3 == 4 then 10 else 5 ->* DInt(5)


Our first extension exposes a problem.
This approach does not allow us to compose existing definitions with new features.
The semantics are **unstable**. 

## Part 1: A Different Approach (I-2)
From okmij:
_Let us go back to the simplest language of integers and increments, and define it differently. Instead of a data type, which then has to be interpreted as `Dom`, let us define a language by directly telling the meaning of each language phrase, and how to make sense of a compound phrase from the meaning of its components_. 

Here we do not delegate evaluation to an `eval` function. Rather each expression definition carries its evaluation semantics.
This interpreter is called I-2.

From okmij:
_For generality, which will come handy later on, we do not fix the domain of denotations to be `Dom`. Rather, we make it variable `d`. All in all, we define a language by a set of definitions, one per syntactic form, each defining the domain mapping for that form. The domain is kept abstract._ 

Thus we define a language as a **trait** in Rust (type class in Haskell):


```rust
// Define the language as a trait - domain is kept abstract
trait EBasic {
    fn int(n: i64) -> Self;
    fn inc() -> Self;
    fn app(e1: Self, e2: Self) -> Self;
}
```

The type is: fn ttinc<D: EBasic>() -> D
Meaning: given an appropriate domain D, ttinc gives the meaning of inc (inc 2) in that domain


```rust
fn ttinc<D: EBasic>() -> D {
    D::app(D::inc(), D::app(D::inc(), D::int(2)))
}
```


Use Dom as defined earlier.
Now we must tell that Dom is 'appropriate' to give meaning to the basic language
Helper for increment lifted to Dom


```rust
fn inc_helper(x: Dom) -> Dom {
    match x {
        Dom::DInt(n) => Dom::DInt(n + 1),
        _ => Dom::DError,
    }
}
```

Now define an instance of EBasicSimple trait for domain `Dom`.


```rust
impl EBasic for Dom {
    fn int(n: i64) -> Self {
        Dom::DInt(n)  // int = DInt
    }
    
    fn inc() -> Self {
        Dom::DFun(Box::new(inc_helper))  // inc = injI succ (lifted successor)
    }
    
    fn app(e1: Self, e2: Self) -> Self {
        match e1 {
            Dom::DFun(f) => f(e2),  // app (DFun f) e2 = f e2
            _ => Dom::DError,       // app _ _ = DError
        }
    }
}

// Test it - now ttinc is reusable!
let result: Dom = ttinc();
println!("Result: inc (inc 2) ->* {:?}", result);
```

    Result: inc (inc 2) ->* DInt(4)


### Adding conditionals compositionally (I-2+)

Now we can add conditionals compositionally without rewriting affecting existing semantics.
Thus, I-2+ is an incremental addition to I-2.


```rust
// Conditionals as a separate trait (domain kept abstract).
trait ECond {
    fn eq() -> Self;              // equality (curried): eq x y -> bool or error
    fn if_(cond: Self, et: Self, ef: Self) -> Self;
}

// A generic term using equality and conditional
fn ttif<D: EBasic + ECond>() -> D {
    // if (3 == inc (inc 2)) then 10 else inc (inc (inc 2))
    let cond = D::app(D::app(D::eq(), D::int(3)), D::app(D::inc(), D::app(D::inc(), D::int(2))));
    let then_branch = D::int(10);
    let else_branch = D::app(D::inc(), D::app(D::inc(), D::app(D::inc(), D::int(2))));
    D::if_(cond, then_branch, else_branch)
}

// Implement ECondSimple for the concrete domain `Dom`
impl ECond for Dom {
    fn eq() -> Self {
        Dom::DFun(Box::new(|x| match x {
            Dom::DInt(n1) => Dom::DFun(Box::new(move |y| match y {
                Dom::DInt(n2) => Dom::DBool(n1 == n2),
                _ => Dom::DError,
            })),
            _ => Dom::DError,
        }))
    }

    fn if_(cond: Self, et: Self, ef: Self) -> Self {
        match cond {
            Dom::DBool(true) => et,
            Dom::DBool(false) => ef,
            _ => Dom::DError,
        }
    }
}

// Test the generic conditional specialized to Dom
let result_if: Dom = ttif();
println!("Demo ttif (specialized to Dom) ->* {:?}", result_if);
```

    Demo ttif (specialized to Dom) ->* DInt(5)


### Adding State (I-3)

Now let's try adding a simple effect to our language: a global mutable memory cell.
This interpreter is I-3.
We define the primitives to access the current state and to change it, returning the new value:


```rust
trait GlobalIntState {
    fn get() -> Self;
    fn put(e: Self) -> Self;
}
```

Now we realize that `get` and `put` need to act on a state. Where will that state be carried?
The solution is to redefine the abstract domain to carry state and define `get` and `put` operations on this domain.


```rust
// Type DomIntState = Int -> (Dom, Int)
// A state transformer: takes current state, returns (value, new_state)
type DomIntState = Box<dyn FnOnce(i64) -> (Dom, i64)>;

impl GlobalIntState for DomIntState {
    // get = \s -> (DInt s, s)
    fn get() -> Self {
        Box::new(|s| (Dom::DInt(s), s))
    }
    
    // put e = \s -> case e s of
    //   (DInt s', _) -> (DInt s', s')
    //   (_, s)       -> (DError, s)
    fn put(e: Self) -> Self {
        Box::new(move |s| {
            match e(s) {
                (Dom::DInt(s_new), _) => (Dom::DInt(s_new), s_new),
                (_, s) => (Dom::DError, s),
            }
        })
    }
}
```

Now we must create an instance of `EBasic` for the domain `DomIntState`.
Note that even though int(n) has nothing to do with state, it must thread state through itself.


```rust
impl EBasic for DomIntState {
    // int x = \s -> (DInt x, s)
    fn int(n: i64) -> Self {
        Box::new(move |s| (Dom::DInt(n), s))  // Thread state even though unused!
    }
    
    // inc = \s -> (DFun(inc_helper), s)
    fn inc() -> Self {
        Box::new(|s| (Dom::DFun(Box::new(inc_helper)), s))  // Thread state!
    }
    
    // app e1 e2 = \s0 -> let (f, s1) = e1 s0
    //                        (x, s2) = e2 s1
    //                    in (app f x, s2)
    fn app(e1: Self, e2: Self) -> Self {
        Box::new(move |s0| {
            let (v1, s1) = e1(s0);
            let (v2, s2) = e2(s1);
            match v1 {
                Dom::DFun(f) => (f(v2), s2),
                _ => (Dom::DError, s2),
            }
        })
    }
}

// Test it: ttinc now means something different!
let result_with_state: DomIntState = ttinc();
let (final_value, final_state) = result_with_state(100);  // Start with state = 100
println!("With state threading: ttinc(100) ->* {:?}, final_state={}", final_value, final_state);
```

    With state threading: ttinc(100) ->* DInt(4), final_state=100


## Part 2: The Key Insight: Effects as Interactions

What have we seen so far?

We started with a simple interpreter **I-0** that we extended with conditionals **I-1**.
However, we had to redefine both the domain and the eval function (unstable denotations).
We then tried a different approach where each expression carried its own meaning in **I-2**. 
This was very useful in incrementally adding conditionals, resulting in **I-2+**, i.e., we got denotional stability on the axis of adding _pure_ features.

Interestingly, when we added a global mutable memory cell, we need to implement `EBasic` again.
Thus denotational stability did not extend to effects.
We will now extend stability to effects.

The solution is to treat effects not as transformations of a semantic domain, but as **interactions** between an expression and its context:

1. **Expressions produce values or make requests**
   - `Done(value)`: A completed computation
   - `Req(request, continuation)`: Ask the context for help, continue when answered

2. **The meaning of `int(2)` never changes** -- it's always `Done(DInt(2))`

3. **The meaning of `app` and `if_` never changes** -- they use `bind` to sequence effects

4. **Only handlers change** -- they interpret requests in different ways

This is what we'll build next! The denotations become **stable**: adding new effects doesn't change existing meanings.


```rust
// Request types
use std::fmt;

#[derive(Debug)]
pub enum StateReq {
    Get,
    Put(i64),
}

#[derive(Debug)]
pub enum ReqT {
    ReqError,
    ReqState(StateReq),
}

impl fmt::Display for ReqT {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ReqT::ReqError => write!(f, "ReqError"),
            ReqT::ReqState(StateReq::Get) => write!(f, "ReqState(Get)"),
            ReqT::ReqState(StateReq::Put(n)) => write!(f, "ReqState(Put({}))", n),
        }
    }
}
```

### Making debugging easier

To allow debugging and inspection of computation in I-3, we create the `DescriptionTree` structure.
It is our **syntax tree for denotations**.
We can print the computation at any stage and inspect the tree.

### Key components:
- **Primitives**: `Int`, `Bool`, `Var` (variables)
- **Lambda abstractions**: `Lam(var, body)`
- **Operations**: `Inc`, `Eq`, `EqArg2` (partially applied equality)
- **Control flow**: `App` (application), `If`
- **State operations**: `Get`, `Put`, `Req`
- **Composition**: `Bind` (monadic bind), `HandleState`


```rust
// Simple global counter for single-threaded Jupyter environment
static mut ABSTRACTION_COUNTER: usize = 0;

fn next_abstraction_id() -> usize {
    unsafe {
        let id = ABSTRACTION_COUNTER;
        ABSTRACTION_COUNTER += 1;
        id
    }
}

// Tree structure for describing computations - formatted lazily on print
#[derive(Clone)]
enum DescriptionTree {
    // Primitives
    Int(i64),
    Bool(bool),
    // Variables
    Var(String),
    // Lambda abstraction: λvar. body
    Lam(String, Box<DescriptionTree>),
    // Operations (with abstraction ID)
    Inc(u64),
    Eq,
    EqArg2(u64, i64),  // (id, first_arg)
    // Control flow
    App(Box<DescriptionTree>, Box<DescriptionTree>),
    If(Box<DescriptionTree>, Box<DescriptionTree>, Box<DescriptionTree>),
    // State
    Get,
    Put(Box<DescriptionTree>),
    Req(Box<DescriptionTree>),  // Req(Put(x))
    // Composition
    Bind(Box<DescriptionTree>, Box<DescriptionTree>),
    #[allow(dead_code)] // Only used in test code
    HandleState(i64, Box<DescriptionTree>),
}

impl DescriptionTree {
    // Helper to create identity: λx. x
    fn identity() -> Self {
        DescriptionTree::Lam("x".to_string(), Box::new(DescriptionTree::Var("x".to_string())))
    }
    
    // Helper to check if this is identity
    fn is_identity(&self) -> bool {
        matches!(self, 
            DescriptionTree::Lam(v, body) 
            if v == "x" && matches!(body.as_ref(), DescriptionTree::Var(bv) if bv == "x")
        )
    }
}

impl fmt::Display for DescriptionTree {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DescriptionTree::Int(n) => write!(f, "int({})", n),
            DescriptionTree::Bool(b) => write!(f, "bool({})", b),
            DescriptionTree::Var(v) => write!(f, "{}", v),
            DescriptionTree::Lam(v, body) => write!(f, "λ{}. {}", v, body),
            DescriptionTree::Inc(id) => write!(f, "λ#{}[λx. x + 1]", id),
            DescriptionTree::Eq => write!(f, "λx. λy. x == y"),
            DescriptionTree::EqArg2(id, n) => write!(f, "λ#{}[λy. {} == y]", id, n),
            DescriptionTree::App(e1, e2) => write!(f, "app({}, {})", e1, e2),
            DescriptionTree::If(c, t, e) => write!(f, "if {}\n  then {}\n  else {}", c, t, e),
            DescriptionTree::Get => write!(f, "get"),
            DescriptionTree::Put(e) => write!(f, "put({})", e),
            DescriptionTree::Req(e) => write!(f, "Req({})", e),
            DescriptionTree::Bind(k1, k2) => write!(f, "bind({}, {})", k1, k2),
            DescriptionTree::HandleState(s, inner) => write!(f, "handleState({}, {})", s, inner),
        }
    }
}

impl fmt::Debug for DescriptionTree {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self)
    }
}
```

### DomC - The Domain of Computation Values

Values in our language can be:
- **Integers** (`DCInt`): the numbers themselves
- **Booleans** (`DCBool`): true/false values  
- **Functions** (`DCFun`): wrapped as `Abstraction` with ID and description

We define the core types together since they're mutually recursive: `Abstraction` contains functions from `DomC` to `Comp`, `DomC` contains `Abstraction`, and `Comp` contains both.

#### Key Design Choices:

**Abstraction wrapper:** Each function is wrapped in an `Abstraction` struct that tracks:
- A unique `id` for debugging and display
- A `kind` (`DescriptionTree`) describing the function body
- The actual Rust closure `f` that implements the function

Note that the following types are mutually recursive:
- Functions (`DCFun`) need to return computations (`Comp`)
- Computations (`Comp`) need to contain values (`DomC`) and continuations (`Abstraction`)
- Abstractions transform values to computations (`DomC -> Comp`)

**Comp structure:** A computation (`Comp`) pairs a `DescriptionTree` (for inspection/debugging) with a `CompInner`:
- `Done(value)`: A completed computation with its final value
- `Req(request, continuation)`: An effect request with a continuation function

This design allows us to:
1. Track and print the structure of computations before running them
2. Separate pure values (`DomC`) from effectful computations (`Comp`)
3. Handle effects uniformly through the request/continuation pattern


```rust
// DomC - domain of computation
pub enum DomC<ReqT> {
    DCInt(i64),
    DCBool(bool),
    // Function from DomC to Comp; used by inc()
    DCFun(Abstraction<ReqT>),
}

// Wrapper for closures with IDs
pub struct Abstraction<ReqT> {
    id: usize,
    kind: DescriptionTree,
    f: Box<dyn FnOnce(DomC<ReqT>) -> Comp<ReqT>>,
}

impl<ReqT> Abstraction<ReqT> {
    fn new<F>(kind: DescriptionTree, f: F) -> Self
    where
        F: FnOnce(DomC<ReqT>) -> Comp<ReqT> + 'static,
    {
        let id = next_abstraction_id();
        Abstraction {
            id,
            kind,
            f: Box::new(f),
        }
    }

    fn call(self, arg: DomC<ReqT>) -> Comp<ReqT> {
        (self.f)(arg)
    }
}

impl<ReqT> fmt::Display for DomC<ReqT> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DomC::DCInt(n) => write!(f, "DCInt({})", n),
            DomC::DCBool(b) => write!(f, "DCBool({})", b),
            // Check if kind already includes λ#id prefix
            DomC::DCFun(c) => match &c.kind {
                // These already have λ#id in their Display
                DescriptionTree::Inc(_) | DescriptionTree::EqArg2(_, _) => {
                    write!(f, "DCFun({})", c.kind)
                }
                // Others need the λ#id prefix
                _ => write!(f, "DCFun(λ#{}[{}])", c.id, c.kind),
            }
        }
    }
}

// Comp holds both the description tree and the computation state
pub struct Comp<ReqT> {
    desc: DescriptionTree,
    inner: CompInner<ReqT>,
}

enum CompInner<ReqT> {
    Done(DomC<ReqT>),
    Req(ReqT, Abstraction<ReqT>),
}

impl<ReqT: fmt::Display> fmt::Debug for CompInner<ReqT> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            CompInner::Done(d) => write!(f, "Done({})", d),
            CompInner::Req(r, k) => write!(f, "Req({}, λ#{}[{}])", r, k.id, k.kind),
        }
    }
}

impl<ReqT> Comp<ReqT> {
    fn done(d: DomC<ReqT>, desc: DescriptionTree) -> Self {
        Comp { desc, inner: CompInner::Done(d) }
    }
    
    fn req(r: ReqT, k: Abstraction<ReqT>, desc: DescriptionTree) -> Self {
        Comp { desc, inner: CompInner::Req(r, k) }
    }
}
```

### Processing Application: The Need for Sequencing

Let's understand what happens when we evaluate `app inc get`.

#### The Challenge

Consider the expression `app inc get`:
- `get` is an **effectful computation** that retrieves the current state
- `inc` is a **function** that increments its argument
- `app` must apply `inc` to the **result** of `get`

But here's the problem: `get` doesn't immediately produce a value! Instead, it produces a computation that will eventually yield a value after interacting with the state handler.

#### What We Need

When evaluating `app inc get`, we need to:
1. **First** evaluate `get` to get the current state value
2. **Then** apply `inc` to that value
3. **Preserve** any effects that `get` might have triggered

This is **sequencing** - we must wait for one computation to complete before continuing with the next.

#### A More Complex Example

Consider: `app inc (app inc get)`
- Evaluate the inner `app inc get`:
    - Get the state (say it's 5)
    - Apply `inc` to get 6
- Apply the outer `inc` to get 7

At each `app`, we must:
- Wait for the left side to produce a function
- Wait for the right side to produce a value  
- Apply the function to the value

#### The Pattern

Every time we have an effectful computation that produces a value we want to use, we need to:
1. **Execute** the computation to get its value
2. **Continue** with the rest of the program using that value
3. **Thread through** any effects the computation generated

This "execute then continue" pattern appears everywhere:
- In `app e1 e2`: execute `e1` to get a function, execute `e2` to get an argument, then apply
- In `if_ cond et ef`: execute `cond` to get a boolean, then execute the appropriate branch
- In `put e`: execute `e` to get a value, then update the state with it

We need a general mechanism to express this "do this computation, then continue with that" pattern. That mechanism is what makes our effects compositional and our denotations stable.

A computation Comp<ReqT> has two shapes:

```rust
enum CompInner<ReqT> {
    Done(DomC<ReqT>),
    Req(ReqT, Abstraction<ReqT>),
}
```
So every computation is either:

* Done(value) – a finished computation that has already produced a value, or,
* Req(request, continuation) – an interaction with the context:
“please handle this request, then resume by calling continuation with the answer”.
bind is the general sequencing operation. 

In math-like notation:
```
bind (Done x) k    = k x
bind (Req r k1) k2 = Req r (\x -> bind (k1 x) k2)
```


```rust
impl<ReqT: 'static + fmt::Display> Comp<ReqT> {
    // Helper to format Comp for logging (used in tests)
    fn format_for_log(&self) -> String {
        format!("{:?}", self.inner)
    }

    // bind :: Comp -> (DomC -> Comp) -> Comp
    // bind (Done x) k    = k x
    // bind (Req r k1) k2 = Req r (\x -> bind (k1 x) k2)
    fn bind(self, k: Abstraction<ReqT>) -> Self {
        let self_desc = self.desc.clone();
        let k_desc = k.kind.clone();
        match self.inner {
            CompInner::Done(x) => {
                k.call(x)
            }
            CompInner::Req(r, k1) => {
                let bind_desc = DescriptionTree::Bind(
                    Box::new(self_desc.clone()),
                    Box::new(k_desc.clone()),
                );
                // The new abstraction: λx. bind(k1(x), k2)
                let k1_desc = k1.kind.clone();
                let k2_desc = k_desc.clone();
                // Format k1 application properly
                let k1_app: DescriptionTree = if k1_desc.is_identity() {
                    // If k1 is identity (λx. x), applying it to x just gives x
                    DescriptionTree::Var("x".to_string())
                } else {
                    // Otherwise show as k1(x)
                    DescriptionTree::App(Box::new(k1_desc), Box::new(DescriptionTree::Var("x".to_string())))
                };
                let body = DescriptionTree::Bind(Box::new(k1_app), Box::new(k2_desc));
                let kind = DescriptionTree::Lam("x".to_string(), Box::new(body));
                let new_abstraction = Abstraction::new(
                    kind,
                    move |x| {
                        Self::bind(k1.call(x), k)
                    }
                );
                Comp::req(r, new_abstraction, bind_desc)
            }
        }
    }
}
```

### EBasic semantics

We define language features as traits with static methods implemented directly on `Comp<ReqT>`. This gives us extensibility - we can add new features without modifying existing ones.

**EBasic:** Core primitives (`int`, `inc`, `app`)  
`app` uses `bind` internally.
In general, semantics of `EBasic` is oblivious to our effect language (i.e., `get`, `put`).


```rust
impl EBasic for Comp<ReqT> {
    // int = Done . DCInt
    fn int(n: i64) -> Self {
        Comp::done(DomC::DCInt(n), DescriptionTree::Int(n))
    }

    // inc = Done . DCFun $ \x -> case x of
    //   DCInt x -> int (succ x)
    //   _       -> err
    fn inc() -> Self {
        let id = next_abstraction_id();
        let closure = Abstraction::new(
            DescriptionTree::Inc(id as u64),
            |x| match x {
                DomC::DCInt(n) => {
                    Self::int(n + 1)
                }
                _ => {
                    // err: send ReqError request
                    let err_closure = Abstraction::new(
                        DescriptionTree::identity(),
                        |_| unreachable!("ReqError continuation should not be called")
                    );
                    Comp::req(ReqT::ReqError, err_closure, DescriptionTree::Req(Box::new(DescriptionTree::Var("error".to_string()))))
                }
            }
        );
        let desc = closure.kind.clone();
        Comp::done(DomC::DCFun(closure), desc)
    }

    // Conceptually, app must handle all these cases:
    //   app (Done (DCFun f)) (Done e2) = f e2          -- apply function to value
    //   app (Done _)         (Done e2) = err            -- type error
    //   app (Req r k) e2               = Req r (\x -> app (k x) e2)  -- e1 has effect
    //   app e1 (Req r k)               = Req r (\x -> app e1 (k x))  -- e2 has effect
    // But bind handles ALL these cases automatically:
    //   app e1 e2 = bind e1 (\f -> bind e2 (\a -> f a))
    fn app(e1: Self, e2: Self) -> Self {
        let e2_desc = e2.desc.clone();
        // First continuation: λf. bind(e2, λa. f(a))
        let inner_lam = DescriptionTree::Lam(
            "a".to_string(),
            Box::new(DescriptionTree::App(
                Box::new(DescriptionTree::Var("f".to_string())),
                Box::new(DescriptionTree::Var("a".to_string())),
            ))
        );
        let k1_desc = DescriptionTree::Lam(
            "f".to_string(),
            Box::new(DescriptionTree::Bind(Box::new(e2_desc), Box::new(inner_lam)))
        );
        e1.bind(Abstraction::new(k1_desc, move |d1| {
            // Second continuation: λa. f(a) where f is the captured function
            let f_desc = match &d1 {
                DomC::DCFun(f) => f.kind.clone(),
                _ => DescriptionTree::Var("_".to_string()),
            };
            let k2_desc = DescriptionTree::Lam(
                "a".to_string(),
                Box::new(DescriptionTree::App(
                    Box::new(f_desc),
                    Box::new(DescriptionTree::Var("a".to_string())),
                ))
            );
            e2.bind(Abstraction::new(k2_desc, move |d2| match d1 {
                DomC::DCFun(f) => {
                    f.call(d2)
                }
                _ => {
                    // err: send ReqError request
                    let err_closure = Abstraction::new(
                        DescriptionTree::identity(),
                        |_| unreachable!("ReqError continuation should not be called")
                    );
                    Comp::req(ReqT::ReqError, err_closure, DescriptionTree::Req(Box::new(DescriptionTree::Var("error".to_string()))))
                }
            }))
        }))
    }
}

```

### Adding State

At this point we keep the core language (`EBasic`) completely unchanged and add state *orthogonally* via a separate trait:

- `EBasic` still only knows about:
    - `int` (integer literals),
    - `inc` (functions over integers),
    - `app` (application using `bind` for sequencing).

- State is introduced by an existing trait:
    - `GlobalIntState` with operations `get` and `put`.

The semantics of `get`/`put` are given **directly on the computation domain** (`Comp<ReqT>`) and expressed in terms of requests:
- `get` issues a `ReqState(Get)` request and continues with the identity continuation.
- `put e` first evaluates `e` using `bind`, then issues a `ReqState(Put n)` request when `e` produces `DCInt(n)`.

This keeps the basic language definitions stable: we don’t touch `EBasic` when adding state, we just enrich the effect language (`ReqT`) and provide an implementation of `GlobalIntState` for `Comp<ReqT>`.


```rust
impl GlobalIntState for Comp<ReqT> {
    // get = Req (ReqState Get) Done
    fn get() -> Self {
        let closure = Abstraction::new(
            DescriptionTree::identity(),
            |d| Comp::done(d, DescriptionTree::Var("x".to_string()))
        );
        Comp::req(ReqT::ReqState(StateReq::Get), closure, DescriptionTree::Get)
    }

    // put e = bind e $ \x -> case x of
    //   DCInt x -> Req (ReqState (Put x)) Done
    //   _       -> err
    fn put(e: Self) -> Self {
        // λx. Req(Put(x))
        let k_desc = DescriptionTree::Lam(
            "x".to_string(),
            Box::new(DescriptionTree::Req(
                Box::new(DescriptionTree::Put(Box::new(DescriptionTree::Var("x".to_string()))))
            ))
        );
        e.bind(Abstraction::new(k_desc, |x| match x {
            DomC::DCInt(n) => {
                let put_n = DescriptionTree::Put(Box::new(DescriptionTree::Int(n)));
                let closure = Abstraction::new(
                    DescriptionTree::identity(),
                    |d| Comp::done(d, DescriptionTree::Var("x".to_string()))
                );
                Comp::req(ReqT::ReqState(StateReq::Put(n)), closure, put_n)
            }
            _ => {
               // err: send ReqError request
                let err_closure = Abstraction::new(
                    DescriptionTree::identity(),
                    |_| unreachable!("ReqError continuation should not be called")
                );
                Comp::req(ReqT::ReqError, err_closure, DescriptionTree::Req(Box::new(DescriptionTree::Var("error".to_string()))))
            }
        }))
    }
}
```

### Interpreting Effect requests using a handler (I-4)

Handlers give meaning to requests by pattern‑matching on them and providing responses. They are defined **separately** from the core language constructs like `inc`, `app`, `get`, and, `put` so we can add new handlers without changing existing code.

`handle_state` is a handler for `Comp<ReqT>` that interprets `ReqState` requests under an integer state:

- Input:
    - initial state `s: i64`
    - computation `comp: Comp<ReqT>`
- Output:
    - a new computation where all `ReqState` requests are handled.

A computation is either:

- `Done(v)` – finished, value `v : DomC<ReqT>`
- `Req(r, k)` – a request `r : ReqT` with continuation `k : DomC<ReqT> -> Comp<ReqT>`

`handle_state` processes these cases as:

1. **Done**  requires no special handling.
2. **State requests**
     - `Get` returns the current state `s` and continues under the same `s`.
     - `Put(new_s)` updates the state to `new_s` and continues under `new_s`.
3. **Errors** stop interpretation.

4. **Other effects** - Non‑state requests are re‑emitted, but their continuation is wrapped so that when they resume, their result is passed back into `handle_state`.
This design assumes a hierarchy of handlers.

This illustrates the **effects-as-interactions** view: computations issue `Req`uest values; handlers decide how to answer them, while the core language stays unchanged. This interpreter is called **I-4**.



```rust
// Pretty print computation tree
fn print_comp_tree(comp: &Comp<ReqT>, indent_level: usize) {
    let indent = "  ".repeat(indent_level);
    println!("{}{}", indent, comp.format_for_log());
}

// handleState :: Int -> Comp -> Comp
// handleState _ (Done x)    = Done x
// handleState s (Req (ReqState r) k) = case r of
//   Get   -> handleState s $ k (DCInt s)
//   Put s -> handleState s $ k (DCInt s)
// -- everything else
// handleState s (Req r k) = Req r (handleState s . k)
fn handle_state(s: i64, comp: Comp<ReqT>) -> Comp<ReqT> {
    // Print the computation tree before handling
    println!("{}Handling state {} for:", "  ".repeat(0), s);
    print_comp_tree(&comp, 1);
    let comp_desc = comp.desc.clone();
    match comp.inner {
        CompInner::Done(x) => {
            Comp::done(x, DescriptionTree::HandleState(s, Box::new(comp_desc)))
        }
        CompInner::Req(ReqT::ReqState(r), k) => match r {
            StateReq::Get => {
                handle_state(s, k.call(DomC::DCInt(s)))
            }
            StateReq::Put(new_s) => {
                handle_state(new_s, k.call(DomC::DCInt(new_s)))
            }
        },
        // Explicit handling for an error request: print and stop
        CompInner::Req(ReqT::ReqError, _k) => {
           println!("error in expression!");
            panic!("ReqError encountered - stopping computation");
        }
        // everything else
        CompInner::Req(r, k) => {
            let kind = DescriptionTree::HandleState(s, Box::new(k.kind.clone()));
            let hs_desc = DescriptionTree::HandleState(s, Box::new(comp_desc.clone()));
            let new_closure = Abstraction::new(
                kind,
                move |x| {
                    handle_state(s, k.call(x))
                }
            );
            Comp::req(r, new_closure, hs_desc)
        }
    }
}
```

#### Test for I-4


```rust
// Test get/put with inc (no conditionals)
fn test_state_basic() -> Comp<ReqT> {
    Comp::put(Comp::int(5))  // This returns DCInt(5) when handled
}

let comp: Comp<ReqT> = test_state_basic();
println!("\n=== Test: put 5");
println!("Before handling:");
print_comp_tree(&comp, 0);

let result = handle_state(0, comp);
println!("\nAfter handling:");
print_comp_tree(&result, 0);
```

    
    === Test: put 5
    Before handling:
    Req(ReqState(Put(5)), λ#1[λx. x])
    Handling state 0 for:
      Req(ReqState(Put(5)), λ#1[λx. x])
    Handling state 5 for:
      Done(DCInt(5))
    
    After handling:
    Done(DCInt(5))


### Sequencing (I-4s)

Sequencing lets us **run two computations one after another** while discarding the result of the first and keeping the second. This is the analogue of `e1; e2` in many languages:

- Evaluate `e1` (which may perform effects like `get`/`put`),
- Ignore its resulting value,
- Then continue with `e2`, preserving any effects that `e1` triggered.

In our setting, both `e1` and `e2` are `Comp<ReqT>` computations that may contain requests. We implement sequencing via a small trait `Seq` with a single operation:

- `seq(e1, e2)` is defined as `bind e1 (\_ -> e2)`

This definition reuses `bind` as the general **effect-aware sequencing primitive**: `bind` runs `e1` to completion (threading all its requests/handlers) and then invokes the continuation `\_ -> e2`, which simply ignores the first result and returns `e2`. The next code block shows this trait and its implementation for `Comp<ReqT>`, followed by a test that sequences several stateful operations.
The I-4 machine with sequence command is called **I-4s**.


```rust
// Sequencing trait - for discarding first result and continuing with second
pub trait Seq {
    fn seq(e1: Self, e2: Self) -> Self;
}

impl Seq for Comp<ReqT> {
    // seq e1 e2 = bind e1 (\_ -> e2)
    // Sequences two computations, discarding the result of the first
    fn seq(e1: Self, e2: Self) -> Self {
        let e2_desc = e2.desc.clone();
        let cont_desc = DescriptionTree::Lam("_".to_string(), Box::new(e2_desc));
        e1.bind(Abstraction::new(cont_desc, move |_| e2))
    }
}
```

#### I-4s test


```rust
fn test_state_basic() -> Comp<ReqT> {
    // put 5; get; put (inc get); get
    Comp::seq(
        Comp::put(Comp::int(5)),
        Comp::seq(
            Comp::get(),
            Comp::seq(
                Comp::put(Comp::app(Comp::inc(), Comp::get())),
                Comp::get()
            )
        )
    )
}

let comp: Comp<ReqT> = test_state_basic();
println!("\n=== Test: put 5; get; put (inc get); get ===");
println!("Before handling:");
print_comp_tree(&comp, 0);

let result = handle_state(0, comp);
println!("\nAfter handling:");
print_comp_tree(&result, 0);

// Assert the final result
match &result.inner {
    CompInner::Done(DomC::DCInt(n)) => {
        assert_eq!(*n, 6, "Expected final state to be 6 (inc 5)");
        println!("✓ Test passed: final value is {}", n);
    }
    _ => panic!("Expected Done(DCInt(6)), got {:?}", result.inner),
}
```

    
    === Test: put 5; get; put (inc get); get ===
    Before handling:
    Req(ReqState(Put(5)), λ#17[λx. bind(x, λ_. bind(get, λ_. bind(bind(bind(get, λa. app(λ#3[λx. x + 1], a)), λx. Req(put(x))), λ_. get)))])
    Handling state 0 for:
      Req(ReqState(Put(5)), λ#17[λx. bind(x, λ_. bind(get, λ_. bind(bind(bind(get, λa. app(λ#3[λx. x + 1], a)), λx. Req(put(x))), λ_. get)))])
    Handling state 5 for:
      Req(ReqState(Get), λ#15[λx. bind(x, λ_. bind(bind(bind(get, λa. app(λ#3[λx. x + 1], a)), λx. Req(put(x))), λ_. get))])
    Handling state 5 for:
      Req(ReqState(Get), λ#13[λx. bind(app(λx. bind(app(λx. bind(x, λa. app(λ#3[λx. x + 1], a)), x), λx. Req(put(x))), x), λ_. get)])
    Handling state 5 for:
      Req(ReqState(Put(6)), λ#19[λx. bind(x, λ_. get)])
    Handling state 6 for:
      Req(ReqState(Get), λ#11[λx. x])
    Handling state 6 for:
      Done(DCInt(6))
    
    After handling:
    Done(DCInt(6))
    ✓ Test passed: final value is 6





    ()



### Add back conditional (I-4s+)

In the implementation of `if_` for `Comp<ReqT>`:

- `cond` is a **computation** that will eventually yield a `DCBool` (or an error), but it may also issue requests (`ReqState`, `ReqError`, …) along the way.
- `bind` is used to *run* this computation and then *decide* which branch to execute once the boolean is available.

The definition:

```rust
fn if_(cond: Self, et: Self, ef: Self) -> Self {
    let et_desc = et.desc.clone();
    let ef_desc = ef.desc.clone();
    // λb. if b then et else ef
    let if_body = DescriptionTree::If(
        Box::new(DescriptionTree::Var("b".to_string())),
        Box::new(et_desc),
        Box::new(ef_desc),
    );
    let cont_desc = DescriptionTree::Lam("b".to_string(), Box::new(if_body));
    cond.bind(Abstraction::new(cont_desc, move |x| match x {
        DomC::DCBool(true)  => et,
        DomC::DCBool(false) => ef,
        _ => { /* send ReqError */ }
    }))
}
```

can be read as the direct encoding of:

```haskell
if_ cond et ef = bind cond $ \b ->
  case b of
    True  -> et
    False -> ef
    _     -> err
```

More concretely:

1. `cond.bind(...)` says:  
   “Run the computation `cond`. When it **finishes** with a value `x`, apply the continuation to `x`.”

2. If `cond` evaluates to `Done (DCBool true)` or `Done (DCBool false)`, the `bind` implementation takes the `Done` branch and calls the continuation:

   ```rust
   match x {
       DomC::DCBool(true)  => et,
       DomC::DCBool(false) => ef,
       _                   => error-request
   }
   ```

   returning either `et` or `ef` *as computations*. Any effects in `et` or `ef` are left untouched.

3. If `cond` is not finished but instead produces a `Req(r, k1)`, `bind` re-emits that request and composes the existing continuation `k1` with the new “if-continuation”. Operationally, this means:

   - The effect handler will see exactly the same requests as if we were evaluating `cond` alone.
   - Once a handler answers a request and `cond` progresses, `bind` will keep threading the new continuation until `cond` eventually returns a `DCBool`, at which point the `match` chooses `et` or `ef`.

Thus, `bind` is the mechanism that:

- **waits** for `cond` to finish producing a boolean,
- **threads** any intermediate effects from `cond` through the same handlers,
- and then **selects** and returns the appropriate branch computation (`et` or `ef`) without changing their definitions.


```rust

impl ECond for Comp<ReqT> {
    // eq = ... like inc
    fn eq() -> Self {
        let closure1 = Abstraction::new(DescriptionTree::Eq, |x| {
            match x {
                DomC::DCInt(n1) => {
                    let id = next_abstraction_id();
                    let closure2 = Abstraction::new(
                        DescriptionTree::EqArg2(id as u64, n1),
                        move |y| {
                            match y {
                                DomC::DCInt(n2) => {
                                    Comp::done(DomC::DCBool(n1 == n2), DescriptionTree::Bool(n1 == n2))
                                }
                                _ => {
                                    // err: send ReqError request
                                    let err_closure = Abstraction::new(
                                        DescriptionTree::identity(),
                                        |_| unreachable!("ReqError continuation should not be called")
                                    );
                                    Comp::req(ReqT::ReqError, err_closure, DescriptionTree::Req(Box::new(DescriptionTree::Var("error".to_string()))))
                                }
                            }
                        }
                    );
                    let desc = closure2.kind.clone();
                    Comp::done(DomC::DCFun(closure2), desc)
                }
                _ => {
                    // err: send ReqError request
                    let err_closure = Abstraction::new(
                        DescriptionTree::identity(),
                        |_| unreachable!("ReqError continuation should not be called")
                    );
                    Comp::req(ReqT::ReqError, err_closure, DescriptionTree::Req(Box::new(DescriptionTree::Var("error".to_string()))))
                }
            }
        });
        Comp::done(DomC::DCFun(closure1), DescriptionTree::Eq)
    }

    // if_ e et ef = bind e $ \x -> case x of
    //   DCBool True  -> et
    //   DCBool False -> ef
    //   _            -> err
    fn if_(cond: Self, et: Self, ef: Self) -> Self {
        let et_desc = et.desc.clone();
        let ef_desc = ef.desc.clone();
        // λb. if b then et else ef
        let if_body = DescriptionTree::If(
            Box::new(DescriptionTree::Var("b".to_string())),
            Box::new(et_desc),
            Box::new(ef_desc),
        );
        let cont_desc = DescriptionTree::Lam("b".to_string(), Box::new(if_body));
        cond.bind(Abstraction::new(cont_desc, move |x| match x {
            DomC::DCBool(true) => {
                et
            }
            DomC::DCBool(false) => {
                ef
            }
            _ => {
                // err: send ReqError request
                let err_closure = Abstraction::new(
                    DescriptionTree::identity(),
                    |_| unreachable!("ReqError continuation should not be called")
                );
                Comp::req(ReqT::ReqError, err_closure, DescriptionTree::Req(Box::new(DescriptionTree::Var("error".to_string()))))
            }
        }))
    }
}
```

#### Test for I-4s+


```rust
// ttinc :: EBasic d => d
// ttinc = inc `app` (inc `app` int 2)
// *EDSLNG> ttinc :: Comp
// Done (DInt 4)
fn ttinc() -> Comp<ReqT> {
    Comp::app(Comp::inc(), Comp::app(Comp::inc(), Comp::int(2)))
}

// Build the complex TTS expression step by step
fn test_tts() -> Comp<ReqT> {
    let condition = Comp::app(
        Comp::app(Comp::eq(), Comp::int(3)),
        Comp::put(ttinc())
    );
    
    let then_branch = Comp::put(Comp::int(10));
    let else_branch = Comp::put(Comp::app(Comp::inc(), Comp::get()));
    
    Comp::if_(condition, then_branch, else_branch)
}

println!("=== Test 4: Complex TTS Expression ===");
let tts = test_tts();
print_comp_tree(&tts, 0);

// handleState 0 ttS should evaluate to Done(DCInt(5))
let result = handle_state(0, tts);
println!("\nAfter handling state:");
print_comp_tree(&result, 0);
```

    === Test 4: Complex TTS Expression ===
    Req(ReqState(Put(4)), λ#29[λx. bind(app(λx. bind(x, λa. app(λ#3[λy. 3 == y], a)), x), λb. if b
      then put(int(10))
      else bind(bind(get, λa. app(λ#20[λx. x + 1], a)), λx. Req(put(x))))])
    Handling state 0 for:
      Req(ReqState(Put(4)), λ#29[λx. bind(app(λx. bind(x, λa. app(λ#3[λy. 3 == y], a)), x), λb. if b
      then put(int(10))
      else bind(bind(get, λa. app(λ#20[λx. x + 1], a)), λx. Req(put(x))))])
    Handling state 4 for:
      Req(ReqState(Get), λ#27[λx. bind(app(λx. bind(x, λa. app(λ#20[λx. x + 1], a)), x), λx. Req(put(x)))])
    Handling state 4 for:
      Req(ReqState(Put(5)), λ#30[λx. x])
    Handling state 5 for:
      Done(DCInt(5))
    
    After handling state:
    Done(DCInt(5))


## Discovering Monads

Monads capture a common pattern:

1. Wrap a pure value into a computation.
2. Sequence computations that may perform effects, where each step can depend on the result of the previous one.

Formally, a monad is defined by:

- A type constructor `M` (here: `Comp<ReqT>`),
- A “return”/“pure” operation (here: `done`),
- A “bind” operation (here: `bind`):

```text
pure  :: a -> M a
bind  :: M a -> (a -> M b) -> M b
```

subject to three laws: **left identity**, **right identity**, and **associativity**.

In this notebook:

- The *values* live in `DomC<ReqT>`.
- The *computations* live in `Comp<ReqT>`.
- `Comp::done : DomC<ReqT> -> Comp<ReqT>` is our “pure”:
    it takes a plain value and produces a finished computation:
    it never issues any requests, it’s just `Done(value)`.
- `Comp::bind(self, k)` is our sequencing operator:
    it runs `self` until it either:
    - finishes with `Done(x)` and then calls `k(x)`, or
    - issues a request `Req(r, k1)`, in which case it *re-emits* the request and composes the original continuation `k1` with `k`:
        when the handler later supplies an answer `x`, the new continuation first resumes via `k1(x)`, then keeps binding with `k`.

Conceptually:

- `done` says: “no effects, here is the value”.
- `bind` says: “run this computation; when it produces a value, continue with this next step; and if there are intermediate requests, keep threading them through”.

### Monadic Laws in Terms of `done` and `bind`

Let `x : DomC<ReqT>`, `m : Comp<ReqT>`, and `f, g : DomC<ReqT> -> Comp<ReqT>`.

We interpret `k_f` and `k_g` as `Abstraction`s built from `f` and `g`.

#### 1. Left Identity

```text
bind (done x) f  =  f x
```

In our implementation:

- `done(x, desc)` builds `Comp { inner: Done(x), desc }`.
- Calling `bind` on `Done(x)` immediately does:

```rust
match self.inner {
        Done(x) => k.call(x)
        ...
}
```

So if `self` is `done(x)`, `bind(self, k_f)` directly evaluates to `f(x)`.
No extra requests or wrapping are introduced, so left identity holds.

#### 2. Right Identity

```text
bind m done  =  m
```

Here we take `k_id` as an abstraction representing the identity continuation:

```text
k_id(x) = done x
```

For each shape of `m`:

- If `m = Done(x)`:
    `bind(Done(x), k_id)` reduces to `k_id(x) = done x`, which is observationally the same as `m` (a completed computation with the same value and description).
- If `m = Req(r, k1)`:
    `bind(Req(r, k1), k_id)` becomes `Req(r, k')` where `k'(x) = bind(k1(x), k_id)`.
    When the handler eventually supplies a value `v` for the request:
    - `k1(v)` is exactly the continuation that `m` would have used,
    - `bind(k1(v), k_id)` then acts as in the `Done` case and does not change the resulting computation.
    Thus, the behaviour of `Req(r, k')` is the same as that of the original `Req(r, k1)` — requests and their answers are handled identically.

So `bind m done` behaves the same as `m`, satisfying right identity.

#### 3. Associativity

```text
bind (bind m f) g  =  bind m (\x -> bind (f x) g)
```

We must consider both shapes of `m`:

- If `m = Done(x)`:
    - Left-hand side: `bind (bind (Done x) f) g`
        - `bind(Done x, f)` gives `f(x)`.
        - Then `bind(f(x), g)` continues with `g` after `f`.
    - Right-hand side: `bind (Done x) (\x -> bind (f x) g)`
        - `bind(Done x, ...)` simply calls the continuation on `x`, giving `bind(f(x), g)`.
    Both sides coincide.

- If `m = Req(r, k1)`:

    Let `bind` be implemented as:

    pure/return:

    ```text
    bind(Done x,   f) = f x
    ```

    effectful step:

    ```text
    bind(Req r k1, f) = Req r (\x -> bind (k1 x) f)
    ```

    This is just pattern matching on the *first* step of the computation:

    - if it is already finished (`Done x`), call `f` on the result;
    - if it is a request (`Req r k1`), re-emit the same request and say:
      “once you get a reply `x`, resume with `k1 x`, and then keep binding with `f`”.

    We now check associativity **by cases on `m`**, not by a full structural induction over all possible computations:

    - the `Done` case was handled above;
    - here we handle the single-step `Req` case, and we do not recurse over longer traces explicitly.

    **Left-hand side**

    ```text
    bind (bind m f) g
    = bind (bind (Req r k1) f) g
    = bind (Req r (\x -> bind (k1 x) f)) g
    = Req r (\y -> bind ((\x -> bind (k1 x) f) y) g)
    = Req r (\y -> bind (bind (k1 y) f) g)
    ```

    **Right-hand side**

    ```text
    bind m (\x -> bind (f x) g)
    = bind (Req r k1) (\x -> bind (f x) g)
    = Req r (\y -> bind (k1 y) (\x -> bind (f x) g))
    ```

    On both sides, the *outer shape* is the same: `Req r (...)`.  
    The only difference lies in the transformed continuation:

    ```text
    k_L(y) = bind (bind (k1 y) f) g
    k_R(y) = bind (k1 y) (\x -> bind (f x) g)
    ```

    These are precisely the left and right sides of the **same associativity law**, but now applied to the *shorter* computation `k1 y`:

    ```text
    bind (bind n f) g    =    bind n (\x -> bind (f x) g)
                 ^                    ^
                 n = k1 y             n = k1 y
    ```

    In other words, for a `Req` step our `bind` does nothing but:

    - keep the request `r` unchanged;
    - replace the continuation `k1` with a new continuation that keeps composing with `f` and `g` in the same order.

    The “work” of associativity is thus pushed inside the continuation, without introducing any reordering of effects.

Because the implementation of `bind` for the `Req` case *only* rewraps the continuation with another call to `bind`, without changing the order, both sides describe the same staged interaction:

1. Emit exactly the same requests in the same order.
2. For each response, resume with `k1`, then `f`, then `g` in the same nesting.

Therefore `bind` is associative.

---

In summary, `Comp<ReqT>` with:

- `done : DomC<ReqT> -> Comp<ReqT>`
- `bind : Comp<ReqT> -> (DomC<ReqT> -> Comp<ReqT>) -> Comp<ReqT>`

forms a monad. This monadic structure is what lets us:

- write pure-looking code (`int`, `inc`, `app`, `if_`),
- layer in effects (`get`, `put`) as *requests*,
- and keep handlers (`handle_state`) separate, all while preserving the monad laws that guarantee predictable sequencing of effects.

A practical introduction to how monads can be used is found [here](https://cs3110.github.io/textbook/chapters/ds/monads.html).

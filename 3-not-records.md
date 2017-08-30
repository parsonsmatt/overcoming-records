# Haskell Records

# Ain't Records
<!-- .element: class="fragment" -->

Note:

So, here's the elephant in the room.
Haskell records?
Aren't.
They have virtually none of the nice properties we talked about records having!

So, what are they?


```haskell
data Human = Human
    { humanName :: Text
    , humanAge  :: Int
    , humanDogs :: [Dog]
    }
```

is sugar for:

```haskell
data Human = Human Text Int [Dog]

humanName :: Human -> Text
humanName (Human n _ _) = n

humanAge :: Human -> Int
humanAge (Human _ a _) = a

humanDogs :: Human -> [Dog]
humanDogs (Human _ _ d) = d
```

Note:

Haskell records are primarily syntax sugar for these record accessor functions.
When you realize that, a lot of the other limitations seem somewhat obvious.


# name overlap

```haskell
data Human = Human { name :: Text }

data Dog = Dog { name :: Text }
```

```haskell
data Human = Human Text

name :: Human -> Text
name (Human n) = n

data Dog = Dog Text

name :: Dog -> Text
name (Dog n) = n
```
<!-- .element: class="fragment" -->

Note:

Beginners to Haskell always try this.
It always fails and bites them. 
Experienced users do it too!
And now, it seems clear why:

We're just generating top level global accessor functions!
Haskell doesn't let you overload ordinary functions.
You need type classes for that.
And these record fields aren't making type classes, nor are they smart enough to.


# record creation

```haskell
data Human = Human
    { humanName :: Text
    , humanAge  :: Int
    , humanDogs :: [Dog]
    }

matt :: Human
matt = Human 
    { humanName = "Matt Parsons"
    , humanAge = 28
    , humanDogs = []
    }

matt2 :: Human
matt2 = Human "Matt Parsons" 28 []
```
<!-- .element: class="fragment" -->

Note:

There's two other bits of syntax sugar that come standard.
The first is record creation.
We can use the record syntax to create instances of records.
This is nice: we don't have to worry about the order we defined the fields in.


# kinda sucks

```haskell
matt3 :: Human
matt3 = Human { humanName = "Matt", humanAge = 28 }
```

```bash
Fields of 'Human' not initialised: humanDogs
In the expression: 
    Human {humanName = "matt", humanAge = 28}
In an equation for 'matt':
    matt = Human {humanName = "matt", humanAge = 28}
```
<!-- .element: class="fragment" -->

Note:

Unfortunately record creation has a serious, ultra, huge, massive problem: it's partial!
If you don't assign a field, then it becomes `undefined`, and will blow up whenever you inspect it.
The compiler will emit a warning if you create a record with missing fields.
However, that really should be a compiler error.


# strict fields!!!

```haskell
{-# LANGUAGE BangPatterns #-}

data Human 
    = Human { humanName :: !Text, humanAge :: !Int }

matt4 = Human { humanName = "Matt, again??" }
```

```bash
Constructor 'Human' does not have the 
    required strict field(s): humanAge
In the expression: Human {humanName = "matt"}
In an equation for 'matt': 
    matt = Human {humanName = "matt"}
```

Note:

Strict fields come to the rescue here, though.
The exclamation marks are a "bang pattern", and tell GHC that the value must be evaluated to "weak head normal form" before the construction of that record will be complete.
Weak head normal form is essentially "outermost constructor".
Strict record fields *will* cause a compile time error if they're uninitialized.


# update syntax

```haskell
rename :: String -> Human -> Human
rename newName human =
    human { humanName = newName }
```

```haskell
renameDogs :: String -> Human -> Human
renameDogs newName human =
    human 
        { humanDogs = 
              map (renameDog newName) (humanDogs human) 
        }
```
<!-- .element: class="fragment" -->

Note:

We also get this record update syntax.
But, it's kind of horrible for anything even remotely large.
It's slightly nicer than writing out the `setName` `setAge` functions yourself, I suppose.


# haskell does not

# have
<!-- .element: class="fragment" -->

# records
<!-- .element: class="fragment" -->

Note:

So it's been easier for me to just pretend that Haskell doesn't have any meaningful records built into the language.
It's fun -- try saying it yourself sometime.


# you're not my real dad

Note:

It kind of feels like saying this, but to, uh, your programming language, which is totally not weird at all.

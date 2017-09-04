# get 

# it 

# back

Note:

So Haskell doesn't have real records. Fine.
How do we end up using them in other languages, so that we can recover this functionality?


# Named Arguments

```ruby
# Ruby
def foo **opts
  puts "#{opts[:name]}, #{opts[:age]}"
end

foo(name: "Matt", age: 28)
```

```javascript
function foo({name, age}) {
    console.log(`${name}, ${age}`);    
}

foo({ name: "Matt", age: 28 });
```

Note:

A common use for records is named (or keyword) arguments.
Ruby, Python, and JavaScript all support this functionality.
Like all good things in dynamically typed OOP, this is inspired by Smalltalk, where named arguments were the *only* way to pass multiple arguments to a method!

There are two big advantages here: Documentation and position independence.
Records are unordered, so it does not matter how I pass name and age.


# Newtypes

```haskell
newtype Name = Name String
newtype Age = Age Int

foo :: Name -> Age -> IO ()
foo (Name name) (Age age) =
  putStrLn (concat [name, ", ", show age])

foo (Name "Matt") (Age 28)
-- Matt, 28
```

Note:

The easiest Haskell-y solution for this is newtypes.
We lose the property of position independence, but Haskell's type system turns incorrect ordering into a very easy to read error message, especially with newtypes.
This removes one of the motivating factors for position independence -- in dynamically typed languages, you don't get much warning if you pass the wrong arguments to a function.
These newtypes might be a little cumbersome to wrap and unwrap, but I've generally found it to be heavily worthwile in terms of type safety and program readability.
I've run into a few production bugs where I mismatched two `Text` parameters, and newtyping them fixed it up quick.


# Optional Arguments

```ruby
def say_hello(name:, prefix: "Comrade", hello: "Hello")
  puts "#{hello}, #{prefix} #{name}!"
end

say_hello(name: "Matt", prefix: "Troll")
# Hello, Troll Matt!
```

Note:

Optional arguments (or arguments with a default) are quite convenient in this context.
We have a default prefix of "Comrade" that we use before saying hello to someone.
If someone prefers a different salutation, then they get that.

This is great for customizing behavior of a function!
Typically these configuration/optional arguments are fed into the start of a bigger method.


# Record Arguments

```haskell
{-# LANGUAGE RecordWildcards #-}

data HelloArgs = HelloArgs
  { hello  :: String
  , prefix :: String
  }

sayHello :: Name -> HelloArgs -> IO ()
sayHello (Name name) HelloArgs{..} =
  putStrLn (concat [salutation, ", ", prefix, " ", name])
```

Note:

So, on the *implementing* side, we can use Haskell's poor records to decent extent here.
We can use the RecordWildcards language extension to make all of the fields of the record available in scope with the braces pattern.
This ends up looking pretty nice!
Unfortunately, it looks pretty ugly on the calling side..


```haskell
main =
  sayHello 
    (Name "Matt") 
    HelloArgs { hello = "Hello", prefix = "Friend" }
```

# :(
<!-- .element: class="fragment" -->

Note:

This isn't great.
We lose the default values.
We need the boilerplate of the `HelloArgs` constructor.
It's not fun.
Fortunately, we can use the Default type class to help:


```haskell
class Default a where
  def :: a

instance Default HelloArgs where
  def = HelloArgs { hello = "Hello", prefix = "Comrade" }

main = do
  sayHello (Name "Matt") def { prefix = "Friend" }
  -- Hello, Friend Matt
  sayHello (Name "Putin") def { hello = "Privet" }
  -- Privet, Comrade Putin
  sayHello (Name "Greg") def
  -- Hello, Comrade Greg

```

Note:

By a wart in Haskell's syntax, record update/create syntax binds tighter than function application.
It's the only thing in the language that does this.
So it *looks* like we're passing an anonymous record to the function, but really, we're modifying `def` and passing that into the function.


# Sharing Fields

Note:

Sharing field names is a pretty common desire for records.


# Nested records

```javascript
var obj = {
    foo: {
        bar: {
            baz: 10     
        }     
    }    
};

obj.foo.bar.baz = 12;
console.log(obj.foo.bar.baz);
// 12
```

Note:

Another thing we often want to do is modify records.
OOP languages have this really nice dot notation for accessing and assigning into deeply nested records.
Haskell's basic syntax for this is... not great:


```haskell
data Obj = Obj { foo :: Foo }
data Foo = Foo { bar :: Bar }
data Bar = Bar { baz :: Int }

obj = Obj { 
  foo = Foo { 
    bar = Bar { 
      baz = 10 } } }

obj' = obj { 
  foo = foo obj {  
    bar = bar (foo obj) {
      baz = 12  } } }

print (baz . bar . foo $ obj)
```

Note:

The assignment is awful.
This is the opposite of fun.
Fortunately there's an incredibly nice solution to this, which dramatically improves on OOP accessors:


<!-- .slide: data-background="lens-1.jpeg" -->

Note:

You probably know what I'm talking about.


<!-- .slide: data-background="lens-2.jpg" -->

Note:

They're considered a little scary.


<!-- .slide: data-background="lens-4.png" -->

Note:

Like most libraries by Edward Kmett.


# Lenses!
<!-- .slide: data-background="lens-3.jpg" -->

Note:

I'm talking about lenses! THey're amazing.

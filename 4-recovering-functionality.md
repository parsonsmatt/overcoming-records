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

```ruby
class Human
  attr_reader :name, :age
end

class Dog
  attr_reader :name, :best_toy
end

def print_name(human_or_dog)
  puts human_or_dog.name
end
```

Note:

Sharing field names is a pretty common desire for records.
These two Ruby classes share the name field.
We can call `name` on any object that has a name method, and it'll respond.


# Sharing Fields

```haskell
class HasName a where
  name :: a -> String

instance HasName Human where
  name = humanName

instance HasName Dog where
  name = dogName
```

Note: 

Haskell uses type classes to overload function and term names.
That we have to implement an explicit class per field is somewhat annoying, and that we have to explicitly define instances is also annoying.


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

### hell yeah
<!-- .element: class="fragment" -->

Note:

I'm talking about lenses! 
They're amazing.
They're somewhat advanced Haskell, but you can use lenses productively without knowing all the fancier tricks.


first lens

```haskell
data Dog = Dog { _name :: String, _age :: Int }
  deriving Show

makeLenses ''Dog

-- Ghci:
>> let dog = Dog "Buddy" 10
Dog { _name = "Buddy", _age = 10 }
>> dog ^. name
"Buddy"
>> let birthday dog = dog & age %~ (+1)
>> birthday dog
Dog { _name = "Buddy", _age = 11 }
```

Note:

I'm not going into super detail on how these work exactly.
But this is the basic idiom: you define your type, name the record fields funny, then call on the `makeLenses` template haskell function.
This generates two "lenses" from the record fields.
Lenses are superpowered dot notation from OOP.

The "hat dot" operator is named view, and is used for accessing the value in a lens.
The ampersand is used to pass a value into a lens that assigns or modifies a value.
The dot operator is exactly function composition.


`lens-aeson`

```haskell
>> :{
>| let someJson = [st|{
>|         "foo": [
>|             { "bar": 3 },
>|             { "bar": 4 },
>|             { "bar": 5 }
>|         ]
>|     } |]
>| :}
>> :{
>| let addThreeToJson = 
>|       _Object.ix "foo"._Array.each.ix "bar"._Number
>|       %~ (\x -> x + 3)
>| :}
>> addThreeToJson modified
"{\"foo\":[{\"bar\":6},{\"bar\":7},{\"bar\":8}]}"
```

Note:

This is a bit of a better example of what lenses can do.
Lenses can encode failure via `Maybe`.
We define a value `someJson`, which is just some strict text.
I'm using a quasiquoter to make the quotation marks more readable.

The first lens `_Object` is polymorphic.
It can work on bytestring, text, string, and also Aeson values.
It "succeeds" if it can parse whatever is given to it into a JSON Object.
Then we use "ix" to get into the index of the object. 
Object's are text indexed, so we pass Foo.
We "index" into the possibility that this is an array.
Then we "index" into each element of the array.
Then we index into the "bar" attribute of each object contained in the array.
Then we index intot he possibility that it is a number.
Finally, we use the percent-squiggle to apply a modification function that adds three.

We can see that the modified JSON is printed out, and all the values have been incremented by three.
Lenses are awesome and we can use the library to effectively conquer much of the record polymorphism problems.


# Row Polymorphism

```haskell
data Human = Human 
  { humanName :: String
  , humanAge  :: Int
  , humanDogs :: [Dog] 
  }

data Dog = Dog
  { dogName    :: String
  , dogAge     :: Int
  , dogGoodBoy :: Bool
  }

makeFields ''Human
makeFields ''Dog
```

Note:

makeFields creates fancier lenses than just makeLenses.
makeFields allows you to start talking about things in a way that is very nearly row polymorphic!
We've defined our records and prefixed them with the type name.
This is going to define the following set of stuff for us:


```haskell
class HasName s a | s -> a where
  name :: Lens' s a

class HasDogs s a | s -> a where
  dogs :: Lens' s a

class HasAge s a | s -> a where
  age :: Lens' s a

class HasGoodBoy s a | s -> a where
  goodBoy :: Lens' s a

instance HasName    Human String where ...
instance HasAge     Human Int    where ...
instance HasDogs    Human [Dog]  where ...
instance HasName    Dog   String where ...
instance HasAge     Dog   Int    where ...
instance HasGoodBoy Dog   Bool   where
  goodBoy = lens (const True) 
    (\(Dog name age _) -> Dog name age True)
```

Note:

This is the code that is generated by the above template Haskell.
We get a new type class `HasFieldName` for each field, and an instance for all the types that define that field.
The type class requests a lens, so it can both view, set, and modify the values contained in the record.
Somehow the template haskell code knows that all dogs are good boys, and generated the correct code for us.
Well done.


```haskell
-- Haskell
yellNameLoudly 
  :: HasName s String 
  => s 
  -> IO ()
yellNameLoudly something = do
  putStrLn (map toUpper (something ^. name))
```

```haskell
-- PureScript
yellNameLoudly 
  :: forall fields
   . { name :: String | fields } 
  -> Eff _ Unit
yellNameLoudly rec = do
  Console.log (map toUpper rec.name)
```

Note:

Now, we can be almost as polymorphic as PureScript's row polymorphism.
The only real problem here is that we have to generate these type classes and lenses for every field we end up caring about.
This is annoying, certainly, but it solves the pain point alright.
The GHC team has considered making all record fields generate lenses instead of accessor functions, which would be amazing.


# Grouping Related Data

```haskell
data Config = Config 
  { _configDb  :: DbConfig
  , _configWeb :: WebConfig
  , _configAws :: AwsConfig
  }

data DbConfig = DbConfig   ...
data WebConfig = WebConfig ...
data AwsConfig = AwsConfig ...

makeClassy ''Config
makeClassy ''DbConfig
makeClassy ''WebConfig
makeClassy ''AwsConfig
```

Note:

Where makeFields gives you row polymorphic like abilities, the other Lens generator creates shared data.
The top level application configuration has access to the database, web, and AWS configuration details.
We create classy lenses for each of these types.
Let's look at the generated code:


```haskell
class HasConfig cfg where
  config    :: Lens' cfg Config

  configDb  :: Lens' cfg DbConfig
  configDb = config . dbConfig

  configWeb :: Lens' cfg WebConfig
  configWeb = config . webConfig

  configAws :: Lens' cfg AwsConfig
  configAws = config . awsConfig

instance HasConfig Config where
  config = id
  configDb f  (Config d w a) =  ...
  configWeb f (Config d w a) = ...
  configAws f (Config d w a) = ...
```

Note:

This is the class that gets generated, along with Config's instance.
Don't worry about the details too much.
We've defined a method on this class for each field on the type.
Then, we define a method on the class that tells us how to get the *whole* type from the thing we're defining the instance on.
For Config, it *is* a Config, so the answer is just identity.
Everything else must be defined in terms of something that eventually reaches a Config.


```haskell
instance HasDbConfig Config where
  dbConfig = configDb

instance HasWebConfig Config where
  webConfig = configWeb

instance HasAwsConfig Config where
  awsConfig = configAws

runQuery 
  :: (HasDbConfig r, MonadReader r m, MonadIO m)
  => Query a
  -> m (Either QueryError a)
runQuery = ...
```

Note:

Now, we also generated classy lenses for the database, web, and AWS configuration.
We want to be able to reach them from the config.
This allows us to write functions like 'runQuery' which rely on a specific slice of the context.
We can now pass anything that "contains" a DbConfig -- either a raw DbConfig, or a larger value like a Config, or another nested value.


# lenses rule

Note:

So, I'm of the opinon that lenses solve nearly every problem associated with Haskell's type system.
If lenses were incorporated as language features with special compiler support for error messaging, I think we'd have one of the best record systems out there.

Naturally, we might still want a fully polymorphic record library solution.

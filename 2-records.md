# Haskell Records

Note:

Let's dig into some records.


```haskell
data Human = Human
    { humanName :: Text
    , humanAge  :: Int
    , humanDogs :: [Dog]
    } deriving (Eq, Show)

data Dog = Dog 
    { dogName    :: Text
    , dogAge     :: Int
    , dogGoodBoy :: Bool
    } deriving (Eq, Show)

data Cat = Cat
    { catName      :: Text 
    , catLivesLeft :: Int
    , catServants  :: [Human]
    } deriving (Eq, Show)
```

Note:

Here we've got a couple of Haskell records.
Everything has a name.
Humans and dogs both have ages.
Humans can own dogs.
Dogs may be good boys.
Cats have servants, etc.


```c
typedef struct DogS {
    char* name;
    int age;
    int isGoodBoy;
} Dog;

typedef struct HumanS {
    char* name;
    int age;
    Dog[] dogs;
} Human;

typedef struct CatS {
   char* name;
   int livesLeft;
   Human[] servants;
} Cat;
```

Note:

Here's the same thing, but in C.
Records are kind of like C structs, but not exactly.


```java
public class Dog {
    public final String name;
    public final int age;
    public final boolean isGoodBoy;

    public Dog(String name, int age) {
        this.name = name;
        this.age = age;
        this.isGoodBoy = true;
    }
}

// etc etc etc lol who has room for java slides
```

Note:

They're kind of like classes in object oriented languages.
Here's a Java implementation of Dog.


```javascript
// lmao javascript
function newDog(name, age) {
    return {
        name: name,
        age: age,
        isGoodBoy: true
    };
};
```

```ruby
# hello, Ruby
def new_dog name, age
  {
    name: name,
    age: age,
    is_good_boy: true
  }
end
```

Note:

They're also similar to objects, hashes, dictionaries, or whatever you want to call them in dynamic languages.
Since dynamic languages lack types, the distinction between a dictionary or hash and a real class is less clear.
JavaScript does away with the distinction entirely.


# What are records?

Note:

I've been using some pretty wishy-washy language thus far to talk about records.
They're "kind of" like this, "sort of" like that.
What is a record?


generalized tuples

$$foo\ =\ (a : Int, b : Char, c, d)$$

$$foo_1 = a : Int$$
$$foo_2 = b : Char$$

Note:

A record is a generalization of a tuple.
A tuple is a collection of elements.
It has a fixed size.
The elements can have different types.
You can project, or index, into a tuple using a natural number.


move over, natural number indexing, let's use labels

$$type\ \ foo = \\{ name : String, age : Int \\} $$
<!-- .element: class="fragment" -->

$$type\ \ bar = \\{ age : Int, name : String \\} $$
<!-- .element: class="fragment" -->

$$foo_{name} : String$$
<!-- .element: class="fragment" -->

$$foo_{age} : Int$$
<!-- .element: class="fragment" -->

Note:

A record is a tuple, but instead of using positional indexing, we use labels.
This has some interesting consequences.
Records do not have a notion of "ordering" -- one label isn't "before" another.
The record types foo and bar here are therefore the same type.


# neat record tricks

Note:

That's not all.
There's some neat features we would naturally want out of records!


# record subtyping

```java
// java
class Bar {
    public String name;    
}

class Foo extends Bar {
    public int age;
}

public void printName(Bar bar) {
    System.out.println(bar.name);    
}
```

Note:

So, one thing we often want to do is express is record subtyping.
If a record has all the same fields as another record, and a few extra, then we should be able to use it in place of the record.
Foo here has a name and an age, while Bar has just a name.
If a function accepts a Bar as input, then we'd want it to also accept a Foo, or anything else that was at least as large.


```java
interface HasName {
    public String getName();
}

class Foo implements HasName {
    public String name;
    public getName() { return this.name };
}
```

Note:

Java doesn't really have this, because you have to either make an explicit subclass or define an interface, and then explicitly implement it.
Java has what is known as "nominal subtyping," where you must explicitly declare that one type is a subtype of another.
Furthermore, subclassing and subtyping are actually totally different -- a superclass is a subtype.
OO type systems are weird and hard.
OCaml has structural subtyping, where you can express that sort of thing.


# row polymorphism

```haskell
type Foo = { name :: String, age :: Int }
type Bar = { name :: String }

asdf 
  :: forall fields. { name :: String | fields } 
  -> IO Unit
asdf r = print r.name
```

Note:

Row polymorphism is similar to subtyping, but it requires explicit polymorphism rather than implicit subtyping rules.
This allows us to say that we can write a function `asdf` that accepts *any* record that has *at least* a `name` field with type String.
Subtyping tends to make type systems get really weird and complex, so the ability to work polymorphically over records like this without subtyping is a huge win.


# adding/dropping fields

```javascript
function extendRecord(obj, field, value) {
    obj[field] = value;    
}

function noMoreName(obj) {
    delete obj.name;
}
```

lol nevermind that
<!-- .element: class="fragment" -->

Note:

We'd also like to be able to potentially add and drop fields to records.
If we're making the comparison to tuples, vectors, dictionaries, and hashes, then I'd expect that we can do similar things.
PureScript lets us do this quite nicely:


# +/- Fields

```haskell
-- PureScript is awesome
foo :: forall fields
     . { | fields }               -- input record
    -> { tag :: String | fields } -- output record
foo = insert (SProxy :: SProxy "tag") "Hello!"
```

Note:

PureScript actually implements this as a library, so it's not super baked in.
Elm also had this feature, but then dropped it for some reason.
We can easily do this in dynamic languages like JavaScript and Ruby, but we want better for our records in typed languages.

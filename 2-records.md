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

$$ foo = \\{ name : String, age : Int \\} $$

$$ bar = \\{ age : Int, name : String \\} $$

$$foo_{name} : String$$

$$foo_{age} : Int$$

Note:

A record is a tuple, but instead of using positional indexing, we use string labels.
This has some interesting consequences.
Records do not have a notion of "ordering" -- one label isn't "before" another.
The record types foo and bar here are the same type.

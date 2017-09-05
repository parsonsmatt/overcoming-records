## Let's Write A

# Record Library

Note:

So, let's write one!
We'll get our hands dirty with some fun type level programming. 


# Underlying Representation

Note:

The most important bit is the underlying representation.
There's a great library called `vinyl` which was the first effort to support extensible records.


# `vinyl`

```haskell
-- | A record is parameterized by a universe @u@, an 
-- interpretation -- @f@ and a list of rows @rs@. The 
-- labels or indices of the record are given by 
-- inhabitants of the kind @u@; the type of values at any 
-- label @r :: u@ is -- given by its interpretation 
-- @f r :: *@.
data Rec :: (u -> *) -> [u] -> * where
  RNil :: Rec f '[]
  (:&) :: !(f r) -> !(Rec f rs) -> Rec f (r ': rs)
```

Note:

This is *really* fancy.
Unfortunately, it's also a singly linked list.
This sort of thing is "correct by construction", which means that if it compiles, it works.
This is both slow and boring.
I don't know about you, but I prefer to live on the wild side.


# `hash-rekt`

```haskell
newtype HashRecord' (k :: [*])
    = HashRecord
    { getHashRecord :: HashMap String Dynamic
    }
```

https://www.github.com/parsonsmatt/hash-rekt

Note:

Records are, morally speaking, maps from strings to values, so let's just do that.
We're going to dig into the highly unsafe and somewhat hilarious record library I wrote.
Along the way, we're going to learn about some neat type level tricks.

Haskell actually has pretty great support for dynamic types, as we'll find out!


# dont

don't actually use hash-rekt

use this instead:

https://github.com/agrafix/superrecord

Note:

Don't actually use this.
I'm probably not going to maintain it.
There's a much more performant library called superrecord which is implemented on top of an even more unsafe foundation.
But it's actually fast and well maintained.


# interface

- `empty`
- `insert`
- `lookup`
- `delete`
- `modify`
- `union`
- `intersect`
- position independent labels!

Note:

We'll want to support an interface that mostly resembles HashMap, but with some added type safety.
Furthermore, it's important that we don't have to worry about the ordering of the labels.
So let's write our first bit.
The empty record!


```haskell
empty :: HashRecord '[]
empty = HashRecord Map.empty
```

Note:

Well, that's easy. We just wrap a hashmap with the right type constructor.
One unfortunate thing about this design is that the type variable is *totally phantom*.
We have no way of asserting the type level list actually corresponds with the contents of the map.
Let's move on to insert. That's where it starts getting fun.


```haskell
insert
  :: forall (label :: Symbol) (value :: *) fields
   . label
  -> value
  -> HashRecord fields
  -> HashRecord (Insert label value fields)

-- used like
... insert "foo" 5 someRecord ...
```

Note:

This is basically the interface we want.
We take a type level string (known as a Symbol), a value, and the record to insert it into.
Then, we say that the type of the new fields has the label and value inserted into it.
Unfortunately, this does not work.
Label is not a value, it's a type, so we can't accept it as an argument.


```haskell
insert
  :: forall (label :: Symbol) (value :: *) fields
   . Proxy label
  -> value
  -> HashRecord fields
  -> HashRecord (Insert label value fields)

-- used like
... insert (Proxy :: Proxy "foo") 5 someRecord ...
```

Note:

Haskell does have a Proxy type, which simply carries a phantom type around.
This can be used to pass type level values to functions.
However, GHC 8 has a new extension called TypeApplications which we can use instead!
It cuts out the Proxy boilerplate:


```haskell
insert
  :: forall (label :: Symbol) (value :: *) fields
   . value
  -> HashRecord fields
  -> HashRecord (Insert label value fields)

-- used like
... insert @"foo" 5 someRecord ...
```

Note:

Type applications is a double edged sowrd.
The ordering of your type variables has now become part of your public API.
Additionally, you'll need to enable the AllowAmbiguousTypes extension to actually get this to compile.
Furthermore, type applications are not first class and can't be passed as arguments.
Anyway, let's get to the meat of this function:


```haskell
insert
  :: forall (label :: Symbol) (value :: *) fields
   . value
  -> HashRecord fields
  -> HashRecord (Insert label value fields)
insert value (HashRecord rec0) = 
  let 
    dynVal = toDyn value 
    label = symbolVal (Proxy @label)
    rec1 = Map.insert label dynVal rec0
  in 
    HashRecord rec1
```

Note:

We unwrap the old record, discarding the type information.
We convert the value to a Dynamically typed value.
Finally, we insert it into the record, keyed by the Label.
Unfortunately, this doesn't compile.
We need some constraints:


```haskell
insert
  :: forall (label :: Symbol) (value :: *) fields
   . (Typeable value, KnownSymbol label)
  => value
  -> HashRecord fields
  -> HashRecord (Insert label value fields)
insert value (HashRecord rec0) = 
  let 
    dynVal = toDyn value 
    label = symbolVal (Proxy @label)
    rec1 = Map.insert label dynVal rec0
  in 
    HashRecord rec1
```

Note:

In order to stuff the value into a Dynamic wrapper, we need to use the Typeable class to carry that runtime type information.
In order to know the string value of the label, we require that the Symbol is an instance of KnownSymbol.
That lets us reflect the string out of the type.
The last piece of the puzzle is that Insert type thing.


# type families

Note:

Haskell has functions from values to values.
These are ordinary functions.
Haskell *also* has functions from types to values.
These are type classes.
Haskell has functions from types to types.
These are called type families for some reason.
Haskell does not have functions from values to types -- these would enable dependently typed programming.

Programming in type families is like programming in a bare bones, strict, purely functional programming language, with no let, where, or other niceties that you're used to.
It sucks.
But it's the best we've got in Haskell land, so we're stuck.


```haskell
data (k :: Symbol) =: (v :: *)

type family Insert key val fields where
  Insert k v '[] =
    '[key =: val]
  Insert k1 a (k2 =: b ': xs) =
    InsertHelper k1 a k2 b xs (CmpSymbol k1 k2)    

type family InsertHelper k1 v1 k2 v2 fields cmp where
  InsertHelper k1 v1 k2 v2 xs 'EQ =
    k1 =: v1 ': xs
  InsertHelper k1 v1 k2 v2 xs 'LT =
    k1 =: v1 ': k2 =: v2 ': xs
  InsertHelper k1 v1 k2 v2 xs 'GT =
    k2 =: v2 ': Insert k1 v1 xs
```

Note:

First, we need a type to represent the pairing of a symbol and a type.
Thus, the equals-colon symbol.
The tick is used to *promote* data constructors from the value to the type level.
These are type level lists and comparisons we're working with!

In order to preserve the position independence invariant, this has to be an insertion sort.
Type families do not have case expressions or if, so we have to write this helper function to decide what to do at each step in the list.


```haskell
type family If cond t f where
  If 'True  t _ = t
  If 'False _ f = f
```

Note:

Why don't we have if?
Remember how I said type families are strict?
If we write this, then it will have to solve or evaluate everything on both sides of the If.
For virtually any type level programming, this causes insane blowups in the compile time.
Alright, let's move to the next function: lookup!


```haskell
lookup 
  :: forall label value fields
   . (Typeable value, KnownSymbol label, ???)
  => HashRecord fields
  -> value
lookup (HashRecord rec) =
  let 
    label = symbolVal (Proxy @label)
    mdyn  = Map.lookup label rec
    dyn   = fromMaybe (error "lmao types") mdyn
    mval  = fromDynamic dyn
    val   = fromMaybe (error "lmao types") mval
  in 
    val
```

Note:

Okay, so heres lookup.
There are two unsafe bits here.
The first is looking up the label in the map.
We're pretty sure that the types guarantee that the label is present, so we can safely error in the Nothing case.
Likewise, we're pretty sure that the types guarantee that the value is the type we claim it to be.
Then we just return it. Nice!

Now, how do we make those types do those things?
We need to assert that the fields type contains the label.
FUrthermore, we need to know what value is contained in the fields.
So let's write a lookup:


```haskell
type family Lookup key fields where
  Lookup key '[] = 'False
  Lookup key (key =: _ ': rest) = 'True
  Lookup key (_ ': rest) = Lookup key rest
```

Note:

OK, so type families have one neat aspect: if you pattern match on two types and give them the same name, then the match only succeeds if the two types are equal.
This is cool.
If we reach the end of the map, then we return type-level False.
If we find it, we return True.
Otherwise, we recurse.
This solves the issue of "does the key exist in the map."
But it does not solve the question of "is the value the right type?"


```haskell
type family Lookup key fields where
  Lookup k '[] = TypeError (
    'Text "The key \"" ':<>: 'Text k 
    ':<>: 'Text "\" did not exist in the map."
    ':$$: 'Text "Thus, we can't get the value out of it."
    )
  Lookup k (k =: a ': xs) = a
  Lookup k (x =: b ': xs) = Lookup k xs
```

Note:

This version actually returns the type that the symbol points to.
It raises a GHC Type Error if we look up on an empty list.
This is a nice little facility that we can use in order to provide good meaningful error messages to our users.


```haskell
lookup 
  :: forall label value fields. 
   ( Typeable value
   , KnownSymbol label
   , Lookup label fields ~ value
   )
  => HashRecord fields
  -> value
lookup (HashRecord rec) =
  let 
    label = symbolVal (Proxy @label)
    mdyn  = Map.lookup label rec
    dyn   = fromMaybe (error "lmao types") mdyn
    mval  = fromDynamic dyn
    val   = fromMaybe (error "lmao types") mval
  in 
    val
```

Note:

Now, with our type family in hand, we've got a fully safe lookup function.
We use the tilde to indicate type equality: when we do a lookup in the map, the type system will verify both that the label exists in the map and that the value is the right type.
Onwards, to the next member in our interface: delete!


```haskell
delete
  :: forall label value fields. 
   ( Typeable value
   , KnownSymbol label
   )
  => HashRecord fields
  -> HashRecords (Remove label fields)
delete (HashRecord r) = HashRecord r
```

Note:

Hah!
We don't actually delete anything.
This is where we kind of give the lie to our interface.
The type safety is just a thin veneer over utter dynamically typed madness.
Let's write that remove type family and be done with this.


```haskell
type family Remove label labels where
  Remove key '[]              = '[]
  Remove key (key =: _ ': xs) = xs
  Remove key (oop =: _ ': xs) = Remove k xs
```

Note:

Easy-peasy.
The code isn't hard to write, but man, it really sucks.
It's very annoying to write higher order functions, since type families and type synonyms must be fully applied, so you end up writing a thousand of these little stupid functions that should be abstractable, but not.
The main virtue of dependent haskell, when it happens, is that it will obviate the need to know about this crap.


# instances!

# gimme the type classes 

## pls

Note:

Alright, so we've defined the core of the interface.
If you're interested, you can lookup how I wrote modify, union, interset, etc and even provide fully type safe polymorphic lenses into these dudes.
It's fun, but it's a little painful.

There's another thing we often want to do with types: give them type class instances!
And how we do that with this weird type level programming thing isn't exactly obvious.
We're going to rely on a thing called "type class induction."
First, we'll define Eq.


equalizer

```haskell
instance Eq (HashRecord '[]) where
  _ == _ = True
```

<pre><code class="lang-haskell hljs" data-trim data-noescape>
instance 
  ( <span class="fragment">Eq v
  , Typeable v
  , KnownSymbol k
  , Eq (HashRecord xs)</span>
  ) => Eq (HashRecord (k =: v ': xs)) where<span class="fragment">
  r0@(HashRecord rec0) == r1@(HashRecord rec1) =
    let this = <span class="fragment">lookup @k r1 == lookup @k r2</span>
        rest = <span class="fragment">(HashRecord rec0 :: HashRecord xs) 
               == (HashRecord rec1 :: HashRecord xs)</span>
    in this && rest</span>
</code></pre>
<!-- .element: class="fragment" -->

Note:

This is the base case of our induction: the empty record!
Two empty records are always equal.
That was easy.

The harder part comes in with the inductive case. (next)
First, we pattern match on the list in the instance type. (next)
We know we're going to need a whole bunch of these constraints in order to do lookups.
We know we're going to need to be able to compare the values for equality. 
And finally, we know we're going to need to recurse down the list. (next)
Finally, we implement. We know that "this" element in the record and the rest of the record must both be equal. (next)
For this, we lookup the value in both records, and compare for equality.
For rest, we delegate to the Eq instance for the rest of the record.

Again, note that I'm not deleting anything out of the hashmap.
I'm just shrinking the size of the record as far as the types are concerned.
If we were going to be deleting entries out of the record, that'd end up doing a ton of copying, and that'd suck for performance.


show me the money

```haskell
instance ToStringMap (HashRecord xs) 
  => Show (HashRecord xs) where
      show = show . toStringMap
```

```haskell
class ToStringMap a where
  toStringMap :: a -> HashMap String String

instance ToStringMap (HashRecord '[]) where
  toStringMap _ = Map.empty

instance (KnownSymbol k, Typeable v, Show v
  , ToStringMap (HashRecord xs)) 
  => ToStringMap (HashRecord (k =: v ': xs)) where
  toStringMap rec@(HashRecord r0) =
    Map.insert 
      (symbolVal (Proxy @k)) 
      (show (lookup @k rec))
      (toStringMap (HashRecord r0 :: HashRecord xs))
```
<!-- .element: class="fragment" -->

Note:

Another really common pattern when working with these types is to define helper classes.
If we want to show a hashrecord, it makes sense to convert it to an easier representation.
After all, we know we have a Map from string to dynamic.
SO if we just show the values in the fields, then we can easily delegate to the map's show instance.


jsonify me, captain

```haskell
instance ToPairs (HashRecord xs) 
  => ToJSON (HashRecord xs) where
  toJSON = object . toPairs

class ToPairs a where
  toPairs :: a -> [Pair]

instance ToPairs (HashRecord '[]) where
  toPairs _ = []

instance ( ToPairs (HashRecord xs) 
         , KnownSymbol k, Typeable v, ToJSON v
         ) => ToPairs (HashRecord (k =: v ': xs)) where
  toPairs rec@(HashRecord r0) =
    let val  = toJSON (lookup @k rec)
        this = Text.pack (symbolVal (Proxy @k)) .= val
        rest = toPairs (HashRecord r0 :: HashRecord xs)
     in this : rest
```

Note:

Converting to and from JSON is a pretty reasonable thing to want from these extensible records.
Let's write those instances now.
We'll delegate to a helper class, again.
We know we'll need to be able to convert each element in the record.

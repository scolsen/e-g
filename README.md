# e-g: Datatypes by Example

**Note: This is not an officially supported Google product.**

e-g lets you define Carp datatypes by using an example value of its component
parts. It helps you ensure the internal representations or data models of other
libraries don't leak into your code while allowing you to reference those
models. Let's look at an example.

Suppose you were using a simple banking API, that defines an `Account` data type
and a few functions for operating on that datatype:

```clojure
(deftype Account [name String balance Int pin Secret])

(module Bank
  (sig open-account (Fn [] Account))
  (sig add-funds (Fn [Account Int] Account))
)
```

Let's say you want to augment an account with additional information, such as an
account history, and you'll use a new datatype to do so. Normally, you'd have to
define it like this:

```clojure
(load Bank)

(deftype Account+ [account Bank.Account history (Array Int)])

(defn deposit-and-record-balance [deposit account]
  (let [account- (Account+.account account)
        old-history (Account+.history account)]
    (do (Account+.set-account account (Bank.add-funds account deposit))
        (Account.+set-history account (push-back old-history deposit)))))
```

Great! But there's a small problem. The `Account+.history` field relies on an
assumption about how Accounts are modeled. Let's now imagine the `Bank` module
designers discover an `Int` is not a good representation of a `balance` after
all. To your surprise, they go ahead and change their `Account` datatype to use
a `Double` instead:

```clojure
(deftype Account [name String balance Double])
```

Unfortunately, your `Account+` type had a direct reference to the type of
`balance`, so this breaks your code, you'll have to use a new conversion
function instead:

```clojure
(deftype Account+ [account Bank.Account history (Array Int)])

(sig convert (Fn [Int] Double))

(defn deposit-and-record-balance [deposit account]
  (let [account- (Account+.account account)
        old-history (Account+.history account)]
    (do (Account+.set-account account (Bank.add-funds account deposit))
        ;; Account.+ history wants an Int, but account balances are now Doubles!
        (Account.+set-history account (push-back old-history deposit)))))
```

e-g helps you avoid this problem. With e-g, types can depend on the *values* of
`Account` rather than the internal details of its type signature, allowing us to
rely on deliberately exposed API functions to define types that depend on
`Account`. Here's how we'd define `Account+` using e-g:

```clojure
(load Bank)

(e-g.product Account+ (account (Bank.open-account)) (history [(e-g.value-of (e-g.select-from Account balance))]))
```

The prior call to `e-g.product` will produce a product type (struct) with the
definition:

```clojure
(deftype Account+ [account Account history (Array Int)])
```

Contrary to the explicit datatype specification, because the `e-g` definition of
`Account+` relies on the functions that return accounts, it changes seamlessly
with upstream changes to the internal specification of `Account`:

```clojure
;; In the Bank module, someone changes Int to Double:
(deftype Account [name String balance Double])

;; Since we used e-g, our `Account+` changes accordingly!
(e-g.product Account+ (account (Bank.open-account)) (history [(e-g.value-of (e-g.select-from Account balance))])])
=> (deftype Account+ [account Account history (Array Double)])
```

Generally speaking, using `e-g` allows you to refer directly to the components
of someone else's types in your own type definitions without risking the
concomitant brittleness. Likewise, it can help you produce correct function
signatures when desired:

```clojure
;; assuming Account.balance is still a Double
(e-g.signature withdraw [(e-g.select-from Account balance)] (e-g.select-from Account balance))
(meta withdraw "sig")
=> (Fn [Double] Double)
```

`e-g` also provides functions for generating sumtypes, interfaces, and
`register` calls from example values. See the [docs](doc/index.html) for more
information.

## Polymorphic Types

e-g has minimal support for defining polymorphic types--the notion of defining
types based on concrete values is inherently opposed to retaining polymorphism
to some extent. That said, any value that itself returns a polymorphic type will
retain its polymorphism. An empty array is a good example of this:

```clojure
(e-g.product Foo (things []))
=> (deftype Foo [things (Array a)])
```

The empty array is polymorphic, but Carp won't accept this definition since the
type variable `a` isn't captured in the head of the definition. `e-g` won't add
polymorphic variables to the head of type definitions automatically. We can fix
the above example by adding an explicit type variable to the head of the name of
the type:

```clojure
(e-g.product (Foo a) (things []))
=> (deftype (Foo a) [things (Array a)])
```

Carp will accept this definition. It's important to note that e-g's type
resolution always maps type variables to quantities--in other words, the type
variable names used in a generated definition will *always* depend on a type's
arity. Consider an empty map, for instance, which has two type inhabitants:

```clojure
(e-g.product (Foo a b) [map {}])
=> (deftype (Foo a b) [map (Map a b)])
```

Since `Map` is a type constructor that takes two arguments, its type variables
are named `a` and `b` respectively. So, we can safely refer to both of these
variables in the head of the type definition. A type constructor with three
variables would generate a type signature like, `(T a b c)`, with four `(T a b c
d)` and so on until the alphabet is exhausted.

One can also define a single uniquely polymorphic member in types generated by
e-g using `e-g.any` which will always resolve to the type variable `a`:

```clojure
(e-g.product (Foo a) [thing e-g.any])
=> (deftype (Foo a) [thing a])
```

Unfortunately, e-g doesn't support introducing any more than a single
polymorphic variable, so the following example that attempts to refer to
different calls to `e-g.any` by different variable names won't work:

```clojure
(e-g.product (Foo a b) (thing-one e-g.any) (thing-two e-g.any))
=> (deftype (Foo a) [thing-one a thing-two a])
```

If you really need significant polymorphic support for a type generated by e-g,
the best you can do is embed the constructor for the type:

```clojure
(deftype (Poly a b c d) [x a y b z c q d])
(e-g.product (Foo a b c d) (nums (Map.init 1 1)) (polymap {}) (poly Poly.init))
=> (deftype (Foo a b c d) [nums (Map Int Int) polymap (Map a b) poly (Fn [a b c d] (Poly a b c d))])
```

You could then construct a `Foo` as follows:

```clojure
(Foo.init {1 1} {} (fn [a b c d] (Poly.init 1 1 1 1)))
```

Which will always return a `(Foo Int Int Int Int)`. In fact, if *any* of the
types in Foo's are generic when Carp attempts to construct it, Carp will
complain. So, if you want particular concrete instances of some polymorphic
type, it's much simpler to pass the constructed value itself.

Note that types like `Foo` also have a *dominant type* that will constrain the
types of other members, for example:

```clojure
(Foo.init {1 \a} {} (fn [a b c d] (Poly.init 1 1 1 1)))
```

Won't work, since we're simultaneously telling the compiler that the variable
`b` is both a `Char` and `Int`.

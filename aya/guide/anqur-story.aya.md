# Motivating Aya features: an incomplete story

Great. I expect you to have basic experience with interactive theorem proving.
This is another Aya tutorial for interactive theorem prover users.
If you find a bug, open an issue on GitHub!

This tutorial will use some basic Aya syntax.
I hope those are sufficiently intuitive, or you can look up [this tutorial](haskeller-tutorial).

Here's a little prelude, which you do not need to understand now.

```aya
prim I prim coe prim coeFill
prim intervalInv
inline def ~ => intervalInv
variable A B : Type
def infix = (a b : A) => [| i |] A { i := b | ~ i := a }
def refl {a : A} : a = a => fn i => a

open data Nat
| zero
| suc Nat
```

## A journey begins

You went to a website and a random person has asked the following question:

```aya
open data Bool | true | false
def not Bool : Bool
| true => false
| false => true

def id (x : Bool) => x

def Goal => (fn x => not (not x)) = id

// And yes, below is the syntax for typed holes in Aya:
// def question : Goal => {??}
```

You realize that there is no way to prove it in your type theory.
However, you are very smart and realized you can instead show the following:

```aya
def Goal' (x : Bool) => not (not x) = id x
```

This is pretty much the same theorem!

Now, suppose we need to show a propositional equality between two records.
This means we have to show they're memberwise equal.
One record has a member `\x => not (not x)`{}, and the other has `id`{}.
This time, you cannot cheat by changing the goal type.
You post the question on some mailing list and people are telling you that
the alternative version of the theorem you have shown does not imply the
original, unless "function extensionality" is a theorem in your type theory.

To have function extensionality as a theorem, you came across two distinct
type theories: observational type theory and cubical type theory.
Aya chose the latter (for now).

## Cubical

Here's the proof of function extensionality in Aya:

```aya
def funExt (f g : A -> B) (p : ∀ a -> f a = g a) : f = g
   => \i a => p a i
```

This is because Aya has a "cubical" equality type that is not inductively defined.
We may also prove the action-on-path theorem, commonly known as `cong`, but
renamed to `pmap`{} to avoid a potential future naming clash:

```aya
def pmap (f : A -> B) {a b : A} (p : a = b) : f a = f b
   => \i => f (p i)
```

We may also invert a path:

```aya
def sym {a b : A} (p : a = b) : b = a => \i => p (~ i)
```

However, we cannot yet define transitivity of equality because we do not have the
traditional elimination rule of the equality type -- the `J` rule.
This will need some advanced proving techniques that are beyond the scope of this
simple tutorial, so I'll skim them. First, we need type-safe coercion:

```aya
def cast (p : A ↑ = B) : A -> B => (\i => p i).coe
```

Then, from `q : b = c` we construct the equivalence `(a = b) = (a = c)`
and coerce along this equivalence:

```aya
def concat {a b c : A} (p : a = b) (q : b = c) : a = c =>
  cast (\i => a = q i) p
```

Note that at this point you can already do a bunch of familiar proofs about
some simple types such as natural numbers or sized vectors.
These are left as exercises, and you are encouraged to try yourself if you are not
very sure about how it feels to prove things in Aya.

## Jesper's master thesis

Here's a bonus feature.
Remember the `+-comm` proof that you need two lemmas?
It is standard to define `+` in the following way:

```aya
example def infix + Nat Nat : Nat
| 0, n => n
| suc m, n => suc (m + n)
```

And then you prove that `a + 0 = a` and `a + suc b = suc (a + b)`.
It is tempting to have `| n, 0 => n` as a computation rule as well,
but this is incompatible with the usual semantics of pattern matching,
which is compiled to elimination principles during type checking.
However, you _can_ do that in Aya. You may also add the other lemma as well.

```aya
overlap def infix + Nat Nat : Nat
| 0, n => n
| n, 0 => n
| suc m, n => suc (m + n)
| m, suc n => suc (m + n)
tighter =
```

This makes all of them definitional equality.
So, `+-comm`{} can be simplified to just one pattern matching:

```aya
def +-comm (a b : Nat) : a + b = b + a
| 0, _ => refl
| suc _, _ => pmap suc (+-comm _ _)
```

## Heterogeneous equality

When working with indexed families, you may want to have heterogeneous equality
to avoid having mysterious coercions.
For example, consider the associativity of sized vector appends.
We first need to define sized vectors and the append operation:

```aya
variable n m o : Nat
// Definitions
open data Vec (n : Nat) (A : Type)
| 0, A => nil
| suc n, A => infixr :< A (Vec n A)
overlap def infixr ++ (Vec n A) (Vec m A) : Vec (n + m) A
| nil, ys => ys
| ys, nil => ys
| x :< xs, ys => x :< xs ++ ys
tighter :< =
```

It is tempting to use the below definition:

```
overlap def ++-assoc (xs : Vec n A) (ys : Vec m A) (zs : Vec o A)
  : (xs ++ ys) ++ zs = xs ++ (ys ++ zs)
| nil, ys, zs => refl
| x :< xs, ys, zs => pmap (x :<) (++-assoc xs ys zs)
```

However, this definition is not well-typed:

+ `(xs ++ ys) ++ zs` is of type `Vec ((n + m) + o) A`{}
+ `xs ++ (ys ++ zs)` is of type `Vec (n + (m + o)) A`{}.

They are not the same!
Fortunately, we can prove that they are propositionally equal.
We need to show that natural number addition is associative,
which is the key lemma of this propositional equality:

```aya
def +-assoc {a b c : Nat} : (a + b) + c = a + (b + c)
| {0} => refl
| {suc _} => pmap suc +-assoc
```

Now we can work on the proof of `++-assoc`.
Here's a lame definition that is well-typed in pre-cubical type theory,
and is also hard to prove -- we `cast`{} one side of the equation to be other side:

```aya
example def ++-assoc-ty (xs : Vec n A) (ys : Vec m A) (zs : Vec o A)
  => cast (↑ pmap (\n => Vec n A) +-assoc) ((xs ++ ys) ++ zs) = xs ++ (ys ++ zs)
```

It is harder to prove because in the induction step, one need to show that
`cast (↑ pmap (\n => Vec n A) (+-assoc {n} {m} {o}))`{implicitArgs=false}
is equivalent to the identity function in order to use the induction hypothesis.
For the record, here's the proof:

```aya
def castRefl (a : A) : cast ↑ refl a = a => sym ((\i => A).coeFill a)
```

But still, with this lemma it is still hard.
Cubical provides a pleasant way of working with heterogeneous equality:

```aya
def Path (A : I -> Type) (a : A 0) (b : A 1) => [| i |] A i { i := b | ~ i := a }
```

So if we have `X : A = B` and `a : A`, `b : B`, then `Path (\i => X i) a b` expresses the heterogeneous
equality between `a` and `b` nicely.

We may then use the following type signature:

```aya
def ++-assoc (xs : Vec n A) (ys : Vec m A) (zs : Vec o A)
  => Path (\i => Vec (+-assoc i) A) ((xs ++ ys) ++ zs) (xs ++ (ys ++ zs))
```

The proof is omitted (try yourself!).

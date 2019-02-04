# Incrementally solving the `Applicative` instance for `Compose`

Here at Stacktrace, our team have been working our way through the incredible (and incredibly challenging) [data61 course](https://github.com/data61/fp-course). Last week we started working on the `Applicative` instance for `Compose`, and boy, it is _much_ harder to implement than you might expect.

Lets have a look:

``` Haskell
newtype Compose f g a = Compose (f (g a))

instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  pure :: a -> Compose f g a
  (<*>) :: Compose f g (a -> b) -> Compose f g a -> Compose f g b
```

There isn't much going on, right? Just some `Compose`, `f`, `g`, `a` and `b`.

Well, I was in agony for what felt like a brain-melting hour before my colleague put me out of my misery and solved the hardest part for me. Just another day in the life of learning Haskell.

In fairness, `pure` is easy enough.

``` Haskell
instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  pure :: a -> Compose f g a
```

We just need to put an `a` in a `g`, and then put that `g a` in an `f`. Given `g` and `f` are `Applicatives`, we can call `pure` to create one of each `Applicative` and then wrap the result in `Compose`: 
``` Haskell
pure a = Compose $ pure $ pure a
```

Or, if we're feeling a bit point free:
``` Haskell
pure = Compose . pure . pure
```

In contrast `<*>` ("apply", or more whimsically, "spaceship") is _hard_.

Lets look at that signature again:

``` Haskell
instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  (<*>) :: Compose f g (a -> b) -> Compose f g a -> Compose f g b
```

It doesn't look that hard, right? The problem is this: how on earth do you reach through `f` __and__ `g`, to get hold of `a -> b` and `a`, so that you can actually apply them to each other and end up with `f (g b)`. 

For the life of me, I couldn't figure out how to solve this incrementally. That left me trying to solve it all at once, _in my head_. I failed.

To practice being more piecemeal with Haskell, let's try to walk through this in a way that my past self would have appreciated. 

To start, lets write the easy stuff. Since `Compose` is a `newtype` wrapper, we need to unwrap the contents of our two `Compose` arguments, and then put the result back in `Compose` at the end:

``` Haskell
instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  (<*>) :: Compose f g (a -> b) -> Compose f g a -> Compose f g b
  (Compose fga2b) <*> (Compose fga) = Compose $ _fgb
```
with the types:
``` Haskell
fga2b :: f (g (a -> b)) 
fga :: f (g a)
_fgb :: f (g b) 
```

The good news is that we're now just dealing with `f`, `g`, `a` and `b`. The bad news is that it's not obvious what to do next. 

Lets try to break the problem down. We need to figure out what `_fgb` should be. Given `f` and `g` are `Applicative`s, we have four choices to start us off (ignoring `pure`, since the last thing we need is _more_ nested `Applicatives`):
1) `_fgb = _ <*> fga2b`
2) `_fgb = _ <*> fga`
3) `_fgb = _ <$> fga2b`
4) `_fgb = _ <$> fga`

It can't be `<$>` ("fmap"), as we'll end up inside `f` twice, once for `fga` and once for `fga2b`. So it must be `<*>`:

``` Haskell
_fgb :: f (g b)
_fgb = _ <*> _
```

Whatever `_fgb` is, it needs to have a type of `f (g b)`. Given we're calling `<*>`, where `<*> :: f (a -> b) -> f a -> f b`, we probably need a non-function on the right. That means we have to start with option (3): `_fgb = ? <*> fga`. 

Now we're getting somewhere:

``` Haskell
instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  (<*>) :: Compose f g (a -> b) -> Compose f g a -> Compose f g b
  (Compose fga2b) <*> (Compose fga) = Compose $ _fga2gb <*> fga
```
with the types:
``` Haskell
fga2b :: f (g (a -> b)) 
fga :: f (g a)
_fga2gb :: f (g a -> g b)
```

We've already used `fga`, so that leaves us with `fga2b`. Can we transform `fga2b :: f (g (a -> b))` into `f (g a -> g b)`?

Lets try to do it in a separate function called `lessScary`:

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ lessScary fga2b <*> fga
  where
     lessScary :: f (g (a -> b)) -> f (g a -> g b)
     lessScary fga2b = _ohboy
```

This seems a little more manageable. We need to transform the contents of `f` from `g (a -> b)` into `g a -> g b`. That feels like a job for `<$>`!

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ lessScary fga2b <*> fga
  where
     lessScary :: f (g (a -> b)) -> f (g a -> g b)
     lessScary fga2b = _wtf <$> fga2b
```
with the types:
``` Haskell
_wtf :: g (a -> b) -> (g a -> g b)
```

Yes yes yes! Lets break that `_wtf` out into another function:

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ lessScary fga2b <*> fga
  where
     lessScary :: f (g (a -> b)) -> f (g a -> g b)
     lessScary fga2b = wtf <$> fga2b
     wtf :: g (a -> b) -> g a -> g b
     wtf ga2b = _soclose
```

Note that I've dropped the brackets around `(g a -> g b)` in `g (a -> b) -> (g a -> g b)`, since that's actually the same as `g (a -> b) -> g a -> g b`.

You know what `g (a -> b) -> g a -> g b` looks like? Yup! It looks like `<*>` for `g`:

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ lessScary fga2b <*> fga
  where
     lessScary :: f (g (a -> b)) -> f (g a -> g b)
     lessScary fga2b = wtf <$> fga2b
     wtf :: g (a -> b) -> g a -> g b
     wtf ga2b = (<*>) ga2b
```

That's it! We did it.

We _can_ take things further though, and condense a bunch of this code.

First, lets drop `ga2b` from `(<*>)` in `wtf`, since it's the first parameter to both:

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ lessScary fga2b <*> fga
  where
     lessScary :: f (g (a -> b)) -> f (g a -> g b)
     lessScary fga2b = wtf <$> fga2b
     wtf :: g (a -> b) -> g a -> g b
     wtf = (<*>)
```

Yum!

Now, lets inline `wtf`:

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ lessScary fga2b <*> fga
  where
     lessScary :: f (g (a -> b)) -> f (g a -> g b)
     lessScary fga2b = (<*>) <$> fga2b
```

Nice!

`lessScary` can be inlined too:

``` Haskell
(Compose fga2b) <*> (Compose fga) = Compose $ ((<*>) <$> fga2b) <*> fga
```

And there we have it. The `Applicative` instance for `Compose`. In incremental steps.

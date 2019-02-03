The `Applicative` instance for `Compose` is much harder than it looks:

```
data Compose f g a = Compose (f (g a))

instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  pure :: a -> Compose f g a
  (<*>) :: Compose f g (a -> b) -> Compose f g a -> Compose f g b
```
I was in agony for what felt like a brain-melting hour before my colleague put me out of my misery, by solving the hardest part for me.

 `pure` is easy enough. We just need to put `a` in a `g` and then in an `f`. `g` and `f` are `Applicatives`, so we can call `pure`  on each `Applicative` and then wrap the result in `Compose`: 
```
pure a = Compose $ pure $ pure a
```
Or if we're feeling a bit point free:
```
pure = Compose . pure . pure
```

In contrast `<*>` ("apply", or more whimsically, "spaceship") is _hard_.

Lets look at that signature again:

```
instance (Applicative f, Applicative g) => Applicative (Compose f g a) where
  (<*>) :: Compose f g (a -> b) -> Compose f g a -> Compose f g b
```

It doesn't look that hard, right? The problem is this: it's not obvious how you can reach through `f` and `g` to get hold of `a -> b` and `a` so that you can actually apply them to each other and end up with `f (g b)`. I couldn't figure out how to solve this incrementally. That left me trying to solve it all at once, _in my head_. I failed.

To practice being more piecemeal with Haskell, let's try to walk through this in a way that my past self would have appreciated. 

To start, lets write the easy stuff. We need to unwrap the contents our our two `Compose` arguments, and then put the result back in `Compose` at the end:

```
(Compose fga2b) <*> (Compose fga) = Compose $ _
```
with the types:
```
_ :: f (g b) 
fga2b :: f (g (a -> b)) 
fga :: f (g a)
(Compose fga2b) <*> (Compose fga) :: f (g b)
```

The good news is that we're now just dealing with `f`, `g`, `a` and `b. The bad news is that it's not obvious what to do next. 

Lets try to break the problem down. Given `f` and `g` are `Applicative`s, we have four choices to start us off (ignoring `pure`, since the last thing we need is _more_ nested `Applicatives`):
1) `? <*> fga2b`
2) `? <*> fga`
3) `? <$> fga2b`
4) `? <$> fga`

It can't be `<$>` ("fmap"), as we'll end up inside `f` twice, once for `fga` and once for `fga2b`. So it must be `<*>`.

We need to return the type `f (g b)`. Given we're calling `<*>`, where `<*> :: f (a -> b) -> f a -> f b`), we probably need a non-function on the right. That means we have to start with option (3): `? <*> fga`. 

Now we're getting somewhere:

```
(Compose fga2b) <*> (Compose fga) = Compose $ _ <*> fga
```
with the types:
```
_ :: f (g a -> g b).
```

We've already used `fga`, so that leaves us to somehow transform `fga2b :: f (g (a -> b)` into `f (g a -> g b)`.

Lets break that out into a function:

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = _
```

This seems a little more manageable. We need to transform the contents of `f` from `g (a -> b)` into `g a -> g b`. That feels like a job for `<$>`!

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = _ <$> fga2b
```
with the types:
```
_ :: g (a -> b) -> (g a -> g b)
```

Yes yes yes! Lets break that `_` out into another function:

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = wtf <$> fga2b
     wtf :: g (a -> b) -> g a -> g b
     wtf ga2b = _
```

I've dropped the brackets around `(g a -> g b)`, since that's actually the same as `g (a -> b) -> g a -> g b`.

You know what `g (a -> b) -> g a -> g b` looks like? Yup! It looks like `<*>` for `g`:

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = wtf <$> fga2b
     wtf :: g (a -> b) -> g a -> g b
     wtf ga2b = (ga2b <*>)
```

That's it! We did it.

But there's a problem. When we come back and look at this code, we'll be able to just read it, and _understand_ it, just like that with no effort. Where's the fun in that?

Lets get our point free on:

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = wtf <$> fga2b
     wtf :: g (a -> b) -> g a -> g b
     wtf = <*>
```

Yum!

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = ((<*>) <$>) fga2b
```

Oh yea!

```
(Compose fga2b) <*> (Compose fga) = Compose $ omg fga2b <*> fga
  where
     omg :: f (g (a -> b)) -> f (g a -> g b)
     omg fga2b = (<*>) <$> fga2b
```

Now we're talking!

```
(Compose fga2b) <*> (Compose fga) = Compose $ ((<*>) <$> fga2b) <*> fga
```

And there we have it. The `Applicative` instance for `Compose`. In incremental steps.

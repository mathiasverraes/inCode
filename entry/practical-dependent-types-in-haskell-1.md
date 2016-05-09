Practical Dependent Types in Haskell: Type-Safe Neural Networks
===============================================================

(Originally posted by Justin Le [https://blog.jle.im/])

Whether you like it or not, programming with dependent types in Haskell
moving slowly but steadily to the mainstream of Haskell programming. In
the current state of Haskell education, dependent types are often
considered topics for “advanced” Haskell users. However, I can
definitely foresee a day where the ease of use of modern Haskell
libraries relying on dependent types as well as their ubiquitousness
forces programming with dependent types to be an integral part of
regular intermediate (or even beginner) Haskell education, as much as
Traversable or Maps.

he point of this post is to show some practical examples of using
dependent types in the real world, and to also walk through the “why”
and high-level philosophy of the way you structure your Haskell
programs. It’ll also hopefully instill an intuition of a dependently
typed work flow of “exploring” how dependent types can help your current
programs.

The first project in this series will build up to type-safe
**[artificial neural
network](https://en.wikipedia.org/wiki/Artificial_neural_network)**
implementations. Hooray!

There are other great tutorials I’d recommend online if you want to
explore dependent types in Haskell further, including [this great
servant
“tutorial”](http://www.well-typed.com/blog/2015/11/implementing-a-minimal-version-of-haskell-servant/).
Also, I should provide a disclaimer — I’m also currently exploring all
of this as I’m going along too. It’s a wild world out there. Join me and
let’s be a part of the frontier!

Neural Networks
---------------

[Artificial neural
networks](https://en.wikipedia.org/wiki/Artificial_neural_network) have
been somewhat of a hot topic in computing recently. At their core they
involve matrix multiplication and manipulation, so they do seem like a
good candidate for a dependent types. Most importantly, implementations
of training algorithms (like back-propagation) are tricky to implement
correctly — despite being simple, there are many locations where
accidental bugs might pop up when multiplying the wrong matrices, for
example.

However, it’s not always easy to gauge before-the-fact what would or
would not be a good candidate for adding dependent types to, and often
times, it can be considered premature to start off with “as powerful
types as you can”. So we’ll walk through a simple implementation
*without*, and see all of the red flags that hint that you might want to
start considering stronger types.

Edwin Brady calls this process “type-driven development”. Start general,
recognize the partial functions and red flags, and slowly add more
powerful types.

### The Network

![Feed-forward ANN
architecture](/img/entries/dependent-haskell-1/ffneural.png "Feed-forward ANN architecture")

We’re going to be implementing a feed-forward neural network, with
back-propagation training. These networks are layers of “nodes”, each
connected to the each of the nodes of the previous layer.

Input goes to the first layer, which feeds information to the next year,
which feeds it to the next, etc., until the final layer, where we read
it off as the “answer” that the network is giving us. Layers between the
input and output layers are called *hidden* layers.

Every node “outputs” a weighted sum of all of the outputs of the
*previous* layer, plus an always-on “bias” term (so that its result can
be non-zero even when all of its inputs are zero). Mathematically, it
looks like:

$$
y_j = b_j + \sum_i^m w_{ij} x_i
$$

Or, if we treat the output of a layer and the list of list of weights as
a matrix, we can write it a little cleaner:

$$
\mathbf{y} = \mathbf{b} + \hat{W} \mathbf{x}
$$

To “scale” the result (and to give the system the magical powers of
nonlinearity), we actually apply an “activation function” to the output
before passing it down to the next step. We’ll be using the popular
[logistic function](https://en.wikipedia.org/wiki/Logistic_function),
$f(x) = 1 / (1 + e^{-x})$.

*Training* a network involves picking the right set of weights to get
the network to answer the question you want.

Vanilla Types
-------------

We can store a network by storing the matrix of of weights and biases
between each layer:

``` {.haskell}
-- source: https://github.com/mstksg/inCode/tree/master/code-samples/dependent-haskell/NetworkUntyped.hs#L12-15
data Weights = W { wBiases :: !(Vector Double)
                 , wNodes  :: !(Matrix Double)
                 }
  deriving (Show, Eq)

```

Now, a `Weights` linking an *n*-node layer to an *m*-node layer has an
*m*-dimensional bias vector (one component for each output) and an
*m*-by-*n* node weight matrix (one column for each output, one row for
each input).

(We’re using the `Matrix` type from the awesome
*[hmatrix](http://hackage.haskell.org/package/hmatrix)* library for
performant linear algebra, implemented using blas/lapack under the hood)

A feed-forward neural network is then just a linked list of these
weights:

``` {.haskell}
-- source: https://github.com/mstksg/inCode/tree/master/code-samples/dependent-haskell/NetworkUntyped.hs#L17-19
data Network = O !Weights
             | !Weights :&~ !Network
  deriving (Show, Eq)

```

So a network with one input layer, two inner layers, and one output
layer would look like:

``` {.haskell}
ih :&~ hh :&~ O ho
```

The first component is the weights from the input to first inner layer,
the second is the weights between the two hidden layers, and the last is
the weights between the last hidden layer and the output layer.

<!-- TODO: graphs using diagrams? -->
We can write simple procedures, like generating random networks:

``` {.haskell}
-- source: https://github.com/mstksg/inCode/tree/master/code-samples/dependent-haskell/NetworkUntyped.hs#L39-49
randomWeights :: MonadRandom m => Int -> Int -> m Weights
randomWeights i o = do
    s1 <- getRandom
    s2 <- getRandom
    let wB = randomVector s1 Uniform o * 2 - 1
        wN = uniformSample s2 o (replicate i (-1, 1))
    return $ W wB wN

randomNet :: MonadRandom m => Int -> [Int] -> Int -> m Network
randomNet i [] o     =     O <$> randomWeights i o
randomNet i (h:hs) o = (:&~) <$> randomWeights i h <*> randomNet h hs o

```

(`randomVector` and `uniformSample` are from the *hmatrix* library,
generating random vectors and matrices from a random `Int` seed. We
configure them to generate them with numbers between -1 and 1)

And now a function to “run” our network on a given input vector,
following the matrix equation we wrote earlier:

``` {.haskell}
-- source: https://github.com/mstksg/inCode/tree/master/code-samples/dependent-haskell/NetworkUntyped.hs#L23-37
logistic :: Double -> Double
logistic x = 1 / (1 + exp (-x))

runLayer :: Weights -> Vector Double -> Vector Double
runLayer (W wB wN) v = wB + wN #> v

runNet :: Network -> Vector Double -> Vector Double
runNet (O w)      !v = cmap logistic (runLayer w v)
runNet (w :&~ n') !v = let v' = cmap logistic (runLayer w v)
                       in  runNet n' v'

```

(`#>` is matrix-vector multiplication)

<!-- TODO: examples of running -->
If you’re a normal programmer, this might seem perfectly fine. If you
are a Haskell programmer, you should already be having heart attacks.
Let’s imagine all of the bad things that could happen:

-   How do we even know that each subsequent matrix in the network is
    “compatible”? We want the outputs of one matrix to line up with the
    inputs of the next, but there’s no way to know unless we have “smart
    constructors” to check while we add things. But it’s possible to
    build a bad network, and things will just explode at runtime.

-   How do we know the size vector the network expects? What stops you
    from sending in a bad vector at run-time?

-   How do we verify that we have implemented `runLayer` and `runNet` in
    a way that they won’t suddenly fail at runtime? We write `l #> v`,
    but how do we know that it’s even correct…what if we forgot to
    multiply something, or used something in the wrong places? We can it
    prove ourselves, but the compiler won’t help us.

### Back-propagation

Now, let’s try implementing back-propagation:

``` {.haskell}
-- source: https://github.com/mstksg/inCode/tree/master/code-samples/dependent-haskell/NetworkUntyped.hs#L51-73
train :: Double -> Vector Double -> Vector Double -> Network -> Network
train rate x0 targ = fst . go x0
  where
    go :: Vector Double -> Network -> (Network, Vector Double)
    go !x (O w@(W wB wN))
        = let y     = runLayer w x
              o     = cmap logistic y
              dEdy  = cmap logistic' y * (o - targ)
              wB'   = wB - scale rate dEdy
              wN'   = wN - scale rate (dEdy `outer` x)
              w'    = W wB' wN'
              delWs = tr wN #> dEdy
          in  (O w', delWs)
    go !x (w@(W wB wN) :&~ n)
        = let y            = runLayer w x
              o            = cmap logistic y
              (n', delWs') = go o n
              dEdy         = cmap logistic' y * delWs'
              wB'          = wB - scale rate dEdy
              wN'          = wN - scale rate (dEdy `outer` x)
              w'           = W wB' wN'
              delWs        = tr wN #> dEdy
          in  (w' :&~ n', delWs)

```
# JAXnet

JAXnet is a neural net library for JAX. 
Other than popular neural net libraries, it is completely functional:
- No mutable weights in modules
- No global compute graph
- No global random key

This is an early version. Expect bugs, sharp edges and breaking changes!

# Overview

Defining networks will look similar to [tensorflow2/keras API](https://www.tensorflow.org/beta/guide/keras/functional):

```
net = Sequential([Dense(10), relu, Dense(10)])
```

`Dense` and `Conv` and `Sequential` are already supported.

To initialize parameter values for a network, call `init_params` on any module (with example inputs and a random key):

```
batch = np.zeros((3, 2))
params = net.init_params(batch, random.PRNGKey(0))
```

It initializes and returns all parameters, accessible via attributes:
```
print(params.layer.layers[0].bias) # [0.00212132 0.01169001 0.00331698 0.00460713]
```

To invoke the network with these `params`:
```
output = net(params, batch)
```

For acceleration use `jit`:

```
output = jit(net)(params, batch)
```

#Defining modules

Modules are functions decorated with `@parameterized`, with parameters defined through default values:

```
from jax import random, numpy as np
from jaxnet import parameterized, Param, glorot, randn

def Dense(out_dim, kernel_init=glorot(), bias_init=randn()):
    @parameterized
    def dense(inputs,
              kernel=Param(lambda inputs: (inputs.shape[-1], out_dim), kernel_init),
              bias=Param(lambda _: (out_dim,), bias_init)):
        return np.dot(inputs, kernel) + bias

    return dense
```

`Param` specifies parameter shape and initialization function. 
`@parameterized` transforms this function to allow usage as above.

# Nesting modules

Modules can be used in other modules through default arguments:

```
@parameterized
def net(inputs, layer1=Dense(10), layer2=Dense(20))
    inputs = layer1(inputs)
    return layer2(inputs)
```

Submodules can also be passed in through collections:

```
def Sequential(layers):
    @parameterized
    def sequential(inputs, layers=layers):
        for module in layers:
            inputs = module(inputs)
        return inputs

    return sequential
```

Arbitrarily nested `tuples`/`list`/`dicts` of modules work. (The same is true for `Param`s.)
Use of parameter-free functions is seamless:

```
def relu(input):
    return np.maximum(input, 0)

layer = Sequential([Dense(10), relu])
```

# Parameter sharing

Parameter sharing will be done by using module or parameter objects multiple times (not yet implemented):

```
shared_net=Sequential([layer, layer])
```

(As a workaround, internal parameter sharing already works:)

```
@parameterized
def shared_net(input, layer=layer):
    return layer(layer(input))
```

For training, you can use JAX' optimizers ([example usage](https://github.com/google/jax/blob/master/examples/mnist_classifier.py)).

# What about [stax](https://github.com/google/jax/blob/master/jax/experimental/stax.py)?
JAXnet is independent of stax.
The main motivation over stax is to simplify nesting modules:
 - Automating `init_params`: delegation to submodules, `output_shape` inference, `rng` passing
 - Seamless use of parameter-free functions as modules
 - Allowing streamlined module/parameter-sharing
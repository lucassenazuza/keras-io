# Introduction to Keras for Researchers

**Author:** [fchollet](https://twitter.com/fchollet)<br>
**Date created:** 2020/04/01<br>
**Last modified:** 2020/04/28<br>
**Description:** Everything you need to know to use Keras & TF 2.0 for deep learning research.


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/https://github.com/keras-team/keras-io/blob/master/guides/intro_to_keras_for_researchers/ipynb/intro_to_keras_for_researchers.py)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/https://github.com/keras-team/keras-io/blob/master/guides/intro_to_keras_for_researchers/intro_to_keras_for_researchers.py)



---
## Setup


```python
import tensorflow as tf
from tensorflow import keras

```

---
## Introduction

Are you a machine learning researcher? Do you publish at NeurIPS and push the
state-of-the-art in CV and NLP? This guide will serve as your first introduction to core
Keras API concepts.

In this guide, you will learn about:

- Creating layers by subclassing the `Layer` class
- Computing gradients with a `GradientTape` and writing low-level training loops
- Tracking losses created by layers via the `add_loss()` method
- Tracking metrics in a low-level training loop
- Speeding up execution with a compiled `tf.function`
- Executing layers in training or inference mode
- The Keras Functional API

You will also see the Keras API in action in two end-to-end research examples:
a Variational Autoencoder, an a Hypernetwork.

---
## The `Layer` class

The `Layer` is the fundamental abstraction in Keras.
A `Layer` encapsulates a state (weights) and some computation
(defined in the call method).

A simple layer looks like this:


```python

class Linear(keras.layers.Layer):
    """y = w.x + b"""

    def __init__(self, units=32, input_dim=32):
        super(Linear, self).__init__()
        w_init = tf.random_normal_initializer()
        self.w = tf.Variable(
            initial_value=w_init(shape=(input_dim, units), dtype="float32"),
            trainable=True,
        )
        b_init = tf.zeros_initializer()
        self.b = tf.Variable(
            initial_value=b_init(shape=(units,), dtype="float32"), trainable=True
        )

    def call(self, inputs):
        return tf.matmul(inputs, self.w) + self.b


```

You would use a `Layer` instance much like a Python function:


```python
# Instantiate our layer.
linear_layer = Linear(units=4, input_dim=2)

# The layer can be treated as a function.
# Here we call it on some data.
y = linear_layer(tf.ones((2, 2)))
assert y.shape == (2, 4)

```

The weight variables (created in `__init__`) are automatically
tracked under the `weights` property:


```python
assert linear_layer.weights == [linear_layer.w, linear_layer.b]

```

You have many built-in layers available, from `Dense` to `Conv2D` to `LSTM` to
fancier ones like `Conv3DTranspose` or `ConvLSTM2D`. Be smart about reusing
built-in functionality.

---
## Weight creation

The add_weight method gives you a shortcut for creating weights:


```python

class Linear(keras.layers.Layer):
    """y = w.x + b"""

    def __init__(self, units=32):
        super(Linear, self).__init__()
        self.units = units

    def build(self, input_shape):
        self.w = self.add_weight(
            shape=(input_shape[-1], self.units),
            initializer="random_normal",
            trainable=True,
        )
        self.b = self.add_weight(
            shape=(self.units,), initializer="random_normal", trainable=True
        )

    def call(self, inputs):
        return tf.matmul(inputs, self.w) + self.b


# Instantiate our lazy layer.
linear_layer = Linear(4)

# This will also call `build(input_shape)` and create the weights.
y = linear_layer(tf.ones((2, 2)))

```

---
## Gradients

You can automatically retrieve the gradients of the weights of a layer by
calling it inside a `GradientTape`. Using these gradients, you can update the
weights of the layer, either manually, or using an optimizer object. Of course,
you can modify the gradients before using them, if you need to.


```python
# Prepare a dataset.
(x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
dataset = tf.data.Dataset.from_tensor_slices(
    (x_train.reshape(60000, 784).astype("float32") / 255, y_train)
)
dataset = dataset.shuffle(buffer_size=1024).batch(64)

# Instantiate our linear layer (defined above) with 10 units.
linear_layer = Linear(10)

# Instantiate a logistic loss function that expects integer targets.
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)

# Instantiate an optimizer.
optimizer = tf.keras.optimizers.SGD(learning_rate=1e-3)

# Iterate over the batches of the dataset.
for step, (x, y) in enumerate(dataset):

    # Open a GradientTape.
    with tf.GradientTape() as tape:

        # Forward pass.
        logits = linear_layer(x)

        # Loss value for this batch.
        loss = loss_fn(y, logits)

    # Get gradients of weights wrt the loss.
    gradients = tape.gradient(loss, linear_layer.trainable_weights)

    # Update the weights of our linear layer.
    optimizer.apply_gradients(zip(gradients, linear_layer.trainable_weights))

    # Logging.
    if step % 100 == 0:
        print("Step:", step, "Loss:", float(loss))

```

<div class="k-default-codeblock">
```
Step: 0 Loss: 2.3826446533203125
Step: 100 Loss: 2.267599582672119
Step: 200 Loss: 2.1371021270751953
Step: 300 Loss: 2.0780434608459473
Step: 400 Loss: 1.9888293743133545
Step: 500 Loss: 1.887470006942749
Step: 600 Loss: 1.8170452117919922
Step: 700 Loss: 1.694450855255127
Step: 800 Loss: 1.653834581375122
Step: 900 Loss: 1.6181517839431763

```
</div>
---
## Trainable and non-trainable weights

Weights created by layers can be either trainable or non-trainable. They're
exposed in `trainable_weights` and `non_trainable_weights` respectively.
Here's a layer with a non-trainable weight:


```python

class ComputeSum(keras.layers.Layer):
    """Returns the sum of the inputs."""

    def __init__(self, input_dim):
        super(ComputeSum, self).__init__()
        # Create a non-trainable weight.
        self.total = tf.Variable(initial_value=tf.zeros((input_dim,)), trainable=False)

    def call(self, inputs):
        self.total.assign_add(tf.reduce_sum(inputs, axis=0))
        return self.total


my_sum = ComputeSum(2)
x = tf.ones((2, 2))

y = my_sum(x)
print(y.numpy())  # [2. 2.]

y = my_sum(x)
print(y.numpy())  # [4. 4.]

assert my_sum.weights == [my_sum.total]
assert my_sum.non_trainable_weights == [my_sum.total]
assert my_sum.trainable_weights == []

```

<div class="k-default-codeblock">
```
[2. 2.]
[4. 4.]

```
</div>
---
## Layers that own layers

Layers can be recursively nested to create bigger computation blocks.
Each layer will track the weights of its sublayers
(both trainable and non-trainable).


```python
# Let's reuse the Linear class
# with a `build` method that we defined above.


class MLP(keras.layers.Layer):
    """Simple stack of Linear layers."""

    def __init__(self):
        super(MLP, self).__init__()
        self.linear_1 = Linear(32)
        self.linear_2 = Linear(32)
        self.linear_3 = Linear(10)

    def call(self, inputs):
        x = self.linear_1(inputs)
        x = tf.nn.relu(x)
        x = self.linear_2(x)
        x = tf.nn.relu(x)
        return self.linear_3(x)


mlp = MLP()

# The first call to the `mlp` object will create the weights.
y = mlp(tf.ones(shape=(3, 64)))

# Weights are recursively tracked.
assert len(mlp.weights) == 6

```

Note that our manually-created MLP above is equivalent to the following
built-in option:


```python
mlp = keras.Sequential(
    [
        keras.layers.Dense(32, activation=tf.nn.relu),
        keras.layers.Dense(32, activation=tf.nn.relu),
        keras.layers.Dense(10),
    ]
)

```

---
## Tracking losses created by layers

Layers can create losses during the forward pass via the `add_loss()` method.
This is especially useful for regularization losses.
The losses created by sublayers are recursively tracked by the parent layers.

Here's a layer that creates an activity regularization loss:


```python

class ActivityRegularization(keras.layers.Layer):
    """Layer that creates an activity sparsity regularization loss."""

    def __init__(self, rate=1e-2):
        super(ActivityRegularization, self).__init__()
        self.rate = rate

    def call(self, inputs):
        # We use `add_loss` to create a regularization loss
        # that depends on the inputs.
        self.add_loss(self.rate * tf.reduce_sum(inputs))
        return inputs


```

Any model incorporating this layer will track this regularization loss:


```python
# Let's use the loss layer in a MLP block.


class SparseMLP(keras.layers.Layer):
    """Stack of Linear layers with a sparsity regularization loss."""

    def __init__(self):
        super(SparseMLP, self).__init__()
        self.linear_1 = Linear(32)
        self.regularization = ActivityRegularization(1e-2)
        self.linear_3 = Linear(10)

    def call(self, inputs):
        x = self.linear_1(inputs)
        x = tf.nn.relu(x)
        x = self.regularization(x)
        return self.linear_3(x)


mlp = SparseMLP()
y = mlp(tf.ones((10, 10)))

print(mlp.losses)  # List containing one float32 scalar

```

<div class="k-default-codeblock">
```
[<tf.Tensor: shape=(), dtype=float32, numpy=0.21055736>]

```
</div>
These losses are cleared by the top-level layer at the start of each forward
pass -- they don't accumulate. `layer.losses` always contains only the losses
created during the last forward pass. You would typically use these losses by
summing them before computing your gradients when writing a training loop.



```python
# Losses correspond to the *last* forward pass.
mlp = SparseMLP()
mlp(tf.ones((10, 10)))
assert len(mlp.losses) == 1
mlp(tf.ones((10, 10)))
assert len(mlp.losses) == 1  # No accumulation.

# Let's demonstrate how to use these losses in a training loop.

# Prepare a dataset.
(x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
dataset = tf.data.Dataset.from_tensor_slices(
    (x_train.reshape(60000, 784).astype("float32") / 255, y_train)
)
dataset = dataset.shuffle(buffer_size=1024).batch(64)

# A new MLP.
mlp = SparseMLP()

# Loss and optimizer.
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)
optimizer = tf.keras.optimizers.SGD(learning_rate=1e-3)

for step, (x, y) in enumerate(dataset):
    with tf.GradientTape() as tape:

        # Forward pass.
        logits = mlp(x)

        # External loss value for this batch.
        loss = loss_fn(y, logits)

        # Add the losses created during the forward pass.
        loss += sum(mlp.losses)

        # Get gradients of weights wrt the loss.
        gradients = tape.gradient(loss, mlp.trainable_weights)

    # Update the weights of our linear layer.
    optimizer.apply_gradients(zip(gradients, mlp.trainable_weights))

    # Logging.
    if step % 100 == 0:
        print("Step:", step, "Loss:", float(loss))

```

<div class="k-default-codeblock">
```
Step: 0 Loss: 6.011272430419922
Step: 100 Loss: 2.6576173305511475
Step: 200 Loss: 2.4374217987060547
Step: 300 Loss: 2.3680849075317383
Step: 400 Loss: 2.360743999481201
Step: 500 Loss: 2.3423142433166504
Step: 600 Loss: 2.313633680343628
Step: 700 Loss: 2.3257956504821777
Step: 800 Loss: 2.306140661239624
Step: 900 Loss: 2.321286201477051

```
</div>
---
## Keeping track of training metrics

Keras offers a broad range of built-in metrics, like `tf.keras.metrics.AUC`
or `tf.keras.metrics.PrecisionAtRecall`. It's also easy to create your
own metrics in a few lines of code.

To use a metric in a custom training loop, you would:

- Instantiate the metric object, e.g. `metric = tf.keras.metrics.AUC()`
- Call its `metric.udpate_state(targets, predictions)` method for each batch of data
- Query its result via `metric.result()`
- Reset the metric's state at the end of an epoch or at the start of an evaluation via
`metric.reset_states()`

Here's a simple example:


```python
# Instantiate a metric object
accuracy = tf.keras.metrics.SparseCategoricalAccuracy()

# Prepare our layer, loss, and optimizer.
model = keras.Sequential(
    [
        keras.layers.Dense(32, activation="relu"),
        keras.layers.Dense(32, activation="relu"),
        keras.layers.Dense(10),
    ]
)
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)

for epoch in range(2):
    # Iterate over the batches of a dataset.
    for step, (x, y) in enumerate(dataset):
        with tf.GradientTape() as tape:
            logits = model(x)
            # Compute the loss value for this batch.
            loss_value = loss_fn(y, logits)

        # Update the state of the `accuracy` metric.
        accuracy.update_state(y, logits)

        # Update the weights of the model to minimize the loss value.
        gradients = tape.gradient(loss_value, model.trainable_weights)
        optimizer.apply_gradients(zip(gradients, model.trainable_weights))

        # Logging the current accuracy value so far.
        if step % 200 == 0:
            print("Epoch:", epoch, "Step:", step)
            print("Total running accuracy so far: %.3f" % accuracy.result())

    # Result the metric's state at the end of an epoch
    accuracy.reset_states()

```

<div class="k-default-codeblock">
```
Epoch: 0 Step: 0
Total running accuracy so far: 0.109
Epoch: 0 Step: 200
Total running accuracy so far: 0.745
Epoch: 0 Step: 400
Total running accuracy so far: 0.819
Epoch: 0 Step: 600
Total running accuracy so far: 0.850
Epoch: 0 Step: 800
Total running accuracy so far: 0.867
Epoch: 1 Step: 0
Total running accuracy so far: 0.938
Epoch: 1 Step: 200
Total running accuracy so far: 0.936
Epoch: 1 Step: 400
Total running accuracy so far: 0.939
Epoch: 1 Step: 600
Total running accuracy so far: 0.940
Epoch: 1 Step: 800
Total running accuracy so far: 0.941

```
</div>
In addition to this, similarly to the `self.add_loss()` method, you have access
to an `self.add_metric()` method on layers. It tracks the average of
whatever quantity you pass to it. You can reset the value of these metrics
by calling `layer.reset_metrics()` on any layer or model.

---
## Compiled functions

Running eagerly is great for debugging, but you will get better performance by
compiling your computation into static graphs. Static graphs are a researcher's
best friends. You can compile any function by wrapping it in a `tf.function`
decorator.


```python
# Prepare our layer, loss, and optimizer.
model = keras.Sequential(
    [
        keras.layers.Dense(32, activation="relu"),
        keras.layers.Dense(32, activation="relu"),
        keras.layers.Dense(10),
    ]
)
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)

# Create a training step function.


@tf.function  # Make it fast.
def train_on_batch(x, y):
    with tf.GradientTape() as tape:
        logits = model(x)
        loss = loss_fn(y, logits)
        gradients = tape.gradient(loss, model.trainable_weights)
    optimizer.apply_gradients(zip(gradients, model.trainable_weights))
    return loss


# Prepare a dataset.
(x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
dataset = tf.data.Dataset.from_tensor_slices(
    (x_train.reshape(60000, 784).astype("float32") / 255, y_train)
)
dataset = dataset.shuffle(buffer_size=1024).batch(64)

for step, (x, y) in enumerate(dataset):
    loss = train_on_batch(x, y)
    if step % 100 == 0:
        print("Step:", step, "Loss:", float(loss))

```

<div class="k-default-codeblock">
```
Step: 0 Loss: 2.372758388519287
Step: 100 Loss: 0.6940608620643616
Step: 200 Loss: 0.2873537838459015
Step: 300 Loss: 0.4446139931678772
Step: 400 Loss: 0.583051860332489
Step: 500 Loss: 0.32000404596328735
Step: 600 Loss: 0.265232115983963
Step: 700 Loss: 0.12978266179561615
Step: 800 Loss: 0.3476303219795227
Step: 900 Loss: 0.24191027879714966

```
</div>
---
## Training mode & inference mode

Some layers, in particular the `BatchNormalization` layer and the `Dropout`
layer, have different behaviors during training and inference. For such layers,
it is standard practice to expose a `training` (boolean) argument in the `call`
method.

By exposing this argument in `call`, you enable the built-in training and
evaluation loops (e.g. fit) to correctly use the layer in training and
inference modes.


```python

class Dropout(keras.layers.Layer):
    def __init__(self, rate):
        super(Dropout, self).__init__()
        self.rate = rate

    def call(self, inputs, training=None):
        if training:
            return tf.nn.dropout(inputs, rate=self.rate)
        return inputs


class MLPWithDropout(keras.layers.Layer):
    def __init__(self):
        super(MLPWithDropout, self).__init__()
        self.linear_1 = Linear(32)
        self.dropout = Dropout(0.5)
        self.linear_3 = Linear(10)

    def call(self, inputs, training=None):
        x = self.linear_1(inputs)
        x = tf.nn.relu(x)
        x = self.dropout(x, training=training)
        return self.linear_3(x)


mlp = MLPWithDropout()
y_train = mlp(tf.ones((2, 2)), training=True)
y_test = mlp(tf.ones((2, 2)), training=False)

```

---
## The Functional API for model-building

To build deep learning models, you don't have to use object-oriented programming all the
time. All layers we've seen so far can also be composed functionally, like this (we call
it the "Functional API"):


```python
# We use an `Input` object to describe the shape and dtype of the inputs.
# This is the deep learning equivalent of *declaring a type*.
# The shape argument is per-sample; it does not include the batch size.
# The functional API focused on defining per-sample transformations.
# The model we create will automatically batch the per-sample transformations,
# so that it can be called on batches of data.
inputs = tf.keras.Input(shape=(16,), dtype="float32")

# We call layers on these "type" objects
# and they return updated types (new shapes/dtypes).
x = Linear(32)(inputs)  # We are reusing the Linear layer we defined earlier.
x = Dropout(0.5)(x)  # We are reusing the Dropout layer we defined earlier.
outputs = Linear(10)(x)

# A functional `Model` can be defined by specifying inputs and outputs.
# A model is itself a layer like any other.
model = tf.keras.Model(inputs, outputs)

# A functional model already has weights, before being called on any data.
# That's because we defined its input shape in advance (in `Input`).
assert len(model.weights) == 4

# Let's call our model on some data, for fun.
y = model(tf.ones((2, 16)))
assert y.shape == (2, 10)

# You can pass a `training` argument in `__call__`
# (it will get passed down to the Dropout layer).
y = model(tf.ones((2, 16)), training=True)

```

The Functional API tends to be more concise than subclassing, and provides a few other
advantages (generally the same advantages that functional, typed languages provide over
untyped OO development). However, it can only be used to define DAGs of layers --
recursive networks should be defined as Layer subclasses instead.

Learn more about the Functional API [here](/guides/functional_api/).

In your research workflows, you may often find yourself mix-and-matching OO models and
Functional models.

Note that the `Model` class also features built-in training & evaluation loops
(`fit()` and `evaluate()`). You can always subclass the `Model` class
(it works exactly like subclassing `Layer`) if you want to leverage these loops
for your OO models.

---
## End-to-end experiment example 1: variational autoencoders.

Here are some of things you've learned so far:

- A `Layer` encapsulate a state (created in `__init__` or `build`) and some computation
(defined in `call`).
- Layers can be recursively nested to create new, bigger computation blocks.
- You can easily write highly hackable training loops by opening a
`GradientTape`, calling your model inside the tape's scope, then retrieving
gradients and applying them via an optimizer.
- You can speed up your training loops using the `@tf.function` decorator.
- Layers can create and track losses (typically regularization losses) via
`self.add_loss()`.

Let's put all of these things together into an end-to-end example: we're going to
implement a Variational AutoEncoder (VAE). We'll train it on MNIST digits.

Our VAE will be a subclass of `Layer`, built as a nested composition of layers that
subclass `Layer`. It will feature a regularization loss (KL divergence).

Below is our model definition.

First, we have an `Encoder` class, which uses a `Sampling` layer to map a MNIST digit to
a latent-space triplet `(z_mean, z_log_var, z)`.


```python
from tensorflow.keras import layers


class Sampling(layers.Layer):
    """Uses (z_mean, z_log_var) to sample z, the vector encoding a digit."""

    def call(self, inputs):
        z_mean, z_log_var = inputs
        batch = tf.shape(z_mean)[0]
        dim = tf.shape(z_mean)[1]
        epsilon = tf.keras.backend.random_normal(shape=(batch, dim))
        return z_mean + tf.exp(0.5 * z_log_var) * epsilon


class Encoder(layers.Layer):
    """Maps MNIST digits to a triplet (z_mean, z_log_var, z)."""

    def __init__(self, latent_dim=32, intermediate_dim=64, **kwargs):
        super(Encoder, self).__init__(**kwargs)
        self.dense_proj = layers.Dense(intermediate_dim, activation=tf.nn.relu)
        self.dense_mean = layers.Dense(latent_dim)
        self.dense_log_var = layers.Dense(latent_dim)
        self.sampling = Sampling()

    def call(self, inputs):
        x = self.dense_proj(inputs)
        z_mean = self.dense_mean(x)
        z_log_var = self.dense_log_var(x)
        z = self.sampling((z_mean, z_log_var))
        return z_mean, z_log_var, z


```

Next, we have a `Decoder` class, which maps the probabilistic latent space coordinates
back to a MNIST digit.


```python

class Decoder(layers.Layer):
    """Converts z, the encoded digit vector, back into a readable digit."""

    def __init__(self, original_dim, intermediate_dim=64, **kwargs):
        super(Decoder, self).__init__(**kwargs)
        self.dense_proj = layers.Dense(intermediate_dim, activation=tf.nn.relu)
        self.dense_output = layers.Dense(original_dim, activation=tf.nn.sigmoid)

    def call(self, inputs):
        x = self.dense_proj(inputs)
        return self.dense_output(x)


```

Finally, our `VariationalAutoEncoder` composes together an encoder and a decoder, and
creates a KL divergence regularization loss via `add_loss()`.


```python

class VariationalAutoEncoder(layers.Layer):
    """Combines the encoder and decoder into an end-to-end model for training."""

    def __init__(self, original_dim, intermediate_dim=64, latent_dim=32, **kwargs):
        super(VariationalAutoEncoder, self).__init__(**kwargs)
        self.original_dim = original_dim
        self.encoder = Encoder(latent_dim=latent_dim, intermediate_dim=intermediate_dim)
        self.decoder = Decoder(original_dim, intermediate_dim=intermediate_dim)

    def call(self, inputs):
        z_mean, z_log_var, z = self.encoder(inputs)
        reconstructed = self.decoder(z)
        # Add KL divergence regularization loss.
        kl_loss = -0.5 * tf.reduce_mean(
            z_log_var - tf.square(z_mean) - tf.exp(z_log_var) + 1
        )
        self.add_loss(kl_loss)
        return reconstructed


```

Now, let's write a training loop. Our training step is decorated with a `@tf.function` to
compile into a super fast graph function.


```python
# Our model.
vae = VariationalAutoEncoder(original_dim=784, intermediate_dim=64, latent_dim=32)

# Loss and optimizer.
loss_fn = tf.keras.losses.MeanSquaredError()
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)

# Prepare a dataset.
(x_train, _), _ = tf.keras.datasets.mnist.load_data()
dataset = tf.data.Dataset.from_tensor_slices(
    x_train.reshape(60000, 784).astype("float32") / 255
)
dataset = dataset.shuffle(buffer_size=1024).batch(32)


@tf.function
def training_step(x):
    with tf.GradientTape() as tape:
        reconstructed = vae(x)  # Compute input reconstruction.
        # Compute loss.
        loss = loss_fn(x, reconstructed)
        loss += sum(vae.losses)  # Add KLD term.
    # Update the weights of the VAE.
    grads = tape.gradient(loss, vae.trainable_weights)
    optimizer.apply_gradients(zip(grads, vae.trainable_weights))
    return loss


losses = []  # Keep track of the losses over time.
for step, x in enumerate(dataset):
    loss = training_step(x)
    # Logging.
    losses.append(float(loss))
    if step % 100 == 0:
        print("Step:", step, "Loss:", sum(losses) / len(losses))

    # Stop after 1000 steps.
    # Training the model to convergence is left
    # as an exercise to the reader.
    if step >= 1000:
        break

```

<div class="k-default-codeblock">
```
Step: 0 Loss: 0.38399988412857056
Step: 100 Loss: 0.12813832219874505
Step: 200 Loss: 0.10099794645214555
Step: 300 Loss: 0.09052002512883903
Step: 400 Loss: 0.08532035229100551
Step: 500 Loss: 0.08204194844631615
Step: 600 Loss: 0.07955289975478129
Step: 700 Loss: 0.07817310818634428
Step: 800 Loss: 0.07688544124606694
Step: 900 Loss: 0.07589594667134486
Step: 1000 Loss: 0.07492802400778342

```
</div>
As you can see, building and training this type of model in Keras
is quick and painless.

Now, you may find that the code above is somewhat verbose: we handle every little detail
on our own, by hand. This gives the most flexibility, but it's also a bit of work.

Let's take a look at what the Functional API version of
our VAE looks like:


```python
original_dim = 784
intermediate_dim = 64
latent_dim = 32

# Define encoder model.
original_inputs = tf.keras.Input(shape=(original_dim,), name="encoder_input")
x = layers.Dense(intermediate_dim, activation="relu")(original_inputs)
z_mean = layers.Dense(latent_dim, name="z_mean")(x)
z_log_var = layers.Dense(latent_dim, name="z_log_var")(x)
z = Sampling()((z_mean, z_log_var))
encoder = tf.keras.Model(inputs=original_inputs, outputs=z, name="encoder")

# Define decoder model.
latent_inputs = tf.keras.Input(shape=(latent_dim,), name="z_sampling")
x = layers.Dense(intermediate_dim, activation="relu")(latent_inputs)
outputs = layers.Dense(original_dim, activation="sigmoid")(x)
decoder = tf.keras.Model(inputs=latent_inputs, outputs=outputs, name="decoder")

# Define VAE model.
outputs = decoder(z)
vae = tf.keras.Model(inputs=original_inputs, outputs=outputs, name="vae")

# Add KL divergence regularization loss.
kl_loss = -0.5 * tf.reduce_mean(z_log_var - tf.square(z_mean) - tf.exp(z_log_var) + 1)
vae.add_loss(kl_loss)

```

Much more concise, right?

By the way, Keras also features built-in training & evaluation loops on its `Model` class
(`fit()` and `evaluate()`). Check it out:


```python
# Loss and optimizer.
loss_fn = tf.keras.losses.MeanSquaredError()
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)

# Prepare a dataset.
(x_train, _), _ = tf.keras.datasets.mnist.load_data()
dataset = tf.data.Dataset.from_tensor_slices(
    x_train.reshape(60000, 784).astype("float32") / 255
)
dataset = dataset.map(lambda x: (x, x))  # Use x_train as both inputs & targets
dataset = dataset.shuffle(buffer_size=1024).batch(32)

# Configure the model for training.
vae.compile(optimizer, loss=loss_fn)

# Actually training the model.
vae.fit(dataset, epochs=1)

```

<div class="k-default-codeblock">
```
1875/1875 [==============================] - 2s 962us/step - loss: 0.0713

<tensorflow.python.keras.callbacks.History at 0x15fcd8c10>

```
</div>
The use of the Functional API and `fit` reduces our example from 65 lines to 25 lines
(including model definition & training). The Keras philosophy is to offer you
productivity-boosting features like
these, while simultaneously empowering you to write everything yourself to gain absolute
control over every little detail. Like we did in the low-level training loop two
paragraphs earlier.

---
## End-to-end experiment example 2: hypernetworks.

Let's take a look at another kind of research experiment: hypernetworks.

A hypernetwork is a deep neural network whose weights are generated by another network
(usually smaller).

Let's implement a really trivial hypernetwork: we'll use a small 2-layer network  to
generate the weights of a larger 3-layer network.




```python
import numpy as np

input_dim = 784
classes = 10

# This is the model we'll actually use to predict labels (the hypernetwork).
outer_model = keras.Sequential(
    [keras.layers.Dense(64, activation=tf.nn.relu), keras.layers.Dense(classes),]
)

# It doesn't need to create its own weights, so let's mark its layers
# as already built. That way, calling `outer_model` won't create new variables.
for layer in outer_model.layers:
    layer.built = True

# This is the number of weight coefficients to generate. Each layer in the
# hypernetwork requires output_dim * input_dim + output_dim coefficients.
num_weights_to_generate = (classes * 64 + classes) + (64 * input_dim + 64)

# This is the model that generates the weights of the `outer_model` above.
inner_model = keras.Sequential(
    [
        keras.layers.Dense(16, activation=tf.nn.relu),
        keras.layers.Dense(num_weights_to_generate, activation=tf.nn.sigmoid),
    ]
)

```

This is our training loop. For each batch of data:

- We use `inner_model` to generate an array of weight coefficients, `weights_pred`
- We reshape these coefficients into kernel & bias tensors for the `outer_model`
- We run the forward pass of the `outer_model` to compute the actual MNIST predictions
- We run backprop through the weights of the `inner_model` to minimize the
final classification loss


```python
# Loss and optimizer.
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-4)

# Prepare a dataset.
(x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
dataset = tf.data.Dataset.from_tensor_slices(
    (x_train.reshape(60000, 784).astype("float32") / 255, y_train)
)

# We'll use a batch size of 1 for this experiment.
dataset = dataset.shuffle(buffer_size=1024).batch(1)


@tf.function
def train_step(x, y):
    with tf.GradientTape() as tape:
        # Predict weights for the outer model.
        weights_pred = inner_model(x)

        # Reshape them to the expected shapes for w and b for the outer model.
        # Layer 0 kernel.
        start_index = 0
        w0_shape = (input_dim, 64)
        w0_coeffs = weights_pred[:, start_index : start_index + np.prod(w0_shape)]
        w0 = tf.reshape(w0_coeffs, w0_shape)
        start_index += np.prod(w0_shape)
        # Layer 0 bias.
        b0_shape = (64,)
        b0_coeffs = weights_pred[:, start_index : start_index + np.prod(b0_shape)]
        b0 = tf.reshape(b0_coeffs, b0_shape)
        start_index += np.prod(b0_shape)
        # Layer 1 kernel.
        w1_shape = (64, classes)
        w1_coeffs = weights_pred[:, start_index : start_index + np.prod(w1_shape)]
        w1 = tf.reshape(w1_coeffs, w1_shape)
        start_index += np.prod(w1_shape)
        # Layer 1 bias.
        b1_shape = (classes,)
        b1_coeffs = weights_pred[:, start_index : start_index + np.prod(b1_shape)]
        b1 = tf.reshape(b1_coeffs, b1_shape)
        start_index += np.prod(b1_shape)

        # Set the weight predictions as the weight variables on the outer model.
        outer_model.layers[0].kernel = w0
        outer_model.layers[0].bias = b0
        outer_model.layers[1].kernel = w1
        outer_model.layers[1].bias = b1

        # Inference on the outer model.
        preds = outer_model(x)
        loss = loss_fn(y, preds)

    # Train only inner model.
    grads = tape.gradient(loss, inner_model.trainable_weights)
    optimizer.apply_gradients(zip(grads, inner_model.trainable_weights))
    return loss


losses = []  # Keep track of the losses over time.
for step, (x, y) in enumerate(dataset):
    loss = train_step(x, y)

    # Logging.
    losses.append(float(loss))
    if step % 100 == 0:
        print("Step:", step, "Loss:", sum(losses) / len(losses))

    # Stop after 1000 steps.
    # Training the model to convergence is left
    # as an exercise to the reader.
    if step >= 1000:
        break

```

<div class="k-default-codeblock">
```
Step: 0 Loss: 1.8061305284500122
Step: 100 Loss: 2.430531589701624
Step: 200 Loss: 2.1146517871364727
Step: 300 Loss: 1.9503340735858263
Step: 400 Loss: 1.8750596672872504
Step: 500 Loss: 1.81695979083944
Step: 600 Loss: 1.7097562247230214
Step: 700 Loss: 1.642795581991716
Step: 800 Loss: 1.6287533775725223
Step: 900 Loss: 1.6253980021383934
Step: 1000 Loss: 1.5546919323841977

```
</div>
Implementing arbitrary research ideas with Keras is straightforward and highly
productive. Imagine trying out 25 ideas per day (20 minutes per experiment on average)!

Keras has been designed to go from idea to results as fast as possible, because we
believe this is
the key to doing great research.

We hope you enjoyed this quick introduction. Let us know what you build with Keras!
# The definition and call of job function

In OneFlow, We encapsulate the train, predict, inference and some other tasks into a function, which is called job function. The job function is used to connect the user's business logic and the computing resource managed by OneFlow

In OneFlow, the function decorated by `@oneflow.global_function` decorator is the OneFlow's job function

We mainly define the structure of the model and choose the optimization in job function.Otherwise, we can also pass some hyperparameters about training and the environment configuration to the job function(like the following example:`get_train_config()`), OneFlow will manage the memory, GPU and some other computing resource according to our config.

In this section, we will specifically learn about:

* how to define and call the job function

* how to get the return value of job function

## The relationship between the job function and the running process of OneFlow

The job function is divided into two phases: definition and call.

It's related to OneFlow's operating mechanism. Briefly, The OneFlow Python layer API simply describes the configuration and the training environment of the model. These information will pass to the C++ backend. After compilation, model graph building and so on, the calculation diagram is obtained. Finally, it will be executed by OneFlow runtime.

The definition of the job function, is actually doing the description of network model and the configuration of training environment in Python. In this phase, there's no data here, we can only define the shape, data type of the model's node, we call it as  **placeholder **, which is convenient to model inference in the compilation and creating graph of OneFlow.

The job function will be called after the OneFlow runtime has started. We can pass the data by calling job function and get the results

The definition and calling method of job functions are described in detail as below

## The definition of job function

We encapsulate the model in Python and use `oneflow.global_function` to decorate. Then the definition is completed.

The job function mainly describes two things:

* The structure of model

* The optimizing target in training phase

In the following code, we build a Multi-Layer Perceptron model and use `flow.nn.sparse_softmax_cross_entropy_with_logits` to compute the cross-entropy loss as our optimizing target.

```python
@flow.global_function(type="train")
def train_job(
    images: tp.Numpy.Placeholder((BATCH_SIZE, 1, 28, 28), dtype=flow.float),
    labels: tp.Numpy.Placeholder((BATCH_SIZE,), dtype=flow.int32),
) -> tp.Callback[tp.Numpy]:
    # mlp
    initializer = flow.truncated_normal(0.1)
    reshape = flow.reshape(images, [images.shape[0], -1])
    hidden = flow.layers.dense(
        reshape,
        512,
        activation=flow.nn.relu,
        kernel_initializer=initializer,
        name="hidden",
    )
    logits = flow.layers.dense(
        hidden, 10, kernel_initializer=initializer, name="output"
    )

    loss = flow.nn.sparse_softmax_cross_entropy_with_logits(
        labels, logits, name="softmax_loss"
    )
    lr_scheduler = flow.optimizer.PiecewiseConstantScheduler([], [0.1])
    flow.optimizer.SGD(lr_scheduler, momentum=0).minimize(loss)
    return loss
```

### The parameters of oneflow.global_function

`oneflow.global_function` decorator accept two parameters, there are `type` and `function_config`. 

- The parameter `type` accept a string, which can only set as `train` or `predict`. When we define a training model, we set it as `train`. We set is as `predict` when we define a testing or inference model. 
- The parameter `function_config` accept a object which is constructed by `oneflow.function_config()`. In `function_config` object, we can use it's method or attribute to config. As the following code. 

```python
def get_train_config():
    config = flow.function_config()
    config.default_data_type(flow.float)
    return config
```

We set the default data type, then, we can pass the `function_config` object to the `global_function` decorator. 

```python
@flow.global_function(type="train", function_config=get_train_config())
def train_job(
    images: tp.Numpy.Placeholder((BATCH_SIZE, 1, 28, 28), dtype=flow.float),
    labels: tp.Numpy.Placeholder((BATCH_SIZE,), dtype=flow.int32),
) -> tp.Numpy:
```

The complete code wo can refer to [Consistent and Mirrored](consistent_mirrored.md)'s  [mixed_parallel_mlp.py](../code/extended_topics/hybrid_parallelism_mlp.py)

### PlaceHolder

Specifically, the `images`、`logits`、`labels`、`loss` and some other objects have no data in our definition of the job function. They are used to describe the **shape** and **attribute** of data, which is called PlaceHolder.

The PlaceHolder in job function's parameter, use `Numpy.Placeholder`、`ListNumpy.Placeholder`、`ListListNumpy.Placeholder` under the `oneflow.typing` to annotate the data type of job function's parameter. As we call the job function, we should pass the `numpy` object

Besides the several types under the `oneflow.typing` in parameter. The variable computed by OneFlow operators or layers, like the `reshape`、`hidden`、`logits`、`loss` and some other in above code, are also PlaceHolder.

Either of the variables mentioned above, They inherit the base class `BlobDef` directly or indirectly, we call this object type as **Blob** in OneFlow. 

The **Blob** has no data in definition of job function. It only plays the role of data placeholder which is convenient to framework inference.

### The return value of the job function

The concept of the data placeholder **Blob** is emphasized above because the return value of the job function cannot be arbitrarily specified. It must be `Blob` type object or the container which only containing the `Blob` object

As the `loss` returned in the above code, it's type is `Blob` object

The return values of job function should be annotated. As an example, `-> tp.Numpy` in above code means return a `Blob` object.

In Another example, we can annotate the return value type as `-> Tuple[tp.Numpy, tp.Numpy]`.It means the function return a `tuple` which contains two `Blob` object

You can refer to [Get the result of the job function](../basics_topics/async_get.md) for specific examples.

## The call of job function

OneFlow use decorator to translate Python function into OneFlow's job function. It is insensitive to the user.

We can call the job function just like we call a Python function. Everytime we call the job function, OneFlow will complete the forward propagation, back propagation, parameter updates, and more in framework

In the code below. When we get the data, we will pass parameter and call the `train_job` function to print the mean loss

```python
    (train_images, train_labels), (test_images, test_labels) = flow.data.load_mnist(
        BATCH_SIZE
    )

    for epoch in range(3):
        for i, (images, labels) in enumerate(zip(train_images, train_labels)):
            loss = train_job(images, labels)
            if i % 20 == 0:
                print(loss.mean())
```

As you can see, by calling the job function `train_job`, the `numpy` data is directly returned.

The method shown above is synchronous. OneFlow also support asynchronous invocation. you can refer to the chapter [Get the result of the job function](../basics_topics/async_get.md).


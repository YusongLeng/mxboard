# Logging MXNet Data for Visualization in TensorBoard
This file demonstrates the use cases where MXBoard logs MXNet data for visualization in TensorBoard.

## Graph
Graphs are visual representations of neural networks. MXBoard supports visualizing MXNet neural
networks in terms of
[Symbol](https://mxnet.incubator.apache.org/api/python/symbol/symbol.html?highlight=symbol#mxnet.symbol.Symbol)
and
[HybridBlock](https://mxnet.incubator.apache.org/api/python/gluon/gluon.html?highlight=hybridblo#mxnet.gluon.HybridBlock).

### Symbol
The following code would plot a toy network defined using symbols.
Users can double click the node block to expand or collapse the node for
exposing or hiding extra sub-nodes of an operator.
```python
import mxnet as mx
from mxboard import SummaryWriter

data = mx.sym.Variable('data')
weight = mx.sym.Variable('weight')
bias = mx.sym.Variable('fc1_bias', lr_mult=1.0)
conv1 = mx.symbol.Convolution(data=data, weight=weight, name='conv1', num_filter=32, kernel=(3, 3))
conv2 = mx.symbol.Convolution(data=data, weight=weight, name='conv2', num_filter=32, kernel=(3, 3))
conv3 = conv1 + conv2
bn1 = mx.symbol.BatchNorm(data=conv3, name="bn1")
act1 = mx.symbol.Activation(data=bn1, name='relu1', act_type="relu")
sum1 = act1 + conv3
mp1 = mx.symbol.Pooling(data=sum1, name='mp1', kernel=(2, 2), stride=(2, 2), pool_type='max')
fc1 = mx.sym.FullyConnected(data=mp1, bias=bias, name='fc1', num_hidden=10, lr_mult=0)
fc2 = mx.sym.FullyConnected(data=fc1, name='fc2', num_hidden=10, wd_mult=0.5)
sc1 = mx.symbol.SliceChannel(data=fc2, num_outputs=10, name="slice_1", squeeze_axis=0)

with SummaryWriter(logdir='./logs') as sw:
    sw.add_graph(sc1)
```
![png](https://github.com/reminisce/web-data/blob/tensorboard_doc/mxnet/tensorboard/doc/summary_graph_symbol.png)

### HybridBlock
To visualize a Gluon model built using `HybridBlock`s, users must first call
[`hybridize()`](https://mxnet.incubator.apache.org/api/python/gluon/gluon.html?highlight=hybridize#mxnet.gluon.Block.hybridize),
[`initialize()`](https://mxnet.incubator.apache.org/api/python/gluon/gluon.html?highlight=hybridize#mxnet.gluon.Block.initialize),
and
[`forward()`](https://mxnet.incubator.apache.org/api/python/gluon/gluon.html?highlight=hybridize#mxnet.gluon.HybridBlock.forward)
functions to generate a graph symbol which will be used later to plot network structures.
```python
from mxnet.gluon import nn
from mxboard import SummaryWriter
import mxnet as mx

net = nn.HybridSequential()
with net.name_scope():
    net.add(nn.Dense(128, activation='relu'))
    net.add(nn.Dense(64, activation='relu'))
    net.add(nn.Dense(10))

# The following three lines hybridize the network, initialize the parameters,
# and run forward pass once to generate a symbol which will be used later
# for plotting the network.
net.hybridize()
net.initialize()
net.forward(mx.nd.ones((1,)))

with SummaryWriter(logdir='./logs') as sw:
    sw.add_graph(net)
```
![png](https://github.com/reminisce/web-data/blob/tensorboard_doc/mxnet/tensorboard/doc/summary_graph_hybridblock.png)

Users can explore more sophisticated network structures provided by
[MXNet Gluon Model Zoo](https://mxnet.incubator.apache.org/api/python/gluon/model_zoo.html?highlight=model_zoo#api-reference).
For example, the following code would plot the network
[Inception V3](http://arxiv.org/abs/1512.00567)
for you.
```python
from mxnet.gluon.model_zoo.vision import get_model
from mxboard import SummaryWriter
import mxnet as mx

net = get_model('inceptionv3')
net.hybridize()
net.initialize()
net.forward(mx.nd.ones((1, 3, 299, 299)))

with SummaryWriter(logdir='./logs') as sw:
    sw.add_graph(net)
```

## Scalar
Scalar values often appear in terms of curves, such as training accuracy as time evolves. Here
is an example of plotting the curve of `y=sin(x/100)` where `x` is in the range of `[0, 2*pi]`.
```python
import numpy as np
from mxboard import SummaryWriter

x_vals = np.arange(start=0, stop=2 * np.pi, step=0.01)
y_vals = np.sin(x_vals)
with SummaryWriter(logdir='./logs') as sw:
    for x, y in zip(x_vals, y_vals):
        sw.add_scalar(tag='sin_function_curve', value=y, global_step=x * 100)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_scalar_sin.png)

Please note that the `tag` parameter in the API is for differentiating plots.
MXBoard allows users to draw multiple curves in the same plot by specifying the same `tag`
and different scalar names for `value`s. The following example demonstrates logging
five curves in the same plot named "curves" in TensorBoard. Each curve has a name
attached except `y5`. It exports logged scalar values into a json file in the end.
```python
import numpy as np
from mxboard import SummaryWriter

xs = np.arange(start=0, stop=2 * np.pi, step=0.01)
y_sin = np.sin(xs)
y_cos = np.cos(xs)
y_exp_sin = np.exp(y_sin)
y_exp_cos = np.exp(y_cos)
y_sin2 = y_sin * y_sin
with SummaryWriter(logdir='./logs') as sw:
    for x, y1, y2, y3, y4, y5 in zip(xs, y_sin, y_cos, y_exp_sin, y_exp_cos, y_sin2):
        sw.add_scalar('curves', {'sin': y1, 'cos': y2}, x * 100)  # log y1 with name 'sin' and y2 with name 'cos'
        sw.add_scalar('curves', ('exp(sin)', y3), x * 100)  # log y3 with name 'exp(sin)'
        sw.add_scalar('curves', ['exp(cos)', y4], x * 100)  # log y4 with name 'exp(cos)'
        sw.add_scalar('curves', y5, x * 100)  # log y5 without specifying scalar name

    sw.export_scalars('scalars.json')
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_scalars.png)


## Histogram
We can visulize the value distributions of tensors by logging `NDArray`s in terms of histograms.
The following code snippet generates a series of normal distributions with smaller and smaller standard deviations.
```python
import mxnet as mx
from mxboard import SummaryWriter


with SummaryWriter(logdir='./logs') as sw:
    for i in range(10):
        # create a normal distribution with fixed mean and decreasing std
        data = mx.nd.normal(loc=0, scale=10.0/(i+1), shape=(10, 3, 8, 8))
        sw.add_histogram(tag='norml_dist', values=data, bins=200, global_step=i)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_histogram_norm.png)


## Image
The image logging API can take MXNet `NDArray` or `numpy.ndarray` of 2-4 dimensions.
It will preprocess the input image and write the processed image to the event file.
When the input image data is 2D or 3D, it represents a single image.
When the input image data is a 4D tensor, which represents a batch of images, the logging
API would make a grid of those images by stitching them together before write
them to the event file. The following code snippet saves 15 same images
for visualization in TensorBoard.
```python
import mxnet as mx
import numpy as np
from mxboard import SummaryWriter
from scipy import misc

# get a raccoon face image from scipy
# and convert its layout from HWC to CHW
face = misc.face().transpose((2, 0, 1))
# add a batch axis in the beginning
face = face.reshape((1,) + face.shape)
# replicate the face by 15 times
faces = [face] * 15
# concatenate the faces along the batch axis
faces = np.concatenate(faces, axis=0)

img = mx.nd.array(faces, dtype=faces.dtype)
with SummaryWriter(logdir='./logs') as sw:
    # write batched faces to the event file
    sw.add_image(tag='faces', image=img)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_image_faces.png)


## Embedding
Embedding visualization enables people to get an intuition on how data is clustered
in 2D or 3D space. The following code takes 2,560 images of handwritten digits
from the [MNIST dataset](http://yann.lecun.com/exdb/mnist/) and log them
as embedding vectors with labels and original images.
```python
import numpy as np
import mxnet as mx
from mxnet import gluon
from mxboard import SummaryWriter


batch_size = 128


def transformer(data, label):
    data = data.reshape((-1,)).astype(np.float32)/255
    return data, label

# training dataset containing MNIST images and labels
train_data = gluon.data.DataLoader(
    gluon.data.vision.MNIST('./data', train=True, transform=transformer),
    batch_size=batch_size, shuffle=True, last_batch='discard')

initialized = False
embedding = None
labels = None
images = None

for i, (data, label) in enumerate(train_data):
    if i >= 20:
        # only fetch the first 20 batches of images
        break
    if initialized:  # after the first batch, concatenate the current batch with the existing one
        embedding = mx.nd.concat(*(embedding, data), dim=0)
        labels = mx.nd.concat(*(labels, label), dim=0)
        images = mx.nd.concat(*(images, data.reshape(batch_size, 1, 28, 28)), dim=0)
    else:  # first batch of images, directly assign
        embedding = data
        labels = label
        images = data.reshape(batch_size, 1, 28, 28)
        initialized = True

with SummaryWriter(logdir='./logs') as sw:
    sw.add_embedding(tag='mnist', embedding=embedding, labels=labels, images=images)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_embedding_mnist.png)


## Audio
The following code generates audio data uniformly sampled in range `[-1, 1]`
and write the data to the event file for TensorBoard to playback.
```python
import mxnet as mx
from mxboard import SummaryWriter


frequency = 44100
# 44100 random samples between -1 and 1
data = mx.random.uniform(low=-1, high=1, shape=(frequency,))
max_abs_val = data.abs().max()
# rescale the data to the range [-1, 1]
data = data / max_abs_val
with SummaryWriter(logdir='./logs') as sw:
    sw.add_audio(tag='uniform_audio', audio=data, global_step=0)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_audio_uniform.png)


## Text
TensorBoard is able to render plain text as well as text in the markdown format.
The following code demonstrates these two use cases.
```python
from mxboard import SummaryWriter


def simple_example(sw, step):
    greeting = 'Hello MXNet from step {}'.format(str(step))
    sw.add_text(tag='simple_example', text=greeting, global_step=step)


def markdown_table(sw):
    header_row = 'Hello | MXNet,\n'
    delimiter = '----- | -----\n'
    table_body = 'This | is\n' + 'so | awesome!'
    sw.add_text(tag='markdown_table', text=header_row+delimiter+table_body)


with SummaryWriter(logdir='./logs') as sw:
    simple_example(sw, 100)
    markdown_table(sw)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_text.png)


## PR Curve
Precision-Recall is a useful metric of success of prediction when the categories are imbalanced.
The relationship between recall and precision can be visualized in terms of precision-recall curves.
The following code snippet logs the data of predictions and labels for visualizing
the precision-recall curve in TensorBoard. It generates 100 numbers uniformly distributed in range `[0, 1]` representing
the predictions of 100 examples. The labels are also generated randomly by picking either 0 or 1.
```python
import mxnet as mx
import numpy as np
from mxboard import SummaryWriter

with SummaryWriter(logdir='./logs') as sw:
    predictions = mx.nd.uniform(low=0, high=1, shape=(100,), dtype=np.float32)
    labels = mx.nd.uniform(low=0, high=2, shape=(100,), dtype=np.float32).astype(np.int32)
    sw.add_pr_curve(tag='pseudo_pr_curve', predictions=predictions, labels=labels, num_thresholds=120)
```
![png](https://github.com/dmlc/web-data/blob/master/mxnet/tensorboard/doc/summary_pr_curve_uniform.png)

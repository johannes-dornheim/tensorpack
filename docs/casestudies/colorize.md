
# Colorize

NOTE: this page is a bit outdated. Will be updated soon.

As creating a neural network for digit classification seems to be a bit outdated, we will create a fictional network that learns to colorize grayscale images. In this case-study, you will learn to do the following using TensorPack.

- DataFlow
    + create a basic dataflow containing images
    + debug you dataflow
    + add custom manipulation to your data such as converting to Lab-space
    + efficiently prefetch data
- Network
    + define a neural network architecture for regression
    + integrate summary functions of TensorFlow
- Training
    + create a training configuration
- Callbacks
    + write your own callback to export predicted images after each epoch

## DataFlow

The basic idea is to gather a huge amount of images, resizing them to the same size and extract
the luminance channel after converting from RGB to Lab. For demonstration purposes, we will split
the dataflow definition into separate steps, though it might more efficient to combine some steps.


### Reading data
The first node in the dataflow is the image reader. You can implement the reader however you want, but there are some existing ones we can use, e.g.:

- use the lmdb files you probably already have for the Caffe framework
- collect images from a specific directory
- read ImageNet dataset if you have already downloaded these images

We will use simply a directory which consists of many RGB images. This is as simple as:

```python
from tensorpack import *
import glob, os

imgs = glob.glob(os.path.join('/media/data/img', '*.jpg'))
ds = ImageFromFile(imgs, channel=3, shuffle=True)
ds = PrintData(ds, num=2) # only for debugging
```

Running this will give you:
```
[0112 18:59:47 @common.py:600] DataFlow Info:
datapoint 0<2 with 1 components consists of
   dp 0: is ndarray of shape (1920, 2560, 3) with range [0.0000, 255.0000]
datapoint 1<2 with 1 components consists of
   dp 0: is ndarray of shape (850, 1554, 3) with range [0.0000, 255.0000]
```

To actually access the datapoints generated by the dataflow, you can use

```python
ds.reset_state()
for dp in ds.get_data():
    print dp[0] # this is an RGB image!
```
This kind of iteration is used behind the scenes to feed data for training.


### Manipulate incoming data
Now, training a ConvNet which is not fully convolutional requires images of known shape, but our
directory may contain images of different sizes. Let us add this to the dataflow:

```python
imgs = glob.glob(os.path.join('/media/data/img', '*.jpg'))
ds = ImageFromFile(imgs, channel=3, shuffle=True)
ds = AugmentImageComponent(ds, [imgaug.Resize((256, 256))])
ds = PrintData(ds, num=2) # only for debugging
```

It's time to convert the RGB image into the Lab space. In python, you would to something like this:

```python
rgb = get_my_image()
lab = cv2.cvtColor(rgb, cv2.COLOR_RGB2Lab)
```

We should add this to our dataflow:

```python
import cv2
imgs = glob.glob(os.path.join('/media/data/img', '*.jpg'))
ds = ImageFromFile(imgs, channel=3, shuffle=True)
ds = AugmentImageComponent(ds, [imgaug.Resize((256, 256))])
ds = MapDataComponent(ds, lambda im: cv2.cvtColor(im, cv2.COLOR_RGB2Lab))
ds = PrintData(ds, num=2) # only for debugging
```

Alternatively, we can also define `rgb2lab` as an augmentor, to make code more compact.
```python
from skimage import color

def get_data():
    augs = [imgaug.Resize((256, 256)),
            imgaug.MapImage(lambda im: cv2.cvtColor(im, cv2.COLOR_RGB2Lab))]

    imgs = glob.glob(os.path.join('/media/data/img', '*.jpg'))
    ds = ImageFromFile(imgs, channel=3, shuffle=True)
    ds = AugmentImageComponent(ds, augs)
    ds = BatchData(ds, 32)
    ds = PrefetchData(ds, 4) # use queue size 4
    return ds
```
Note that we've also added batch and prefetch, so that the dataflow now generates images of shape (32, 256, 256, 3), and faster.

But wait! The alert reader makes a critical observation! For input we need the L channel *only* and we should add the RGB image as ground-truth data. Let's fix that.

```python
from tensorpack import *
import glob, os
import cv2

def get_data():
    imgs = glob.glob(os.path.join('/media/data/img', '*.jpg'))
    ds = ImageFromFile(imgs, channel=3, shuffle=True)
    ds = AugmentImageComponent(ds, [imgaug.Resize((256, 256))])
    ds = MapData(ds, lambda dp: [cv2.cvtColor(dp[0], cv2.COLOR_RGB2Lab)[:,:,0], dp[0]])
    ds = BatchData(ds, 32)
    ds = PrefetchData(ds, 4) # use queue size 4
		ds = PrintData(ds, num=2)	# only for debug
    return ds
```

Here, we simply apply a mapping function to the datapoint, transform the single component to two components: the first is the L color space, and the second is just itself.
The output when using `PrintData` should be like:

```
datapoint 0<2 with 2 components consists of
   dp 0: is ndarray of shape (32, 256, 256) with range [0, 100.0000]
   dp 1: is ndarray of shape (32, 256, 256, 3) with range [0, 221.6387]
datapoint 1<2 with 2 components consists of
   dp 0: is ndarray of shape (32, 256, 256) with range [0, 100.0000]
   dp 1: is ndarray of shape (32, 256, 256, 3) with range [0, 249.6030]
```

Well, this is probably not the most efficient way to encode this process. But it clearly demonstrates how much flexibility the `dataflow` gives.
You can easily insert you own functions, and utilize the pre-defined modules at the same time.

## Network

If you are surprised how far we already are, you will enjoy how easy it is to define a network model. The most simple model is probably:

```python
class Model(ModelDesc):
    def _get_inputs(self):
        pass

    def _build_graph(self, input_vars):
        self.cost = 0
```

The framework expects:
- a definition of inputs in `_get_inputs`
- a computation graph containing the actual network layers in `_build_graph`
- In single-cost optimization problem, a member `self.cost` representing the loss function we would like to minimize.

### Define inputs
Our dataflow produces data which looks like `[(32, 256, 256), (32, 256, 256, 3)]`.
The first entry is the luminance channel as input and the latter is the original RGB image with all three channels. So we will write

```python
def _get_inputs(self):
        return [InputDesc(tf.float32, (None, 256, 256), 'luminance'),
                InputDesc(tf.int32, (None, 256, 256, 3), 'rgb')]
```

This is pretty straight forward, isn't it? We defined the shapes of the input and give each entry a name.
You can certainly use 32 instead of `None`, but since the model itself doesn't really need to know
the batch size, using `None` offers the extra flexibility to run inference with a different batch size in the same graph.

From now, the `input_vars` in `_build_graph(self, input_vars)` will be the tensors of the defined shapes in the method `_get_inputs`. We can therefore write

```python
class Model(ModelDesc):

    def _get_inputs(self):
        return [InputDesc(tf.float32, (None, 256, 256), 'luminance'),
                InputDesc(tf.int32, (None, 256, 256, 3), 'rgb')]

    def _build_graph(self, input_vars):
        luminance, rgb = input_vars  # (None, 256, 256), (None, 256, 256, 3)
        self.cost = 0
```


### Define Architecture
So all we need to do is to define a network layout
```math
f\colon \mathbb{R}^{b \times 256 \times 256} \to \mathbb{R}^{b \times 256 \times 256 \times 3}
```
mapping our input to a plausible rgb image.

The process of coming up with such a network architecture is usually a soup of experience,
a lot of trials and much time laced with magic or simply chance, depending what you prefer.
We will use an auto-encoder with a lot of convolutions to squeeze the information through a bottle-neck
(encoder) and then upsample from a hopefully meaningful compact representation (decoder).

Because we are fancy, we will use a U-net layout with skip-connections.

```python
NF = 64
with argscope(Conv2D, kernel_shape=4, stride=2,
							nl=lambda x, name: LeakyReLU(BatchNorm('bn', x), name=name)):
		# encoder
		e1 = Conv2D('conv1', luminance, NF, nl=LeakyReLU)
		e2 = Conv2D('conv2', e1, NF * 2)
		e3 = Conv2D('conv3', e2, NF * 4)
		e4 = Conv2D('conv4', e3, NF * 8)
		e5 = Conv2D('conv5', e4, NF * 8)
		e6 = Conv2D('conv6', e5, NF * 8)
		e7 = Conv2D('conv7', e6, NF * 8)
		e8 = Conv2D('conv8', e7, NF * 8, nl=BNReLU)  # 1x1
with argscope(Deconv2D, nl=BNReLU, kernel_shape=4, stride=2):
		# decoder
		e8 = Deconv2D('deconv1', e8, NF * 8)
		e8 = Dropout(e8)
		e8 = ConcatWith(e8, 3, e7)

		e7 = Deconv2D('deconv2', e8, NF * 8)
		e7 = Dropout(e7)
		e7 = ConcatWith(e7, 3, e6)

		e6 = Deconv2D('deconv3', e7, NF * 8)
		e6 = Dropout(e6)
		e6 = ConcatWith(e6, 3, e5)

		e5 = Deconv2D('deconv4', e6, NF * 8)
		e5 = Dropout(e5)
		e5 = ConcatWith(e5, 3, e4)

		e4 = Deconv2D('deconv5', e65, NF * 4)
		e4 = Dropout(e4)
		e4 = ConcatWith(e4, 3, e3)

		e3 = Deconv2D('deconv6', e4, NF * 2)
		e3 = Dropout(e3)
		e3 = ConcatWith(e3, 3, e2)

		e2 = Deconv2D('deconv7', e3, NF * 1)
		e2 = Dropout(e2)
		e2 = ConcatWith(e2, 3, e1)

		prediction = Deconv2D('prediction', e2, 3, nl=tf.tanh)
```

There are probably many better tutorials about defining your network model. And there are definitely [better models](../../examples/GAN/image2image.py). You should check them later. A good way to understand layers from this library is to play with those examples.

It should be noted that you can write your models using [tfSlim](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/slim)
which comes along [architectures and pre-trained models](https://github.com/tensorflow/models/tree/master/slim/nets) for image classification.
TensorPack automatically handles regularization and batchnorm updates from tfSlim. And you can directly load these pre-trained checkpoints from state-of-the-art models in TensorPack. Isn't this cool?

The remaining part is a boring L2-loss function given by:

```python
self.cost = tf.nn.l2_loss(prediction - rgb, name="L2 loss")
```

### Pimp the TensorBoard output

It is a good idea to track the progress of your training session using TensorBoard.
TensorPack provides several functions to simplify the output of summaries and visualization of intermediate states.

The following two lines

```python
add_moving_summary(self.cost)
tf.summary.image('colorized', prediction, max_outputs=10)
```

add a plot of the moving average of the cost tensor, and add some intermediate results to the tab of "images" inside TensorBoard. The summary is written after each epoch.
Note that you can certainly use `tf.summary.scalar(self.cost)`, but then you'll only see a single cost value (rather than moving average) which is much less informative.

## Training

Let's summarize: we have a model and data.
The missing piece which stitches these parts together is the training protocol.
It is only a [configuration](http://tensorpack.readthedocs.io/en/latest/modules/tensorpack.train.html#tensorpack.train.TrainConfig)

For the dataflow, we already implemented `get_data` in the first part. Specifying the learning rate is done by

```python
lr = symbolic_functions.get_scalar_var('learning_rate', 1e-4, summary=True)
```

This essentially creates a non-trainable variable with initial value `1e-4` and also track this value inside TensorBoard.
You can certainly just use `lr = 1e-4`, but then you'll lose the ability to modify it during training (through callbacks).
Let's have a look at the entire code:

```python
def get_config():
    logger.auto_set_dir()
    dataset = get_data()
    lr = symbolic_functions.get_scalar_var('learning_rate', 2e-4, summary=True)
    return TrainConfig(
        dataflow=dataset,
        optimizer=tf.train.AdamOptimizer(lr),
        callbacks=[PeriodicCallback(ModelSaver(), 3)],
        model=Model(),
        steps_per_epoch=dataset.size(),
        max_epoch=100,
    )
```

There is not really new stuff.
The model was implemented, and `max_epoch` is set to 100.
The alert reader who almost already had gone to sleep makes some noise: "Where is `dataset.size()` coming from?"
This method is implemented by `ImageFromFile` and is forwarded by all mappings.
If you have 42 images in your directory, then this value would be 42.
Satisfied with this answer, the alert reader went out of the room.
But he will miss the most interesting part: the callback section. We will cover this in the next section.

## Callbacks

Until this point, we spoke about all necessary parts of deep learning pipelines which are common for GANs, image-recognition and embedding learning.
But sometimes you want to add your own code to do something extra. We will now add a functionality which will export some entries of the tensor `prediction`.
Remember, this tensor is the result of the decoder part in our network.

To modularize the code, there is a plug-in mechanism called callbacks. Our callback looks like

```python
class OnlineExport(Callback):
    def __init__(self):
        pass

    def _setup_graph(self):
        pass

    def _trigger_epoch(self):
       pass
```

So it has 3 methods, although there are some more.
TensorPack is conservative regarding the computation graph.
After the network is constructed and all callbacks are initialized the graph is read-only.
So once you started training, there is no way of modifying the graph, which we actually want to do for inference.
You'll need to define the whole graph before training starts.

Let us fill in some parts:

```python
class OnlineExport(Callback):
    def __init__(self):
        self.cc = 0
        self.example_input = color.rgb2lab(cv2.imread('myimage.jpg')[:, :, ::-1])[:, :, 0] # read rgb image and extract luminance

    def _setup_graph(self):
        self.predictor = self.trainer.get_predictor(['luminance'], ['prediction/output'])

    def _trigger_epoch(self):
        pass
```

Can you remember the method `_get_inputs` in our model? We used the name `luminance` to identify one input.
If not, this is the best time to go back in this text and read how to specify input variables for the network.
In the deconvolution step there was also:

```python
prediction = Deconv2D('prediction', e2, 3, nl=tf.tanh) # name of the tensor is 'prediction/output'
```
Usually the name of the output tensor of a layer is in the API documentation. If you are uncertain,
you can simply `print(prediction)` to find out the name.

These two names allows us to build the inference part of the network in
```python
self.trainer.get_predictor(['luminance', 'prediction/output'])
```

This is very convenient because in the `_tigger_epoch` we can use:
```python
def _trigger_epoch(self):
        hopefully_cool_rgb = self.predictor([self.example_input])[0]
```

to do inference. Together this looks like

```python
class OnlineExport(Callback):
    def __init__(self):
        self.cc = 0
        self.example_input = color.rgb2lab(cv2.imread('myimage.jpg')[:, :, [2, 1, 0]])[:, :, 0]

    def _setup_graph(self):
        self.trainer.get_predictor(['luminance', 'prediction/output'])

    def _trigger_epoch(self):
        hopefully_cool_rgb = self.pred([[self.example_input]])[0][0]
        cv2.imwrite("export%04i.jpg" % self.cc, hopefully_cool_rgb)
        self.cc += 1
```

One note about the predictor: it allows any number of inputs or outputs, so it takes a list of names
to create a predictor, and the predictor takes list of inputs and returns list of outputs.
Also, remember that our graph takes and returns a batch but we only have one. This explains the double brackets `[[self.example_input]]`
and the double indexing `[0][0]` above.

Finally do not forget to add `OnlineExport` to you callbacks in your `TrainConfig`. Then the
inference will be executed after every epoch.

This part shows a simple callback, but there are a lot more it can do. It can manipulate summary statistics, run inference,
dump parameters, modify hyperparameters, etc. You can checkout the pre-defined callbacks to see how
to implement a powerful callback.

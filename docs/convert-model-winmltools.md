---
author: wschin
title: Convert ML models to ONNX with WinMLTools
description: Learn how to use WinMLTools to convert ML models into ONNX format.
ms.author: wechi
ms.date: 11/7/2018
ms.topic: article
keywords: windows 10, windows machine learning, WinML, WinMLTools, ONNX, ONNXMLTools, scikit-learn, Core ML, Keras
ms.localizationpriority: medium
---

# Convert ML models to ONNX with WinMLTools

[WinMLTools](https://pypi.org/project/winmltools/) is an extension of [ONNXMLTools](https://github.com/onnx/onnxmltools) to convert ML models to ONNX format to use with Windows ML. By default, WinMLTools converts models to ONNX **version 1.2.2**. WinMLTools currently supports conversion from the following frameworks:

- Apple CoreML
- scikit-learn (subset of models convertible to ONNX)
- xgboost
- libSVM
- Keras

To learn how to export from other ML frameworks, take a look at [ONNX tutorials](https://github.com/onnx/tutorials) on GitHub.

In this article, we demonstrate how to use WinMLTools to:

- Convert CoreML models into ONNX
- Convert scikit-learn models into ONNX
- Create custom ONNX operators
- Convert floating point models

## Install WinMLTools

WinMLTools is a Python package (winmltools) that supports Python versions 2.7 and 3.6. If you are working on a data science project, we recommend installing a scientific Python distribution such as Anaconda.

> [!NOTE]
> WinMLTools does not currently support Python 3.7.

WinMLTools follows the [standard python package installation process](https://packaging.python.org/installing/). From your python environment, run:

```console
pip install winmltools
```

WinMLTools has the following dependencies:

- numpy v1.10.0+
- onnxmltools 1.0.0.0+
- protobuf v.3.1.0+

To update the dependent packages, please run the pip command with ‘-U’ argument.

```console
pip install -U winmltools
```

Please follow [onnxmltools](https://github.com/onnx/onnxmltools) on GitHub for further information on onnxmltools dependencies.

Additional details on how to use WinMLTools can be found on the package specific documentation with the help function.

```console
help(winmltools)
```

## Convert CoreML models

Here, we assume that the path of an example Core ML model file is *example.mlmodel*.

~~~python
from coremltools.models.utils import load_spec
# Load model file
model_coreml = load_spec('example.mlmodel')
from winmltools import convert_coreml
# Convert it!
# The automatic code generator (mlgen) uses the name parameter to generate class names.
model_onnx = convert_coreml(model_coreml, name='ExampleModel')   
~~~

The `model_onnx` is an ONNX [ModelProto](https://github.com/onnx/onnxmltools/blob/0f453c3f375c1ae928b83a4c7909c82c013a5bff/onnxmltools/proto/onnx-ml.proto#L176) object. We can save it in two different formats.

~~~python
from winmltools.utils import save_model
# Save the produced ONNX model in binary format
save_model(model_onnx, 'example.onnx')
# Save the produced ONNX model in text format
from winmltools.utils import save_text
save_text(model_onnx, 'example.txt')
~~~

**Note**: Core MLTools is a Python package provided by Apple, but is not available on Windows. If you need to install the package on Windows, install the package directly from the repo:

```console
pip install git+https://github.com/apple/coremltools
```

## Convert CoreML models with image inputs or outputs

Because of the lack of image types in ONNX, converting Core ML image models (i.e., models using images as inputs or outputs) requires some pre-processing and post-processing steps.

The objective of pre-processing is to make sure the input image is properly formatted as an ONNX tensor. Assume *X* is an image input with shape [C, H, W] in Core ML. In ONNX, the variable *X* would be a floating-point tensor with the same shape and *X*[0, :, :]/*X*[1, :, :]/*X*[2, :, :] stores the image's red/green/blue channel. For gray scale images in Core ML, their format are [1, H, W]-tensors in ONNX because we only have one channel.

If the original Core ML model outputs an image, manually convert ONNX's floating-point output tensors back into images. There are two basic steps. The first step is to truncate values greater than 255 to 255 and change all negative values to 0. The second step is to round all pixel values to integers (by adding 0.5 and then truncating the decimals). The output channel order (e.g., RGB or BGR) is indicated in the Core ML model. To generate proper image output, we may need to transpose or shuffle to recover the desired format.

Here we consider a Core ML model, FNS-Candy, downloaded from [GitHub](https://github.com/likedan/Awesome-CoreML-Models), as a concrete conversion example to demonstrate the difference between ONNX and Core ML formats. Note that all the subsequent commands are executed in a python environment.

First, we load the Core ML model and examine its input and output formats.

~~~python
from coremltools.models.utils import load_spec
model_coreml = load_spec('FNS-Candy.mlmodel')
model_coreml.description # Print the content of Core ML model description
~~~

Screen output:

~~~console
...
input {
    ...
      imageType {
      width: 720
      height: 720
      colorSpace: BGR
    ...
}
...
output {
    ...
      imageType {
      width: 720
      height: 720
      colorSpace: BGR
    ...
}
...
~~~

In this case, both the input and output are 720x720 BGR-image. Our next step is to convert the Core ML model with WinMLTools.

~~~python
# The automatic code generator (mlgen) uses the name parameter to generate class names.
from onnxmltools import convert_coreml
model_onnx = convert_coreml(model_coreml, name='FNSCandy')    
~~~

An alternative method to view the model input and output formats in ONNX, is to use the following command:

~~~python
model_onnx.graph.input # Print out the ONNX input tensor's information
~~~

Screen output:

~~~console
...
  tensor_type {
    elem_type: FLOAT
    shape {
      dim {
        dim_param: "None"
      }
      dim {
        dim_value: 3
      }
      dim {
        dim_value: 720
      }
      dim {
        dim_value: 720
      }
    }
  }
...
~~~

The produced input (denoted by *X*) in ONNX is a 4-D tensor. The last 3 axes are C-, H-, and W-axes, respectively. The first dimension is "None" which means that this ONNX model can be applied to any number of images. To apply this model to process a batch of 2 images, the first image corresponds to *X*[0, :, :, :] while *X*[1, :, :, :] corresponds to the second image. The blue/green/red channels of the first image are *X*[0, 0, :, :]/*X*[0, 1, :, :]/*X*[0, 2, :, :], and similar for the second image.

~~~python
model_onnx.graph.output # Print out the ONNX output tensor's information
~~~

Screen output:

~~~console
...
  tensor_type {
    elem_type: FLOAT
    shape {
      dim {
        dim_param: "None"
      }
      dim {
        dim_value: 3
      }
      dim {
        dim_value: 720
      }
      dim {
        dim_value: 720
      }
    }
  }
...
~~~

As you can see, the produced format is identical to the original model input format. However, in this case, it's not an image because the pixel values are integers, not floating-point numbers. To convert back to an image, truncate values greater than 255 to 255, change negative values to 0, and round all decimals to integers.

## Convert Scikit-learn models

The following code snippet trains a linear support vector machine in scikit-learn and converts the model into ONNX.

~~~python
# First, we create a toy data set.
# The training matrix X contains three examples, with two features each.
# The label vector, y, stores the labels of all examples.
X = [[0.5, 1.], [-1., -1.5], [0., -2.]]
y = [1, -1, -1]

# Then, we create a linear classifier and train it.
from sklearn.svm import LinearSVC
linear_svc = LinearSVC()
linear_svc.fit(X, y)

# To convert scikit-learn models, we need to specify the input feature's name and type for our converter. 
# The following line means we have a 2-D float feature vector, and its name is "input" in ONNX.
# The automatic code generator (mlgen) uses the name parameter to generate class names.
from winmltools import convert_sklearn
from onnxmltools.convert.common.data_types import FloatTensorType
linear_svc_onnx = convert_sklearn(linear_svc, name='LinearSVC',
                                  input_features=[('input', FloatTensorType([1, 2]))])    

# Now, we save the ONNX model into binary format.
from winmltools.utils import save_model
save_model(linear_svc_onnx, 'linear_svc.onnx')

# If you'd like to load an ONNX binary file, our tool can also help.
from winmltools.utils import load_model
linear_svc_onnx = load_model('linear_svc.onnx')

# To see the produced ONNX model, we can print its contents or save it in text format.
print(linear_svc_onnx)
from winmltools.utils import save_text
save_text(linear_svc_onnx, 'linear_svc.txt')

# The conversion of linear regression is very similar. See the example below.
from sklearn.svm import LinearSVR
linear_svr = LinearSVR()
linear_svr.fit(X, y)
linear_svr_onnx = convert_sklearn(linear_svr, name='LinearSVR', 
                                  input_features=[('input', FloatTensorType([1, 2]))])   
~~~

Users can replace `LinearSVC` with other scikit-learn models such as `RandomForestClassifier`. Please note that [mlgen](mlgen.md) uses the `name` parameter to generate class names and variables. If `name` is not provided, then a GUID is generated, which will not comply with variable naming conventions for languages like C++/C#.

## Convert Scikit-learn pipelines

Next, we show how scikit-learn pipelines can be converted into ONNX.

~~~python
# First, we create a toy data set.
# Notice that the first example's last feature value, 300, is large.
X = [[0.5, 1., 300.], [-1., -1.5, -4.], [0., -2., -1.]]
y = [1, -1, -1]

# Then, we declare a linear classifier.
from sklearn.svm import LinearSVC
linear_svc = LinearSVC()

# One common trick to improve a linear model's performance is to normalize the input data.
from sklearn.preprocessing import Normalizer
normalizer = Normalizer()

# Here, we compose our scikit-learn pipeline. 
# First, we apply our normalization. 
# Then we feed the normalized data into the linear model.
from sklearn.pipeline import make_pipeline
pipeline = make_pipeline(normalizer, linear_svc)
pipeline.fit(X, y)

# Now, we convert the scikit-learn pipeline into ONNX format. 
# Compared to the previous example, notice that the specified feature dimension becomes 3.
# The automatic code generator (mlgen) uses the name parameter to generate class names.
from winmltools import convert_sklearn
from onnxmltools.convert.common.data_types import FloatTensorType, Int64TensorType
pipeline_onnx = convert_sklearn(linear_svc, name='NormalizerLinearSVC',
                                input_features=[('input', FloatTensorType([1, 3]))])   

# We can print the fresh ONNX model.
print(pipeline_onnx)

# We can also save the ONNX model into binary file for later use.
from winmltools.utils import save_model
save_model(pipeline_onnx, 'pipeline.onnx')

# Our conversion framework provides limited support of heterogeneous feature values.
# We cannot have numerical types and string type in one feature vector. 
# Let's assume that the first two features are floats and the last feature is integer (encoded a categorical attribute).
X_heter = [[0.5, 1., 300], [-1., -1.5, 400], [0., -2., 100]]

# One popular way to represent categorical is one-hot encoding.
from sklearn.preprocessing import OneHotEncoder
one_hot_encoder = OneHotEncoder(categorical_features=[2])

# Let's initialize a classifier. 
# It will be right after the one-hot encoder in our pipeline.
linear_svc = LinearSVC()

# Then, we form a two-stage pipeline.
another_pipeline = make_pipeline(one_hot_encoder, linear_svc)
another_pipeline.fit(X_heter, y)

# Now, we convert, save, and load the converted model. 
# For heterogeneous feature vectors, we need to seperately specify their types for all homogeneous segments.
# The automatic code generator (mlgen) uses the name parameter to generate class names.
another_pipeline_onnx = convert_sklearn(another_pipeline, name='OneHotLinearSVC',
                                        input_features=[('input', FloatTensorType([1, 2])),
                                        ('another_input', Int64TensorType([1, 1]))])
save_model(another_pipeline_onnx, 'another_pipeline.onnx')
from winmltools.utils import load_model
loaded_onnx_model = load_model('another_pipeline.onnx')

# Of course, we can print the ONNX model to see if anything went wrong.
print(another_pipeline_onnx)
~~~

## Create custom ONNX operators

When converting from a Keras or a Core ML model, you can write a custom operator function to embed custom [operators](https://github.com/onnx/onnx/blob/master/docs/Operators.md) into the ONNX graph. During the conversion, the converter invokes your function to translate the Keras layer or the Core ML LayerParameter to an ONNX operator, and then it connects the operator node into the whole graph.

1. Create the custom function for the ONNX sub-graph building.
2. Call `winmltools.convert_keras` or `winmltools.convert_coreml` with the map of the custom layer name to the custom function.S
3. If applicable, implement the custom layer for the inference runtime.  

The following example shows how it works in Keras.

~~~python
# Define the activation layer.
class ParametricSoftplus(keras.layers.Layer):
    def __init__(self, alpha, beta, **kwargs):
    ...
    ...
    ...

# Create the convert function.
def convert_userPSoftplusLayer(scope, operator, container):
      return container.add_node('ParametricSoftplus', operator.input_full_names, operator.output_full_names,
        op_version=1, alpha=operator.original_operator.alpha, beta=operator.original_operator.beta)

winmltools.convert_keras(keras_model,
    custom_conversion_functions={ParametricSoftplus: convert_userPSoftplusLayer })
~~~

## Convert to floating point 16

Most models are represented in floating point 32, but if you prefer model efficiency over accuracy, then you can convert your model to floating point 16.

~~~python
import winmltools
from winmltools.utils import convert_float_to_float16
new_onnx_model = convert_float_to_float16(onnx_model)
~~~

With `help(winmltools.utils.convert_float_to_float16)`, you can find more details about this tool. The floating data 16 in WinMLTools currently only complies with [IEEE 754 floating point standard (2008)](https://en.wikipedia.org/wiki/Half-precision_floating-point_format).

Here is a full example if you want to convert directly from an ONNX binary file. 

~~~python
from winmltools.utils import convert_float_to_float16
from winmltools.utils import load_model, save_model
onnx_model = load_model('model.onnx')
new_onnx_model = convert_float_to_float16(onnx_model)
save_model(new_onnx_model, 'model_fp16.onnx')
~~~

[!INCLUDE [help](includes/get-help.md)]

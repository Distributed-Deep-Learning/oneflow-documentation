In [data_input](../basics_topics/data_input.md) section， we know that because of the support like Multi-threading, scheduling of resources in OneFlow, the efficiency of processing data will be highter in OneFlow data-pipeline. Also, we learn about the operation in data-pipeline.

In [OFRecord](ofrecord.md) section, we learn about the storage format of OFRecord files.

In this section, we will focus on the loading and producing of OneFlow's OFRecord dataset. Mainly including:

* The organization of OFRecord dataset

* The multiple ways of loading OFRecord dataset

* The transition of OFRecord dataset to other dataformats

## What is OFRecord dataset
In [OFRecord](ofrecord.md) section. We have introduced the storage format of `OFRecord file `, and we also have known about what is `OFRecord file`

OFRecord dataset is **the collection of OFRecord files **.According to the OneFlow convention file format, we store `multiple OFRecord files` in the same directory, then we will get OFRecord dataset.

By default, The files in OFRecord dataset directory are uniformly named in the way of `part-xxx`, where "xxx" is the file id starting from zero, and there can be two choices of completement or non-completement.

Here is the example of using non-completement name style:
```
mnist_kaggle/train/
├── part-0
├── part-1
├── part-10
├── part-11
├── part-12
├── part-13
├── part-14
├── part-15
├── part-2
├── part-3
├── part-4
├── part-5
├── part-6
├── part-7
├── part-8
└── part-9
```

Here is the example of using completement name style:
```
mnist_kaggle/train/
├── part-00000
├── part-00001
├── part-00002
├── part-00003
├── part-00004
├── part-00005
├── part-00006
├── part-00007
├── part-00008
├── part-00009
├── part-00010
├── part-00011
├── part-00012
├── part-00013
├── part-00014
├── part-00015
```
OneFlow adopts this convention, which is consistent with the default storage filename in `spark`, it is convenient to make and convert to OFRecord data by spark.

Actually, we can specify the filename prefix(`part-`)，whether to complete the filename id, how many bits to complete.We just need to keep the same parameters when loading dataset(described below)

OneFlow provides the API interface to load OFRecord dataset, so that we can enjoy the Multi-threading, pipeline and some other advantages brought by OneFlow framework by specifying the path of dataset directory.

## The method to load OFRecord dataset
We usually use `decode_ofrecord` to load and decode dataset; or use `ofrecord_reader` to load and preprocess dataset.

### `decode_ofrecord`
We can use `flow.data.decode_ofrecord` to load and decode the dataset at the same time. `decode_ofrecord` 的调用接口如下：
```python
def decode_ofrecord(
    ofrecord_dir,
    blobs,
    batch_size=1,
    data_part_num=1,
    part_name_prefix="part-",
    part_name_suffix_length=-1,
    shuffle=False,
    buffer_size=1024,
    name=None,
)
```

它的常用非必需参数及其意义如下：

* batch_size： 一次训练所选取的数据个数

* data_part_num： 数据集中 OFRecord 文件的个数

* part_name_prefix： 数据集中 OFRecord 文件的文件名前缀

* part_name_suffix_length： 数据集中 OFRecord 文件编号的补齐长度，如 `part-00001` 这种文件名，其 `part_name_suffix_length` 应该设置为5，-1表示无补齐

* shuffle：数据获取时顺序是否随机打乱

* buffer_size： 数据流水线中样本的数量，比如，若设置为1024表示缓冲区中一共1024个样本，则以上参数 shuffle 为 True 时，是针对缓冲区中的1024个样本进行打乱

其中必需参数 `ofrecord_dir` 为数据集目录的路径，`blobs` 为一个tuple，tuple 中存有需要读取数据集中的`Feature`(参考[OFrecord数据格式](ofrecord.md))，我们将在下文结合实例，介绍如何定义 `blobs` 参数。

完整代码：[decode_ofrecord.py](../code/extended_topics/decode_ofrecord.py)

```python
import oneflow as flow

def get_train_config():
  config = flow.function_config()
  config.default_data_type(flow.float)
  return config


@flow.global_function(get_train_config())
def train_job():
  images = flow.data.BlobConf("images", 
          shape=(28, 28, 1), 
          dtype=flow.float, 
          codec=flow.data.RawCodec())
  labels = flow.data.BlobConf("labels", 
          shape=(1, 1), 
          dtype=flow.int32, 
          codec=flow.data.RawCodec())

  return flow.data.decode_ofrecord("./dataset/", (images, labels),
                                data_part_num=1,
                                batch_size=3)

def main():
  check_point = flow.train.CheckPoint()
  check_point.init()

  f0, f1 = train_job().get()
  print(f0.ndarray(), f1.ndarray())
  print(f0.shape, f1.shape)

if __name__ == '__main__':
  main()
```

以上的代码，加载[OFrecord数据格式](ofrecord.md)一文中"将 OFRecord 对象写入文件"中所写入的数据集。

运行后得到类似如下结果：
```text
... [[0.5941235 ]
   [0.27485612]
   [0.4714867 ]
   ... [0.21632855]
   [0.15881447]
   [0.65982276]]]] [[[2]]

 [[3]]

 [[1]]]
(3, 28, 28, 1) (3, 1, 1)
```

可以看到，我们使用 `flow.data.BlobConf` 声明与数据集中 `Feature` 对应的占位符，`BlobConf` 的必需参数有：
```python
 BlobConf(name, shape, dtype, codec)
```

* name：在制作 OFRecord 文件时，Feature 所对应的 Key；

* shape：数据对应的形状，需要与 Feature 中元素个数一致。如上文中的`(28, 28, 1)`修改为`(14, 28*2, 1)`或者`(28, 28)`均可；

* dtype：数据类型，需要与写入数据集中的 Feature 数据类型一致；

* codec： 解码器，OneFlow 内置了诸如 `RawCodec`、`ImageCodec`、`BytesListCodec`等解码器。上例中我们使用 `RawCodec`。

使用 `BlobConf` 得到占位符后，我们可以使用 `decode_ofrecord` 方法，从数据集中获取数据。
```python
    flow.data.decode_ofrecord("./dataset/", (images, labels),
                            data_part_num=1,
                            batch_size=3)
```

通过以上例子可以总结使用 `decode_ofrecord` 的基本步骤：

* 通过 `BlobConf` 定义占位符，用于提取数据集中的 `Feature`

* 调用 `decode_ofrecord`，将上一步定义的占位符传递给 `decode_ofrecord` ，并设置相关参数，获取数据集中的数据

使用 `decode_ofrecord` 的方式提取数据中的 `Feature` 虽然方便，但是支持的预处理方式和解码器种类有限。如果需要更灵活的数据预处理方式，包括自定义用户 op，推荐使用 `ofrecord_reader`。

### `ofrecord_reader`
在[数据输入](../basics_topics/data_input.md)一文中，我们已经展示了如何使用 `ofrecord_reader` 接口加载 OFRecord 数据，并进行数据预处理：

完整代码：[of_data_pipeline.py](../code/basics_topics/of_data_pipeline.py)

```python
import oneflow as flow

@flow.global_function(flow.function_config())
def test_job():
  batch_size = 64
  color_space = 'RGB'
  with flow.scope.placement("cpu", "0:0"):
    ofrecord = flow.data.ofrecord_reader('/path/to/ImageNet/ofrecord',
                                         batch_size = batch_size,
                                         data_part_num = 1,
                                         part_name_suffix_length = 5,
                                         random_shuffle = True,
                                         shuffle_after_epoch = True)
    image = flow.data.OFRecordImageDecoderRandomCrop(ofrecord, "encoded",
                                                     color_space = color_space)
    label = flow.data.OFRecordRawDecoder(ofrecord, "class/label", shape = (), dtype = flow.int32)
    rsz = flow.image.Resize(image, resize_x = 224, resize_y = 224, color_space = color_space)

    rng = flow.random.CoinFlip(batch_size = batch_size)
    normal = flow.image.CropMirrorNormalize(rsz, mirror_blob = rng, color_space = color_space,
                                            mean = [123.68, 116.779, 103.939],
                                            std = [58.393, 57.12, 57.375],
                                            output_dtype = flow.float)
    return normal, label

if __name__ == '__main__':
  images, labels = test_job().get()
  print(images.shape, labels.shape)
```

`ofrecord_reader` 的接口如下：
```python
def ofrecord_reader(
    ofrecord_dir,
    batch_size=1,
    data_part_num=1,
    part_name_prefix="part-",
    part_name_suffix_length=-1,
    random_shuffle=False,
    shuffle_buffer_size=1024,
    shuffle_after_epoch=False,
    name=None,
)
```

使用 `ofrecord_reader` 的好处在于可以用数据处理流水线的方式进行数据预处理，而且可以通自定义预处理 op，拥有很高的灵活性和扩展性。

* 关于数据流水线及预处理可以参考[数据输入](../basics_topics/data_input.md)

* 关于自定义OP可以参考[用户自定义op](user_op.md)

## 其它格式数据与 OFRecord 数据集的相互转化
参考[OFrecord数据格式](ofrecord.md)中 OFRecord 文件的存储格式及本文开头介绍的 OFRecord 数据集的文件名格式约定，我们完全可以自己制作 OFRecord 数据集。

不过为了更加方便，我们提供了 Spark 的 jar 包，方便 OFRecord 与常见数据格式(如 TFRecord、json)进行相互转化。

### spark 的安装与启动
首先，下载 spark 及 spark-oneflow-connector：

* 在 spark 官网下载[spark-2.4.0-bin-hadoop2.7](https://archive.apache.org/dist/spark/spark-2.4.0/spark-2.4.0-bin-hadoop2.7.tgz)

* 在[这里](https://oneflow-static.oss-cn-beijing.aliyuncs.com/oneflow-tutorial-attachments/spark-oneflow-connector-assembly-0.1.0_int64.jar)下载 jar 包，spark 需要它来支持 ofrecord 格式

接着，解压 `spark-2.4.0-bin-hadoop2.7.tgz`，并配置环境变量 `SPARK_HOME`:
```shell
export SPARK_HOME=path/to/spark-2.4.0-bin-hadoop2.7
```

然后，通过以下命令启动 pyspark shell：
```shell
pyspark --master "local[*]"\
 --jars spark-oneflow-connector-assembly-0.1.0_int64.jar\
 --packages org.tensorflow:spark-tensorflow-connector_2.11:1.13.1
```

```text
... Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.4.0
      /_/

Using Python version 3.6.10 (default, May  8 2020 02:54:21)
SparkSession available as 'spark'. >>> 
```

在启动的 pyspark shell 中，我们可以完成 OFRecord 数据集与其它数据格式的相互转化。

### 使用 spark 查看 OFRecord 数据集
使用以下命令可以查看 OFRecord 数据：
```
spark.read.format("ofrecord").load("file:///path/to/ofrecord_file").show()
```
默认显示前20条数据:
```
+--------------------+------+
|              images|labels|
+--------------------+------+
|[0.33967614, 0.87...|     2|
|[0.266905, 0.9730...|     3|
|[0.66661334, 0.67...|     1|
|[0.91943026, 0.89...|     6|
|[0.014844197, 0.0...|     6|
|[0.5366513, 0.748...|     4|
|[0.055148937, 0.7...|     7|
|[0.7814437, 0.228...|     4|
|[0.31193638, 0.55...|     3|
|[0.20034336, 0.24...|     4|
|[0.09441255, 0.07...|     3|
|[0.5177533, 0.397...|     0|
|[0.23703437, 0.44...|     9|
|[0.9425567, 0.859...|     9|
|[0.017339867, 0.0...|     3|
|[0.827106, 0.3122...|     0|
|[0.8641392, 0.194...|     2|
|[0.95585227, 0.29...|     3|
|[0.7508129, 0.464...|     4|
|[0.035597708, 0.3...|     9|
+--------------------+------+
only showing top 20 rows
```


### 与 TFRecord 数据集的相互转化
以下命令可以将 TFRecord 转化为 OFRecrod：

```python
reader = spark.read.format("tfrecords")
dataframe = reader.load("file:///path/to/tfrecord_file")
writer = dataframe.write.format("ofrecord")
writer.save("file:///path/to/outputdir")
```
以上代码中的 `outputdir` 目录会被自动创建，并在其中保存 ofrecord 文件。在执行命令前应保证 outputdir 目录不存在。

此外，还可以使用以下命令，在转化的同时，将数据切分为多个 ofrecord 文件：
```python
reader = spark.read.format("tfrecords")
dataframe = reader.load("file:///path/to/tfrecord_file")
writer = dataframe.repartition(10).write.format("ofrecord")
writer.save("file://path/to/outputdir")
```
以上命令执行后，在 outputdir 目录下会产生10个 `part-xxx` 格式的ofrecord文件。

将 OFRecord 文件转为 TFRecord 文件的过程类似，交换读/写方的 `format` 即可：
```python
reader = spark.read.format("ofrecord")
dataframe = reader.load("file:///path/to/ofrecord_file")
writer = dataframe.write.format("tfrecords")
writer.save("file:///path/to/outputdir")
```

### 与 JSON 格式的相互转化
以下命令可以将 JSON 格式数据转为 OFRecord 数据集:
```python
dataframe = spark.read.json("file:///path/to/json_file")
writer = dataframe.write.format("ofrecord")
writer.save("file:///path/to/outputdir")
```

以下命令将 OFRecord 数据转为 JSON 文件：
```python
reader = spark.read.format("ofrecord")
dataframe = reader.load("file:///path/to/ofrecord_file")
dataframe.write.json("file://path/to/outputdir")
```

### 其它脚本及工具
除了以上介绍的方法，使得其它数据格式与 OFRecord 数据格式进行转化外，在[oneflow_toolkit](https://github.com/Oneflow-Inc/oneflow_toolkit/tree/master/ofrecord)仓库下，还有各种与 OFRecord有关的脚本：

* 为 BERT 模型准备 OFRecord 数据集的脚本

* 下载 ImageNet 数据集并转 OFRecord 数据集的工具

* 下载 MNIST 数据集并转 OFRecord 数据集的工具

* 将微软 Coco 转 OFRecord 数据集的工具


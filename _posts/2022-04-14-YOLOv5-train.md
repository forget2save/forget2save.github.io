---
layout: default
---

# YOLO v5训练源码阅读

参考根目录下`train.py`文件

## 常量

使用pathlib来管理路径，`FILE`就是`train.py`文件所在的路径，`ROOT`是`train.py`文件所在文件夹的路径。
`LOCAL_RANK`，`RANK`，`WORLD_SIZE`是几个与分布式训练有关的常量，不影响对核心代码的理解。

## 主流程

1. 调用`parse_opt()`获取可选参数的值。
2. 进入`main()`主程序，此步只是一个简单的函数包装。
3. 判断是继续训练还是从头开始：
	1. 继续训练。读取存放在`opt.resume`或者默认位置中的checkpoint，根据相对位置查询到`opt.yaml`文件（存放可选参数），并读取到变量`opt`中。
	2. 从头开始。检查由可选参数指定的`data`，`cfg`，`hyp`，`weights`，`project`是否合法。
4. 分布式数据并行模式（DDP mode），仍然是与分布式训练相关的准备操作，不是核心代码。
5. 正式训练环节，`opt.evolve`是用来进行超参数调优的选项，在一次常规的训练过程中，一般直接使用已经由官方调优过的超参数，所以只需要看第一个分支，其中核心的只有`train()`函数。
	1. 将可选参数中的关键参数读取到局部变量，同时进行一些预处理，比如从超参文件中读取超参数，构成一个`hyp`命名空间。
	2. 直到91行`Config`部分前的代码都不是很关键，可以简单地理解为一些准备操作。96行的`data_dict`很关键，它指示了数据集的位置，它的值是通过`check_dataset(data)`函数返回的。在后续几行的代码中，我们可以看到`train_path`，`val_path`分别指示了训练集和验证集的地址，`names`指示了数据集的各个类别，`is_coco`指示了数据集是否为COCO2017，这些信息全存在`data_dict`的键值对中。从后续的代码中会发现`is_coco`只会影响最后的评估环节，与训练过程无关，因此并不重要。
	3. 103行进入模型加载环节，当`weights`中存在`.pt`文件时，就会采取预训练模式，它会将`weights`所指示的模型文件中的参数加载进来，唯一例外的是`anchor`是从超参文件中加载的。若不存在则构建一个新模型，`anchor`同样从超参文件中加载。
	4. 119行是冻结模型参数的选项，通常出于减少训练时长的目的，往往取决于finetune过程中当前数据集与预训练数据集的差异大小，只有在较小的情况下才会考虑冻结，默认不需要。
	5. to be continue

## 关键函数

由于目前最直接的需求是替换一个数据集进行训练，所以最需要关注的是dataset这一部分的代码，其次是optimizer的一些超参数，最后才是model本身的结构。

1. 数据集配置（91~101行）

`opt.data`或者`data`：程序可选参数，默认值是工程目录下的`data/coco128.yaml`文件
`train_path`：记录在yaml文件中，指向存放训练集图片的文件夹
`val_path`：记录在yaml文件中，指向存放验证集图片的文件夹
`nc`：数据集的类别总数，COCO是80类
`names`：数据集各类别的名称
`check_dataset(data)`：负责校验、读取、下载`opt.data`指向的数据集配置文件
	yaml中的`path`指示了数据集的根目录，可以是绝对路径也可以是相对路径
	yaml中的`train`，`val`，`test`指示了各子集的相对于数据集根目录的路径；若`val`中的位置不存在数据集，那么会从yaml中`download`所指的网址下载并解压到`val`中的位置

2. 数据加载器配置（208~215行）

`create_dataloader()`构造了`DataLoader`和`Dataset`两个类的实例，前者是后者的高级接口。进入函数后，可以发现这又是一层包装，核心在于`utils/datasets.py`中的`LoadImagesAndLabels`类，它继承自`Dataset`类，且构造了`DataLoader`的`collate_fn`，这决定了每次加载时，图像与标签的构造方式。
`LoadImagesAndLabels.__init__()`：初始化过程：
	1. 从`path`中读取图片，如果`path`是一个文件夹，那么它就会读取该文件夹包括其所有子目录下的所有图片格式文件；如果`path`是一个文本文件，那么它其中的每一行都应是一张图片的路径
	2. 读取`label`，每张图片的标签要求以`.txt`的形式保存在图片所在路径的对应位置，例如：图片位置`xxx/images/train/1.png`，标签文件位置`xxx/labels/train/1.txt`，其实就是替换了文件格式和`images`和`labels`的文件夹名称
	3. 由于图片的标签是以文件格式存储，每次使用都从硬盘读取会很不方便，所以需要读取到内存中，第一次读取时会使用`LoadImagesAndLabels.cache_labels`创建npy格式的键值对文件，里面以`np.ndarray`格式存储标签，其格式为`label(int:0~nc-1), xc(float:0~1), yc(float:0~1), w(float:0~1), h(float:0~1)`
`LoadImagesAndLabels.__getitem__()`：单次读取过程：
	1. to be continue


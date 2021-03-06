# 基于PaddleX的X光安检机打火机检测

针对X光安检机扫描结果进行目标检测，检测目标为打火机。

# 一、项目背景

项目构想始于训练营第一课，四位导师从工作、生活、爱好三个方面阐述了优秀的项目创意来源，该项目也是始于生活发现，七月底国内疫情反复，高铁站、地铁等公共交通场所实施严格的安检，本人从武汉回家时亲身体验了安检场所的拥堵问题，而该项目如果落地，则从一定程度上提高了安检效率，减少了人力消耗，同时也减少了安检场所的人员聚集现象，有助于疫情防控。

# 二、数据集简介

本项目使用AIStudio公开数据集：[安检X光图片](http://aistudio.baidu.com/aistudio/datasetdetail/67989),数据集已经提前下载好了。

<font size="3" color="red">特别注意，该数据集中没有划分预测集，且两个划分数据集的split files中‘/’都是反的，直接使用会找不到路径报错，这里使用PaddleX重新划分数据集</font>




```python
# 解压数据集（解压一次即可，请勿重复解压）
!unzip -oq /home/aistudio/D0002.zip -d /home/aistudio/data/
```


```python
# 安装PaddleX
!pip install paddlex
```


```python
# 按照7:2:1重新划分数据集
!paddlex --split_dataset --format VOC --dataset_dir data/D0002 --val_value 0.2 --test_value 0.1
```


```python
# 查看数据集文件结构
!tree /home/aistudio/data/D0002 -L 1
```

# 三、模型选择

本项目使用PaddleX套件yolov3_mobilenetv1模型进行训练


# 四、开始训练

## 首先预处理图片数据集


```python
#设置训练GPU
import os
os.environ['CUDA_VISIBLE_DEVICES'] = '0'
```


```python
from paddlex.det import transforms

# 定义训练和验证时的transforms，这里使用了mixup,randomdistort,resize,normalize四个处理手段，()内留空为使用默认参数设置
# API说明 https://paddlex.readthedocs.io/zh_CN/develop/apis/transforms/det_transforms.html

train_transforms = transforms.Compose([
    transforms.MixupImage(mixup_epoch=250), transforms.RandomDistort(),
    transforms.Resize(target_size=608, interp='RANDOM'), 
    transforms.Normalize()
])

eval_transforms = transforms.Compose([
    transforms.Resize(
        target_size=608, interp='CUBIC'), transforms.Normalize()
])
```


```python
import paddlex as pdx
# 定义训练和验证所用的数据集
# API说明：https://paddlex.readthedocs.io/zh_CN/develop/apis/datasets.html#paddlex-datasets-vocdetection
train_dataset = pdx.datasets.VOCDetection(
    data_dir='data/D0002',
    file_list='data/D0002/train_list.txt',
    label_list='data/D0002/labels.txt',
    transforms=train_transforms,
    shuffle=True)

eval_dataset = pdx.datasets.VOCDetection(
    data_dir='data/D0002',
    file_list='data/D0002/val_list.txt',
    label_list='data/D0002/labels.txt',
    transforms=eval_transforms)
```

## 调整参数 开始炼丹！

调参是影响训练结果的关键，本案例调参相对简单（因为没学到家），大家可以反复尝试，了解各参数训练过程产生的影响


```python
# 初始化模型
# API说明: https://paddlex.readthedocs.io/zh_CN/develop/apis/models/detection.html#paddlex-det-yolov3
model = pdx.det.YOLOv3(num_classes=len(train_dataset.labels), backbone='MobileNetV1')
```


```python
# 模型训练
# API说明: https://paddlex.readthedocs.io/zh_CN/develop/apis/models/detection.html#id1
# 各参数介绍与调整说明：https://paddlex.readthedocs.io/zh_CN/develop/appendix/parameters.html
model.train(
    num_epochs=120,
    train_dataset=train_dataset,
    train_batch_size=8,
    eval_dataset=eval_dataset,
    learning_rate=0.0125,
    lr_decay_epochs=[210, 240],
    save_dir='output/yolov3_mobilenetv1')
```

## 模型导出 再预测一下

这里就不做部署了（因为还没学会）


```python
#导出最佳模型 可用于后续部署
#部署说明： https://paddlex.readthedocs.io/zh_CN/release-1.3/deploy/export_model.html
!paddlex --export_inference --model_dir=./output/yolov3_mobilenetv1/best_model --save_dir=./inference_model
```


```python
# 模型预测 单张图片预测

import paddlex as pdx
model = pdx.load_model('./inference_model')
image_name = 'data/D0002/JPEGImages/000366401017449.jpg'
result = model.predict(image_name)
pdx.det.visualize(image_name, result, threshold=0.5, save_dir='./inference_model')
```

# 五、总结与升华


本项目还有很多需要改进之处，比如检测对象不限于打火机，还可以有其他的危险物品；比如模型参数、模型部署还有很多可以调整完善的地方等等，个人认为这个项目落地后实际价值相当不俗，各位大牛如果有心可以在这个方向上继续推进。


通过一个月不到的AI达人创造营的活动，我从一个彻彻底底的零基础门外汉，成长到现在能够自己在AIStudio上进行模型训练的小菜鸟，这十几二十天的学习使我受益匪浅，感谢老师们的教学，尤其是最后两天顾茜老师和郑院士的“手把手教学”，前面很多没理解的地方一下有了眉目，在这里提一个小小的建议，像安全帽检测的全流程实战这样的课程其实可以放在最前面讲，可以让我们对PaddlePaddle有一个完整地认知，后面的学习可能会更轻松一些。





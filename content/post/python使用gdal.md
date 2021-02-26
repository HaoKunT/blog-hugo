+++
title = "python使用gdal"  # 文章标题
date = 2021-02-26T15:04:16+08:00  # 自动添加日期信息
draft = false  # 设为false可被编译为HTML，true供本地修改
tags = ["gdal", "遥感影像"]  # 文章标签，可设置多个，用逗号隔开。Hugo会自动生成标签的子URL
categories = ["遥感"]
toc = true
preserveTaxonomyNames = true
disablePathToLower = true

+++

> 本文为原创文章，转载注明出处，欢迎关注网站[https://hkvision.cn](https://hkvision.cn)

## 缘起

一个项目需要用GDAL这个库，然后去稍微深入看了下这玩意，感觉还是有很多坑的。



## 简介

就不介绍了，需要用这玩意的肯定知道这是啥



## 读写

最重要的肯定是读写

读:

```python
from osgeo import gdal
ds = gdal.Open("{image_path}")
"""
有很多种Open的方法
OpenShared
OpenEx
还可以设置只读
flags=gdal.GA_ReadOnly
"""
```



写:

```python
drv = gdal.GetDriverByName("GTiff")
drv.Create("{filepath}", xsize, ysize, bds_num, eType. options=[])
```

看起来一切都很美好，但是这里面**蕴含着一个很大的坑**，读影像还好，但是写影像需要指定路径，并且，**gdal没有额外的方式能创建一个空的`DataSet`**，这意味着如果你想弄一些临时的`DataSet`（这非常普遍，稍微复杂一点的算法都会需要有这个），你必须要存在硬盘上一份文件，**这简直太蠢了**。



## VRT

当然这也不是没法解决的，GDAL其实有解决方案，那就是`VRT`文件，这个是GDAL的一种虚拟文件，是对影像的一种抽象表述，这个设定是很有意思的，给大家看个VRT文件的例子

```xml
<VRTDataset rasterXSize="4000" rasterYSize="4000" subClass="VRTPansharpenedDataset">
  <Metadata domain="IMAGE_STRUCTURE">
    <MDI key="INTERLEAVE">PIXEL</MDI>
  </Metadata>
  <VRTRasterBand dataType="Byte" band="1" subClass="VRTPansharpenedRasterBand">
    <ColorInterp>Red</ColorInterp>
  </VRTRasterBand>
  <VRTRasterBand dataType="Byte" band="2" subClass="VRTPansharpenedRasterBand">
    <ColorInterp>Green</ColorInterp>
  </VRTRasterBand>
  <VRTRasterBand dataType="Byte" band="3" subClass="VRTPansharpenedRasterBand">
    <ColorInterp>Blue</ColorInterp>
  </VRTRasterBand>
  <BlockXSize>512</BlockXSize>
  <BlockYSize>512</BlockYSize>
  <PansharpeningOptions>
    <Algorithm>WeightedBrovey</Algorithm>
    <AlgorithmOptions>
      <Weights>0.3333333333333333,0.3333333333333333,0.3333333333333333</Weights>
    </AlgorithmOptions>
    <Resampling>Cubic</Resampling>
    <SpatialExtentAdjustment>Union</SpatialExtentAdjustment>
    <PanchroBand>
      <SourceFilename relativeToVRT="1">pan_mask_opencv.tif</SourceFilename>
      <SourceBand>1</SourceBand>
    </PanchroBand>
    <SpectralBand dstBand="1">
      <SourceFilename relativeToVRT="1">origin_mask_opencv.tif</SourceFilename>
      <SourceBand>1</SourceBand>
    </SpectralBand>
    <SpectralBand dstBand="2">
      <SourceFilename relativeToVRT="1">origin_mask_opencv.tif</SourceFilename>
      <SourceBand>2</SourceBand>
    </SpectralBand>
    <SpectralBand dstBand="3">
      <SourceFilename relativeToVRT="1">origin_mask_opencv.tif</SourceFilename>
      <SourceBand>3</SourceBand>
    </SpectralBand>
  </PansharpeningOptions>
</VRTDataset>

```

这里我们定义了一个Pansharpened文件，也就是一个影像融合之后的影像，但是实际上在硬盘上的只是一个这样的描述文件，我如果想把他变成真正的影像文件，我可以这样做

```python
from osgeo import gdal

ds = gdal.Open("pansharpened.vrt")
drv = gdal.GetDriverByName("GTiff")
drv.CreateCopy("pansharpened.tif", ds)
```

这样我就将一个很小的vrt文件变成了tif文件。



聪明的你肯定看出来了，这个vrt文件在实际编程的时候和这个tif文件没啥区别，但是不一样的是，vrt文件只占用这么小的空间，这是很爽的事情。



那么我们可以更进一步，这个vrt文件我都不想看到，怎么办？也可以，你只需要这样做

```python
# doing something

# now we want to create a new DataSet
drv = gdal.GetDriverByName("VRT")
ds = drv.Create("", xoff, yoff, bds_num, eType. options=[])
```

是的，如果是vrt文件，大可以在filename这个参数上填空串，这样就不会出现这个文件了。至于这个文件是放在了内存中还是再怎么样，我也就不关心了（GDAL也有内存文件，可以参考[这个](https://gdal.org/user/virtual_file_systems.html)）

## 一些命令

GDAL除了提供库以外，还提供了一些命令行程序，可以对影像进行一些简单的操作，具体的内容就不细说了，看[这里](https://gdal.org/programs/index.html)

B站上还有个视频教程，不过是全英文无字幕的，看着操作还是能明白是什么意思，在这里

[part1](https://www.bilibili.com/video/BV15K411g7Sy)

[part2](https://www.bilibili.com/video/BV1ST4y1K7Sy)


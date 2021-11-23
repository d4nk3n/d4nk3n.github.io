---
title: "使用SDR模拟GPS信号"
categories:
  - "Life"
tags:
  - "GPS"
  - "SDR"
---

如果你在开发工作中不满足于Android的模拟位置，而你正好有一个SDR开发板的话，可以试试使用SDR模拟GPS信号。

美国的GPS民用卫星信号是没有加密的，而且卫星的星历很好获取，信号频率为1575.42 MHz，大部分SDR开发板都支持这个频率。

## 1. 需要准备的硬件和软件

### 1.1 硬件

硬件上需要一个SDR开发板（比如Hackrf One，其他更高级的也可以）。另外像Hackrf One这样的设备，它自带的晶振的误差太大，不能模拟GPS信号，需要买一个外部晶振（最好是0.1ppm的）。

晶振的安装方法请见：<https://github.com/osqzss/gps-sdr-sim/blob/master/extclk/hackrf_gpssim.jpg>

### 1.2 软件

对于Hackrf One，需要一个生成卫星信号的`gps-sdr-sim`和用于发射信号的`hackrf_transfer`，Linux上用gcc和apt就搞定了，下面讲讲Windows上怎么办：

* `gps-sdr-sim`可以从GitHub上获取：<https://github.com/osqzss/gps-sdr-sim>，按README上说明用Visual Studio编译；
* [Hackrf](https://github.com/greatscottgadgets/hackrf)没有提供Windows上的`hackrf_transfer`的二进制版本，不过[PothosSDR](https://downloads.myriadrf.org/builds/PothosSDR/)自带了Hackrf的二进制程序，可以安装它。之后可以在程序目录下找到`hackrf_transfer.exe`；

## 2. 操作步骤

首先要知道你要模拟的位置的经纬度。以四川外国语大学的西区运动场为例子，在Google地图上拾取坐标：

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/google-map-satellite.png){: width="400"}

坐标位置是`29.572615 N, 106.430159 E`，不清楚当地海拔多少，先设置为200m吧。

我们还需要GPS的星历表，它包含了GPS卫星的位置、速度、时钟等信息，可以从NASA的网站上下载（需要注册账号），下面链接是2023年的卫星导航文件，里面是按一年的第几天分类的：
<https://cddis.nasa.gov/archive/gnss/data/daily/2023/>。

里面有很多文件，我们可以看一下<https://cddis.nasa.gov/archive/gnss/00readme>里的说明。

可以知道其中`DDD/YYd/YYn/brdcDDD0.YYn.gz`也就是我们需要的YYYY年DDD日的GPS星历文件，比如<https://cddis.nasa.gov/archive/gnss/data/daily2023/070/23n/brdc0700.23n.gz>：

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/nasa-gps-nav-data.png){: width="500"}

接着使用`gps-sdr-sim`可以生成我们需要模拟的GPS信号：

`gps-sdr-sim.exe -e brdc0700.23n -l 29.572615,106.430159,200 -b 8`

其中参数
* `-l 29.572615,106.430159,200`自然就是经纬度和高度；
* `-b 8`是I/Q数据的位数，对于HackRF One来说是8，这是它的DAC的位数；
* `-e brdc0700.23n`自然就是使用的星历（注意我们下载的是压缩包，需要解压）；

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/gps-sdr-sim.png){: width="500"}

其他的参数（比如模拟的时间长度等等）可以自行查阅程序的帮助信息。

生成好信号之后我们调用`hackrf_transfer`发送它：

`hackrf_transfer.exe -t gpssim.bin -f 1575420000 -s 2600000 -a 1 -x 47 -R`

其中参数
* `-t gpssim.bin`是`gps-sdr-sim`生成的GPS信号文件；
* `-f 1575420000`是GPS卫星的频率1575.42 MHz；
* `-s 2600000`是采样率2.6MHz；
* `-a 1`是TX RF放大器，设置为1开启；
* `-x 47`是TX VGA (IF)增益，设置为47分贝；
* `-R`是反复发送；

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/hackrf-transfer.png){: width="800"}

这时候Hackrf One就开始发射信号了，将你的手机放到它旁边，打开GPS定位软件（这里用的是GPS Test和高德地图），如果你手机好几分钟都没有接收到GPS信号的话，可以重启试试。

显示的GPS信号的信噪比：可以看到这时已经成功定位了：

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/gps-test-signal.jpg){: width="250"}

GPS Test显示的定位的位置的经纬度，和我们需要模拟的位置很接近：

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/gps-test-location.jpg){: width="250"}

最后再看看高德地图里显示的定位：

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/amp-app.jpg){: width="250"}

## 3. 坐标系转换

需要注意的是，我们前面提到的经纬度坐标是WGS84坐标。之前我们是从Google地图的卫星地图获取的坐标数据，Google的卫星地图使用的是WGS84坐标，没有问题。

但是下面情况使用的坐标系不一样：
* Google地图在中国内地（也就是不含港澳台）的路网坐标（使用GCJ-02坐标系）；
* 百度地图的坐标（使用BD-09坐标系）；
* 高德地图的坐标（使用GCJ-02坐标系）；

比如深圳和香港边界，你可以看到深圳这边的路网和卫星地图不重合，而香港这边的没有问题。

![](/uploads/img/posts/2023-03-13-gps-sdr-sim/shenzhen-hongkong.png){: width="600"}

使用不同的坐标系是出于国防安全需要（作用微乎其微）和商业数据保护（喜欢整护城河）。

如果要进行坐标转换的话，GitHub上也有开源实现，也可以使用这个网页<https://atool.vip/lnglat/>，注意遵守法律法规。

## 4. 结语与参考链接

除了模拟静止位置，实际上我们还可以模拟运动时的GPS信号，需要的话可以查询gps-sdr-sim项目的README。

如果想了解这几个坐标系的关系，可以看看最后一个参考链接里的文章。

* [gps-sdr-sim的GitHub仓库](https://github.com/osqzss/gps-sdr-sim)
* [hackrf的GitHub仓库](https://github.com/greatscottgadgets/hackrf)
* [高德地图坐标拾取（GCJ-02）](https://lbs.amap.com/tools/picker)
* [A Short Guide To The Chinese Coordinate System](https://abstractkitchen.com/blog/a-short-guide-to-chinese-coordinate-system/)

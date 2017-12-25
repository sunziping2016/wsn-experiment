<style>
@media print {    
    [data-cmd=toc] + ul {
        display: none !important;
    }
}
</style>

<h1 style="text-align:center;">WSN实验报告</h1>
<p style="text-align:center;">孙子平 2015013249 车行 2015013241 李在弦 2015080121</p>


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

* [1 题目一：多跳WSN数据采集](#1-题目一多跳wsn数据采集)
	* [1.1 关于](#11-关于)
		* [1.1.1 项目特色](#111-项目特色)
		* [1.1.1 项目结构](#111-项目结构)
		* [1.1.2 使用方式](#112-使用方式)
	* [1.2 实现](#12-实现)
		* [1.2.1 节点实现](#121-节点实现)
		* [1.2.2 可视化实现](#122-可视化实现)
* [2 题目二：多点协作Data Aggregation实验](#2-题目二多点协作data-aggregation实验)
	* [2.1 关于](#21-关于)
		* [2.1.1 项目特色](#211-项目特色)
		* [2.1.2 项目结构](#212-项目结构)
		* [2.1.3 使用方式](#213-使用方式)
	* [2.2 实现](#22-实现)
* [3 项目分工](#3-项目分工)

<!-- /code_chunk_output -->

## 1 题目一：多跳WSN数据采集

### 1.1 关于

#### 1.1.1 项目特色

* 自由组网：可自由组网成树状的拓扑结构，支持任意数目节点组网，可视化支持任意数目节点
* 可靠传输：几乎0丢包率，即使节点掉线一阵子，重连之后，依旧不会丢包
* 实时交互的可视化界面：基于Web的可视化，实时统计丢包数目、重包数目等，统计图表可交互显示

#### 1.1.1 项目结构

项目的根目录位于`task1`下，包括以下文件、文件夹：

* `Node/`：Telosb节点的代码，我们所有的节点都采用同一份代码
  * `ForwarderC.nc`：泛型传输模块，负责在父子节点之间建立可靠传输
  * `SensorC.nc`：传感器模块，负责收集数据
  * `AppC.nc`：顶层模块
* `Visualizer/`：数据可视化的Web服务端
  * `main.py`：服务端代码
* `nodes.csv`：配置文件，用以为不同的节点产生不同的编译选项

#### 1.1.2 使用方式

首先编辑`nodes.csv`，格式如下方的表格。如果节点是`Sensor`，则会收集传感器数据；如果节点是`Base Station`，则其上游通信不是采用无线而是采用串口。

| ID | Father ID | Sensor | Base Station |
|:---:|:----:|:----:|:---:|
| 0 | 0 | 0 | 1 |
| 1 | 0 | 1 | 0 |
| 2 | 1 | 1 | 0 |

之后可以烧录程序，运行可视化界面：

```bash
cd task1
# 编译节点代码，<node_id>为节点的ID
make compile,<node_id>
# 烧录节点程序，<node_id>为节点的ID
make install,<node_id>
# 不停烧录节点程序直至成功，<node_id>为节点的ID
./FuckMake install,<node_id>
# 安装可视化的依赖
pip install -r Visualizer/requirements.txt
# 启用可视化，后面可跟串口参数，默认是serial@/dev/ttyUSB0:115200
python Visualizer/main.py
```

这时，可以浏览器打开`http://localhost:8050/`看到可视化的效果。这里3个节点的Sensor都开启了，所有的湿度传感器都有问题，而1号节点的光照传感器也有问题。

“刷新间隔”是指轮询服务器的间隔（实时改变），“显示时长”是下方各个表显示的x轴的范围（实时改变），“采样间隔”是每个节点的采样计时器触发的时间间隔（需要点击发送按钮改变）。

“手动刷新”用于触发一次轮询。“同步时间”会将下一匹收到的包作为新的基准时间，推测新的包的真实发送时间，应当在网络条件较好的情况下再点击。“清除数据”会清空显示的数据。

所有的图都可以缩放，查看细节数据。

`result.txt`是收到的最原始数据，注意我们始终用append模式打开这个文件。重复的包也会被记录下来。

![Visualizer](https://cdn.pbrd.co/images/GZJr77e.png)		* [1.2.2 可视化实现](#122-可视化实现)


### 1.2 实现

#### 1.2.1 节点实现

**可靠传输：** 首先，我们发送的数据包分为两类，往根节点传的数据消息和往叶节点传的控制消息。前者默认拥有512的循环队列，而后者默认拥有32的缓冲区。而后我们借助Tinyos自带的PackageAcknowledges借口确保1个数据包被成功发送，如果没成功发送则重试。整个发送等价滑窗为1的GBN。

**组网：** 组网我们采用静态的路由表，而后以编译参数的方式编译到程序中。数据发送我们都是指定接受者，而非广播。对于多个子节点的情况，我们是挨个发个去并确认收到的。

**数据采集：** 数据采集我们有两种方案，一种是等待全部采集完毕再采集下一匹（默认），另一种是有个采集队列，不必等待采集完成就发起下一次采集（解除`task1/Node/Makefile`中`RECORD_QUEUE`选项的注释即可启用）。实际证明，后者在采集速度上几乎等价于前者，呈现出来的时间则会有偏差，故不采用。实际节点采样频率在280~320ms左右。

#### 1.2.2 可视化实现

我们采用的数据可视化框架是[Dash by plotly](https://plot.ly/products/dash/)。数据采集则用的是TinyOS自带的Python SDK。具体实现上，我们启动了额外的线程专门用于串口通信。

**包数目的统计：** 对于每个收到的包，我们会查看是否已经收到过这个节点的包。如果还未收到，我们则初始化这个节点的相关数据，同步时间，并开始计算“真实采样间隔”。如果已经收到，我们会检查包的序号。如果序号超前不多（16以内），我们会计算“包丢失数”；如果序号落后的不多（16以内），则会计入“包重复数”；如果序号变化实在是太大，则我们会计入“重置数”，并重新同步时间，重新计算“真实采样间隔”。

**时间同步：** 由于收到的包的时间戳，是相对于发送节点的开启时刻。而我们没有复杂的同步协议。所以时间同步就以收到的该节点的第一个包的时间为准。点击“同步时间”和“清除数据”，都会强制以下一个包收到的时间作为基准。

**真实采样频率：** 采样频率的估计是用指数移动平均，具体公式如下，其中，$\hat{t}_k$是收到第$k$个包时对真实采样间隔的估计，$t_k$是收到第$k$个包时计算得到的此刻采样间隔，$r$是系数，这里取0.9。

$$\hat{t}_k = r \cdot t_k + (1 - r) \cdot \hat{t}_{k-1}$$

## 2 题目二：多点协作Data Aggregation实验

### 2.1 关于

#### 2.1.1 项目特色

* 自由组合：支持任意数目的节点，以任意方式分配任务
* 多种策略：可以选择是边收包边询问peers未收到的包还是等收完后再询问，可以选择中位数的算法

#### 2.1.2 项目结构

项目的根目录位于`task2`下，包括以下文件、文件夹：

* `Master/`：模拟的助教节点，负责发送数据，接受答案并返回ACK
* `Slave/`：本项目真正的代码所在，负责协作计算
  * `CalculateC.nc`：负责计算数据的模块
  * `TransportC.nc`：负责接受数据、协商丢失的数据、汇总计算结果的模块
  * `SlaveAppC.nc`：顶层模块
  * `Makefile`：包含了若干的编译选项，用以改变策略
* `nodes.csv`：配置文件，用以为不同的节点产生不同的编译选项

#### 2.1.3 使用方式

### 2.2 实现

## 3 项目分工

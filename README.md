


# 2019太原理工大学TRoMaC战队英雄自动瞄准开源
---
TRoMaC战队的开源程序借鉴了官方ICRA开源程序与大疆工程师陈建先生在卡尔曼滤波做预判上的思路，在下面给出链接：

官方ICRA开源程序：https://github.com/RoboMaster/RoboRTS
卡尔曼滤波在云台控制上的思路：https://zhuanlan.zhihu.com/p/38745950

联系方式：余树源
qq: 690208412
欢迎大家交流

---
## 目录

* [1.运行环境](#1运行环境)

* [2.方案介绍](#2方案介绍)

* [3.装甲检测](#3装甲检测)

* [4.时间同步](#4时间同步)

* [5.预测](#5预测)

* [6.通信协议](#6通信协议)

* [7.代码整体框架](#7代码整体框架)

* [8.使用说明](#8使用说明)

* [9.总结与改进方向](#9总结与改进方向)

---

## 1.运行环境
### 硬件环境
* NUC8I5BEK miniPC
* 国产度申科技U3VT132-H （648 x 480下最高442FPS，价格1880￥）
* usb转ttl串口通信模块（测试中CH341比CH340要稳定，感谢大佬给我焊的341）
### 软件环境
* 系统：ubuntu 16.04.5
* IDE：kDevelop
* 编译方式：cmake3 GCC5
* 依赖库：Eigen3 OpenCV3.4.3

---

## 2.方案介绍
###发射控制方案
国赛TRoMaC英雄为大小发射共用一个云台，单个工业摄像头放置于小枪管侧上方，随云台移动。采用三种模式控制方法，第一种是优先大枪管瞄准，大子弹剩余热量小于100后小强管瞄准，第二种第三种为只控制一个枪管瞄准，打击模式的切换由操作手控制。
![枪管](https://github.com/)

###控制逻辑
控制逻辑是纯算法控制，电控负责发送云台yaw的imu角度值、pitch编码值、以及热量、系统时间、小子弹预期射速给算法，miniPC将解算检测到的装甲板偏离摄像头中心角度与云台此刻角度向加转化为云台坐标，经过预判将yaw和pitch的期望和时间发给电控，电控置期望与发射。逻辑如下图

![控制逻辑](https://github.com/)

## 3.装甲检测
装甲检测借鉴的官方ICRA开源程序思路，利用亮度和颜色外加数字检测以提高鲁棒性，对于一幅1284x1024的完整图片，能够在6ms以内完成识别，对于640x480的图片可以在2ms以内完成识别。因为大家的装甲识别程序都相对成熟了，所以在这里不在多去叙述装甲识别的逻辑了。

![装甲识别效果图1](https://github.com/)
![装甲识别效果图2](https://github.com/)

## 4.时间同步
###介绍
在进行检测的过程中，由于摄像头取图并传送至电脑存在延迟，并且由于fincontours的存在，识别程序的识别时间并不稳定，所以在做自瞄的时候比较倾向于将装甲在摄像头的角度返回给电控，在电控端融合成绝对坐标并进行预判，以减少通信过程中的误差。在去年下半年学习vslam的过程中，我看到一张VIO（视觉与惯性测量单位融合的里程计）的逻辑图，他将图像和imu进行融合，并在同一个时间上进行计算，虽然最后了解到这不是它的原理，但还是给了我很大的启发，如果我们能让电控和算法公用一个时间轴，然后取得图像的时间和云台状态的时间，那么可以将他们融合在一起，降低融合坐标的误差。
###程序时间过程分析
为了减少电控和算法之间的误差，分析程序运行过程各个时间的过程。

    1.电控读取系统时间，并紧接着读取can信号返回的imu角度值和mpu编码值，并交给DMA，此过程时间可以忽略不计。
    2.DMA将串口数据发给usb转ttl模块，USB转ttl模块在缓存区未满的情况下会直接把串口数据转发给miniPC，这个过程所用时间可以大体上看做是串口通信的时间，消耗时间=发送比特数/波特率
    3.摄像头取得图像时间是一个不确定的，不同的摄像头性质不同，以我们队所用的摄像头为例，其取图分为三个步骤，第一步触发取图，整个过程大约为十几微妙，可忽略不计，第二步进行曝光，并将数字图像保存在缓存区，这一步所用时间由设置决定，我们队设置的2ms，第三步将图像从缓存区经usb3.0传送移至miniPC，这一步所用时间比较难计算，因为不清楚采用的图像压缩方式和加速方式。但是正好我们买的摄像头包含触发功能，利用获取图像的时间减去触发时间，我们的摄像头大概在曝光设置为2ms是取图为4-6ms（804x600像素下）。
    4.检测预测完后，串口发送数据到usb转ttl在转发给电控的过程，此过程同2，时间计算方法一致。
###时间同步方案
根据以上的时间过程分析，提出以下方案：

    1.miniPC第一次接受到电控数据，将电控传来系统时间戳当成开始时间，并减去串口通信的时间，就可以得到一个接近电控控的时间开始点。
    2.由于晶振的不同，电控与算法之间的时间轴会产生偏差，每次读取的时候检测偏差量，当连续大于一定值（我取的6ms）3次，就进行同步时间。
    3.摄像头读取图像之后，减去取图时间到曝光时间的值，不同摄像头取图计算方法不同。
    4.发送给主控后的时间加上串口延时，保证时间一致。
###时间同步的验证
我们队验证时间同步的方法如下：

    1.电控和算法同时各自打印接收到的时间，我们在测试的时候算法和电控都能在一段时间内保证到2ms之内的误差。
    2.将一个已经标定好并去除畸变的摄像头（获得像素点偏离摄像头光轴角度计算准确）固定在imu上，并去检测装甲板，进行融合转化为绝对坐标，快速以摄像头镜头末端为圆心左右转动摄像头，如果打印出来装甲板的世界imu坐标保持不变，或变化为最小，则说明摄像头时间获得准确。
###减少同步时间误差的方法
时间同步误差来源主要为以下几种，并提出一些解决办法的建议:

    1.串口通信的速度跟不上电控发送速度，导致DMA发送不完，会造成一定的丢帧和时间误差，并且USB转ttl的缓存区也会积累也会产生误差，建议可以降低电控发送频率，保证每次都完全发送后在进行下次发送。
    2.由于大多数工业摄像头驱动会单拉一个线程出去取图，所以会造成在经过识别和预判的循环后已经有张图在缓存区内，正在取下一张图的情况，这时候仍可以直接读图，但是会造成直接用减法时间不准确的情况，建议可以遇到能直接读图的情况时不用此张图而是选择下一张图进行时间戳同步，也可以加快自己的识别与预判速度，保证每次识别预判串口发送时间合起来小于取图时间，这样取得图也是可以使用的（比如120FPS摄像头取图8.6ms以上，只要保证识别预判与通信小于8.6ms就能较为准确计算出取图时间）
    3.若无法知道摄像头取图的特性，也可以之间用时间同步验证中2的方法反推尝试出取图时间。
## 5.预测
   预测采用kalman滤波的状态向量进行预测，利用Eigen库编写的kalman滤波对绝对的pitch和yaw值进行预测更新，这点同大疆工程师介绍的方案是一样的。
   预测时利用状态方程的位置和速度结合地方距离进行预测。
   需要注意的是对输出进行限幅。
## 6.通信协议
通信写在uart文件夹下，
算法发送给电控数据如下，共16字节:

    【0】：帧头0x3d
    【1】：帧头0xd3
    【2】 : 0：未检测到 1-5弃用 6：检测到
    【3】：大发射发射 0：不发射 1：发射
    【4】：小发射发射 同上
    【5-8】：联合体 imu角度值期望
    【9.10】：联合体 pitch编码值期望
    【11-14】：联合体 算法返回的时间戳
    【15】：异或校验位
电控发送给算法数据如下，共23字节:

    【0】：帧头0x2d
    【1】：帧头0xd2
    【2】 : 模式 1：自动选择瞄准 17：17mm瞄准 42:42mm瞄准
    【3-6】：联合体 电控读回的imu角度值
    【7.8】：联合体 电控读回的pitch编码值
    【9-12】：联合体 电控返回的时间戳
    【13.14】：联合体 大发射热量
    【15-16】：已弃用
    【17】：小发射预期速度
    【18-21】：已弃用
    【22】：异或校验位
##7.代码整体框架
代码文件树如下：
### 文件树

```
hero/
├── main 
│   ├── ThreadControl.h 
│   ├── ThreadControl.cpp     //线程管理程序
│   └── main.cpp
├── camera 
│   ├── camera.h 
│   ├── camera.cpp            //摄像头驱动与图像去畸变
│   └── intrinsics.yml
├── decision
│   ├── Decision.h            
│   └── Decision.cpp          //发射的决策
├── fusion
│   ├── CoordinatesFusion.h
│   └── CoordinatesFusion.cpp //将相对坐标转为绝对坐标
├── detector
│   ├── ArmorDetector.h
│   ├── ArmorDetector.cpp     //装甲识别
│   ├── big_armor.xml
│   └── classification.xml
├── predictor
│   ├── KalmanFilter.h
│   ├── KalmanFilter.cpp      //卡尔曼滤波函数
│   ├── Predictor.h
│   └── Predictor.cpp         //预测函数
├── tools
│   ├── MathTools.h
│   └── MathTools.cpp         //几个为了方便的数学工具
└── uart
    ├── serial.h
    └── serial.cpp            //串口通信
```
### 重点文件与类说明
|文件名|包含类|作用|
|:--|:--|:--|
|camera.cpp|MyCamera|摄像头驱动与像素转角度|
|ArmorDetectpr.cpp|Lamp，Armor，ArmorDetector|Lamp为灯条的类，Armor为装甲的类，均为简化逻辑，Armordetector为检测器|
|ThreadControl.cpp|无|用来控制每个线程完成的任务|
##8.使用说明
### 运行前改动

1.打开ThreadControl.cpp,有三个文件的路径需更改，分别代表摄像头标定参数、小装甲SVM分类器、大装甲SVM分类器路径，可重新制作分类器，分类器大小能在ArmorDetector.cpp的构造函数中找到，若不想使用数字，将ArmorDetector.h中的#define USE_ID注释掉即可。

2.camera中未使用工业摄像头的SDK，而是使用VideoCapture，若想更改为工业摄像头，即对camera类中的open函数和getFream函数进行修改并添加定义即可。

3.使用前需要更改decision.cpp中的tvec17或者tvec42，它表示枪管与摄像头之间的平移向量，以达到枪管瞄准而不是摄像偶瞄准的目的。

4.对枪管与弹道的偏差调整可以使用camera.cpp内的pix2angle函数中的补偿部分对枪管角度进行补偿。

4.我们是在IDE kDevelop下利用cmake进行编译的，也可直接进入build文件夹下进行编译，cmakeLists.txt在文件夹内
##9.总结与改进方向
从第一版程序到现在代码一直在缩短，去掉了好多繁琐的步骤，但现在看起来还是不太简洁。
利用算法在同步的时间轴上控制的绝对坐标自动瞄准算是今年的一个小点子，在分区赛中取得了一定的成效，分区赛后在哨兵上尝试在电控上绝对坐标转换和kalman预判，也取得了不错的结果，不过最终还是选择了调习惯了的以前的做法。但是在比赛的过程中也发现了一些问题：

1.双向通信错误率还是比单向高，由于usb2.0为半双工串口，在通信频率过快的时候难免会挤掉一些数据导致丢帧。

2.由于整套程序的编写迭代只有我一个人，而且还有哨兵和步兵的自动瞄准需要考虑，所以程序的测试时间还不足够，在场上的表现为一场能用一场不能用。

3.代码调参不方便，由于没写上位和调参文件，程序参数又比较多，调参会显得很麻烦。

4.在算法上控制没有在主控上控制程序那么简洁，毕竟多了一个通信的步骤，所以程序的稳定性更需要功夫去调整。

5.吐槽，当摄像头够快的时候，检测程序够快，传回来的云台值甚至可以直接和检测解算出的角度进行融合，误差也能接受...甚至不用拉两个线程...

总之，在算法上做时间同步和预判一定是需要更多的时间检查BUG和调整程序，使程序能够更好debug和调参时目前攻克的目标。
当然，在算法上做的也是有一定优点的，可以编写更复杂的预判程序，在对时间同步后，小陀螺的解算可能会误差更小。
这次开源是给大家一个新的思路，也希望这个思路能对大家有些启发，共同学习。

最近在家里没办法给大家拍一些效果图了，开学了我尽量补上。
最后祝明年学弟学妹们改掉今年的坏毛病，让TRoMaC明年翻身，也祝大家能在明年收货更多，RM越办越好。


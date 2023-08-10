# TF-A First Impression

#### 系统源码获取
路径：1、程序源码 --> ST官方原版LinuxFirst源码 --> en.SOURCES-stm32mp1-openstlinux-5-4-dunfell-mp1-20-06-24.tar.xz 


直接在/home/yz/linux下mkdir，创建atk-mp1/

将压缩包拷贝进去，解压得同名文件夹

文件夹第二层目录内ls，有uboot源码、linux源码和tf-a源码

进入tf-a-stm32mp-2.2.r1-r0/，得同名压缩包，解压。


>**ARM官网原版** [documentation](https://trustedfirmware-a.readthedocs.io/en/latest/index.html) 

>**就。。。不如跳到p10直接上手操作吧**

------


## TF-A系统移植

### 为何要移植

tf-a是ARM官方出品的软件包，半导体厂商（比如ST）得到这个软件包之后，**需要把自己家的SOC芯片添加进去**，然后打包好做成SDK给客户用。

### 移植步骤

#### （乱入的体验活动，无实际意义）打补丁，添加MP1系列
在资料光盘中，ST官方源码压缩包的路径是 
>1、程序源码→5、ST 官方原版 Linux 源码→en.SOURCES-stm32mp1-openstlinux-5-4-dunfell-mp1-20-06-24.tar.xz

解压后，得到`stm32mp1-openstlinux-5.4-dunfell-mp1-20-06-24`文件夹，内部第二层有


|file|usage|
|:---|:---|
|**u-boot-stm32mp-2020.01-r0 uboot** | **uboot源码, 版本号为 2020.01**|
|**linux-stm32mp-5.4.31-r0 linux** | **linux源码, 版本号为 5.4.31**|
|**tf-a-stm32mp-2.2.r1-r0 tf-a** | **tf-a源码, 版本号为 2.2**|
|tf-a-stm32mp-ssp-2.2.r1-r0 tf-a | tf-a源码, ssp=secure secret provisioning|
|optee-os-stm32mp-3.9.0.r1-r0 optee |optee系统源码, 版本为 3.9.0|

正点原子教程只用到前三个，本次tf-a移植只用到第三个 `tf-a-stm32mp-2.2.r1-r0 tf-a`。 这是ST的官方包，支持所有的MP1系列芯片。同时，ST官方也有自己的开发板，正点原子就参考了STM32MP157C-EV1开发板。

在`tf-a-stm32mp-2.2.r1-r0 tf-a`文件夹内，有：

|file/dir|usage|
|:---|:---|
|**0001-st-update-v2.2-r2.0.0.patch**|**TF-a 补丁文件**|
|Makefile.sdk|编译 TF-A 用的 MAKEFILE文件|
|README.HOW_TO.txt| ST 官方 TF-A 的编译文档|
|series| 存放补丁名字的文件|
|**tf-a-stm32mp-2.2.r1-r0.tar.gz** |**TF-A 的ARM官方源码压缩包**|

先解压tf-a的源码包，然后为它打补丁，命令为
```
for p in `ls -1 ../*.patch`; do patch -p1 < $p; done
```
执行好以后，ARM官方的tf-a代码就添加了STM32MP1系列的补丁，可以对本系列SOC芯片进行authentication

-----

### A BIG BUT！！！
**正点原子的开发板虽然是参考ST的官方开发板制作的，但是由于有很大改动，所以上一部分中打好补丁的ST官方tf-a文件，是无法在正点开发板上运行的！！！**

**所以，我们从下一部分开始，直接使用正点原子修改好的tf-a文件**


------

### 移植和烧录正点原子的tf-a

#### 准备工作

##### 安装stm32wrapper4dbg工具

* 从`5、开发工具→ stm32wrapper4dbg-master.zip`中复制出来这个压缩包，解压缩到单独的文件夹中
* 直接`make`，得到`stm32wrapper4dbg`工具
* 将工具拷贝到/usr/bin：`sudo cp stm32wrapper4dbg /usr/bin`
* 拷贝好了以后在终端中用`stm32wrapper4dbg -s`查看是否安装成功

##### 安装ARM device-tree 编译器

* 直接在终端中使用`sudo apt-get install device-tree-compiler`命令安装

#### 开始
* 从`1、程序源码→ 1、正点原子 Linux 出厂系统源码→ tf-a-stm32mp-2.2.r1-g463d4d8-v1.0.tar.bz2`中拷贝出来正点原子的tf-a压缩包，解压后得到：
|file/dir|usage|
|:---|:---|
|Makefile.sdk| 编译使用的makefile文件 |
|tf-a-stm32mp-2.2.r1| 类似于上节中ST官方的文件夹 |

#### 修改Makefile.sdk

* 很久很久以前，我们在ubuntu系统中安装了交叉编译器`arm-none-linux-gnueabihf-gcc`，然而`Makefile.sdk`中默认使用的是ST官方的交叉编译器`arm-ostl-linux-gnueabihf-gcc`，所以要修改一下

* 在文件夹位置打开终端，`gedit Makefile.sdk`，然后`Ctrl+F`搜索`CROSS_COMPILE=`，将编译器名称进行修改

#### 编译tf-a

* 先进入tf-a文件夹：`cd tf-a-stm32mp-2.2.r1/`，然后`make -f ../Makefile.sdk all`，如果编译速度较慢，可以使用`-jxx`命令使用多核编译。

* 编译完成后会在上层目录中生成一个`build`文件夹，内有`optee`、`serialboot`和`trusted`三个文件夹，打开`trusted`，内有MP1所有型号的tf-a固件，包括正点原子开发板使用的`tf-a-stm32mp157d-atk-trusted.stm32`文件

#### 将tf-a烧录到开发板EMMC

使用`STM32CubeProgrammer`，通过`USB_OTG`接口进行USB烧写：

* 新建`images`文件夹

* 拷贝文件1：`tf-a-stm32mp157d-atk-serialboot.stm32`，路径：`8、系统
镜 像 →2 、 出 厂 系 统 镜 像→ 1 、 STM32CubeProg 烧 录 固 件 包 →tf-a→ tf-a-stm32mp157d-atk-
serialboot.stm32`

* 拷贝文件2：`u-boot.stm32`，路径：`8、系统镜像→2、出厂系统镜像
→ 1、STM32CubeProg 烧录固件包→uboot→u-boot.stm32`

* 拷贝文件3：刚才编译好的`tf-a-stm32mp157d-atk-trusted.stm32`

> 这三个文件中，`serialboot`和启动有关，用来初始化USB和DDR等外设。DDR初始化完毕后即可运行`uboot`。`uboot`负责初始化EMMC、NAND等外设，同时`uboot`也能提供EMMC的操作指令，通过相关的指令将`tf-a-stm32mp157d-atk-trusted.stm32`写入EMMC或者NAND中。

#### 准备FlashLayout

* `FlashLayout`是`.tsv`格式的文件，用于引导上节中三个文件的烧写（比如烧写地址是EMMC的哪个区域）

* 正点原子提供了FlashLayout文件，路径为：`8、系统镜像→2、出厂系统镜
像→1、STM32CubeProg 烧录固件包→flashlayout→atk_emmc-stm32mp157d-atk-qt.tsv` 

* 将其拷贝到 images文件夹中，重命名为 `tf-a.tsv`。
* 使用Notepad++修改，只保留`fsbl1-boot`、`ssbl-boot`、`fsbl1`和`fsbl2`，其余删除，同时修改好`Binary`下的路径地址即可

#### 烧写
* 将开发板拨码开关设置为`000`，从USB启动，然后复位开发板
* 将`USB_OTG`和`USB_TTL`连接到电脑，在windows系统下打开MobaXterm查看串口信息
* 打开STM32CubeProgrammer，选择USB连接，在`Open file`中选择`images`目录下的`tf-a.tsv`
* 设置好Binaries Path中的路径地址，点击右上角Download

#### TF-A的运行
* 将拨码开关设置为`010`，从EMMC启动，然后复位开发板
* 从MobaXterm中查看启动log，从INFO栏中的编译时间可知运行的是自己编译的TF-A文件
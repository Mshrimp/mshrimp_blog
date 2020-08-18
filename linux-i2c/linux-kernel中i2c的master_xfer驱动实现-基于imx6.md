## linux-kernel中i2c的master_xfer驱动实现----基于imx6

### 目录

[TOC]

在I2C驱动中，每个适配器i2c_adapter都有自己的I2C通信方法，在struct i2c_algorithm结构中的master_xfer()函数中实现；i2c_algorithm结构中的关键函数master_xfer()用于产生I2C访问周期需要的信号，以struct i2c_msg结构的格式进行数据传送；

未完成



<!--more-->



**注：**本文含有一些从《IMX6SDLRM.pdf》手册中获取的一些数据信息截图，如有侵权，纯属无意，还请告知，立即删除！



### 简介



http://read.pudn.com/downloads664/sourcecode/embedded/2692975/IMX6SDLRM.pdf











#### 寄存器描述

![I2C memory map]([https://github.com/Mshrimp/mshrimp_blog/blob/master/linux-i2c/Linux-kernel%E4%B8%ADI2C%E7%9A%84master_xfer%E9%A9%B1%E5%8A%A8%E5%AE%9E%E7%8E%B0-%E5%9F%BA%E4%BA%8Eimx6/I2C-memory-map.png](https://github.com/Mshrimp/mshrimp_blog/blob/master/linux-i2c/linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2C-memory-map.png))





##### I2C Address Register

![I2C-Address-Register2](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2C-Address-Register2.png)



##### I2C Frequency Divider Register

![I2C Frequency Divider Register](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2C-Frequency-Divider-Register.png)



##### I2C Control Register

![I2C Control Register](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2C-Control-Register.png)

![I2Cx_I2CR field descriptions](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2Cx_I2CR-field-descriptions.png)

![I2Cx_I2CR field descriptions2](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2Cx_I2CR-field-descriptions2.png)



##### I2C Status Register

![I2C Status Register](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2C-Status-Register.png)

![I2Cx_I2SR field descriptions](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2Cx_I2SR-field-descriptions.png)

![I2Cx_I2SR field descriptions2](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2Cx_I2SR-field-descriptions2.png)

![I2Cx_I2SR field descriptions3](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2Cx_I2SR-field-descriptions3.png)





##### I2C Data I/O Register

![I2C Data IO Register](linux-kernel中i2c的master_xfer驱动实现-基于imx6/I2C-Data-IO-Register.png)




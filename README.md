# FPGA-Project-Identifying-Numbers-using-Vivado
FPGA-Project-Identifying-Numbers-using-Vivado  
2020东南大学-XILINX暑期学校数字识别项目  

Vivado版本：2018.3  
开发板：sea-s7  
芯片：xilinx Spartan-7  
外设：ov5647摄像头 miniHDMI-to-HDMI连接线 HDMI显示器  
images为README中相关的图片  
Sourcecode为Vivado工程文件  
ExecutableFiles为比特流文件  

## 一、设计概述
### 1.设计目的：
  识别白底黑色的数字，通过OV5647摄像头采集图像，SEA读取图像信息，根据不同数字的特征，将0-9数字进行分类，然后识别数字，并在HDMI显示器上进行显示。  
### 2.应用领域：
  可以应用于各类报表以及票据的数字识别并自动录入，减轻人手动输入的工作量；在体育项目中，自动识别运动员身上号码布的个人编号，方便检录以及成绩记录；可以拓展应用到车牌的自动识别。  

## 二、系统组成及功能说明
### MIPI摄像头的驱动：
  1.csi2_d_phy_rx_0 Data_Read();这是数据读取的实例化；  
  2.csi_to_axis_0 Data_To_csi();这是数据转化成csi的实例化；  
	3.system_rgb2dvi_0_1 rgb2dvi_0();这是csi转化成dvp的实例化；  
	4.system_bayer2rgb_0_0 bayer2rgb_0 ();这是将bayer阵列获取的数据转化为RGB格式图像的实例化；  
	以上的各个模块都封装成IP，在system函数中直接例化就可以直接使用。  
### OV5647摄像头的初始化：
  系统采用MicroBlaze软核对摄像头进行控制，在软核上编程C语言程序实现OV5647摄像头的初始化； 
```
struct config_word_t cfg_ov5647_init[] =
{
		{0x0100, 0x00},
		{0x0103, 0x01},
		……
		{0x5a00, 0x08},
		{0x0100, 0x01},
		{0xffff, 0xff},
};
```
  该结构体表示的是I2C的传输状态机。  
  再使用以下的for循环进行初始化。  
```
for(i = 0; cfg_ov5647_init[i].addr != 0xffff; i++)
		writeReg(cfg_ov5647_init[i].addr, cfg_ov5647_init[i].data);
```
## 摄像头数据转换模块：
  摄像头的数据采用的是bayer颜色格式，奇数行包括green和red颜色的像素，偶数行包括blue和green颜色的像素。奇数列包括green和blue颜色的像素，偶数列包括red和green颜色的像素。  
所以每个像素点的颜色值由4个单色像素点组成，为两个G像素、一个R像素、一个B像素。通过对像素颜色值进行读取和转换，把R、B作为当前RGB的R、B值，G值为G1和G2值取平均。  

### 图像的灰度处理和二值化模块：
  对于数字识别而言，不需要使用到彩色数据，只要对于图像的形状进行分辨即可。为了简化数据处理，对RGB格式数据进行灰度处理，将RGB值转化为灰度值，这样三维的颜色数据就变为了一维的数据，因为摄像头采集到的绿色原始数据多，而且人眼对于绿色更为敏感，所以相应的绿色像素权值更大，最后得到的灰度值为0.299R+0.587G+0.114B。  
  此时也不能很好的将边界信息给分割开来，我们使用二值化处理灰度值图像，以50作为阈值，将灰度值和设定的阈值进行比较，灰度值大于50的为1，小于等于50为0，这样就把图像分为了前景和背景两部分，分别用0和1表示，图形的形状也就好确定了。 
  
### 数字识别模块：
  根据图像处理后的二值化01值对图形进行识别。首先需要对图像边界进行确定。设定一个识别框范围，大小为中心点上下左右偏移200像素的矩形框，即{(H_Center-200，V_Center-200),(H_Center+200,V_Center+200)}的矩形框中，在此范围内的像素进行检测，设定识别框的目的是为了排除周围无关像素对数字识别的干扰。然后对1像素进行统计，找出像素的边界存储下来，然后四个边界各向外扩大一个像素作为此数字的边框。确定三条检测直线对数字进行检测，通过和数字的交点个数和位置进行数字的判断。数字特征信息的提取基于打印体，如下图1,图2,图3所示，以图3数字5举例，红框是数字5的水平和竖直的上下左右边界。X1在竖直方向的2/5处的水平线，x2在竖直方向的2/3处的水平线，y在水平方的1/2处的水直线。我们以此特征来统计x1,x2,y与数字5的交叉点。  
  以交叉统计法来区分0-9数字的特征如下表1所示：  
  由于2,3,5的数字特征统计表一样，无法区分所以我们继续增加数字特征以区分2,3,5。如表2：  
  通过判断跳变位置和左右边界的距离大小来判断靠左还是靠右。  
  
### HDMI显示模块：
  图像经过二值化后进入HDMI进行显示，二值化为1则显示0xffffff的白色像素，否则为0的黑色像素，但当前显示像素的位置在识别框内时，替换显示颜色为红色0xff0000，这样可以在HDMI显示器上标注出红色识别框，当显示像素的位置到右上角{（1150，40），（1250，160）}的矩形框内时，替换显示内容为0~9的数字图像数据，通过相应的选择显示模块对数据进行读取显示当前的数字。将图像数据送入rgb2dvi显示驱动进行显示。

### 实现流程：
  摄像头初始化设置 -> 图像格式转换 -> 图像灰度处理和二值化 -> 数字识别 -> HDMI显示

### 操作说明：
  将待识别的数字放入红色识别框内，需要保证框内数字显示清晰，并且正向放置单个数字，屏幕右上角白色框中会显示绿色数字字符识别结果。

## 完成情况及性能参数
### 实现的模块：
1、摄像头采集图像的实现；  
2、图像的灰度处理和二值化；  
3、数字的识别；  
4、数字的HDMI显示屏显示。  
### 完成的功能：
实现了对单个正向放置的清晰数字的识别和显示。  

参考工程[https://github.com/liuweistrong/Digital-Recognition.git]

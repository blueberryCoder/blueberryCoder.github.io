---
title: 色彩模型
date: 2025-08-25 20:02:24
tags:
- [graphic]
categories:
- [graphic]
mathjax: true

---
# 光谱
1666年牛顿发现太阳光经三棱镜的折射后可呈现彩色光，称为光的色散现象。

<img src="/images/color_newton.png" alt="newton" width="600"> 
由牛顿的色散实验结果，可知白光是由多种颜色的光所组成。太阳光呈现白色，当它通过三棱镜折射后，将形成由红、橙、黄、绿、蓝、靛、紫（或红、橙、黄、绿、青、蓝、紫）顺次连续分布的彩色光谱，覆盖了大约在390到770纳米的可见光区。

<img src="/images/color_spectrum.png" alt="spd" width="600"> 

## SPD(谱功率密度)
可以用谱功率密度（Spectral Power Distribution (SPD)）用来描述光，用来描述光在不同波长之间的能量，它的单位是辐射度单位/波长，例如(watts/nm)。
比如下面这幅图中，太阳部分光线的SPD和蓝天部分的SPD如下：

<img src="/images/color_eg_spd.png" alt="spd" width="600"> 
再如下图，不同光的不同的SPD:
<img src="/images/color_spd2.png" alt="spd" width="600"> 

## 线性性质
SPD具有线性性质，比如蓝色和黄色的光加起来得到的SPD，就是蓝色和黄色分别得SPD相加。
<img src="/images/color_spd_linear.png" alt="spd" width="600"> 

# 什么是颜色？
颜色是人眼感知得到的，它并不是光的一种物理属性。不同波长的光也不是颜色。

## 人眼的结构
颜色的结构类似一个摄像机，光线通过晶状体聚焦子视网膜上，视网膜再将收到的光信号转化为发送给大脑的电信号。

<img src="/images/color_eye.png" alt="spd" width="600"> 
人眼有大约1200万的视干细胞，它们能给大脑提供亮度信息，600～700万左右的视锥细胞，它们能给大脑提供颜色信息。（从这个数量上我们也能看出来，人眼对亮度信息更敏感）。

<img src="/images/color_eye_cones.png" alt="spd" width="600"> 
视锥细胞也可以分为三种：S型、M型、L型，它们对不同波长的响应强度如下：
<img src="/images/color_cone_resp.png" alt="spd" width="600"> 
下图是从12给不同人中统计的视觉细胞的类型分布，注意不同类型视锥细胞的比例存在高度差异。这也就是说，不同的人感知到的颜色其实是有很大差异的（每个人的看到的世界是不一样的🐶）。
<img src="/images/color_cones_distribute.png" alt="spd" width="600"> 

# 三色理论
了解以上理论支持，人们就可以用一种数学模型来描述人眼对光的感知。人眼感知到的光的能量等于人眼的三种细胞对不同波长的强度乘以光源的SPD中不同波长的强度然后相加，可以用如下数学公式描述：

<img src="/images/color_spd_cones.png" alt="spd" width="600"> 

## 同色异谱
有以上积分的定义，我们可以得出对于人眼感知到的颜色，光源其实可以拥有不同的频谱。（只要保证最后积分和的结果是一致的就行）。
如下图中，不同的频谱，其实最终人眼感知到的能量是一样的。
<img src="/images/color_metamerism.png" alt="spd" width="600"> 
下图中2个太阳的频谱虽然不同，但是最终人眼感知到的是一样的。
<img src="/images/color_sunlight.png" alt="spd" width="600"> 

# 加色模型
如图，我们可以用三个信号，它们的SPD分别为：sR(λ), sG(λ), sB(λ)来表示三种原始光，我们分别调节不同原始光的强度就可以得到不同的颜色。最终的到的颜色为：R sR(λ) + G sG(λ) + B sB(λ)， R,G,B代表它们的强度。

<img src="/images/color_rgb_additive.png" alt="spd" width="600"> 

## CIE RGB匹配函数

CIE通过用以下3个原色的光做实验，得到一个颜色匹配函数。
<img src="/images/color_cie_primary.png" alt="spd" width="600"> 
颜色匹配函数如下，对于任意的频谱，人眼感知到到的颜色能通过这个函数的到人眼感知到的颜色（定义在CIE RGB中的颜色）
<img src="/images/color_cie_rgb.png" alt="spd" width="600"> 
（匹配函数中，有负数的情况，这种可以在实验时向目标色加一个红色强度的光来实现，即实现了对原色的减法）

# 颜色空间
色彩模型（color model）是用一组数组（每个颜色用3个或4个数字来表示）来描述颜色的数学模型。

# CIE XYZ
CIE XYZ中定义的X、Y、Z是并不存在的颜色。其中Y是表示的亮度(luminance)，另外2个是颜色相关的虚拟量。这个定义消除了负数的情况。
<img src="/images/color_cie_xyz.png" alt="spd" width="600"> 

## 色域
为了让我们更好的理解CIE XYZ，通过一下数据手段将三维的颜色空间显示在了二维平面。人眼能看见的颜色就是下图的这个马蹄形状。
它通过统一亮度Y，然后对X、Y、Z归一化，然后痛殴下面的数学公式的到x,y,z，因为x+y+z = 1 所以我们可以在二维平面呈现这个颜色空间。
<img src="/images/color_cie_gamut.png" alt="spd" width="720"> 

下图是一些常见的绝对色彩空间的色域图：

<img src="/images/color_gamut.png" alt="spd" width="720"> 

## 颜色是相对的

观察下面的图：
<img src="/images/color_eg_relative0.png" alt="spd" width="720"> 

<img src="/images/color_eg_relative1.png" alt="spd" width="720"> 
你能看出黄色是一直的吗？（你的大脑一直在欺骗你🐶）

# RGB色彩模型
RGB色彩模型是由红(R)、绿(G)、蓝(B)组成的一种加法颜色模型。它是我们在电视机、计算机中种常用的颜色模型。RGB色彩模型是一种设备依赖(device-dependent)的色彩模型，不同的设备因为厂商有不同的物理实现（比如：使用不同的荧光粉、不同的染料），所提供的红、绿、蓝的实现等级不一样，甚至同一台设备随着时间推移老化提供出的颜色也不是不一样的。所以如果没有色彩管理(color management)，不同的设备将不能提供出一致的颜色。

## 加法颜色模型
要用RGB生成一个颜色，需要将3种基础颜色（红、绿、蓝）叠加在一起（如：可以通过发射出3种光线直接进入人眼、或在白纸上照射3种光，通过反射进入人眼）。每束基础光都有自己的强度，假设从（0.0 到 1.0），混合在一起形成一种颜色。

<img src="/images/color_model_rgb.png" alt="RGB加法" width="360">

从上面这幅图我们可以看出，红、绿、蓝可以叠加成白色、红和绿可以叠加成黄色、红和蓝可以叠加成品红色、绿和蓝可以叠加成蓝绿色。
我们把这3个基础色也称之为颜色的分量。如果所有的分量（基础颜色）的强度都为0，我们将只能得到黑色。如果所有的分量都为1我们将得到白色，该点也是该颜色系统的白点(white point)。当所有分量的强度都相等时，我们得到是一种灰色，强度越低颜色越暗，强度越高颜色越亮。当所有分量强度不同时，我们将得到具有色相的颜色；其饱和度取决于所用原色中最强与最弱分量强度之间的差异——差异越大，饱和度越高；差异越小，饱和度越低。

RGB色彩模型并没有规定红、绿、蓝具体的实现、即没有定义原色本身是什么（比如它相对于人眼感知到的结果有多红？、有多蓝？、有多绿？、强度分布是怎么样的？）。所以它是一种相对的色彩空间。只有当红、绿、蓝这3种基础颜色并精准定义后，这个颜色空间才会变为绝对颜色空间，如sRGB，AdobeRGB。

原色的选择与人眼的生理结构有关，“良好”的原色的定义应当是能最大化人眼视锥细胞对不同波长的影响差异（在色度图中形成更大的颜色三家形）。人眼中正常的三类对光敏感的感光受体（锥体细胞）分别对黄色（长波，L）、绿色（中波，M）和紫色（短波，S）光最为敏感（各自的峰值波长分别约为 570 nm、540 nm 和 440 nm）。三类受体所接收到信号的差异，使得大脑能够区分非常广泛的颜色范围；总体上，人眼对黄绿色光以及绿到橙区间的色相差异最为敏感。

<img src="/images/color_cone_cell_resp.png" alt="cone" width="360"> 

仅使用三种原色并不足以复现所有颜色；只有位于由三原色色度所定义的颜色三角形内的那些颜色，才能通过对这三种光非负量进行加法混合加以再现。

如图，是sRGB颜色模型在色域图中的分布。

<img src="/images/color_srgb.png" alt="sRGB" width="360">

## RGB设备
常见的RGB显示设备有：阴极射线管(CRT)、液晶显示（LCD）、OLED等设备。屏幕上的每个像素由三个微小、彼此非常接近但仍相互独立的 RGB 发光单元驱动构成。在通常的观看距离下，这些独立的光源不可分辨，人的眼睛会把它们感知为某一种均匀的颜色。所有像素在矩形屏幕表面共同排列，便构成了彩色图像。在数字图像处理过程中，每个像素都可以在计算机内存或接口硬件（例如显卡）中，用红、绿、蓝三个颜色分量的二进制数值来表示。经过适当的管理，这些数值会通过伽马校正转换为强度或电压，以校正某些设备固有的非线性，从而在显示设备上再现预期的亮度/强度。

CRT显示：

<img src="/images/color_crt_tv.webp" alt="crt" width="360">

LCD显示设备：

<img src="/images/color_lcd.jpg" alt="lcd" width="360">

### 打包格式
设备内存中存储常用RGB_888、RGBA_8888、RGB_565等来存储RGB颜色。RGB_888即R、G、B的强度分别用8位二进制来表示。在这种体系下，允许16,777,216（256³ 或 2²⁴） 种离散的 R、G、B 组合。为增加明暗层次，人们采用了多种做法；一些格式（如 .png、.tga 等）会使用第四个灰度颜色通道作为遮罩层，通常称为 RGB32（即带有额外通道的 RGB），在内存中的一般为RGBA_8888格式。RGB_565适用越内存比较紧张的场景、即用5位二进制表示红色和蓝色，用6位二进制来表示绿色（颜色对绿色比较敏感）。

## 几何表示
由于颜色通常由三个分量来定义——不仅在 RGB 模型中如此，在诸如 CIELAB、Y'UV 等其他色彩模型中也是如此——因此可以把这些分量值视作欧氏空间中的普通笛卡尔坐标，从而描述一个三维体积。对 RGB 模型而言，这对应于一个使用 0–1 非负取值的立方体：将黑色赋予位于顶点 (0, 0, 0) 的原点，沿三条坐标轴分量强度递增，直到对角顶点 (1, 1, 1) 的白色。
一个 RGB 三元组 (r, g, b) 表示该颜色在立方体（或其面、棱）中的三维坐标。这样一来，只需计算两种 RGB 颜色之间的距离就能评估它们的颜色相似度：距离越短，相似度越高。超出色域的计算也可以用这种方式进行。

<img src="/images/color_rgb_cube.png" alt="rgb_cube" width="360">

# YUV色彩模型
Y′UV（也写作 YUV）是模拟彩色电视标准中采用的色彩模型。一种颜色用一个 Y′ 分量（亮度信号，luma）和两个色度分量 U、V 来描述。撇号（′）表示这个亮度Y′是从经过伽马校正的RGB输入计算得到的，因此与真实的亮度（luminance）不同。如今，在计算机行业中，“YUV”这一术语常被用来泛指以 YCbCr 编码的各种色彩空间。

在电视制式里，色彩信息（U与V）通过副载波单独加入，这样黑白电视机也能接收并以其原生的黑白方式显示彩色节目的传输，而无需额外的传输带宽。

Y′UV是在工程师们希望在黑白电视基础设施上实现彩色电视时发明的。他们需要一种既兼容黑白电视、又能加入色彩的传输方式。亮度分量（Y′）本就作为黑白信号而存在；他们在此基础上加入 U、V 色度信号作为解决方案。
之所以选用 U、V 来表示色度，而不是直接传输 R、B，是因为 U、V 是色差信号。换言之，U、V 告诉电视机在不改变亮度的情况下把某一点的颜色偏移到何处，或者以此消彼长地调整两种原色的相对亮度，以及应偏移多少。当 U、V 值越高（或为负时数值越低），该点就越饱和（更“有色”）；当 U、V 越接近 0，色偏越小，意味着红、绿、蓝的亮度更为均衡，产生更灰的外观。这就是使用色差信号的好处：与其直接说明某颜色里“有多少红”，不如说明它“比绿或蓝多多少红”。
由此可知，当 U、V 为 0 或不存在时，显示的就是灰度图像。若当初采用的是 R、B 信号，那么即使在黑白场景中它们也不会为零，于是仍需三路数据承载。这一点在彩色电视的早期尤为重要：旧式黑白电视信号不包含 U、V，彩电直接按黑白方式显示即可。另一方面，黑白接收机可以取用 Y′，忽略 U、V，使 Y′UV 与既有黑白设备（输入与输出）向后兼容。

## 与RGB的转换
Y′UV 信号通常由经过伽马校正的RGB（红、绿、蓝）做为源生成。先对 R、G、B 设定权重并求和得到 Y′，它表示整体亮度（luma/近似于 luminance）。随后将 U、V 计算为 B、R 与 Y′ 的差值按比例缩放后的结果。

### ITU-R BT.601 转换
用于标清电视（SDTV）的Y′CbCr形式由 ITU-R BT.601（原 CCIR 601）为数字分量视频定义，其来源是相应的 RGB 空间（ITU-R BT.470-6 System M 的原色）。常数为：
$$
K_R=0.299,\quad K_G=0.587,\quad K_B=0.114.
$$
由上述常数与公式，可推得 BT.601 下的关系。

#### RGB转YUV
**模拟信号R′G′B′转模拟 YPbPr**
$$
\begin{aligned}
Y'  &= 0.299\,R' + 0.587\,G' + 0.114\,B',\\
P_B &= -0.168736\,R' - 0.331264\,G' + 0.5\,B',\\
P_R &= \phantom{-}0.5\,R' - 0.418688\,G' - 0.081312\,B'.
\end{aligned}
$$
矩阵形式为：
$$
\begin{bmatrix}
Y' \\[6pt]
P_B \\[6pt]
P_R
\end{bmatrix}
=
\begin{bmatrix}
0.299 & 0.587 & 0.114 \\
-0.168736 & -0.331264 & 0.5 \\
0.5 & -0.418688 & -0.081312
\end{bmatrix}
\begin{bmatrix}
R' \\[6pt]
G' \\[6pt]
B'
\end{bmatrix}
$$

#### 逆变换
**模拟信号YUV转RGB**
$$
\begin{bmatrix}
R'\\ G'\\ B'
\end{bmatrix}
=
\begin{bmatrix}
1 & 0 & 1.402\\
1 & -0.344136 & -0.714136\\
1 & 1.772 & 0
\end{bmatrix}
\begin{bmatrix}
Y'\\ P_B\\ P_R
\end{bmatrix}
$$

**模拟 R′G′B′转数字 Y′CbCr（每分量 8 位）：**
$$
\begin{aligned}
Y'  &= 16 + \bigl(65.481\,R' + 128.553\,G' + 24.966\,B'\bigr),\\
C_B &= 128 + \bigl(-37.797\,R' - 74.203\,G' + 112.0\,B'\bigr),\\
C_R &= 128 + \bigl(112.0\,R' - 93.786\,G' - 18.214\,B'\bigr).
\end{aligned}
$$
矩阵形式（带偏置项）：

$$
\begin{bmatrix}
Y' \\[6pt]
C_B \\[6pt]
C_R
\end{bmatrix}
=
\begin{bmatrix}
65.481 & 128.553 & 24.966 \\
-37.797 & -74.203 & 112.0 \\
112.0 & -93.786 & -18.214
\end{bmatrix}
\begin{bmatrix}
R' \\[6pt]
G' \\[6pt]
B'
\end{bmatrix}
+
\begin{bmatrix}
16 \\[6pt]
128 \\[6pt]
128
\end{bmatrix}
$$

此种 Y′CbCr 形式主要用于较早的标清电视系统，其所用 RGB 模型贴合老式 CRT 荧光粉的发光特性。

### ITU-R BT.709 转换
ITU-R BT.709标准为HDTV主要指定了另一种形式的Y′CbCr。这种较新的形式也用于一些面向计算机显示的应用，在BT.709中，常数 $K_b$ 与 $K_r$ 的取值不同，但使用它们的公式相同。对BT.709，有：
$$
K_B = 0.0722,\quad K_R = 0.2126,\quad K_G = 1-K_B-K_R = 0.7152.
$$
这种 Y′CbCr 形式所基于的RGB模型更贴合较新CRT及其他现代显示设备的荧光粉发光特性。

#### RGB转YUV
**模拟R′G′B′转模拟 YPbPr：**
$$
\begin{bmatrix}
Y'\\ C_B\\ C_R
\end{bmatrix}
=
\begin{bmatrix}
0.2126 & 0.7152 & 0.0722\\
-0.09991 & -0.33609 & 0.436\\
0.615 & -0.55861 & -0.05639
\end{bmatrix} \\
\begin{bmatrix}
R'\\ G'\\ B'
\end{bmatrix}
$$


#### 逆变化
**模拟YPbPr转模拟 R′G′B′**
$$
\begin{bmatrix}R'\\G'\\B'\end{bmatrix}
=
\begin{bmatrix}
1 & 0        & 1.5748 \\
1 & -0.21482  & -0.38059 \\
1 & 2.12798   & 0
\end{bmatrix}
\begin{bmatrix}Y'\\C_B\\C_R\end{bmatrix}  
$$

### 打包格式
RGB文件通常以每像素 8、12、16 或 24 位进行编码。下面的示例我们假定每个像素24bits，记作 RGB888。标准的字节顺序很简单：
r0, g0, b0, r1, g1, b1, ...

YCbCr 的打包像素格式往往被称为“YUV”。这类文件可以用 每像素 12、16 或 24 位编码。根据色度抽样不同，大致可描述为 4:2:0p、4:2:2、4:4:4。Y 后面的撇号（′）常被省略，YUV420p 里的 p（planar，平面）也常被省略。就实际文件格式而言，4:2:0 最常见，因为数据量更小，文件扩展名通常为 “.YUV”。数据率与抽样（A:B:C）之间的关系由 Y 与 U/V 通道采样数之比定义。
从 RGB 到 YUV 或相反方向转换，最简单是使用RGB888与4:4:4。对于4:1:1、4:2:2、4:2:0，需要先转换/重建到 4:4:4 再处理。

**4:4:4**

4:4:4 很直接，因为不分组像素；差别只在于每个通道分配多少位以及字节排列方式。基础的 YUV3 方案每像素用 3 字节，顺序为：
y0, u0, v0, y1, u1, v1, ...
（这里用 u 代表 Cb、v 代表 Cr；以下相同。）

**4:1:1**

4:1:1 较少使用。像素按每 4 个水平分组。

**4:2:0**

4:2:0 使用极其广泛。主要格式有 IMC2、IMC4、YV12、NV12。这些格式都是平面型（planar），即把 Y、U、V 各自成片集中存放，而不是交错；在 8 位通道下总计每像素 12 位。
- NV12：可能是最常见的 8-bit 4:2:0 格式；Android 相机预览的默认格式。先写完整 Y 平面，然后写交织的色度行：U0, V0, U1, V1, ...
- I420：更简单、也更常用。先写完整 Y 平面，再写完整 U 平面，最后写完整 V 平面。

<img src="/images/yuv420.svg.png" alt="yuv420" width="720">

- IMC2：先写完整的 Y 图；色度每行按 V0…Vn, U0…Un 排列，其中 n 为该行的色度采样数（等于 Y 宽度的一半）。

- IMC4：与 IMC2 类似，但色度行顺序为 U0…Un, V0…Vn。

- YV12：与 I420 相同，只是 U/V 平面的顺序对调。

# HSL和HSV色彩模型
HSL 和 HSV 是在 RGB 色彩模型中用来表示颜色点的两种最常见的圆柱坐标表示。与笛卡尔（立方体）表示相比，这两种表示通过重新组织 RGB 的几何结构，力图更直观、更符合人类感知。它们于 20 世纪 70 年代为计算机图形学应用所开发；如今广泛用于取色器、图像编辑软件，而在图像分析和计算机视觉中使用相对较少。
HSL 代表 Hue（色相）、Saturation（饱和度）、Lightness（明度），也常被写作 HLS。
HSV 代表 Hue（色相）、Saturation（饱和度）、Value（值），也常被称为 HSB（其中 B 指 Brightness，亮度）。
第三种在计算机视觉中常见的模型是 HSI，即 Hue（色相）、Saturation（饱和度）、Intensity（强度）。

<img src="/images/color_hsl_hsv_models.svg.png" alt="hsl_hsv" width="720">

在这两种“圆柱体”模型中，绕中央垂直轴的角度对应“色相（hue）”，离轴心的距离对应“饱和度（saturation）”，沿轴向的位置对应“明度（lightness）”“值（value）或“亮度（brightness）”。需要注意的是，虽然 HSL 与 HSV 中的“色相”指的是同一属性，但它们对“饱和度”的定义差异很大。由于 HSL 与 HSV 都是对设备依赖的 RGB 模型做的简单变换，它们所定义的物理颜色取决于设备或特定 RGB 空间中红、绿、蓝三原色的具体颜色，以及用于表示这些原色分量大小的伽马校正。因此，每一台 RGB 设备都有其独特的 HSL 与 HSV 空间；相同的 HSL/HSV 数值在不同的基准 RGB 空间中会表示不同的颜色。

在这两种圆柱体几何中，围绕中央垂直轴的角度对应色相（hue）：从 0° 的红原色出发，经过 120° 的绿原色与 240° 的蓝原色，最后在 360° 回到红。两种几何的中轴都由中性、无彩或灰色组成：自上而下从 亮度 1（或值 1）的白到 亮度 0（或值 0）的黑。
在两种几何中，加色的原色与二次色——红、黄、绿、青、蓝、洋红——以及它们之间相邻成分的线性混合（有时称为纯色）都以饱和度 1排布在圆柱的外缘。这些饱和色在 HSL 中的明度(lightness)为 0.5，而在 HSV 中的值(value)为 1。把这些纯色与黑混合（得到所谓的 shade，加黑）时，饱和度不变。在 HSL 中，与白混合（tint，加白）时饱和度也不变；只有同时与黑和白混合（tone，加灰）时，饱和度才会小于 1。而在 HSV 中，仅加白就会降低饱和度。

另外可以观察到，在HSL中他最顶部的整个区域都是白色、最底部整个区域都是黑色。HSV中最底部都是黑色。所以为了方便认为调色，它们还有另外一种表示方式：用双圆锥型表示HSL，用单圆锥表示HSV模型。

<div class="img-row">
  <img src="/images/color_hsl_cone.png" alt="hsl_cone" width="260">
  <img src="/images/color_hsv_cone.png" alt="hsv_cone" width="260">
</div>

## 色彩模型转换

### RGB转HSL

$$
\begin{aligned}
&\textbf{归一化:}\quad
r=\frac{R}{255},\quad g=\frac{G}{255},\quad b=\frac{B}{255} \\[4pt]
&\textbf{极值与差值:}\quad
C_{\max}=\max(r,g,b),\quad
C_{\min}=\min(r,g,b),\quad
\Delta=C_{\max}-C_{\min} \\[6pt]
&\textbf{Lightness:}\quad
L=\frac{C_{\max}+C_{\min}}{2} \\[6pt]
&\textbf{Saturation:}\quad
S=\begin{cases}
0, & \Delta=0 \\[6pt]
\dfrac{\Delta}{1-\lvert 2L-1\rvert}, & \Delta\neq 0
\end{cases} \\[10pt]
&\textbf{Hue:}\quad
H=
\begin{cases}
0, & \Delta=0 \\[6pt]
60^\circ\!\times\!\big((\tfrac{g-b}{\Delta})\bmod 6\big), & C_{\max}=r \\[8pt]
60^\circ\!\times\!\big(\tfrac{b-r}{\Delta}+2\big), & C_{\max}=g \\[8pt]
60^\circ\!\times\!\big(\tfrac{r-g}{\Delta}+4\big), & C_{\max}=b
\end{cases}
\end{aligned}
$$


### RGB转HSV
**先将RGB（假设8bit）归一化。**
$$
\begin{aligned}
&\text{归一化：}\quad
r=\frac{R}{255},\quad g=\frac{G}{255},\quad b=\frac{B}{255} \\[4pt]
&\text{极值与差值：}\quad
C_{\max}=\max(r,g,b),\;
C_{\min}=\min(r,g,b),\;
\Delta=C_{\max}-C_{\min}
\end{aligned}
$$

**Hue（色相）：**

$$
H=
\begin{cases}
0, & \Delta=0 \\[6pt]
60^\circ\!\times\!\big((\frac{g-b}{\Delta})\bmod 6\big), & C_{\max}=r \\[8pt]
60^\circ\!\times\!\Big(\frac{b-r}{\Delta}+2\Big), & C_{\max}=g \\[8pt]
60^\circ\!\times\!\Big(\frac{r-g}{\Delta}+4\Big), & C_{\max}=b
\end{cases}
$$

**Saturation（饱和度）：**

$$
S=
\begin{cases}
0, & C_{\max}=0 \\[6pt]
\dfrac{\Delta}{C_{\max}}, & C_{\max}\neq 0
\end{cases}
$$

**Value（明度）：**

$$
V=C_{\max}.
$$

### 转换代码

#### rgb转hsv
```glsl
vec3 rgb2hsv(vec3 c)
{
    vec4 K = vec4(0.0, -1.0/3.0, 2.0/3.0, -1.0);
    vec4 p = mix(vec4(c.bg, K.wz), vec4(c.gb, K.xy), step(c.b, c.g));
    vec4 q = mix(vec4(p.xyw, c.r), vec4(c.r, p.yzx), step(p.x, c.r));
    float d = q.x - min(q.w, q.y);
    float e = 1.0e-10;
    return vec3(abs(q.z + (q.w - q.y) / (6.0 * d + e)), d / (q.x + e), q.x);
}

vec3 hsv2rgb(vec3 c)
{
    vec4 K = vec4(1.0, 2.0/3.0, 1.0/3.0, 3.0);
    vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
    return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

```
#### rgb转hls
```glsl
vec3 rgb2hls(vec3 c)
{
    vec4 K = vec4(0.0, -1.0/3.0, 2.0/3.0, -1.0);
    vec4 p = mix(vec4(c.bg, K.wz), vec4(c.gb, K.xy), step(c.b, c.g));
    vec4 q = mix(vec4(p.xyw, c.r), vec4(c.r, p.yzx), step(p.x, c.r));

    float Cmax = q.x;
    float Cmin = min(q.w, q.y);
    float d    = Cmax - Cmin;           // Δ (chroma)
    float L    = 0.5 * (Cmax + Cmin);   // HLS 的 Lightness
    float e    = 1.0e-10;
    float H = abs(q.z + (q.w - q.y) / (6.0 * d + e));
    float S = d / (1.0 - abs(2.0 * L - 1.0) + e);
    return vec3(H, L, S);
}

vec3 hls2rgb(vec3 hls)
{
    float H = hls.x;
    float L = hls.y;
    float S = hls.z;

    float C = (1.0 - abs(2.0 * L - 1.0)) * S;
    vec3 t = clamp(abs(mod(H * 6.0 + vec3(0.0, 4.0, 2.0), 6.0) - 3.0) - 1.0, 0.0, 1.0);
    return L + C * (t - 0.5);
}

```
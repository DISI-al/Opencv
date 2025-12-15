# OpenCV 彩色图像直方图绘制学习笔记

## 一、图像直方图的原理及作用

1. **直方图原理**

图像直方图是图像灰度分布的统计图表，横坐标表示图像的灰度级（彩色图像为每个通道的灰度级，范围 0~255），纵坐标表示对应灰度级的像素数量占总像素数的比例或像素数量。
它将二维的图像像素分布，转化为一维的统计曲线，核心是对图像中不同灰度级的像素进行频次统计，不关注像素的空间位置，只关注灰度值的分布特征。

2.  **直方图的核心作用**

1. **图像灰度分布分析**：直观判断图像的亮暗程度，若直方图峰值集中在高灰度区（右侧），图像偏亮；集中在低灰度区（左侧），图像偏暗；分布均匀则图像对比度适中。

2. **图像增强的依据**：是直方图均衡化、直方图规定化等图像增强算法的核心基础，通过调整直方图的分布形态，提升图像的对比度和视觉效果。

3. **特征提取与匹配**：在目标检测、图像检索中，直方图可作为图像的全局特征，用于相似图像的匹配与识别（如彩色直方图可区分不同颜色分布的图像）。

4. **阈值分割的参考**：通过直方图的波峰波谷，可选择合适的阈值，实现前景与背景的分离（如二值化处理）。

## 二、核心功能

基于 OpenCV C++ 实现彩色图像的 RGB 三通道直方图计算与可视化，通过统计每个通道不同灰度级的像素数量，以折线图形式直观展示图像的灰度分布特征。

## 三、核心 API 详解

**1. split(InputArray src, OutputArrayOfArrays mv)**

~~~
参数	说明	
src	输入的多通道图像（此处为 CV_8UC3 彩色图像）	
mv	输出的单通道图像向量，按 B→G→R 顺序存储分离后的通道	
功能	将彩色图像拆分为蓝、绿、红三个独立的单通道灰度图	
~~~
代码示例
~~~c++
vector<Mat>bgr_plane;
split(image, bgr_plane);
~~~

**2.  calcHist(const Mat* images, int nimages, const int* channels, InputArray mask, OutputArray hist, int dims, const int* histSize, const float ranges, bool uniform=true, bool accumulate=false)**

直方图计算的核心函数，用于统计图像中指定灰度级的像素数量

~~~
参数 说明 
images 输入图像的指针（此处为单通道图像的地址，如 &bgr_plane[0]） 
nimages 输入图像的数量，此处固定为 1 
channels 指定计算直方图的通道索引，单通道设为 {0} 
mask 掩码图像，非零区域参与计算，传 Mat() 表示无掩码 
hist 输出的直方图矩阵，为 1 维 float 类型矩阵 
dims 直方图维度，灰度直方图设为 1 
histSize 分箱数（bins），此处设为 {256}，对应 0~255 所有灰度级 
ranges 灰度值取值范围，此处为 {0.0f, 256.0f}（左闭右开，覆盖 0~255） 
uniform 是否为均匀直方图，设为 true 表示每个分箱区间宽度相同 
accumulate 是否累加直方图，设为 false 表示每次计算重置直方图 
~~~

代码示例

```c++
const int channels[1] = { 0 };
const int bins[1] = { 256};
static const float hranges[2] = { 0.0f ,256.0f};
const float* ranges[1] = { hranges };
Mat b_hist;
Mat g_hist;
Mat r_hist;

calcHist(&bgr_plane[0], 1, channels, Mat(), b_hist, 1, bins, ranges,true, false);
calcHist(&bgr_plane[1], 1, channels, Mat(), g_hist, 1, bins, ranges,true, false);
calcHist(&bgr_plane[2], 1, channels, Mat(), r_hist, 1, bins, ranges, true,false);
```

**3. normalize(InputArray src, OutputArray dst, double alpha=0, double beta=1, int norm_type=NORM_L2, int dtype=-1, InputArray mask=noArray())**

将直方图数值缩放到画布高度范围内，避免超出显示区域

~~~
参数 说明 
src 输入的原始直方图矩阵 
dst 归一化后的输出矩阵（可与 src 同地址，原地修改） 
alpha 归一化最小值，此处设为 0 
beta 归一化最大值，此处设为画布高度 histImage.rows 
norm_type 归一化类型，NORM_MINMAX 表示将数值缩放到 [alpha, beta] 区间 
dtype 输出矩阵数据类型，设为 -1 表示与输入相同 
mask 掩码，传 Mat() 表示无掩码 
~~~
代码示例
```c++
normalize(b_hist, b_hist, 0, histImage.rows, NORM_MINMAX, -1, Mat());
normalize(g_hist, g_hist, 0, histImage.rows, NORM_MINMAX, -1, Mat());
normalize(r_hist, r_hist, 0, histImage.rows, NORM_MINMAX, -1, Mat());

```

**4.  line(InputOutputArray img, Point pt1, Point pt2, const Scalar& color, int thickness=1, int lineType=LINE_8, int shift=0)**

在画布上绘制线段，用于连接相邻分箱的直方图数值点

```
参数 说明 
img 绘制的目标画布（histImage） 
pt1 线段起点坐标 
pt2 线段终点坐标 
color 线段颜色，B 通道设为 Scalar(255,0,0)、G 通道设为 Scalar(0,255,0)、R 通道设为 Scalar(0,0,255) 
thickness 线段宽度，此处设为 2 
lineType 线条类型，设为 8 表示 8 连通线 
shift 坐标点小数位数，设为 0 表示整数坐标 
```

代码示例

```c++
for (int i = 1; i < bins[0];i++)
{
    line(histImage, 
        Point(bin_w * (i - 1),hist_h - cvRound(b_hist.at<float>(i - 1))),
        Point(bin_w * (i), hist_h - cvRound(b_hist.at<float>(i))), 
         Scalar(255, 0, 0),2,8,0);
}
```

**5. namedWindow(const string& winname, int flags=WINDOW_AUTOSIZE)****

```
参数 说明 
winname 窗口名称（此处为 "Histogram Demo"） 
flags 窗口属性，WINDOW_AUTOSIZE 表示窗口大小随图像自动调整 
功能 创建用于显示直方图的窗口 
```

代码示例

```c++
namedWindow("Histogram Demo", WINDOW_AUTOSIZE);
```

**6. imshow(const string& winname, InputArray mat)**

```
参数 说明 
winname 目标窗口名称 
mat 要显示的图像（此处为绘制好的直方图画布 histImage） 
功能 在指定窗口显示图像 
```

代码示例

```
imshow("Histogram Demo", histImage);
```

**7. cvRound(double value)**

```
参数 说明 
value 需要取整的浮点数 
功能 对浮点数进行四舍五入取整，将归一化后的直方图浮点数值转为整数坐标，避免绘图位置偏差 
```

代码示例

```c++
// 配合line函数使用，计算纵轴坐标
hist_h - cvRound(b_hist.at<float>(i - 1))
hist_h - cvRound(g_hist.at<float>(i - 1))
hist_h - cvRound(r_hist.at<float>(i - 1))
```

## 四、直方图绘制完整流程

### 步骤 1：通道分离

调用 **split(image, bgr_plane)** 将输入的彩色图像拆分为蓝、绿、红三个单通道图像，存入 **vector<Mat>** 类型的 **bgr_plane** 中，因为彩色图像需要按通道分别计算直方图。

### 步骤 2：设置直方图参数

• **通道索引**：**const int channels[1] = {0}**，指定计算单通道直方图。

• **分箱数**：**const int bins[1] = {256}**，每个灰度级（**0~255**）对应一个分箱，保证统计精度。

• **灰度范围**：**static const float hranges[2] = {0.0f, 256.0f}**，采用左闭右开区间覆盖所有灰度值；         **const float* ranges[1] = {hranges}** 为 **calcHist** 提供范围参数。

### 步骤 3：计算三通道直方图

分别对蓝、绿、红通道调用 **calcHist** 函数，得到三个 1 维直方图矩阵 **b_hist**、**g_hist**、**r_hist**，矩阵中每个元素的值代表对应灰度级的像素数量

### 步骤 4：创建直方图画布

**• 定义画布尺寸**：宽度 **hist_w=512**，高度 **hist_h=400**。

**• 计算分箱宽度**：**bin_w = cvRound((double)hist_w / bins[0])**，即每个分箱在画布横轴上占据的像素宽度（此处 **bin_w=2**）。

**• 创建黑色画布**：**Mat histImage = Mat::zeros(hist_h, hist_w, CV_8UC3)**，初始为全黑的 3 通道图像。

### 步骤 5：直方图归一化

调用 **normalize** 函数，将三个通道的直方图数值缩放到 **[0, 400]**（画布高度）区间，确保所有数值都能适配画布显示，避免因像素数量过大导致折线超出画布。

### 步骤 6：绘制直方图折线

通过循环遍历每个分箱**（i 从 1 到 255）**，为每个通道绘制相邻分箱的线段：

**1. 横轴坐标**：第 **i-1** 个分箱的横轴位置为 **bin_w*(i-1)**，第 **i** 个分箱为 **bin_w*i **。

**2. 纵轴坐标**：因图像坐标系原点在左上角，需用 **hist_h - cvRound(hist.at<float>(i))** 计算，实现直方图向上绘制。

**3. 颜色区分**：蓝、绿、红通道分别使用对应颜色的线段，确保可视化结果可区分。

### 步骤 7：显示直方图

调用 **namedWindow** 创建窗口，**imshow** 显示绘制好的直方图画布。注意：需在 **imshow** 后添加 **waitKey(0)**，否则窗口会一闪而过。

## 五、关键注意事项

1. 输入图像必须为 **CV_8UC3** 彩色图像，若为灰度图需删除通道分离逻辑，直接计算单通道直方图。

2. 分箱数设为 **256** 是最合理的选择，可精准匹配 **0~255** 的灰度级范围。

3. 坐标系转换是绘图的关键：图像左上角为原点，需通过减法将直方图的“底部”对齐到画布横轴。

## 六、Demo

**图像直方图 Histogram**

![](C:\Users\DISI\Desktop\HOME\Pictures\histogram.png)

**原图  original**

<img src="C:\Users\DISI\Desktop\HOME\Pictures\original.png"  />


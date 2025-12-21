# OpenCV 图像直方图均衡化学习笔记

##  一、 直方图均衡化底层**数学原理**及**作用**

直方图均衡化的核心是 将原始图像的灰度分布映射为**均匀分布**，以此提升图像**整体对比度**，尤其适用     于灰度分布集中的图像（如**雾天图像**、**低曝光图像**）。其数学推导、公式作用及图像变化效果如下：

1. 计算原始灰度**直方图**
    设图像灰度级范围为 [0, L-1]（L=256 对应 8 位图像），**n_i** 为灰度级 **i** 的像素数量，图像**总像素数** 
   **N** = n_i  **求和**
   灰度级 i 的**概率密度**（归一化直方图）公式：
  
    **pr(ri)  = n_i/N**

​        **公式作用**：量化每个灰度级在图像中的**占比**，直观反映图像灰度分布状态（如**暗图**的 p_r 集中在**低灰度**区间）。

  图像关联效果：通过该公式可判断图像偏亮/偏暗——若**低灰度**值的 **p_r 大**，图像**整体偏暗**；若**高灰度**值的 **p_r 大**，图像**整体偏亮**。

2. 计算**累积分布**函数（**CDF**）
  累积分布函数描述**灰度级小于等于 i 的像素占比**，公式：

  ![CDF](.\image\CDF.jpg)

   **公式作用**：建立原始灰度到新灰度的映射关系基础，将离散的灰度占比累积为连续的分布曲线。

图像关联效果：CDF 曲线的斜率决定灰度拉伸程度——斜率大的区间，原始灰度会被拉伸为更宽范围的新灰度，对应图像中该区间的对比度提升。

​        **3. 灰度映射与归一化**
​        将累积分布函数的值映射回 [0, L-1] 灰度范围，得到均衡化后的灰度值：

​       ![format](.\image\format.jpg)
​        公式作用：将 CDF 的归一化值（0~1)转换为 0~255 的有效灰度值，完成最终的灰度映射。

​        图像关联效果：**原始灰度集中的区间会被拉伸**，**分散的区间会被压缩**，最终图像的灰度**分布更均匀**，视觉上**细节更清晰**（如**暗部隐藏的纹理会显现**）。

二、 核心代码解析（C++ 版本）
~~~c++
void QuickDemo:: Histogram_eq_demo(Mat& image)
{
	Mat gray;
	cvtColor(image, gray, COLOR_BGR2GRAY);
	imshow("gray_Demo", gray);
	Mat dst;
	equalizeHist(gray, dst);

	imshow("equalizeHist_Demo", dst);
}

~~~
2. 关键 API 解析
   函数 作用 参数说明 
   `cvtColor(image, gray, COLOR_BGR2GRAY) `彩色图像转灰度图 `image`：输入彩色图；`gray`：输出灰度图；`COLOR_BGR2GRAY`：色彩空间转换标识 
   `equalizeHist(gray, dst)` 灰度图直方图均衡化 `gray`：输入 8 位单通道灰度图；`dst`：输出均衡化后的图像 

   ### 三、 局部直方图均衡化实现

​       全局直方图均衡化的缺陷：会**过度增强图像中的噪声**（如**暗区域的微小噪声会被放大**），且对**局部细节提升有限**。
​       局部直方图均衡化思路：将图像划分为多个子块，对**每个子块单独做均衡化**，并通过**插值消除块间边界效应**，适用于**需要保留局部细      节**的场景（如**医学影像**、**遥感图像**）。

1. C++ 实现代码
~~~c++
   void QuickDemo::Local_Histogram_eq_demo(Mat& image) {
   Mat gray;
   cvtColor(image, gray, COLOR_BGR2GRAY);
   imshow("gray_Demo", gray);

   // 1. 全局均衡化作为对比
   Mat dst_global;
   equalizeHist(gray, dst_global);

   // 2. 创建 CLAHE 对象，设置关键参数
   Ptr<CLAHE> clahe = createCLAHE(2.0, Size(8, 8));
   Mat dst_local;
   // 3. 局部直方图均衡化
   clahe->apply(gray, dst_local);

   imshow("Global_equalizeHist", dst_global);
   imshow("Local_CLAHE", dst_local);
   }
~~~
2. 关键参数及作用
   CLAHE 是什么
   CLAHE 是 OpenCV 中定义的一个类，全称是 Contrast Limited Adaptive Histogram      
   Equalization（对比度受限的自适应直方图均衡   化），专门用于实现局部直方图均衡化。
   这个类里封装了局部均衡化的核心算法，比如分块处理、对比度限制、插值去块效应等逻辑。

​      `Ptr<CLAHE>` 是什么

​      `Ptr` 不是普通容器，而是 OpenCV 提供的智能指针模板类，作用是自动管理内存，避免手动   
​      `new/delete` 导致的内存泄漏。
​      `Ptr<CLAHE>` 表示一个指向 CLAHE 类对象的智能指针，等价于“能自动释放内存的 `CLAHE*` 指针

3. 为什么不用 `new CLAHE()` 直接创建对象
   OpenCV 中 `CLAHE `类的构造函数是受保护的（不能直接 new 实例化），必须通过官方提供的工厂函数 `createCLAHE()` 来创建对象，这个函数的返回值就是 `Ptr<CLAHE> `智能指针。

3. `createCLAHE()` 工厂函数（核心）

   这是创建 CLAHE 对象的唯一方式，参数决定局部均衡化的效果：
   `clipLimit` 对比度限制阈值 限制每个子块的最大对比度，超过阈值的像素会被裁剪，避免噪声过度增强 2.0~5.0（值越大对比度越强，噪声越明显） 
   `tileGridSize` 子块网格大小将图像划分为 `Size(w,h)` 的子块矩阵，每个子块单独做均衡化 `Size(8,8)/Size(16,16)`（子块越小，局部细节越精细） 

    函数返回值：`Ptr<CLAHE> `智能指针，指向一个初始化好的 CLAHE 对象。

    智能指针的优势：函数执行完后，无需手动释放 `clahe` 对象，`Ptr` 会自动在对象生命周期结束时释放内存。

   2.  `clahe->apply(gray, dst_local) `成员函数

    是智能指针的成员访问运算符，用法和普通指针一样。

    作用：调用 CLAHE 对象中封装的局部均衡化算法，对输入灰度图 gray 处理，结果存入` dst_local`。

    要求：输入必须是 8位单通道灰度图（和 `equalizeHist` 要求一致）。

    **通俗类比**
   你要做局部均衡化（买一台专用机器）→ `CLAHE` 是机器的型号 → `createCLAHE()` 是厂家的“定制机器”服务 → 你告诉厂家参数`（clipLimit/tileGridSize）`→ 厂家给你一台现成的机器（`Ptr<CLAHE> `智能指针）→ 你按下机器的“启动键”（`apply` 函数）→ 机器自动处理图像。

   **作用**：子块越小，局部细节提升越精细，但计算量会增加；需根据图像分辨率调整

3. 图像变化效果

  处理后图像的局部细节更清晰（如纹理边缘更锐利），且噪声不会被过度增强，整体视觉效果更自然。

### 四 、指定局部区域的实现直方图均匀化

如果想只对图像的某一个部分（比如 ROI 区域）做均衡化，需要分 3 步：

1. 定义感兴趣区域（ROI）：确定要处理的局部区域坐标。

2. 创建掩膜（Mask）：掩膜和原图大小一致，ROI 区域设为白色（255），其余区域设为黑色（0）。

3. 结合掩膜做均衡化：只对掩膜白色区域执行均衡化，黑色区域保持原图不变。

~~~c++
void QuickDemo::ROI_Histogram_eq_demo(Mat& image) {
    Mat gray;
    cvtColor(image, gray, COLOR_BGR2GRAY);
    imshow("Original Gray", gray);

    // 1. 定义 ROI 区域：比如左上角 200x200 的区域
    Rect roi(0, 0, 200, 200);
    // 2. 创建和原图大小一致的掩膜，初始全黑
    Mat mask = Mat::zeros(gray.size(), CV_8UC1);
    // 3. ROI 区域设为白色（255），表示要处理的区域
    mask(roi) = 255;

    // 4. 对灰度图做均衡化，但只作用于掩膜白色区域
    Mat dst = gray.clone(); // 复制原图，避免覆盖
    equalizeHist(gray, dst); // 先全局均衡化
    // 5. 掩膜融合：只保留 ROI 区域的均衡化结果，其余区域用原图
    gray.copyTo(dst, ~mask); // ~mask 是掩膜取反，黑色变白色

    imshow("ROI EqualizeHist", dst);
}
~~~


CLAHE	    全图分块，每块均衡化	            提升整幅图像的局部细节（如医学影像、纹理图像）	
掩膜+ROI	只处理用户指定的局部区域	    针对特定区域增强（如人脸检测中只增强人脸区域）	

  ###  五、 彩色图像直方图均衡化实现

  禁忌：不能直接对 RGB 三通道分别做均衡化——会导致通道间灰度映射不一致，出现严重的色彩失真。
  标准做法：转换到 YUV/YCrCb 等亮度-色度分离的色彩空间，仅对亮度通道均衡化，再转换回 RGB 空间，兼顾对比度提升和色彩保真。

1. C++ 实现代码
~~~c++
   void QuickDemo::Color_Histogram_eq_demo(Mat& image) {
   imshow("Original_Color", image);

   Mat ycrcb;
   // 1. BGR 转 YCrCb 空间（Y=亮度，Cr/Cb=色度）
   cvtColor(image, ycrcb, COLOR_BGR2YCrCb);

   // 2. 通道分离，提取亮度通道 Y
   vector<Mat> channels;
   split(ycrcb, channels);

   // 3. 仅对亮度通道做均衡化，保持色度通道不变
   equalizeHist(channels[0], channels[0]);

   // 4. 通道合并，转回 BGR 空间
   merge(channels, ycrcb);
   Mat dst_color;
   cvtColor(ycrcb, dst_color, COLOR_YCrCb2BGR);

   imshow("Color_equalizeHist", dst_color);
   }
~~~
2. 核心原理及图像效果

• YCrCb 空间中，Y 代表亮度，Cr/Cb 代表色度；仅均衡化 Y 通道，不会改变色彩信息。

• 处理后图像 对比度提升，色彩还原准确，适用于彩色监控图像、航拍图像的增强处理。

### 六、Demo

![equal-histogram](.\image\equal-histogram.jpg)

![gray](.\image\gray.png)

![histogram_equal](.\image\histogram_equal.png)
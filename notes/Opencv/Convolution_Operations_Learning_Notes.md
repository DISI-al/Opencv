## 常见图像卷积操作学习笔记

### 1. 简介
卷积是一种普遍使用的图像处理操作，旨在通过核（Kernel）对图像中的每个像素进行加权和，在去噪、边缘检测、模糊化等应用中具有重要作用。本笔记涵盖了若干常见卷积操作：模糊化、边缘检测、锐化等。

### 2. 示例代码与说明
以下代码展示了OpenCV中实现不同卷积效果的操作示例：

#### 2.1 均值模糊 (Blur)
```cpp
void QuickDemo::blur_demo(Mat& image)
{
    Mat dst;
    blur(image, dst, Size(15,15), Point(-1, -1));
    imshow("original", image);
    imshow("blur_demo", dst);
}
```
**作用：** 去除图像中的噪声，使画面显得更加平滑。

#### 2.2 高斯模糊 (GaussianBlur)
```cpp
void QuickDemo::gaussian_blur_demo(Mat& image)
{
    Mat dst;
    GaussianBlur(image, dst, Size(15, 15), 0, 0);
    imshow("original", image);
    imshow("gaussian_blur_demo", dst);
}
```
**作用：** 相比均值模糊，产生更自然的模糊效果，常用于去除高斯噪声。

#### 2.3 中值模糊 (MedianBlur)
```cpp
void QuickDemo::median_blur_demo(Mat& image)
{
    Mat dst;
    medianBlur(image, dst, 15);
    imshow("original", image);
    imshow("median_blur_demo", dst);
}
```
**作用：** 对于椒盐噪声有显著的去除效果。

#### 2.4 锐化滤波 (Sharpening)
```cpp
void QuickDemo::sharpen_demo(Mat& image)
{
    Mat kernel = (Mat_<float>(3,3) << 0, -1, 0, -1, 5, -1, 0, -1, 0);
    Mat dst;
    filter2D(image, dst, -1, kernel);
    imshow("original", image);
    imshow("sharpen_demo", dst);
}
```
**作用：** 增强图像边缘，使细节更加清晰。

#### 2.5 边缘检测 (Canny)
```cpp
void QuickDemo::canny_demo(Mat& image)
{
    Mat dst, gray;
    cvtColor(image, gray, COLOR_BGR2GRAY);
    Canny(gray, dst, 50, 150);
    imshow("original", image);
    imshow("canny_demo", dst);
}
```
**作用：** 在处理图像边缘提取时十分高效。

### 3. 效果说明
卷积操作的效果取决于核的大小、具体的操作类型以及参数的调整。例如：
1. 模糊核越大，图像越模糊。
2. 边缘检测的两个阈值影响提取的边缘强度。

### 4. 常见应用场景
- 去噪：模糊操作（均值、高斯、中值）可有效减轻噪声。
- 边缘提取：Canny操作是边缘检测中的经典算法。
- 图像锐化：在增强视觉效果方面，锐化操作大有可为。

### 5. 示例图片
添加您自己的实验图像，展示卷积不同操作的具体效果。

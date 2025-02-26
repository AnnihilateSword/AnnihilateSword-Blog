---
title: TinyRenderer 学习笔记 (上)
tags: [计算机图形学, Software Renderer]
date: 2025-02-26
updated: 2025-02-26
cover: /res/img/post/03-tinyrenderer-learn-note-01/3-4-1.jpg
top_img: /res/img/site/background.png
---

![](/res/img/post/03-tinyrenderer-learn-note-01/3-4-1.jpg)

<br>

# 前言

本文配套代码：

👉https://github.com/AnnihilateSword/TinyRenderer

其中添加了更为详细的注释，帮助更快理解掌握知识！

<br>
<br>

# 前置知识 (如果了解可跳过)

## Bresenham's line algorithm 简介

**Bresenham's line algorithm** 是一种绘制线条的算法，它确定了 n 维栅格中应该选择的点，以便在两点之间形成近似直线。它通常用于在位图图像中绘制线原语 (例如在计算机屏幕上)，因为它只使用整数加法，减法和位移位，所有这些操作在历史上常见的计算机体系结构中都是非常便宜的操作。它是一种增量误差算法，是计算机图形学领域发展最早的算法之一。原始算法的扩展称为中点圆算法可用于绘制圆。

虽然 Wu 算法等算法也经常用于现代计算机图形学，因为它们可以支持抗锯齿，但 Bresenham 的线算法仍然很重要，因为它的速度和简单性。该算法用于绘图仪等硬件和现代图形卡的图形芯片中。它也可以在许多软件图形库中找到。由于该算法非常简单，因此通常在现代显卡的固件或图形硬件中实现。

"Bresenham" 标签今天用于扩展或修改 Bresenham 原始算法的一系列算法。

更多细节可以查看维基百科：👉https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm


## 如何确定线段中像素点的位置？

下面以第一象限，斜率在 0 到 1 之间的线段展开讨论：

<img src="/res/img/post/03-tinyrenderer-learn-note-01/1.jpg" width=800>

<br>

<img src="/res/img/post/03-tinyrenderer-learn-note-01/2.jpg" width=800>

<br>
<br>

但是，如上所述，这仅适用于八分位数零，即从原点开始的线条，斜率在 0 和 1 之间，其中 x 每次迭代恰好增加 1，y 增加 0 或 1。

扩展后的完整解决方案的代码示例可以看 Lesson 1 第五次尝试的代码。

> 了解思路后相信 Lesson 1 可以轻松拿下了。

<br>
<br>

# Lesson 1: Bresenham’s Line Drawing Algorithm

第1课：Bresenham 的线条绘制算法

## 第一次尝试

简单的沿着线段迭代，在 x<sub>0</sub>, x<sub>1</sub> 和 y<sub>0</sub>, y<sub>1</sub> 之间线性插值

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    for (float t = 0.0; t < 1.0; t += 0.01)
    {
        int x = x0 + (x1 - x0) * t;
        int y = y0 + (y1 - y0) * t;
        image.set(x, y, color);
    }
}
```

![](/res/img/post/03-tinyrenderer-learn-note-01/1-1-1.png)

除了效率低下，这段代码的问题是常数的选择，如果我们取它等于 0.1，我们的线段看起来像这样：

![](/res/img/post/03-tinyrenderer-learn-note-01/1-1-2.png)

所以这里存在的主要问题是：需要手动选择参数控制间隙

<br>

## 第二次尝试

x 轴上的步长为 1 像素，然后对 y 轴进行线性插值，这样我们就不用担心间隙存在了

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    // x 每一步加 1
    for (int x=x0; x<=x1; x++)
    {
        // 在 0 到 1 的范围内得到从 x0 到 x1 的当前距离
        float t = (x-x0) / (float)(x1-x0);
        // 用 0 到 1 之间的值对 y 进行线性插值
        int y = y0*(1.0 - t) + y1*t;
        image.set(x, y, color);
    }
}
```

但是让我们进行对称性测试，绘制如下线条，会得到下面的结果：

```cpp
line(13, 20, 80, 40, image, white);
line(20, 13, 40, 80, image, red);
line(80, 40, 13, 20, image, red);
```

![](/res/img/post/03-tinyrenderer-learn-note-01/1-2-1.png)

结果是一条线是好的，第二条线有间隙，根本没有第三条线。注意，第一行和第三行（在代码中）用不同的颜色和不同的方向绘制同一条线（源点和目标点翻转）。我们已经看过白色的了，画得很好。我希望把白线的颜色改成红色，但是做不到。这是对对称性的测试：绘制线段的结果不应该取决于点的顺序：（a,b）线段应该与（b,a）线段完全相同。

<br>

## 第三次尝试

我们通过交换点来修复缺失的红线，使 x<sub>0</sub> 总是小于 x1

> 因为如果 x<sub>0</sub> < x<sub>1</sub> 就不会进入 for 循环的逻辑

然后如果直线的斜率大于 1 (也就是更陡峭) 我们交换两点的 x 和 y 的值，用更长的计算 t，然后如果计算 t 前有做交换，在最终绘制的时候再翻转回去

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    bool steep = false;
    if (std::abs(x0-x1) < std::abs(y0-y1))
    { // if the line is steep, we transpose the image
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if (x0>x1)
    { // make it left−to−right
        std::swap(x0, x1);
        std::swap(y0, y1);
    }
    for (int x=x0; x<=x1; x++)
    {
        float t = (x-x0) / (float)(x1-x0);
        int y = y0*(1.0 - t) + y1*t;
        if (steep)
        {
            image.set(y, x, color); // if transposed, de−transpose
        }
        else
        {
            image.set(x, y, color);
        }
    }
}
```

Holy cow! 天哪！

![](/res/img/post/03-tinyrenderer-learn-note-01/1-3-1.png)

<br>

## 第四次尝试 (直接看第五部分尝试)

> 可以先看第五部分尝试，这部分算法很少使用，并且第五部分的算法不用 float 类型

```shell
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 69.16      2.95     2.95  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 19.46      3.78     0.83 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
  8.91      4.16     0.38 207000000     0.00     0.00  TGAColor::TGAColor(TGAColor const&) 
  1.64      4.23     0.07        2    35.04    35.04  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
  0.94      4.27     0.04                             TGAImage::get(int, int)
```

tinyrenderer 的作者对程序进行测试，发现 10% 的时间花在复制颜色上。但是 70% 的时间都花在了调用 line() 上！这就是我们要优化的地方。

我们应该注意到每个除法都有相同的除数。让我们把它排除在循环之外。误差变量给出了从当前 (x, y) 像素到最佳直线的距离。每次误差大于一个像素时，我们增加 (或减少) y 一个像素，同时误差也减少一个像素。

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    bool steep = false;
    if (std::abs(x0-x1)<std::abs(y0-y1))
    {
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if (x0>x1)
    {
        std::swap(x0, x1);
        std::swap(y0, y1);
    }
    int dx = x1-x0;
    int dy = y1-y0;
    float derror = std::abs(dy / float(dx));
    float error = 0;
    int y = y0;
    for (int x=x0; x<=x1; x++)
    {
        if (steep)
        {
            image.set(y, x, color);
        }
        else
        {
            image.set(x, y, color);
        }
        error += derror;
        if (error > 0.5)
        {
            y += (y1 > y0 ? 1 : -1);
            error -= 1.0;
        }
    }
}
```

tinyrenderer 作者再次对程序进行测试：

```shell
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 38.79      0.93     0.93  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 37.54      1.83     0.90 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
 19.60      2.30     0.47 204000000     0.00     0.00  TGAColor::TGAColor(int, int) 
  2.09      2.35     0.05        2    25.03    25.03  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
  1.25      2.38     0.03                             TGAImage::get(int, int) 
```

<br>

## 第五次也是最后一次尝试

为什么我们需要浮点数？唯一的原因是在循环体中除以 dx 并与。5进行比较。我们可以通过用另一个变量替换误差变量来消除浮点数。我们叫它 error2，假设它等于 error * dx * 2。

先搞懂 `Bresenham 直线绘制算法` 可以更好的理解下面的代码：

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    bool steep = false;
    if (std::abs(x0-x1) < std::abs(y0-y1))
    {
        // if the line is steep, we transpose the image
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if (x0>x1)
    {
        // make it left−to−right
        std::swap(x0, x1);
        std::swap(y0, y1);
    }
    int dx = x1-x0;
    int dy = y1-y0;
    int derror2 = std::abs(dy)*2;
    int error2 = 0;
    int y = y0;
    for (int x=x0; x<=x1; x++)
    {
        if (steep)
        {
            // if transposed, de−transpose
            image.set(y, x, color);
        }
        else
        {
            image.set(x, y, color);
        }
        error2 += derror2;
        if (error2 > dx)
        {
            y += (y1 > y0 ? 1 : -1);
            error2 -= dx*2;
        }
    }
}
```

```shell
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 42.77      0.91     0.91 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
 30.08      1.55     0.64  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 21.62      2.01     0.46 204000000     0.00     0.00  TGAColor::TGAColor(int, int) 
  1.88      2.05     0.04        2    20.02    20.02  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
```

现在，在函数调用期间通过引用传递颜色来删除不必要的副本就足够了（或者只是启用编译标志 -O3），这样就完成了。代码中没有一个乘法或除法。执行时间从 2.95 秒减少到 0.64 秒。

<br>

## 线框渲染

所以现在我们准备好创建一个线渲染了。您可以在此处找到代码和测试模型的快照。我使用文件的 wavefront obj 格式来存储模型。渲染所需的只是从文件中读取以下类型的顶点数组：

```txt
v 0.608654 -0.568839 -0.416318
```

对该文件格式不熟悉的推荐看看维基百科：👉[Wavefront .obj file (wikipedia)](https://en.wikipedia.org/wiki/Wavefront_.obj_file)

例如下面的格式表示一个面的属性：

```txt
f 1193/1240/1193 1180/1227/1180 1179/1226/1179
```

我们感兴趣的是每个空格后的第一个数字。它是我们之前读过的数组中顶点的个数。因此，这条线表示由第 1193、1180 和 1179个顶点组成一个三角形。注意，在 obj 文件中，索引从 1 开始，这意味着您应该分别查找第 1192、1179 和 1178 个顶点。cpp文件包含一个简单的解析器。将以下循环写入我们的 main.cpp，瞧，我们的 wire 渲染器已准备就绪。

```cpp
for (int i = 0; i < model->nfaces(); i++)
{
    std::vector<int> face = model->face(i);
    for (int j=0; j<3; j++)
    {
        Vec3f v0 = model->vert(face[j]);
        Vec3f v1 = model->vert(face[(j+1)%3]);
        int x0 = (v0.x+1.)*width/2.;
        int y0 = (v0.y+1.)*height/2.;
        int x1 = (v1.x+1.)*width/2.;
        int y1 = (v1.y+1.)*height/2.;
        line(x0, y0, x1, y1, image, white);
    }
}
```

![](/res/img/post/03-tinyrenderer-learn-note-01/1-6-1.jpg)

<br>
<br>

# Lesson 2: Triangle rasterization and back face culling

第2课：三角形光栅化和背面剔除

## 填充三角形

Hi, everyone. It’s me.

### 老派的方法：扫描画线

学会了绘制线段，再用线段绘制三角形就比较简单了

```cpp
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color)
{
    line(t0, t1, image, color);
    line(t1, t2, image, color);
    line(t2, t0, image, color);
}

// ...

Vec2i t0[3] = {Vec2i(10, 70),   Vec2i(50, 160),  Vec2i(70, 80)};
Vec2i t1[3] = {Vec2i(180, 50),  Vec2i(150, 1),   Vec2i(70, 180)};
Vec2i t2[3] = {Vec2i(180, 150), Vec2i(120, 160), Vec2i(130, 180)};
triangle(t0[0], t0[1], t0[2], image, red);
triangle(t1[0], t1[1], t1[2], image, white);
triangle(t2[0], t2[1], t2[2], image, green);
```

![](/res/img/post/03-tinyrenderer-learn-note-01/2-1-1.png)

怎么填充呢？

可以用一种扫描画线的绘制方法，可以选择用水平线或者竖直线去填充这个三角形。

下面假定我们用水平线去填充，如果用水平线，我们需要知道水平线段的最左端点和最右端点

![](/res/img/post/03-tinyrenderer-learn-note-01/2-1-2.png)

有相似的性质我们可以知道，n 条线段左端点 x, y 满足

$$
\frac{x - x_2}{x_1 - x_2} = \frac{y - y_2}{y_1 - y_2}
$$

右端点可以看出当 y < y<sub>1</sub> 时在线段 t<sub>0</sub>t<sub>1</sub> 上， y > y<sub>1</sub> 时在线段 t<sub>2</sub>t<sub>1</sub> 上。

所以我们可以先绘制下半部分，再绘制上半部分：

```cpp
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color)
{
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!)
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int y=t0.y; y<=t1.y; y++)
    {
        // 这里就是公式了 对应带入进去即可
        int segment_height = t1.y-t0.y+1;
        float alpha = (float)(y-t0.y)/total_height;
        float beta  = (float)(y-t0.y)/segment_height; // be careful with divisions by zero
        Vec2i A = t0 + (t2-t0)*alpha;
        Vec2i B = t0 + (t1-t0)*beta;
        if (A.x>B.x) std::swap(A, B);
        for (int j=A.x; j<=B.x; j++)
        {
            image.set(j, y, color); // attention, due to int casts t0.y+i != A.y
        }
    }
    for (int y=t1.y; y<=t2.y; y++)
    {
        // 这里就是公式了 对应带入进去即可
        int segment_height =  t2.y-t1.y+1;
        float alpha = (float)(y-t0.y)/total_height;
        float beta  = (float)(y-t1.y)/segment_height; // be careful with divisions by zero
        Vec2i A = t0 + (t2-t0)*alpha;
        Vec2i B = t1 + (t2-t1)*beta;
        if (A.x>B.x) std::swap(A, B);
        for (int j=A.x; j<=B.x; j++)
        {
            image.set(j, y, color); // attention, due to int casts t0.y+i != A.y
        }
    }
}
```

![](/res/img/post/03-tinyrenderer-learn-note-01/2-1-3.png)

这可能就足够了，但我不喜欢看到相同的代码两次。这就是为什么我们会让它的可读性降低一点，但对修改/维护更方便：

```cpp
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color)
{
    if (t0.y==t1.y && t0.y==t2.y) return; // I dont care about degenerate triangles
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!)
    if (t0.y>t1.y) std::swap(t0, t1);
    if (t0.y>t2.y) std::swap(t0, t2);
    if (t1.y>t2.y) std::swap(t1, t2);
    int total_height = t2.y-t0.y;
    for (int i=0; i<total_height; i++)
    {
        bool second_half = i>t1.y-t0.y || t1.y==t0.y;
        int segment_height = second_half ? t2.y-t1.y : t1.y-t0.y;
        float alpha = (float)i/total_height;
        float beta  = (float)(i-(second_half ? t1.y-t0.y : 0))/segment_height; // be careful: with above conditions no division by zero here
        Vec2i A =               t0 + (t2-t0)*alpha;
        Vec2i B = second_half ? t1 + (t2-t1)*beta : t0 + (t1-t0)*beta;
        if (A.x>B.x) std::swap(A, B);
        for (int j=A.x; j<=B.x; j++)
        {
            image.set(j, t0.y+i, color); // attention, due to int casts t0.y+i != A.y
        }
    }
}
```

### 作者还有另一种方法

略

## 平面阴影渲染

我们已经知道如何用空三角形绘制模型。让我们用随机的颜色填充它们。这将帮助我们了解我们对三角形的编码填充的程度。这是代码：

```cpp
#include "model.h"

Model* model = nullptr;

int main(int argc, char** argv)
{
    if (2==argc)
    {
        model = new Model(argv[1]);
    }
    else
    {
        model = new Model("../obj/african_head.obj");
    }
    TGAImage image(width, height, TGAImage::RGB);

    for (int i=0; i<model->nfaces(); i++)
    {
        std::vector<int> face = model->face(i);
        Vec2i screen_coords[3];
        for (int j=0; j<3; j++)
        {
            Vec3f world_coords = model->vert(face[j]);
            screen_coords[j] = Vec2i((world_coords.x+1.)*width/2., (world_coords.y+1.)*height/2.);
        }
        triangle(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(rand()%255, rand()%255, rand()%255, 255));
    }

    image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
    image.write_tga_file("output.tga");
    return 0;
}
```

这很简单：就像以前一样，我们迭代所有三角形，将世界坐标转换为屏幕坐标并绘制三角形。我将在以下文章中提供各种坐标系统的详细描述。当前的图片看起来像这样：

![](/res/img/post/03-tinyrenderer-learn-note-01/2-2-1.jpg)

让我们摆脱这些小丑色，放一些照明。显而易见的是：“在相同的光强度下，当与光方向正交时，多边形最明亮地照亮。”

![](/res/img/post/03-tinyrenderer-learn-note-01/2-2-2.jpg)

![](/res/img/post/03-tinyrenderer-learn-note-01/2-2-3.jpg)

如果多边形平行于光向量，我们得到零照明。换句话说：照明强度等于光向量和给定三角形法线的标量积。这个三角形的法线可以简单地计算为它两边的叉乘。

```cpp
#include "model.h"

Model* model = nullptr;

int main(int argc, char** argv)
{
    if (2==argc)
    {
        model = new Model(argv[1]);
    }
    else
    {
        model = new Model("../obj/african_head.obj");
    }
    TGAImage image(width, height, TGAImage::RGB);

    Vec3f light_dir(0,0,-1); // define light_dir

    for (int i=0; i<model->nfaces(); i++)
    {
        std::vector<int> face = model->face(i);
        Vec2i screen_coords[3];
        Vec3f world_coords[3];
        for (int j=0; j<3; j++)
        {
            Vec3f v = model->vert(face[j]);
            screen_coords[j] = Vec2i((v.x+1.)*width/2., (v.y+1.)*height/2.);
            world_coords[j]  = v;
        }
        Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]);
        n.normalize();
        float intensity = n*light_dir;
        if (intensity>0)
        {
            triangle(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(intensity*255, intensity*255, intensity*255, 255));
        }
    }

    image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
    image.write_tga_file("output.tga");
    return 0;
}
```

但是点积可以是负的。这是什么意思？这意味着光来自多边形的后面。如果场景建模良好(通常是这种情况)，我们可以简单地抛弃这个三角形。这允许我们快速移除一些不可见的三角形。这被称为背面剔除。

![](/res/img/post/03-tinyrenderer-learn-note-01/2-2-4.jpg)

注意口腔的内腔在嘴唇的上方。这是因为我们对看不见的三角形进行了肮脏的剪裁：它只适用于凸形状。下次编码z缓冲区时，我们将去掉这个工件。

这是渲染的当前版本。你有没有发现我脸部的图像更细致了？好吧，我有点作弊：它有25万个三角形，而这个人造头部模型大约有1000个。但我的脸确实是用上面的代码渲染的。我向您保证，在接下来的文章中，我们将为这个图像添加更多的细节。

<br>
<br>

# Lesson 3: Hidden faces removal (z buffer)

第3课：删除隐藏的人脸（z缓冲区）

> 这一小节大部分解释我都写在代码上了，非常详细，相信足够看懂了！

从理论上讲，我们可以在不丢弃任何三角形的情况下绘制所有三角形。如果我们正确地 **_从后到前_** 进行正确的操作，则前面将删除后面的方面。它称为 **_画家的算法_**。不幸的是，它带有高计算成本：对于每个相机运动，我们都需要重新分配所有场景。然后是动态场景...这甚至不是主要问题。主要问题是，并非总是有可能确定正确的顺序。

> 画家算法可能解决这个问题，但是性能成本太大了，有时候图形 (例如三角形之间) 还会相交

所以我们的想法是，我们要跟踪每个像素在场景中的深度，然后我们就可以很容易地知道这个像素是在另一个像素的前面还是后面，如果在后面，我们就把它扔掉，如果在前面，我们就绘制它。

<br>

## 让我们尝试渲染一个简单的场景

想象一下一个由三个三角形组成的简单场景：相机看起来很最新，我们将彩色三角形投影到白色屏幕上：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-1-1.png)

渲染应该看起来像这样：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-1-2.png)

蓝色方面 - 它是在红色后面还是在红色的前面？画家算法在这里不起作用。可以将蓝色的面分为两个（一个在红色方面，一个后面）。然后，红色前面的一个要分为两分 - 一个在绿色三角形的前面，一个在后面...我认为您会遇到问题：在有数百万三角形的场景中，计算真的很昂贵。可以使用 BSP 树将其完成。顺便说一句，这种数据结构对于移动相机是恒定的，但确实很混乱。生活太短了，无法使它变得凌乱。

<br>
<br>

## 甚至更简单：让我们失去一个维度。Y-buffer !

让我们失去一段时间的尺寸，并沿着黄色的平面切割上述场景：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-2-1.png)

我的意思是，现在我们的场景是由三个线段（黄色平面和每个三角形的交点）组成的，最终渲染的宽度正常，但是1像素高：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-2-2.png)

```cpp
    { // just dumping the 2d scene (yay we have enough dimensions!)
        TGAImage scene(width, height, TGAImage::RGB);

        // scene "2d mesh"
        line(Vec2i(20, 34),   Vec2i(744, 400), scene, red);
        line(Vec2i(120, 434), Vec2i(444, 400), scene, green);
        line(Vec2i(330, 463), Vec2i(594, 200), scene, blue);

        // screen line
        line(Vec2i(10, 10), Vec2i(790, 10), scene, white);

        scene.flip_vertically(); // i want to have the origin at the left bottom corner of the image
        scene.write_tga_file("scene.tga");
    }
```

如果我们侧向看它，这就是我们的2D场景的样子：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-2-3.png)

让我们渲染它。回想一下渲染是 1 像素高度。在源代码中，我创建图像 16 像素高度，以便于在高分辨率屏幕上读取。 rasterize() 函数仅在图像渲染的第一行中写入

```cpp
        TGAImage render(width, 16, TGAImage::RGB);
        int ybuffer[width];
        for (int i=0; i<width; i++)
        {
            ybuffer[i] = std::numeric_limits<int>::min();
        }
        rasterize(Vec2i(20, 34),   Vec2i(744, 400), render, red,   ybuffer);
        rasterize(Vec2i(120, 434), Vec2i(444, 400), render, green, ybuffer);
        rasterize(Vec2i(330, 463), Vec2i(594, 200), render, blue,  ybuffer);
```

因此，我声明了一个具有维度 (width, 1) 的神奇数组 ybuffer。这个数组初始化为 -∞。然后我用这个数组和图像渲染作为参数调用 rasterize() 函数。函数是什么样的？

```cpp
/** 在图像上绘制一条线段，并更新深度缓冲区 ybuffer */
void rasterize(Vec2i p0, Vec2i p1, TGAImage &image, TGAColor color, int ybuffer[])
{
    if (p0.x>p1.x)
    {
        std::swap(p0, p1);
    }
    for (int x=p0.x; x<=p1.x; x++)
    {
        float t = (x-p0.x)/(float)(p1.x-p0.x);
        // y 是通过线性插值计算的当前 x 对应的 y 坐标，其中 +0.5 是为了四舍五入到最近的整数
        int y = p0.y*(1.-t) + p1.y*t + .5;
        if (ybuffer[x]<y)
        {
            ybuffer[x] = y;
            image.set(x, 0, color);
        }
    }
}
```

这确实是非常简单的：我迭代 p0.x 和 p1.x 之间的所有 x 坐标，并计算段的相应 y 坐标。然后，我使用当前的 X 索引检查了我们的数组 ybuffer 中的内容。如果当前的 y 值比 ybuffer 中的值更靠近相机，则我将其绘制在屏幕上并更新 ybuffer 。

我们会得到：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-2-4.png)

<br>
<br>

## 回到3D

### 质心坐标

计算质心坐标的实现：

```cpp
/**
 * @brief 计算质心坐标
 * @return 如果返回的坐标 (u,v,w) 满足 u,v,w >0，则点 P 在三角形内部
 */
Vec3f barycentric(Vec3f A, Vec3f B, Vec3f C, Vec3f P)
{
    Vec3f s[2];
    // 计算 [AC,AB,PA] 的 x 和 y 分量
    // s[0] 存储 x 分量的 AC,AB,PA
    // s[1] 存储 y 分量的 AC,AB,PA
    for (int i=2; i--; )
    {
        s[i][0] = C[i]-A[i]; // 向量 AC 的分量
        s[i][1] = B[i]-A[i]; // 向量 AB 的分量
        s[i][2] = A[i]-P[i]; // 向量 PA 的分量（注意负号）
    }

    // 这里 cross(s[0], s[1]) 相当于 AC 叉乘 AB，结果是 u 垂直于这两个向量
    // 它的 z 分量 u.z 表示三角形 ABC 所在平面的法向量的 z 分量
    // 叉积 |axb| = |a||b|sin(α)  值为 0 时角度为 0，即 ABC 三点共线
    // u.x = s[0].y · s[1].z - s[0].z · s[1].y
    // u.y = s[0].z · s[1].x - s[0].x · s[1].z
    // u.z = s[0].x · s[1].y - s[0].y · s[1].x
    Vec3f u = cross(s[0], s[1]);

    // 三点共线时，会导致 u[2] 为 0，此时返回 (-1,1,1)
    // 判断是否大于 0.01
    if (std::abs(u[2])>1e-2)
        //若 1-u-v，u，v全为大于 0 的数，表示点在三角形内部
        // P = (1-u-v)*A + u*B + v*C
        // u.x 和 u.y 是点 P 相对于三角形边 AB 和 AC 的贡献
        // 所以 u = u.y/u.z, v = u.x/u.z
        return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z);

    // 在这种情况下生成负坐标，它将被栅格化器丢弃
    return Vec3f(-1,1,1);
}
```

更多参考：👉https://en.wikipedia.org/wiki/Barycentric_coordinate_system

<br>

### 继续 3D

对于在2D屏幕上绘制，z缓冲区必须是二维的：

```cpp
int *zbuffer = new int[width*height];
```

<br>

### 深度测试

```cpp
void triangle(Vec3f *pts, float *zbuffer, TGAImage &image, TGAColor color)
{
    Vec2f bboxmin( std::numeric_limits<float>::max(),  std::numeric_limits<float>::max());
    Vec2f bboxmax(-std::numeric_limits<float>::max(), -std::numeric_limits<float>::max());
    // clamp 用于限制包围盒在图像范围内
    // clamp：表示图像的最大 x 和 y 坐标 (从 0 开始，所以需要减 1)。
    Vec2f clamp(image.get_width()-1, image.get_height()-1);
    for (int i=0; i<3; i++)
    {
        for (int j=0; j<2; j++)
        {
            // 计算三角形的包围盒
            // 保证不超出图像的左边界和上边界
            bboxmin[j] = std::max(0.f,      std::min(bboxmin[j], pts[i][j]));
            // 保证不超出图像的右边界和下边界
            bboxmax[j] = std::min(clamp[j], std::max(bboxmax[j], pts[i][j]));
        }
    }

    Vec3f P;

    // 遍历一个二维包围盒（Bounding Box）中的每一个像素点，并判断这些像素点是否位于给定的三角形内。
    // 如果像素点在三角形内，则计算其深度值（z 值），并进行深度测试（Z-Buffer 测试），
    // 最后更新帧缓冲区和深度缓冲区。

    // 遍历边框中的每一个像素点
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++)
    {
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++)
        {
            // 获取质心坐标
            Vec3f bc_screen  = barycentric(pts[0], pts[1], pts[2], P);
            // 质心坐标有一个负值，说明点在三角形外
            if (bc_screen.x<0 || bc_screen.y<0 || bc_screen.z<0) continue;
            P.z = 0;
            //计算 zbuffer，并且每个顶点的 z 值乘上对应的质心坐标分量
            // P.z = u.x*pts[0][2] + u.y*pts[1][2] + u.z*pts[2][2];
            // 其中 pts[0][2] 即 pts[0].z
            // 根据质心坐标 (u,v,w) 和三角形顶点的深度值 pts[i][2]，插值计算 P 的深度值：
            for (int i=0; i<3; i++)
                P.z += pts[i][2]*bc_screen[i];

            // 深度测试
            if (zbuffer[int(P.x+P.y*width)] < P.z)
            {
                zbuffer[int(P.x+P.y*width)] = P.z;
                image.set(P.x, P.y, color);
            }
        }
    }
}
```

最终代码：

```cpp
#include <vector>
#include <cmath>
#include <cstdlib>
#include <limits>
#include "tgaimage.h"
#include "model.h"
#include "geometry.h"

const TGAColor white = TGAColor(255, 255, 255, 255);
const TGAColor red   = TGAColor(255, 0,   0,   255);
Model *model = NULL;
const int width  = 800;
const int height = 800;

void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    bool steep = false;
    if (std::abs(x0-x1)<std::abs(y0-y1))
    {
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if (x0>x1)
    {
        std::swap(x0, x1);
        std::swap(y0, y1);
    }

    for (int x=x0; x<=x1; x++)
    {
        float t = (x-x0)/(float)(x1-x0);
        int y = y0*(1.-t) + y1*t;
        if (steep)
        {
            image.set(y, x, color);
        }
        else
        {
            image.set(x, y, color);
        }
    }
}

/**
 * @brief 计算质心坐标
 * @return 如果返回的坐标 (u,v,w) 满足 u,v,w >0，则点 P 在三角形内部
 */
Vec3f barycentric(Vec3f A, Vec3f B, Vec3f C, Vec3f P)
{
    Vec3f s[2];
    // 计算 [AC,AB,PA] 的 x 和 y 分量
    // s[0] 存储 x 分量的 AC,AB,PA
    // s[1] 存储 y 分量的 AC,AB,PA
    for (int i=2; i--; )
    {
        s[i][0] = C[i]-A[i]; // 向量 AC 的分量
        s[i][1] = B[i]-A[i]; // 向量 AB 的分量
        s[i][2] = A[i]-P[i]; // 向量 PA 的分量（注意负号）
    }

    // 这里 cross(s[0], s[1]) 相当于 AC 叉乘 AB，结果是 u 垂直于这两个向量
    // 它的 z 分量 u.z 表示三角形 ABC 所在平面的法向量的 z 分量
    // 叉积 |axb| = |a||b|sin(α)  值为 0 时角度为 0，即 ABC 三点共线
    // u.x = s[0].y · s[1].z - s[0].z · s[1].y
    // u.y = s[0].z · s[1].x - s[0].x · s[1].z
    // u.z = s[0].x · s[1].y - s[0].y · s[1].x
    Vec3f u = cross(s[0], s[1]);

    // 三点共线时，会导致 u[2] 为 0，此时返回 (-1,1,1)
    // 判断是否大于 0.01
    if (std::abs(u[2])>1e-2)
        //若 1-u-v，u，v全为大于 0 的数，表示点在三角形内部
        // P = (1-u-v)*A + u*B + v*C
        // u.x 和 u.y 是点 P 相对于三角形边 AB 和 AC 的贡献
        // 所以 u = u.y/u.z, v = u.x/u.z
        return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z);

    // 在这种情况下生成负坐标，它将被栅格化器丢弃
    return Vec3f(-1,1,1);
}

void triangle(Vec3f *pts, float *zbuffer, TGAImage &image, TGAColor color)
{
    Vec2f bboxmin( std::numeric_limits<float>::max(),  std::numeric_limits<float>::max());
    Vec2f bboxmax(-std::numeric_limits<float>::max(), -std::numeric_limits<float>::max());
    // clamp 用于限制包围盒在图像范围内
    // clamp：表示图像的最大 x 和 y 坐标 (从 0 开始，所以需要减 1)。
    Vec2f clamp(image.get_width()-1, image.get_height()-1);
    for (int i=0; i<3; i++)
    {
        for (int j=0; j<2; j++)
        {
            // 计算三角形的包围盒
            // 保证不超出图像的左边界和上边界
            bboxmin[j] = std::max(0.f,      std::min(bboxmin[j], pts[i][j]));
            // 保证不超出图像的右边界和下边界
            bboxmax[j] = std::min(clamp[j], std::max(bboxmax[j], pts[i][j]));
        }
    }

    Vec3f P;

    // 遍历一个二维包围盒（Bounding Box）中的每一个像素点，并判断这些像素点是否位于给定的三角形内。
    // 如果像素点在三角形内，则计算其深度值（z 值），并进行深度测试（Z-Buffer 测试），
    // 最后更新帧缓冲区和深度缓冲区。

    // 遍历边框中的每一个像素点
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++)
    {
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++)
        {
            // 获取质心坐标
            Vec3f bc_screen  = barycentric(pts[0], pts[1], pts[2], P);
            // 质心坐标有一个负值，说明点在三角形外
            if (bc_screen.x<0 || bc_screen.y<0 || bc_screen.z<0) continue;
            P.z = 0;
            //计算 zbuffer，并且每个顶点的 z 值乘上对应的质心坐标分量
            // P.z = u.x*pts[0][2] + u.y*pts[1][2] + u.z*pts[2][2];
            // 其中 pts[0][2] 即 pts[0].z
            // 根据质心坐标 (u,v,w) 和三角形顶点的深度值 pts[i][2]，插值计算 P 的深度值：
            for (int i=0; i<3; i++)
                P.z += pts[i][2]*bc_screen[i];

            // 深度测试
            if (zbuffer[int(P.x+P.y*width)] < P.z)
            {
                zbuffer[int(P.x+P.y*width)] = P.z;
                image.set(P.x, P.y, color);
            }
        }
    }
}

Vec3f world2screen(Vec3f v)
{
    return Vec3f(int((v.x+1.)*width/2.+.5), int((v.y+1.)*height/2.+.5), v.z);
}

int main(int argc, char** argv)
{
    if (2==argc)
    {
        model = new Model(argv[1]);
    }
    else
    {
        model = new Model("../../res/obj/african_head/african_head.obj");
    }

    float *zbuffer = new float[width*height];
    for (int i=width*height; i--; zbuffer[i] = -std::numeric_limits<float>::max());

    TGAImage image(width, height, TGAImage::RGB);

    Vec3f light_dir(0,0,-1); // define light_dir

    for (int i=0; i<model->nfaces(); i++)
    {
        std::vector<int> face = model->face(i);
        Vec3f world_coords[3];
        for (int j=0; j<3; j++)
        {
            Vec3f v = model->vert(face[j]);
            world_coords[j]  = v;
        }
        Vec3f A = world_coords[2] - world_coords[0];
        Vec3f B = world_coords[1] - world_coords[0];
        // 叉积求面的法向量
        Vec3f n = Vec3f(A.y*B.z - A.z*B.y, A.z*B.x - A.x*B.z, A.x*B.y - A.y*B.x);

        n.normalize();
        // 计算光线强度
        float intensity = n*light_dir;
        if (intensity>0)
        {
            Vec3f pts[3];
            for (int i=0; i<3; i++) pts[i] = world2screen(model->vert(face[i]));
            triangle(pts, zbuffer, image, TGAColor(intensity*255, intensity*255, intensity*255, 255));
        }
    }

    image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
    image.write_tga_file("output.tga");
    delete model;
    return 0;
}

```

最后运行程序：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-3-1.jpg)

<br>
<br>

## Okay, we just interpolated the z-values. What else can we do?

接下来我们给我们的非洲老哥加上纹理！

为了添加纹理颜色，我们首先需要读入纹理坐标，采样纹理颜色。

所以我们首先需要修改 `model.h` 以及 `model.cpp`：

`model.h`：此处做出的修改如下：

1. 每个小三角形面片的数据类型为 `std::vector<Vec3i>`，每个三角形由三个 `Vec3i` 组成，分别代表三个顶点，每个顶点存储三个整数值，分别对应 顶点索引 / 纹理坐标索引 / 法线向量索引;

2. 增加纹理坐标、法线向量、漫反射贴图这些成员变量，并对应增加一些成员函数；

```cpp
#ifndef __MODEL_H__
#define __MODEL_H__

#include <vector>
#include "geometry.h"
#include "tgaimage.h"

class Model {
private:
	std::vector<Vec3f> verts_;
	std::vector<std::vector<Vec3i> > faces_;
	std::vector<Vec3f> norms_;
	std::vector<Vec2f> uvs_;

	TGAImage diffuseMap_;
	void load_texture(std::string filename, const char* suffix, TGAImage& img);
public:
	Model(const char *filename);
	~Model();
	int nverts();
	int nfaces();
	Vec3f vert(int i);
	std::vector<int> face(int idx);
	Vec2i uv(int iface, int nvert);
	TGAColor diffuse(Vec2i uv);
};

#endif //__MODEL_H__

```

`model.cpp`：针对我们对 .h 文件做出的修改，对 .cpp 文件的实现也做出修改。

```cpp
#include <iostream>
#include <string>
#include <fstream>
#include <sstream>
#include <vector>
#include "model.h"

Model::Model(const char *filename) : verts_(), faces_() {
    std::ifstream in;
    in.open (filename, std::ifstream::in);
    if (in.fail()) return;
    std::string line;
    while (!in.eof()) {
        std::getline(in, line);
        std::istringstream iss(line.c_str());
        char trash;
        if (!line.compare(0, 2, "v ")) {
            iss >> trash;
            Vec3f v;
            for (int i=0;i<3;i++) iss >> v[i];
            verts_.push_back(v);
        }
        else if (!line.compare(0, 3, "vt ")) {
            iss >> trash >> trash;
            Vec2f uv;
            for (int i=0;i<2;i++) iss >> uv[i];
            uvs_.push_back(uv);
        }
        else if (!line.compare(0, 3, "vn ")) {
            iss >> trash >> trash;
            Vec3f normal;
            for (int i=0;i<3;i++) iss >> normal[i];
            norms_.push_back(normal);
        }
        else if (!line.compare(0, 2, "f ")) {
            std::vector<Vec3i> f;
            Vec3i tmp;
            iss >> trash;
            while (iss >> tmp[0] >> trash >> tmp[1] >> trash >> tmp[2]) {
                // in wavefront obj all indices start at 1, not zero
                for (int i = 0; i < 3; i++) tmp[i]--;
                f.push_back(tmp);
            }
            faces_.push_back(f);
        }
    }
    std::cerr << "# v# " << verts_.size() << " f# "  << faces_.size() << " vt# " << uvs_.size() << " vn# " << norms_.size() << std::endl;
    // 加载纹理
    load_texture(filename, "_diffuse.tga", diffuseMap_);
}

Model::~Model() {
}

int Model::nverts() {
    return (int)verts_.size();
}

int Model::nfaces() {
    return (int)faces_.size();
}

std::vector<int> Model::face(int idx) {
    std::vector<int> face;
    std::vector<Vec3i> tmp = faces_[idx];
    for (int i = 0; i < tmp.size(); i++)
        face.push_back(tmp[i][0]);
    return face;
}

Vec3f Model::vert(int i) {
    return verts_[i];
}

void Model::load_texture(std::string filename, const char* suffix, TGAImage& img)
{
    std::string texfile(filename);
    size_t dot = texfile.find_last_of(".");
    if (dot != std::string::npos) {
        texfile = texfile.substr(0, dot) + std::string(suffix);
        std::cerr << "texture file " << texfile << " loading " << (img.read_tga_file(texfile.c_str()) ? "ok" : "failed") << std::endl;
        img.flip_vertically();
    }
}

TGAColor Model::diffuse(Vec2i uv)
{
    return diffuseMap_.get(uv.x, uv.y);
}

Vec2i Model::uv(int iface, int nvert)
{
    int idx = faces_[iface][nvert][1];
    return Vec2i(uvs_[idx].x * diffuseMap_.get_width(), uvs_[idx].y * diffuseMap_.get_height());
}

```

接下来我们需要在 `main.cpp` 中做出如下修改：

triangle 函数：

```cpp
void triangle(Vec3f *pts, Vec2i* uvs, float *zbuffer, TGAImage &image, float intensity)
{
    Vec2f bboxmin( std::numeric_limits<float>::max(),  std::numeric_limits<float>::max());
    Vec2f bboxmax(-std::numeric_limits<float>::max(), -std::numeric_limits<float>::max());
    // clamp 用于限制包围盒在图像范围内
    // clamp：表示图像的最大 x 和 y 坐标 (从 0 开始，所以需要减 1)。
    Vec2f clamp(image.get_width()-1, image.get_height()-1);
    for (int i=0; i<3; i++)
    {
        for (int j=0; j<2; j++)
        {
            // 计算三角形的包围盒
            // 保证不超出图像的左边界和上边界
            bboxmin[j] = std::max(0.f,      std::min(bboxmin[j], pts[i][j]));
            // 保证不超出图像的右边界和下边界
            bboxmax[j] = std::min(clamp[j], std::max(bboxmax[j], pts[i][j]));
        }
    }

    Vec3f P;
    Vec2i uvP;

    // 遍历一个二维包围盒（Bounding Box）中的每一个像素点，并判断这些像素点是否位于给定的三角形内。
    // 如果像素点在三角形内，则计算其深度值（z 值），并进行深度测试（Z-Buffer 测试），
    // 最后更新帧缓冲区和深度缓冲区。

    // 遍历边框中的每一个像素点
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++)
    {
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++)
        {
            // 获取质心坐标
            Vec3f bc_screen  = barycentric(pts[0], pts[1], pts[2], P);
            // 质心坐标有一个负值，说明点在三角形外
            if (bc_screen.x<0 || bc_screen.y<0 || bc_screen.z<0) continue;
            P.z = 0;
            //计算 zbuffer，并且每个顶点的 z 值乘上对应的质心坐标分量
            // P.z = u.x*pts[0][2] + u.y*pts[1][2] + u.z*pts[2][2];
            // 其中 pts[0][2] 即 pts[0].z
            // 根据质心坐标 (u,v,w) 和三角形顶点的深度值 pts[i][2]，插值计算 P 的深度值：
            for (int i=0; i<3; i++)
                P.z += pts[i][2]*bc_screen[i];

            // 获取该像素点的 uv 坐标
            uvP = uvs[0] * bc_screen.x + uvs[1] * bc_screen.y + uvs[2] * bc_screen.z;

            // 深度测试
            if (zbuffer[int(P.x+P.y*width)] < P.z)
            {
                zbuffer[int(P.x+P.y*width)] = P.z;
                TGAColor texColor = model->diffuse(uvP);
                image.set(P.x, P.y, texColor * intensity);
            }
        }
    }
}
```

main 函数：

```cpp
int main(int argc, char** argv)
{
    if (2==argc)
    {
        model = new Model(argv[1]);
    }
    else
    {
        model = new Model("../../res/obj/african_head/african_head.obj");
    }

    float *zbuffer = new float[width*height];
    for (int i=width*height; i--; zbuffer[i] = -std::numeric_limits<float>::max());

    TGAImage image(width, height, TGAImage::RGB);

    Vec3f light_dir(0,0,-1); // define light_dir

    for (int i=0; i<model->nfaces(); i++)
    {
        std::vector<int> face = model->face(i);
        Vec3f world_coords[3];
        for (int j=0; j<3; j++)
        {
            Vec3f v = model->vert(face[j]);
            world_coords[j]  = v;
        }
        Vec3f A = world_coords[2] - world_coords[0];
        Vec3f B = world_coords[1] - world_coords[0];
        // 叉积求面的法向量
        Vec3f n = Vec3f(A.y*B.z - A.z*B.y, A.z*B.x - A.x*B.z, A.x*B.y - A.y*B.x);

        n.normalize();
        // 计算光线强度
        float intensity = n*light_dir;
        if (intensity>0)
        {
            Vec3f pts[3];
            for (int i=0; i<3; i++) pts[i] = world2screen(model->vert(face[i]));

            // 获取 uv
            Vec2i uv[3];
            for (int j = 0; j < 3; j++) uv[j] = model->uv(i, j);
            triangle(pts, uv, zbuffer, image, intensity);
        }
    }

    image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
    image.write_tga_file("output.tga");
    delete model;
    return 0;
}
```

结果如下：

![](/res/img/post/03-tinyrenderer-learn-note-01/3-4-1.jpg)

<br>
<br>

# 参考

[1] 👉[Github tinyrenderer wiki (原作者 wiki)](https://github.com/ssloy/tinyrenderer/wiki)
[2] 👉[Bresenham's line algorithm (wikipedia)](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm)
[3] 👉[Wavefront .obj file (wikipedia)](https://en.wikipedia.org/wiki/Wavefront_.obj_file)
[4] 👉[Barycentric coordinate (wikipedia)](https://en.wikipedia.org/wiki/Barycentric_coordinate_system)
[5] 👉[从零构建光栅器，tinyrenderer笔记（上）](https://zhuanlan.zhihu.com/p/399056546)
[6] 👉[计算机图形学补充1：重心坐标(barycentric coordinates)详解及其作用](https://zhuanlan.zhihu.com/p/144360079)
[7] 👉[Lesson 3: Hidden faces removal (z buffer)](https://github.com/ssloy/tinyrenderer/wiki/Lesson-3:-Hidden-faces-removal-(z-buffer))
[8] 👉[TinyRenderer从零实现（四）：lesson 3 z-buffer实现与纹理映射](https://zhuanlan.zhihu.com/p/523503467)

<br>

The end.

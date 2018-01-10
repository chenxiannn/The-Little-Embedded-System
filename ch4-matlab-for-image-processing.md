#### **1.先聊聊基于模型的设计**

可以先看看知乎这篇文章《[基于模型设计——电力电子的利器](https://zhuanlan.zhihu.com/p/23149544)》

最开始我们做小的系统，自己想怎么码就怎么码，想怎么命名就怎么命名，因为系统小，不管怎么折腾复杂度都可控，但是随着代码量的增加，我们开始切分函数，然后切分模块，（参见ch1），这时候我们开始考虑模块化设计，考虑模块的封装与耦合，尽量高内聚低耦合，随着系统复杂度进一步增加，我们开始几个人协同开发，考虑分层，考虑应用层，系统层，驱动层，数据服务是不是要独立出来，通讯是不是要单独出来，是裸跑，RTOS还是Linux，接口如何划分和设计，这时候就开始系统级地去进行软件设计，实现尽可能正交的系统，减少冗余代码。随着系统进一步复杂，代码量和复杂度都在不断攀升，这时候又有什么应对措施呢？？

还有一个问题，我们始终避免不了，就是嵌入式系统的测试与验证，如果按照瀑布模型走的话，等所有代码都完成了，再进行验证测试的话，如果一旦出现问题，再返工重新设计，那不拖延项目才怪呢，所以我们尽可能的将验证测试提前，越早期发现错误，那风险也就越低。于是尝试喷泉模型，在每一个阶段都能进行验证测试，但是嵌入式系统软件和一般软件系统不太一样，因为要与实物配合才能进行测试验证。往往硬件还没出来，我们都无法进行软件的调试测试，然后整个项目就卡在这里，有没有什么好的解决方案呢？

其实，嵌入式系统里，有一部分是纯逻辑与控制算法，这是系统的核心，大量的精力应该放在这里，但是实际中我们更多的时间被硬件接口，驱动，RTOS所累，最后对控制模型反而有点心有余而力不足。那是不是有让我们更专注于模型设计与控制算法的设计方法呢，以此提高我们的工作效率，更专注于核心。

正是基于以上几点，MBD（Model Based Design-基于模型的设计）闪亮登场，于是Matlab/Simulink进入了我们的工具箱，不过这里面分三种玩法：

* 土鳖的玩法：Simulink搭建模型与算法，验证测试通过后，然后再徒手实现C代码
* 折中的玩法：利用mexfunction写C的控制算法，Simulink模型配合mex构建的控制算法，仿真验证
* 高级的玩法：Simulink搭建模型与算法，验证测试通过后，自动生成C代码（高级是高级，但是生成的代码没法看）

考虑到代码的移植以及可读性，我们采用的是折中玩法，下面我以智能车的图像处理来简单介绍一下。

**最低效率的方式（比土鳖还土鳖）**

徒手写图像处理的C代码下载到车里，放到赛道上跑两圈，如果好了一把成，但如果出了问题，只能抱着笔记本，连着J-Link调试，趴在那里，一点点查找程序错误，这中间哪怕一个很小的bug问题，都会消耗极大的体力和脑力，因为不受控的因素太多了。

**改进版的方式（土鳖玩法）**

先大量采集赛道图片，覆盖90%以上路况的情况，然后在Matlab里对这些图片进行处理，实现图像处理算法，等验证通过后，再把Matlab徒手翻译为C代码，然后上赛道测试，依然会出现部分bug，但是80%的低级的原则性的bug在Matlab验证阶段已经被消灭了。

**目前要做的方式（折中玩法）**

在改进版中，matlab中验证的是matlab代码，车上跑的是C代码，由于这中间存在人工转化的过程，意味着依然可能会引入未知的错误。为何不直接在matlab里直接验证我的C代码呢？对了，这就是交叉编译，在matlab里直接调用C语言，OK，这样就能保证最终车里跑的代码，是在matlab里最终验证通过的了，Yeah。下面我们就跟着老司机一起开车喽。

#### 2**.交叉编译环境搭建**

Matlab 的软件版本推荐2011a（我比较钟爱老版本，哈哈，因为占地小）

C&C++编译器，推荐VS2010

第一步，先装好Matlab 2011a和VS2010（大家度娘解决吧）。

第二步，在Matlab下配置C&C++交叉编译器。

1.在matlab的工作台中输入mex–setup命令（mex就是matlab支持交叉编译的工具）

![](/assets/EmbeddedSystem_S4_P0.png)

2.回车后，会输出一段话，最后是“Would you  like mex to locate installed compilers\[y\]/n”是问你确定要为mex配置编译器？请输入y表示确定。

![](/assets/EmbeddedSystem_S4_P1.png)

3.此时，会显示matlab搜索到已经安装好的VS2010，你要在Compiler：后面输入相应的选择序号1，回车.

![](/assets/EmbeddedSystem_S4_P2.png)

4.在上图里，显示了你配置的编译器，最后需要输入y确认此配置，这样就OK了。

![](/assets/EmbeddedSystem_S4_P3.png)

比如我们实现最简单的y=a+b这样一个加法操作。

用matlab的函数function如何做呢？看下面：

```
function y=add(a,b)
    y=a+b;
```

将上面的代码保存为add.m，这样在matlab的工作台就可以调用这个函数了，比如：

![](/assets/EmbeddedSystem_S4_P4.png)

那这样一个简单的加法，我用C语言怎么实现呢？也很简单

```
double  add(double  a, double  b)
{
    return a+b;
}
```

既然我实现了C语言函数，那在matlab里怎么样调用这个C函数呢？这时候，我们前面配置好的mex交叉编译工具就上场了。

看下面这段代码，看不懂没关系，后面会一一解释的。

```
void mexFunction ( int nlhs, mxArray *plhs[], int nrhs, const mxArray *prhs[] )
{
    double *Y;
    double A, B;

    //输入接口绑定
    A = *(mxGetPr(prhs[0]));
    B = *(mxGetPr(prhs[1]));

    //输出接口绑定
    plhs[0] = mxCreateDoubleMatrix(1, 1, mxREAL); 
    Y = mxGetPr(plhs[0]);

    //做你该做的事
    *Y = add(A, B);
}
```

将上面的这段代码，创建为了new\_add.c，然后在工作台上输入mex new\_add.c命令，编译无错，就可以使用了。

![](/assets/EmbeddedSystem_S4_P5.png)

我们到底做了什么？其实，你只是把这个函数需要的两个参数a和b从matlab倒腾到C语言里面，进行了相应运算之后，再把输出结果从C语言里面倒腾到Matlab里而已。

不要怕麻烦，为了不熬夜调车，为了不做码农，刚开始势必会麻烦一点，等熟练了就好了，等你的代码到2000行或者更多的时候，你依然可以有时间优哉游哉地玩耍。

其实Matlab的大侠们，已经给这个倒腾过程，建立了一个专门的接口函数叫mexFunction，这个函数有四个参数分别为：

```
int nlhs                  输出变量个数
mxArray *plhs[]           输出变量指针数组
int nrhs                  输入变量个数
const mxArray *prhs[]     输入变量的指针数组
```

比如看我们上面的例子，当我调用new\_add\(3,4\)时，

```
int     nlhs              输出变量个数为1
mxArray *plhs[]           输出变量指针数组，plhs[0]对应求和结果y的变量地址
int     nrhs              输入变量个数为2
const  mxArray *prhs[]    输入变量的指针数组有两个，prhs[0]为参数a=3对应的地址，prhs[1]为参数b=4对应的地址。

//接下来就是把输入的两个参数读取到C变量里暂存，mxGetPr是获取数组地址，*就是获取地址里的内容。
//输入接口绑定
A = *(mxGetPr(prhs[0]));
B = *(mxGetPr(prhs[1]));

//再下面是输出接口的绑定，输出未分配存储空间，所以必须先申请存储空间，用mxCreateDoubleMatrix，然后用C指针变量指向这个地址。
//输出接口绑定
plhs[0] = mxCreateDoubleMatrix(1, 1, mxREAL); 
Y = mxGetPr(plhs[0]);

//然后就是你要做的操作，调用add，实现加操作
*Y = add(A, B);
```

有几点注意事项：

* matlab默认数据类型为double，如果你直接调用的接口数据的话，你的C语言声明的类型必须与之对应，否则Matlab会挂掉。
* 像我们图像一般都是0-255单字节的灰度或者二值化图，那应该怎么办呢？你要在matlab下将二维数组转化为uint8类型，然后再传给mexFunction接口，同时C语言中，一定要用uint8\*类型的指针去操作。
* Matlab中矩阵的排列与C语言中的排列不太一样，这一点也要注意，matlab的存储是按照列来顺序存储，不是C语言中的按照行顺序存储，如图6所示。
* C的数组起始是0，matlab的话是1，这点一定要注意。

* 交叉编译不好的地方，就是不能进行单步调试，那出了问题，怎么办呢？用mexPrintf函数，与你用C里面的printf一样。

![](/assets/EmbeddedSystem_S4_P6.png)

图6.Matlab矩阵顺序图

这里附上源代码

```
#include "mex.h"

double add(double a,double b)
{
    return (a+b);
}

void mexFunction ( int nlhs, mxArray *plhs[], int nrhs, const mxArray *prhs[] )
{
    double *Y;
    double A, B;

    A = *(mxGetPr(prhs[0]));
    B = *(mxGetPr(prhs[1]));


    plhs[0] = mxCreateDoubleMatrix(1, 1, mxREAL); 
    Y = mxGetPr(plhs[0]);

    *Y = add(A, B);
}
```

#### 3.智能车比赛图像处理

图像处理模块的结构图如图7所示，其中图像处理部分主要是imProc完成，与Matlab的mex接口由imCar完成。

![](/assets/EmbeddedSystem_S1_P2.png)

图7.图像处理模块结构图

整个图像处理模块imProc的内部结构如图8所示，输入信号为图像数据Image\_Data（用于寻找中线），Speed和寻找到的中线接合起来用于计算方向偏差，之所以要跟速度相关，因为车速快了的话，需要用更远的图像信息去计算方向偏差，加大提前量，近了则用更近的图像数据去计算，最终计算的中线偏差有三个值：

* gDir\_Near：近距离方向偏差，暂时未使用
* gDir\_Mid：中距离方向偏差，主要用于转向PD控制
* gDir\_Far：远距离方向偏差，主要用于识别入弯和出弯，提前进行加减速控制

![](/assets/EmbeddedSystem_S4_P7.png)

图8.imProc模块图

整个代码实现了一个基本的寻线处理和计算中线偏差的思路，具体过程如下：

1. 判断有没有出界，如果出界，则不做处理，保持原来的方向偏差不变，否则开始寻找新的中线
2. 逐行扫，先寻找中间位置，然后向左右寻找左右边界
3. 根据左右边界计算左右边界的斜率变化，然后递推得到最终的左右边界值
4. 根据左右边界，计算中线
5. 将中线做均值滤波
6. 滤波后的中线，映射到实际的物理坐标上（单位为cm）
7. 根据速度，计算三个中线偏差值

imProc的几个函数的功能：

* int Graph\_JudgeOut\(void\)：判断是否出界
* void Graph\_FindMidLine\(void\)：寻找中线
* void Graph\_AverageMBound\(void\)：均值滤波函数
* void Graph\_Cam2Real\_BoundM\(void\)：将中线映射到真是物理坐标
* int Graph\_Real2Cam\(int D\)：将真实距离映射到图像位置
* int Graph\_Cam2Real\(int H\)：将图像位置映射到真实距离

* void Graph\_Calculate\_Dir\(int Speed\)：计算方向偏差

图像处理好之后，下一步就是实现Matlab结合C编程的接口imCar，代码如下，

```
#include "mex.h"
#include "imProc.h"
#include "imType.h"

imUINT8  Image_Data[CAMERA_H][CAMERA_W];
extern imUINT8  Image_DataF[CAMERA_H][CAMERA_W];
extern imINT32  gDir_Near;
extern imINT32  gDir_Mid;
extern imINT32  gDir_Far;
extern imINT16  HBoundL[CAMERA_H];
extern imINT16  HBoundR[CAMERA_H];
extern imINT16  HBoundM[CAMERA_H];
extern imINT16  HBoundM_F[CAMERA_H];
extern imINT16  HBoundM_REAL[CAM_MAX_LENGTH_CM+1];


void mexFunction ( int nlhs, mxArray *plhs[], int nrhs, const mxArray *prhs[] )
{
    imUINT8 *imIn;
    imUINT8 *imOut;
    int H,W;
    imINT16 *bound;
    imINT32 *dir;
    imINT32 CarSpeed;

    imIn=mxGetPr(prhs[0]);
    for(H=0;H<CAMERA_H;H++)
    {    
        for(W=0;W<CAMERA_W;W++)
        {
        Image_Data[H][W]=imIn[H*CAMERA_W+W];
            //mexPrintf("%d ",Image_Data[H][W]);
        }
    }
    CarSpeed = *mxGetPr(prhs[1]);
    ControlParam_Init();
    Graph_FindMidLine();
    Graph_Calculate_Dir(CarSpeed);
    plhs[0]=mxCreateNumericMatrix(CAMERA_H,1,mxINT16_CLASS,mxREAL);
    plhs[1]=mxCreateNumericMatrix(CAMERA_H,1,mxINT16_CLASS,mxREAL);
    plhs[2]=mxCreateNumericMatrix(CAMERA_H,1,mxINT16_CLASS,mxREAL);
    plhs[3]=mxCreateNumericMatrix(3,1,mxINT32_CLASS,mxREAL);
    plhs[4]=mxCreateNumericMatrix(CAMERA_W,CAMERA_H,mxUINT8_CLASS,mxREAL);
    plhs[5]=mxCreateNumericMatrix(CAMERA_H,1,mxINT16_CLASS,mxREAL);
    plhs[6]=mxCreateNumericMatrix(CAM_MAX_LENGTH_CM+1,1,mxINT16_CLASS,mxREAL);

    bound=mxGetPr(plhs[0]);
    for(H=0;H<CAMERA_H;H++)
    {    
        bound[H]=HBoundL[H];
    }
    bound=mxGetPr(plhs[1]);
    for(H=0;H<CAMERA_H;H++)
    {    
        bound[H]=HBoundR[H];
    }
    bound=mxGetPr(plhs[2]);
    for(H=0;H<CAMERA_H;H++)
    {    
        bound[H]=HBoundM_F[H];
    }
    dir=mxGetPr(plhs[3]);
    dir[0]=gDir_Near;
    dir[1]=gDir_Mid;
    dir[2]=gDir_Far;
    imOut=mxGetPr(plhs[4]);
    for(H=0;H<CAMERA_H;H++)
    {
       for(W=0;W<CAMERA_W;W++)
       {
           imOut[H*CAMERA_W+W]=Image_Data[H][W];
       }
    }

    bound=mxGetPr(plhs[5]);
    for(H=0;H<CAMERA_H;H++)
    {    
        bound[H]=HBoundM_F[H];
    }
    bound=mxGetPr(plhs[6]);
    for(H=0;H<CAM_MAX_LENGTH_CM;H++)
    {    
        bound[H]=HBoundM_REAL[H];
    }
}
```

matlab代码调用接口函数的代码如下：

```
clc;
clear mex
mex -I"../ControlLib/Inc" ...,
    imCar.c ...,
    imProc.c ...,
    imCom.c ...,
    ../ControlLib/ControlParam.c
CarSpeed=200;%单位为cm/s
CAMERA_W=160;
CAMERA_H=120;
for i=1:127
    try
        imfilename=strcat('.\Image_txt\Imag',int2str(i),'.txt');%输入图片
        svfilename=strcat('.\Image_txt\solve\Imag',int2str(i),'.bmp');%输出图片
        %img=uint8(not(imread(imfilename))*255)';%加载BMP格式图片
        img=uint8(load(imfilename))';      %加载txt文本格式图片
        [W H]=size(img);
        if W ~=CAMERA_W && H~= CAMERA_H
            continue
        end
        [L R M  dir imOut M_F M_Real]=imCar(img,CarSpeed);
        imshow(imOut) 
        hold on
        plot(1:1:CAMERA_H,[L R M],'-r')
        saveas(gcf,svfilename)
        close all
        clear mex
    catch e
        e
        continue
    end
end
```

imCar一共返回7个变量，分别代表的含义是：

* L：左边界
* M：中线
* R：右边界
* dir：\[gDir\_Near gDir\_Mid  gDir\_Far\]
* imOut:输入图像重新返回
* M\_F：中线滤波后的结果
* M\_Real：中线滤波后，映射到实际距离

这里要说一下，由于imProc用到了ControParam模块的配置参数，来实现方向偏差的计算，所以mex的命令如下：

```
mex -I"../ControlLib/Inc" ...,%包含ControlParam.h
    imCar.c ...,
    imProc.c ...,
    imCom.c ...,
    ../ControlLib/ControlParam.c%编译ControlParam.c文件
```

将matlab的工作目录设置为Graphic，然后运行Compile.m，默认选择的是txt文本图像（Image\_txt文件夹），1分钟之后，127张图像就全部处理结束啦（在Image\_txt下的solve文件下），速度是不是很快呀，哈哈哈，如图10所示。然后你就可以针对不同的路况，去优化算法，立刻就可以在Matlab上验证，知道所有的路况全部验证通过之后，再把代码烧到单片机里，进行真实赛道测试。

， ![](/assets/EmbeddedSystem_S4_P10.png)、

图10.处理结果

这里要提一下，Graphic下目前有两个保存图像的文件夹，分别为Image\__txt_和Image\_bmp\_，\_Image\_Txt是我们用串口再赛道上每隔10cm采集的部分图像，Image\_bmp是山外自带的bmp格式图像。大家可以根据自己情况自由选择图片格式。

如果要处理Image\_bmp文件夹下的图像请将compile文件修改为如下：

```
clc;
clear mex
mex -I"../ControlLib/Inc" ...,
    imCar.c ...,
    imProc.c ...,
    imCom.c ...,
    ../ControlLib/ControlParam.c
CarSpeed=200;%单位为cm/s
CAMERA_W=160;
CAMERA_H=120;
for i=1:1000
    try
        imfilename=strcat('.\Image_bmp\fire',int2str(i),'.bmp');%输入图片
        svfilename=strcat('.\Image_bmp\solve\fire',int2str(i),'.bmp');%输出图片
        img=uint8(not(imread(imfilename))*255)';%加载BMP格式图片
        %img=uint8(load(imfilename))';      %加载txt文本格式图片
        [W H]=size(img);
        if W ~=CAMERA_W && H~= CAMERA_H
            continue
        end
        [L R M  dir imOut M_F M_Real]=imCar(img,CarSpeed);
        imshow(imOut) 
        hold on
        plot(1:1:CAMERA_H,[L R M],'-r')
        saveas(gcf,svfilename)
        close all
        clear mex
    catch e
        e
        continue
    end
end
```

这一小节，中间略掉了很多细节但是行文依然比较长，希望能够帮助到大家。

相关代码已经上传到[Github](https://github.com/chenxiannn/The-Little-Embedded-System)，如果对你有帮助的话，可以点个赞支持一下在下。


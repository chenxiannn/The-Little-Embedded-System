#### 1.分层与模块化

说起分层，让我想起了大学刚毕业去的那家小公司，当时自己维护空调控制板，代码是前辈做的，功能相当棒，但是当我看到代码的那一刻，差点吐血，因为整个代码就两个文件shuikongtiao.c和shuikongtiao.h，里面的变量和函数定义全部用汉语拼音，哈哈哈，一万只草泥马从心中飞过。我擦，终于见识到了国产一线小厂的实力水平。看到代码的那一刻，就在想，这玩意居然好使，写代码的人是爽了，维护者怎么搞呀，一个.c文件上万行。当然这些都是内心戏，说出来，前辈肯定把我废了。

我大学时学C语言，刚开始码代码就是一个main.c然后用一个main函数搞定所有的事，比如下面：

```
#include<stdio.h>
#include<stdlib.h>

int main(int argc,char**argv)
{
    //定义变量，balabala
    int a,b;
    int sum;
    //输入数据或者读取文件
    scanf("%d %d",&a,&b);
    //处理逻辑
    sum=a+b;
    //打印输出
    printf("%d+%d=%d",a,b,sum);
    return 0;
}
```

这样的单文件单函数处理一些比如简单计算，文件读写，代码行不超过两个屏还不错，当代码超过2屏（大概150行），就要开始切分模块分割函数了，于是main.c变成了下面的样子。

```
#include<stdio.h>
#include<stdlib.h>
struct data{
    int a;
    int b;
}
int read_data(char*filename,struct data*d)
{
    //读取数据
}
int process_data(struct data*d)
{
    //处理数据和逻辑
}
int print_data(struct data*d)
{
    //输出数据
}

int main(int argc,char**argv)
{
    //定义变量，balabala
    struct data d;

    read_data("input.dat",&d)
    process_data(&d);
    print_data(&d);

    return 0;
}
```

通过将部分功能模块化抽离出函数，原本上千行的代码被切割为几个50-200行的代码，既方便阅读，又方便处理，随着功能继续完善，我们会产生不同的数据处理方式，比如添加，删除，修改，查看，查找，排序等，这时候我们就需要把处理数据的部分单独拿出来成为一个独立的模块，于是我们产生了新的模块process\_data.c和process\_data.h，其中.c文件负责模块的代码实现，.h负责模块的对外接口声明，其他的模块也类似，于是我们的代码变成了下面几个文件main.h，process\_data.c和process\_data.h，read\_data.c和read\_data.h，print\_data.c和print\_data.h。

```
//process_data.h
#ifndef __PROCESS_DATA_DEF__
#def __PROCESS_DATA_DEF__

//数据元素
struct data{
    int a;
    int b;
}
typedef struct data* dat;

//数据链表
struct data_list{
    struct data d;
    dat    next;
}
typedef struct data_list* dat_list;
extern int new_list   (dat_list dl);
extern int add_data   (dat_list dl,dat d);
extern int update_data(dat_list dl,int index,dat d);
extern int delete_data(dat_list dl,int index);
extern dat select_data(dat_list dl,char* cmd);
extern int sort_data  (dat_list dl);
extern int search_data(dat_list dl,dat d);

#endif
```

```
//process_data.c
#include "process_data.h"


int new_list   (dat_list dl)
{
}

int add_data   (dat_list dl,dat d)
{
    //添加数据
}
int update_data(dat_list dl,int index,dat d)
{
    //更新数据
}
int delete_data(dat_list dl,int index)
{
    //删除数据
}
dat select_data(dat_list dl,char* cmd)
{
    //查找数据
}
int sort_data  (dat_list dl)
{
    //排序数据
}
int search_data(dat_list dl,dat d)
{
    //查找数据
}

#endif
```

这时候的main.c就把process\_data，read\_data，print\_data包含进来，即可以使用该模块，main.c的代码进一步缩减，框架和结构更清晰明了。

```
#include <stdio.h>
#include <stdlib.h>
#include "process_data.h"
#include "read_data.h"
#include "print_data.h"

int main(int argc,char**argv)
{
    //定义变量，balabala
    struct data d;
    dat_list dl;
    //输入数据
    new_list(dl);
    while(read_data("input.dat",&d) != 0)
         add_data(&d);

    //一系列的数据处理过程
    select_data(dl,d);

    //个性化显示数据
    print_data(&d);

    return 0;
}
```

随着系统功能进一步复杂，输入设备会有各种各样，输出设备与模式也会有各种各样的适配，为了控制系统的复杂度，会进一步进行分层，整体的进化流程就如图1所示。

![](/assets/EmbeddedSystem_S1_P0.png)图1.模块与分层进化图

其实说到底，最初其分成几个函数，到后面的模块化，再到最后的分层设计，都是在简化系统的复杂度，做到局部可控，这样才能hold住全场，让我们同一时刻只关注有限的信息量，毕竟都是人类，谁能一下子接受那么多code，更何况是凌乱的呢。善待code，善待自己，请从模块和分层开始。其实分层不是绝对的完美，所有的分层都会带来效率的降低，比如额外增加的函数调用时间损耗，但是为了可读性和可维护性，牺牲一点效率又能怎么样呢。不过千万不要过度分层，那是在装逼，不是在设计。

#### 2.整个系统的总体设计

智能车系统的模块与分层划分，总体上分为三层，如图2所示。

* 控制与图像层（转速与转向控制器，图像处理）
* 嵌入式平台层（信号采集，器件驱动，任务调度）
* 硬件与机械层（舵机，电机，硬件驱动，编码器，电源，摄像头，巴拉巴拉）

这三层划分，在一般的项目中正好对应四类工程师，控制与图像层对应控制与算法工程师，嵌入式平台层对应嵌入式软件工程师，硬件与机械层，对应嵌入式硬件工程师和机械设计工程师。如果要想实现一个完备的嵌入式系统产品，需要凑齐这四类人才才能够有备无患。

![](/assets/EmbeddedSystem_S1_P1.png)

图2.整体系统模块结构图

##### 控制与图像层

系统中图像处理模块图如图3所示，主要实现图像的处理，寻找中线，以及与Matlab2011a和VS2010配合实现对算法的快速仿真验证，大大提高开发效率，后文会重点介绍这里。

* imCom：图像处理的公用模块
* imProc：图像处理找中线和计算方向偏差的算法实现
* imType：自定义数据类型
* imCar：与Matlab的接口模块，用来快速批量验证算法

![](/assets/EmbeddedSystem_S1_P2.png)

图3.图像处理模块图

系统中控制算法部分的模块图如图4所示，主要负责实现转速和转向控制，其中转速控制会结合Matlab/Simulink 进行仿真，寻找合理的PI控制参数，后面会详细展开如何设计PI控制器。

* ControlVar：所有的共享全局变量
* ControlParam：所有的全局配置参数
* ControlGraphTask：图像和方向控制任务模块
* ControlSpeedTask：速度控制任务模块

Control子模块介绍：

* EIT\_PID：PID控制器模块
* EIT\_SpeedL：左轮速度控制器
* EIT\_SpeedR：右轮速度控制器

![](/assets/EmbeddedSystem_S1_P3.png)

图4.控制算法模块图

##### 嵌入式平台层

嵌入式平台层，负责整个嵌入式软件系统的初始化，信号采集以及驱动执行，模块结构如图5所示，其中本小书会详细介绍EITLIb库中的电机驱动与编码器和摄像头采集部分。

* CarDisplay：显示模块
* CarSystem：系统初始化模块
* CarTest：主循环模块
* IntHandler：中断处理模块
* Board：Vcan山外的K60核心板库
* EITLib：自定义的硬件驱动库
* Chip：Vcan山外实现的 K60的部件库
* CMSIS：CMSIS支持库

![](/assets/EmbeddedSystem_S1_P4.png)

图5.嵌入式平台模块结构图

**硬件与机械层**

硬件采用山外的K60核心板，其他部分，控制核心板和驱动电源板都是自制。模块结构如图6所示，其中主要的是Power，Camera，Motor，Sensor，LCD和Key模块图。

![](/assets/EmbeddedSystem_S1_P5.png)图6.系统硬件结构图

机械部分在ch2再做介绍。

到现在为止，对整个智能车系统有了一个总体的了解，下面我们会分模块进行详细的介绍。


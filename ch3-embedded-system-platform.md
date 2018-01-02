#### 1.从任务调度说起

最开始我们在单片机写代码的样子是怎样的呢？在ch1那一章我们对模块和分层进行了讨论，模块是对功能代码的封装，分层是在平台层面封装，都是在解决项目复杂度控制的问题，但是我们拿单片机最主要的目的是来执行任务Task帮我们做事的，比如读取ADC采样数据，读取键盘按键，输出PWM，I2C通讯，运行PID控制，等等。

那在单片机里如何组织任务调度的设计？

##### 大循环调度

最初的最初，我们的任务调度简单直接——也就是大循环方式，示例代码如下：

```
int main()
{
    Dis_Interrupt();
    System_Init();
    En_Interrupt();

    while(1)
    {
        Task0_Run();
        Task1_Run();
        Task2_Run();
        Task3_Run();
        Task4_Run();
    }
}

void Task0_Run(void)
{
    Pot1Calc(); //加速器信号计算
    Pot2Calc(); //制动器信号计算（保留）
    TempCalc(); //电机及控制器温度计算
}
.....
```

大循环方式的任务调度如图1所示，优点就是简单直接，适合比较简单的系统，带来的不好的地方：

* 每个任务的调度周期和时间是不固定的\(if else的存在\)，无法保证确定的周期性执行任务
* 随着任务数量的增加，系统会越来越慢
* 如果遇上长时间任务，会拖累整个系统变慢

![](/assets/EmbeddedSystem_S3_P0.png)图1.大循环任务调度图

##### 定时任务调度

为了克服大循环方式的缺点（任务调度周期性无法保证，任务数量增加系统会变慢），提出了定时的任务调度的方式，不过需要使用单片机一个定时器，来实现一个简单的任务调度器，利用定时器将CPU切割为一个等周期的时间片调度单元，然后利用标志位控制在每个时间片只调用一个任务。整个系统代码结构如下所示：

```
#define TASK_MAX_LENGTH 10
typedef struct
{
    Int16 Flag[TASK_MAX_LENGTH];
    Int16 Timer;
    Int32 Number;
} USERTASK;
USERTASK UserTask0={0,0,0,0,0,0,0,0,0,0,0,TASK_MAX_LENGTH};//任务初始化

//任务调度函数
void TaskScheduler(USERTASK* v)
{
    v->Flag[v->Timer++] = 1;
    if(v->Timer >= v->Number)
    {
        v->Timer = 0;
    }
}
//主函数
int main()
{
    Dis_Interrupt();
    System_Init();
    En_Interrupt();

    while(1)
    {
        Task0_Run();
        Task1_Run();
        Task2_Run();
        Task3_Run();
        Task4_Run();
    }
}

//1ms定时中断
__interrupt void Timer0_INT_MapedISR(void)
{
    TaskScheduler(&UserTask0);
}

//单个任务示例函数
void Task0_Run(void)
{
    if(UserTask0.Flag[0])
    {
        Pot1Calc();                    //加速器信号计算
        Pot2Calc();                    //制动器信号计算（保留）
        TempCalc();                    //电机及控制器温度计算

        UserTask0.Flag[0] = 0;
    }
}
......
```

定时任务调度的流程图如图2所示。与大循环调度方式对比，这种方式能够实现周期性的任务调度，同时随着任务的增加，依然能够保证调度的周期性，基本上这种调度能够应对大多数的控制系统，比如TI的PMSM电机控制器，一般小的家电控制器，都可以搞定是有。但是使用时有几点要注意：

1. 单个任务的最长时间长度务必保证不超过单个时间片，否则会导致周期性延迟
2. 对于严格实时的控制周期任务，定时调度器不能够保证
3. 对于长周期任务（比如通讯等待等），定时任务调度器要么把任务切割为小任务，要么安排几个连续的空闲周期来执行

![](/assets/EmbeddedSystem_S3_P1.png)图2.定时任务调度图

针对第1点，需要测试或者预估任务的最长执行时间，这个可以采用IO测试的方式解决（具体参见ch6）。

针对第2点，对于实时性要求高，并且周期控制快的任务，只能将这个任务放到定时中断里做，示例代码如下：

```
//1ms定时中断
__interrupt void Timer0_INT_MapedISR(void)
{
    TaskScheduler(&UserTask0);
    Task_SpeedPID_Control();
}

//实时性要求高的任务，示例函数，如果控周期慢的话，也可以选择加入if(UserTask0.Flag[0])做判断
void Task_SpeedPID_Control(void)
{
   SpeedPID_Input();                    //读取输入指令和反馈信号
   SpeedPID_Run();                      //运行PID
   SpeedPID_Output();                   //输出PWM控制
}
......
```

针对第3点，我们可以将长周期任务放在最后面，如图3所示，可以把最后几个空闲周期都留给Task4执行。但是要注意，如果有多个长周期任务，依然会拖慢整个调度周期，于是就出现了基于优先级的任务调度方式，高优先级的任务可以中断低优先级的任务，在保证长周期任务调度的同时，短周期任务的调度依然能够保证，这就是RTOS。

![](/assets/EmdeddedSystem_S3_P2.png)图3.长周期调度方式

##### 实时操作系统RTOS调度

实时操作系统，常用的小型RTOS有uCosII，FreeRTOS，Rt-thread，主要是任务优先级的调度方式不一样，这里感兴趣的同学，可以参见相关的专业书籍，对RTOS内核代码不做详细介绍。RTOS的对任务的调度方式如图4所示。

![](/assets/EmdeddedSystem_S3_P3.png)图4.RTOS任务调度方式




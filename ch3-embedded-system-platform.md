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

定时任务调度的流程图如图2所示。与大循环调度方式对比，这种方式能够实现周期性的任务调度，同时随着任务的增加，依然能够保证调度的周期性，这种调度能够应对大多数的控制系统，比如TI的PMSM电机控制器，一般小的家电控制器，都可以搞定。但是使用时有几点要注意：

1. 单个任务的最长时间长度务必保证不超过单个时间片，否则会导致周期性延迟
2. 对于严格实时的控制周期任务，定时调度器不能够保证
3. 对于长周期任务（比如通讯等待等），定时任务调度器要么把任务切割为小任务，要么安排几个连续的空闲周期来执行

![](/assets/EmbeddedSystem_S3_P1.png)图2.定时任务调度图

针对第1点，需要测试或者预估任务的最长执行时间，这个可以采用IO测试的方式解决（具体参见ch6）。

针对第2点，对于实时性要求高，并且周期控制快的任务（比如PID控制），只能将这个任务放到定时中断里做，示例代码如下：

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

实时操作系统，常用的小型RTOS有uCosII，FreeRTOS，Rt-thread，主要是任务优先级的调度方式不一样，这里感兴趣的同学，可以参见相关的专业书籍，对RTOS内核代码不做详细介绍。RTOS的对任务的调度方式如图4所示。Task0的优先级高，可以中断优先级低的Task1，等Task0执行完，然后RTOS会切换到Task1继续执行。

![](/assets/EmdeddedSystem_S3_P3.png)图4.RTOS任务调度方式图

#### 2.智能车总体任务调度

智能车调度平台总体上只有两个任务SpeedControlTask和ControlGraphTask，考虑到系统简单，没有用RTOS和任务调度器，直接中断配合While实现，代码示例如下，运行时序如图5所示。

```
//main主循环
void main(void)
{                                                               
   DisableInterrupts;  
   CarSystem_Init();
   EnableInterrupts;
   Car_Test();//主循环在这里
   while(1);                                                  
}

//主循环
void  Car_Test(void)
{ 
    while(1)
    {  
       if(ImageOver)//图像DMA传输结束
       {
           ImageOver=0;
           img_extract((uint8 *)Image_Data, (uint8 *)imgbuff0, CAMERA_SIZE);//解压图像
           ControlGraphTask();//图像处理任务

           DataLog_Add();//数据记录
           if(DataLog_CheckEN())
              DataLog_Print();
      }
    }
}

//中断
#define CAM_VSYNC 29

void PORTA_handler(void)
{
    uint32 flag = PORTA_ISFR;
    PORTA_ISFR  = ~0; 
   if(flag & (1 << CAM_VSYNC))                                 //PTA29触发摄像头帧中断
   {
       ImageOver=0;                                            //清除图像采集标志                                  
       camera_vsync();
       gVar.time++;
   }
}
//DMA传输图像数据
void DMA0_IRQHandler()
{
    camera_dma();
    ImageOver=1;
}

//定时10ms中断，执行速度PID控制任务
void PIT0_IRQHandler(void)
{
   SpeedControlTask();
   PIT_Flag_Clear(PIT0);
}
```

![](/assets/EmdeddedSystem_S3_P4.png)图5.系统任务时序图

总体思路就是，每一幅图像的帧中断VSYNC触发PORTA\_handler\(PA29\)中断函数，此时ImageOver清零，同时DMA开始传输图像，当DMA传输结束触发DMA0\_IRQHandler中断，此时ImageOver=1，如果ControlGraphTask检测到的话，那就开始执行，如果ControlGraphTask在VSYNC到来清零ImageOver之前没有开始执行的话，那只能等待下一次DMA中断。最终测试结果，每两帧触发一次ControlGraphTask执行，控制周期为13.33ms。

#### 3.嵌入式驱动层设计

嵌入式驱动层大部分复用了Vcan山外的板级库，新加入比较重要的库有EITMotorL，EITMotor\_R，EIT\_Steer和EIT\_Log，封装在EITLib文件夹，总体的思路就是，.h负责接口，.c负责功能实现。

这里以Motor库为例，介绍一下嵌入式驱动库的封装。

考虑到速度控制用到Motor和Encode，所以把二者集成放到了一起，整体MotorR代码解析如下所示。

```
#ifndef __EIT_MOTORR_DEF__
#define __EIT_MOTORR_DEF__
#include "include.h"
/*Motor Driver*/
#define    MOTORR_PWM_MAX   1000                   //PWM范围：-1000到1000
#define    MOTORR_PWM_MIN   (-1000)
#define    MOTORR_PWM_FREQ  15000                  //PWM工作频率
#define    MOTORR_FTM    FTM0
#define    MOTORR_EN     PTA24
#define    MOTORR_PWMA   FTM_CH3
#define    MOTORR_PWMB   FTM_CH4
#define    MOTORR_PWMAIO PTA6
#define    MOTORR_PWMBIO PTA7


/*Encode */
#define    MOTORR_ENCODE_FTM             FTM2         //左编码器用FTM1
#define    MOTORR_GEAR_N                 36           //B车电机自带齿轮齿轮数
#define    ENCODR_GEAR_N                 40           //B车主动轴齿轮齿轮数
#define    WHEELR_GEAR_N                 105          //编码器比例系数
#define    ENCODR_CYCLE                  2000         //编码器一圈触发2000个脉冲
#define    WHEELR_LENGTH                 18           //17.8cm车轮周长

#define    SPEEDR_FS                     100          //速度采样频率Hz，周期10ms

extern void MotorR_Init(void);                        //电机初始化
extern void MotorR_Run(int32 pwm);                    //电机PWM控制
extern void MotorR_Brake(void);                       //电机刹车
extern void MotorR_Slip(void);                        //电机滑行

extern int32 MotorR_GetWheelSpeed(int32 CntInTs);     //speed单位为cm/s
extern int32 MotorR_GetTsCount(void);                 //10ms周期内，编码器脉冲计数值

#endif
```




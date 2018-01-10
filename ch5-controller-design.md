#### 1.转速PID控制

关于PID控制器的解释可以看看知乎的这篇回答[https://www.zhihu.com/question/23088613](https://www.zhihu.com/question/23088613)

PID控制器应该怎么设计，各种玩家各种玩法，

* 土鳖玩法：不停地试凑PID参数，改一次，烧一次程序，然后实际测试，跟着感觉走，老铁
* 折中玩法：先搞到系统模型，然后Simulink搭建仿真环境，在仿真里试凑，试得差不多了，再放到实际环境进行真实测试
* 高级玩法：硬件在环或者直接MBD设计（基于模型的设计，频域和时域都有不错的玩法）

下面就用智能车的转速PID控制器举例，来跟大家说一下PID到底怎么玩，这里采用的是折中玩法，首先是测得被控对象的模型，被控对象输入控制量是PWM，输出是车速，那系统模型就是一个PWM占空比与到车速之间的关系，如果要推公式的话，那电压电流，转矩，摩擦系数，叽里呱啦一大堆，有没有什么简单易行的方法呢？？废话，当然有呀，我们要得到系统的模型，无非是想知道给这个系统输入什么，它会输出什么反应。我们可以给系统加不同的激励输入，然后测输出反应，根据输入输出反应，就能反推出系统模型呀。

这里我们车的加速和减速性能，所以我们选择加阶跃输入，就是突然给车加一个电压，看车速怎么变化。具体玩法：

1. 做一条长约10-20m的长直赛道，土豪可以再长点
2. 智能车方向控制要有，保证车沿长直赛道行驶
3. 代码设定PWM占空比25%，也就是250（不要太大或者太小），固定不变开环控制，不加入任何速度控制
4. 系统上电，车开始加速行驶，直到速度稳定
5. 从系统上电开始，每隔一个时间在Log里记录一下当前速度（可以选定10ms间隔）
6. 全部跑完之后，将Log记录的数据导出到电脑里，matlab开始画图建模

这里就用到了测试Log模块，会在ch6会详细解释。由于轮胎表面处理对摩擦系数影响比较大，建议测试前适当处理，尽量模拟真实赛况下的轮胎。

测试结束后，我们会在Matlab中画出这样一张车速随时间变化，如图1所示，最后凹下去一大坑又飚起来，是因为车走到终点被抓住速度降了，拿起来空转速度又飚起来了。

![](/assets/EmbeddedSystem_S5_P0.png)

图1.车速开环阶跃响应测试图

根据这张阶跃响应测试图，我们就可以用1阶或者2阶模型去做建模，传递函数形式：

![](/assets/EmbeddedSystem_S5_P1.png)

在这里选的二阶模型建模，Wn=1.5rad/s，zeta=1.6，最终拟合出来的系统模型是：

![](/assets/EmbeddedSystem_S5_P2.png)

在Simulink搭建模型，同样加阶跃响应，可以测试得到实测图与仿真模型的对比结果，如图2所示。

![](/assets/EmbeddedSystem_S5_P3.png)

图2.建模测试对比图（蓝色实测，红色建模）

下一步就是搭建PID控制模块，我们直接来上Simulink仿真模型图，如图3所示，PI控制器的控制效果图如图4所示。

* Test Motor B Car Data：实地测试的B车车速数据
* Model Motor B Car Data：仿真建模的模型阶跃输出
* PI Control Data：PI控制器的输出
* Set PWM：测试设定PWM值（量程-1000至1000）
* Set Speed：设定速度数据（单位为cm/s）

![](/assets/EmbeddedSystem_S5_P4.png)

图3.控制模型图

这里要简单说一下，在反馈回路加了三个部件，一个是Delay环节，因为我们10ms测一次速度，延时一半5ms，RateTransition ZOH是采样率转换，因为前后两级采样率不一致，必须加一个零阶保持器，FIR Filter是均值滤波器，4阶，把车速的高频抖动滤除掉再进控制器。

![](/assets/EmbeddedSystem_S5_P8.png)

图4.PI控制效果图（浅绿色线就是控制效果图，阶跃响应的上升时间从4s降到0.8s左右，效果还可以）

下面重点介绍一下PI Controller，之所以没有加D微分，因为实测速度抖动太厉害，再加微分不抖死呀，目前PI用着就不错。PI的控制模型用的是：

![](/assets/EmbeddedSystem_S5_P5.png)

离散化后的差分方程（采用欧拉前向差分）是：

![](/assets/EmbeddedSystem_S5_P6.png)

对应到Simulink的PI Controller模块设置，下面图5中的几处设置，务必要注意：

* Controller：选择PI
* Form：选择Parallel，并型模式
* Time domain:Discrete-time离散时间域，因为我们是要仿真10ms控制一次
* Integrator method：积分的差分方法，前向欧拉
* Sample time：采样时间Ts=10ms
* Proportional\(P\)：比例系数=4
* Integral\(I\)：积分系数=2.5
* Compensator formula：模块公式与我们上面的差分公式一模一样，**Kp=P，Ki=I**

![](/assets/EmbeddedSystem_S5_P7.png)

图5.PI Controller设置

下面我们就看看代码吧：

```
//EIT_PID.h接口文件
typedef struct _PID
{
    /*In*/
    int32   spVal;
    int32   spValRamp;
    int32   spUpRate;
    int32   spDnRate;
    int32   fbValFilterLast;
    int32   fbValFilter;
    int32   fbValFilterDiff;
    int32   fbVal_k0;
    int32   fbVal_k1;
    int32   fbVal_k2;
    int32   fbVal_k3;
    /*Out*/
    int32   outVal;
    /*Var*/
    int32   err;
    int32   P;
    float   I;
    int32   D;

    /*Param*/
    int32   MAX_Val;
    int32   MIN_Val;

    float   Kp;
    float   Ki;
    float   Kd;
}PID;
typedef  PID*   PID_t;
extern void   PID_InitFbVal(PID_t tPID,int32 fbVal);
extern void   PID_SetFbVal(PID_t tPID,int32 fbVal);
extern void   PID_Run_STD(PID_t tPID);
extern void   PID_Run_PID(PID_t tPID);


//EIT_PID.c模块代码文件
void   PID_SetFbVal(PID_t tPID,int32 fbVal)
{
        tPID->fbVal_k3 =tPID->fbVal_k2;
        tPID->fbVal_k2 =tPID->fbVal_k1;
        tPID->fbVal_k1 =tPID->fbVal_k0;
        tPID->fbVal_k0 =fbVal;
        tPID->fbValFilterLast=tPID->fbValFilter;
        tPID->fbValFilter    =(fbVal+tPID->fbVal_k1+tPID->fbVal_k2+tPID->fbVal_k3)/4;//FIR滤波器
        tPID->fbValFilterDiff=tPID->fbValFilter-tPID->fbValFilterLast;
}

//采用只对反馈值进行微分的PID控制器，本文采用的这种方法，将Kd设置为0，去掉微分
void  PID_Run_PID(PID_t tPID)
{
    int32 err;
    //指令加了Ramp平滑处理
    if(tPID->spVal-tPID->spValRamp > tPID->spUpRate)
        tPID->spValRamp+= tPID->spUpRate;
    if(tPID->spVal-tPID->spValRamp < tPID->spDnRate)
        tPID->spValRamp+= tPID->spDnRate;

    //计算error偏差
    err=tPID->spValRamp-tPID->fbValFilter;
    tPID->err = err; 

    tPID->P =   (int32)(tPID->Kp*err);//比例计算

    tPID->D =  (int32)(tPID->Kd*tPID->fbValFilterDiff);//微分计算

    tPID->outVal = tPID->P + (int32)(tPID->I)+tPID->D;//控制量计算
    tPID->outVal = PID_MaxMin(tPID,tPID->outVal);

    tPID->I =   (int32)(tPID->I  +  tPID->Ki*err);    //前向差分计算积分
    tPID->I =  PID_MaxMinFloat(tPID,tPID->I);  
}
//标准PID控制器
void  PID_Run_STD(PID_t tPID)
{
    int32 err;
    if(tPID->spVal-tPID->spValRamp > tPID->spUpRate)
        tPID->spValRamp+= tPID->spUpRate;
    if(tPID->spVal-tPID->spValRamp < tPID->spDnRate)
        tPID->spValRamp+= tPID->spDnRate;

    err=tPID->spValRamp-tPID->fbValFilter;
    tPID->err = err;

    tPID->P =  (int32)(tPID->Kp*err);

    tPID->D =  (int32)(tPID->Kd*(tPID->fbVal_k0-tPID->fbVal_k1));

    tPID->outVal = (int32)(tPID->P + tPID->I+tPID->D);
    tPID->outVal = PID_MaxMin(tPID,tPID->outVal);

    tPID->I =  (int32)(tPID->I  +  tPID->Ki*err);
    tPID->I =  PID_MaxMinFloat(tPID,tPID->I);
}


Controlparam设置
/*B car just one Motor-Right Motor*/    
gParam.MotorR_PID_KP=4.0;    
gParam.MotorR_PID_KI=2.5;        
gParam.MotorR_PID_KD=0.0;          
gParam.MotorR_PID_Ts=MOTOR_PID_TS; /*Unit: s   */
gParam.MOtroR_PID_UpRate = 1000;/*指令最大m/s^2*/
gParam.MOtroR_PID_DnRate = -2000;/*指令最大m/s^2*/
```

整个速度控制的Simulink模型和C代码已经上传到[github](https://github.com/chenxiannn/SmartCarSpeedControl-PI)。

#### 2.转向PD控制器

没有用什么高大上的算法，就是用最基本的，好使够用。之前在ch4节中，我们通过对赛道图像处理得到了3个gDir的偏差值，分别为gDir\_Near，\_gDir\_Mid和gDir\_Far，大概含义如图6所示。分别选择不同远近区域的中线偏差做平均得到。

* gDir\_Far：用于识别入弯和出弯，因为Far距离远，可以入弯提前减速
* gDir\_Mid：用于方向PD跟踪控制
* gDir\_Near：暂时未使用

![](/assets/EmbeddedSystem_S5_P9.png)

图6.三个gDir的计算区域

整体控制，就将所有赛道路况就分为2种，一种就是直道，另一种就是弯道，根据gDir\_Far以及它的变化率进行识别，具体代码如下：

```
//gDir的滤波处理，这个必须做，因为图像识别算法没处理好的话，很容易出现突变，再一微分，那分分钟搞死
void gDir_Filter(void)
{
   static int MidDir[5];
   static int FarDir[15];
   //gDir_Mid滤波
   MidDir[4]=MidDir[3];
   MidDir[3]=MidDir[2];
   MidDir[2]=MidDir[1];
   MidDir[1]=MidDir[0];
   MidDir[0]=gDir_Mid;

   gDir_MidFilterLast=gDir_MidFilter;
   gDir_MidFilter=(MidDir[0]+MidDir[1]+MidDir[2]+MidDir[3]+MidDir[4])/5;
   gDir_MidFilterDiff=gDir_MidFilter-gDir_MidFilterLast;

   //gDir_Far滤波
   FarDir[9]=FarDir[8];
   FarDir[8]=FarDir[7];
   FarDir[7]=FarDir[6];
   FarDir[6]=FarDir[5];
   FarDir[5]=FarDir[4];
   FarDir[4]=FarDir[3];
   FarDir[3]=FarDir[2];
   FarDir[2]=FarDir[1];
   FarDir[1]=FarDir[0];
   FarDir[0]=gDir_Far;

   //普通滤波5次加权
   gDir_FarFilterLast=gDir_FarFilter;
   gDir_FarFilter=(FarDir[0]+FarDir[1]+FarDir[2]+FarDir[3]+FarDir[4])/5;
   gDir_FarFilterDiff=gDir_FarFilter-gDir_FarFilterLast;

   //慢速滤波5次加权，更慢也更平滑
   gDir_FarFilterSlowLast=gDir_FarFilterSlow;
   gDir_FarFilterSlow=gDir_FarFilter/2+(FarDir[9]+FarDir[8]+FarDir[7]+FarDir[6]+FarDir[5])/10;
   gDir_FarFilterSlowDiff=gDir_FarFilterSlow-gDir_FarFilterSlowLast;

   //入弯和出弯识别
   switch(gVar.InAngle)
   {
       case 0:
          //长直道，如果gDir_far大于某正阀值，并且还在增加，那就是右入弯
          if(gDir_FarFilterDiff>0 && gDir_FarFilter>gParam.InAngle_FarDir )
             gVar.InAngle=1;

          //长直道，如果gDir_far小于某负阀值，并且还在减小，那就是左入弯
          if(gDir_FarFilterDiff<0 && gDir_FarFilter<-gParam.InAngle_FarDir)
             gVar.InAngle=1;
          if(gVar.InAngle == 1)
             MotorR_PID.I = MotorR_PID.I/3;
       break;
       case 1:
           //出弯，可以根据入弯类推
          if(gDir_FarFilterDiff<0 && gDir_FarFilter<gParam.OutAngle_FarDir && gDir_FarFilter>0)
             gVar.InAngle=0;
          if(gDir_FarFilterDiff>0 && gDir_FarFilter>-gParam.OutAngle_FarDir && gDir_FarFilter<0)
             gVar.InAngle=0;
          if(gVar.InAngle == 0)
             MotorR_PID.I = MotorR_PID.I*3;
     break;
   }

}
```

根据gDir\_Far识别出直道和弯道的标志位InAngle，然后根据这个标志来确定车速和方向PD控制参数：

* 车速指令：直道一个速度，弯道一个速度，就两个速度设置参数，简单直接有效
* 转向控制：直道一套PD控制参数，弯道一套PD控制参数，一样简单直接

首先我们看车速指令代码，就两个参数设置gParam.MinSpeed和gParam.MaxSpeed：

```
void GetSetPointMaxSpeed(void)
{
    int MidSpeed;

    if( gVar.InAngle)
    {
        MidSpeed = gParam.MinSpeed;//是不是太简单直接了
    }
    else
    {
        MidSpeed =gParam.MaxSpeed;

    }
    spSpeedL = CarSpeed2LSpeed(MidSpeed,angle);//考虑到转弯半径问题，左右轮速度和车速必须折算一下
    spSpeedR = CarSpeed2RSpeed(MidSpeed,angle);

    //MotorLPID_SetSpeed(spSpeedL);
    MotorRPID_SetSpeed(spSpeedR);
}
```

然后我们转向控制代码，死区控制必须加，否则长直道容易抖，具体大小要实验测试，直道和弯道两套控制参数很有必要，弯道Kp大点更有助于转弯，直道Kp小点，行驶会更平滑：

```
void SteerDirControl(void)
{   
    int MidDir;

    MidDir=gDir_Mid;
    //必须加入死区，大大减少直道抖动
    if(int_abs(MidDir)>=gParam.DIR_Dead)
    {
        if(MidDir>0)
           MidDir-=gParam.DIR_Dead;
        else
           MidDir+=gParam.DIR_Dead;
    }
    else
    {
        MidDir=0;
    }
    //直道和弯道两套控制参数，其中微分参数一致，比例参数是两个参数设置值
    if(gVar.InAngle)
       angle =(int32)((float)(MidDir)*gParam.DIR_KpInAngle+ (float)(gDir_MidFilterDiff)*gParam.DIR_Kd);
    else
       angle =(int32)((float)(MidDir)*gParam.DIR_Kp+ (float)(gDir_MidFilterDiff)*gParam.DIR_Kd);

    //角度限幅操作，防止舵机转角过大，转弯卡死
    if (angle >gParam.AngleMax)
       angle =gParam.AngleMax;
    else if (angle <-gParam.AngleMax)
       angle =-gParam.AngleMax;

    //角度变化率限幅操作，指令必须能够有效执行，不能乱下指令
    angle=int_delta_Limit(angle,angleLast, gParam.AngleDeltaMax);
    angleLast=angle;

    //输出给舵机
    Steer_Run(gParam.SteerMid,angle*gParam.SteerDeltaMax/gParam.AngleMax);//新车需要加负号
}
```

整体的控制思路如图7所示，没有什么过于复杂的地方，都是简单不能再简单的最小系统实现，基本的PID控制配合滤波器，简单的找中线处理，配合上Matlab/Simulink后，工作效率会大大提高。

![](/assets/EmbeddedSystem_S5_P10.png)图7.整体控制思路图


#### 1.为什么分层

说起分层，让我想起了大学刚毕业去的那家小公司，当时自己维护空调控制板，代码是前辈做的，功能相当棒，但是当我看到代码的那一刻，差点吐血，因为整个代码就两个文件shuikongtiao.c和shuikongtiao.h，里面的变量和函数定义全部用汉语拼音，哈哈哈，一万只草泥马从心中飞过。我擦，终于见识到了国产一线小厂的实力水平。看到代码的那一刻，我就在想，这玩意居然好使，写代码的人是爽了，维护者怎么搞呀，一个.c文件上万行，就像一坨屎一样。当然这些都是内心戏，说出来，前辈肯定把我废了。

我大学时学C语言，刚开始码代码是一个main.c然后用一个main函数搞定所有的事，比如下面：

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

这样的单文件单函数处理一些比如计算，文件读写，代码行不超过两个屏还不错，代码超过2屏，大概150行，就要开始切分模块分割函数了，于是main.c变成了下面的样子。

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
int process_data()
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

通过将部分功能模块化抽离出函数，原本上千行的代码被切割为几个50-200行的代码，既方便阅读，又方便处理，随着功能继续完善，我们会产生不同的数据处理方式，比如添加，删除，修改，查看等，这时候我们就需要把

#### 2.分层与模块化

#### 3.整个系统的总体设计




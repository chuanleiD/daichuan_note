### 引用

$j$是$i$的别名，$i$与$j$是同一个内存单元

定义引用时必须立即对它初始化，不能定义完成后再赋值

```c++
int i;
int &j=i;
```

为引用提供的初始值可以是一个变量或另一个引用，如：

```c++
int i=5;
int &j1=i;
int &j2=j1;
```

函数的调用方法：

```c++
//函数：
void swap(int &m,int &n)
{
    int temp;
    temp=m;
    m=n;
    n=temp;
}
//调用
swap(x,y);
```



### 指针

指针合法操作

```c++
int *p;
p+k;
p-k;
```

指针指向元素

```c++
int i=1,*p,j;
j=*(p+i);
```

指针函数调用：

```c++
//函数：
void swap(int *m,int *n)
{
    int temp;
    temp=*m;
    *m=*n;
    *n=temp;
}
//调用
swap(&x,&y);
```


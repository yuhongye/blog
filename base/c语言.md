# 1. 基础语法

### 基础

1. 全局变量只能使用常量或者常量表达式(也就是必须是编译器可以计算出来的)来初始化，默认是零值。

2. 可以在函数体内声明函数，但是不能定义函数

   ```c
   int f() {
     void f2(int, int); // 声明函数, ok
     f2(1, 2);
     return 3;
   }
   
   int f() {
     // error: 不能定义函数
     void f2(int a, int b) {
       printf("a: %d, b: %d\n", a, b);
     }
     f2(1, 2); 
   }
   ```

3. goto语句适用的场景：一个函数中任何地方出错都可以跳转到末尾做错误处理，处理完之后函数返回。

### 结构体(是一种递归定义，因此可以嵌套)

1. 结构的定义

   ```c
   // 1. 最基础的定义
   struct Pointer {
     double x, y;
   }; // 注意分号
   
   // 2. 定义的同时定义两个变量
   struct Pointer {
     double x, y;
   }z1, z2;
   
   // 3. 只定义变量，无法再次引用结构体
   struct {
     double x, y
   } z1, z2;
   
   // 可以在方法内部定义结构体
   int main() {
     struct Pointer { double x, y; } p1;
     p1.x = 1.0;
     p1.y = 2.0;
   }
   
   ```

2. 结构体的初始

   ```c
   // 1. 初始化的数据与结构体的定义一一对应
   struct Pointer p1 = {1, 2,}; // 多个逗号没关系
   // 2. 初始化的数据比结构体成员少，未指定的成员将用零值
   struct Pointer p2 = {1,};
   // 3. 如果多余结构体成员，则报错
   // struct Pointer p2 = {1, 2, 3}; error!
   ```

3. 结构体支持的操作: 目前看支持赋值操作`p1 = p2`

### 枚举类型

枚举类型是用户在程序中定义的__整数类型__, 它可以指定初始值，也可以不指定初始值(不指定初始值时第一个值为0，依次+1), 并且枚举中的不同常量可以拥有相同的值，未指定值的常量等于前一个值加1
```c
// 各个值:0  1  2  0    1  2   0   1  2   1   2  3  4
enum ch {A, B, C, D=0, E, F, G=0, H, I, J=1, K, L, M}
```

#### bool

在C99中定义了`_Bool`类型，`_Bool`实际上是无符号整数类型(所以，算数运算对它是合法的，但是不建议这么做)，但是和一般的整型不同，`_Bool`只能赋值为0或者1，一般来说，往`_Bool`变量中存储非零值会导致变量赋值为1。

在C99中提供了新的头文件`<stdbool.h>`，它提供了3个宏:

* `bool`
* `true`
* `false`

因此使用bool类型的地方都应该引入该头文件。

### void

void *可以赋给任何类型的对象指针，且不要强制转换:
```c
int a = 4;
int *p = &a;
void *v = p;
int *i = v;
```

**指针的本质**

指针的值是一块内存的地址，通过指针可以操作那块内存的值。内存的地址是什么？一个整数.
*指向指针的指针的...指针*
在C语言中，指针是精华，指向指针的指针也比较常用。其实指针的概念扩展开来可以有无限层的指针，但是本质上都指针：其值都是另一块内存的地址。例如下面可以定义4层指针

```c
/* 多重指针并不该被使用，只是为了理解C语言的原则*/
int var = 5;          //存在内存0x23处
int *p = &a;          //指向0x23处的内存，存在内存0x100处
int **pp = &p;        //指向0x100处的内存，存在0x200处
int ***ppp = &pp;     //指向0x200处，存在0x300处
int ****pppp = &ppp;  //指向0x300处，存在0x400处
```

**指向函数的指针**

程序中的每个函数都位于内存中的某个位置，所以存在指向那个位置的指针是完全可能的。在实际的函数调用过程中，也是首先把函数名转换成函数指针。

```c
/* 首先要定义好函数原型 */
int compare(void const *,void const *);
//定义指向函数的指针
int (*p_compare)(void const *,void const *);
//指针指向函数
p_compare = &compare;
//起始&是可以省略的,因为函数名在使用时，编译器总是先把它转换成函数指针
p_compare = compare;
```

如何调用？

* 直接使用函数名compare(p,q)：函数名首先被转换成函数指针，它指向内存中函数的地址，然后执行开始于这个地址的代码
* 使用函数指针(*p_compare)(p,q):解引用获得函数名，函数名再被转换成函数指针，这种调用方式是不是最慢的？它的效果和使用函数名是一样的
* 函数指针 p_compare(p,q):直接使用函数指针，这是最直接的方式

注意和普通的函数区分开来：

* `int *p_compare(void const *,void const *)`p_compare是函数名，它返回整型指针。它的结合顺序是：p_compare首先跟()结合，表明它使一个指针，然后再和*结合，表明它的返回值是一个指针，int表明它使一个指向int型的指针
* `int (*p_compare)(void const *，void const *)`p_compare是指向函数的指针，它的结合顺序是：首先跟`(*p_compare)`表明它使一个指针，然后跟`(void const *,void const *)`结合，表明它使指向函数的指针，int表明它的返回值

函数指针数组，首先它使一个数组，它的元素都是指向函数的指针，那么该如何定义它呢？试着用一些原则来逐步定义函数指针数组

* 它是一个数组，数组的定义：`p_function_array[]`
* 元素是指针，指针数组的定义：`*p_fuction_array[]`
* 指向的是函数,`(*p_function_array[])(void *,void *)`
* 再加上返回值,`int (*p_function_array[])(void *,void *)`

######函数指针的使用
函数指针的作用场景是作为函数的参数，也就是回调函数，看一段代码

```c
通过使用函数指针，可以使用与类型无关的查找方法。
不同的类型有不同的比较方法，但是查找的逻辑都是相同的,
所以可以通过为不同的类型提供不同的比较方法，就可以实现统一的查找逻辑。
p_compare就是回调函数，只要函数的声明符合条件，都可以作为search_list的参数
Java中如何实现回调函数？
Node *search_list(List L,void *value,int (*p_compare)(void const *,void const *)) {
    for(;;)
        if(p_compare(,) == 0)
            do something
    return 节点;
}
```

函数指针的场景是转移表，它可以实现switch的功能，考虑一个计算器的功能，把加减乘除抽象成函数，根据不同的命令，通过switch来选择相应的函数。转移表就可以完成这个操作：

```c
double add(double a,double b);
double sub(double a,double b);
double mul(double a,double b);
double div(double a,double b);

//函数指针数组
double (*calc[])(double,double) = {add,sub,mul,div};

//使用
calc[0](a,b)
```

通过分析一些技巧的含义能让我们更好的理解C语言的原则,例如只要把握住指针的本质，在分析多层指针的时就不会有问题。

#### memcpy
**函数原型** `void *memcpy(void *dest,const void *src,size_t n)`

**功能** 由src指向的地址为起始地址，拷贝n个字节到dest指向的地址为起始地址的空间内，一定会拷贝n个字节。 

**说明**

* memcpy是逐字节拷贝的，看的原型声明，它可以接收任何类型的指针，在内部应该是将`void *`转换成`char *`，不论字节的内容是什么，只管拷贝
* 一定会拷贝n个字节，它是mem拷贝函数，同strcpy不同的地方就是，memcpy不会判断字节的内容，而strcpy则是遇到'\0'就停止拷贝
* 不安全，它不会判断dest是否有足够的空间，这需要由调用方来保证，不管怎么，它就拷贝n个字节

```c
/* 在c语言的一般实现中，函数的局部变量被分配在栈上，且由高往低
 * 数组的分配：连续n个元素，xxx[0]是地址最小的
 */
void memcpy_test() {
    /* memcpy只管拷贝n个字节，不会判断，也不会添加\0 */
    char src1[] = "aaaaaaaaaaaaaaaaaaaaaa";
    char dest1[] = "bbbbbbbbbbbbbbbbbbbbbb";
    memcpy(dest1,src1,2);
    //调用结束后，dest1的内容：aabbbbbbbbbbbbbbbbbbbb

    /* 一定会拷贝n个字节，且不判断dest1是否有足够的空间*/
    char tmp_src[] = "abcdefg";
    char src_2[] = "hijklmn";
    char tmp_dest[] = "opqrst"
    char dest_2[] = "uvwxyz";
    /* 4个数组的内存布局,由于有对齐的要求，每个组数中间会空几个字节
     * 假设是16字节对齐
     * tmp_src，由高到低：gfedcba
     * ...间隔8个字节:n1
     * src_2,由高到低：nmlkjih
     * ...间隔9的字节:n2
     * tmp_dest,由高到低:tsrqpo
     * ...间隔9个字节:n3
     * dest_2,由高到低：zyxwvu
     * 上面函数的内存布局
        |分配给该函数的栈帧|
        |------------------|
        |填充的8个字节     |
        |gfedcba           |
        |填充的8个字节     |
        |nmlkjih           |
        |填充的9个字节     |
        |tsrqpo            |
        |填充的9个字节     |
        |zyxwvu            |
     */

     memcpy(dest_2,src_2,sizeof(src_2) + n1 + sizeof(tmp_src));
     /* 上面的函数会拷贝sizeof(src_2) + n1 + sizeof(tmp_src)=24个字节
      * 1)把src_2,tmp_src和两个数组之间的填充内容都拷贝到dest_2指向的内存
      * 2)dest_2指向的内存内容：hijklmn\0 n1个填充字节 abcdefg\0，实际上
      *   dest_2已经写越界了，把tmp_dest的内容也覆盖了
      * 3)tmp_dest的内容就变成了：abcdefg\0,g和'\0'都已经超出了tmp_dest的空间
      */
}
```

#### 柔性数组成员(flexible array member)
**不完整类型(incomplete type)**是一种缺乏足够信息去描述一个完整的对象。
柔性数组成员就是一种不完整类型，它的声明：ElementType buf[];有几点需要解释的：

* C99中，结构体的最后一个元素允许是未知大小的数组，结构体中的柔性数组成员前面至少有一个成员，且flexible array member必须是结构体的最后一个成员
* 柔性数组成员只作为一个符号地址存在，它不占用空间，sizeof(结构体)返回的大小并不包含柔性数组成员的内存，因为它压根就不占用内存
* 柔性数组成员不仅可以用作字符数组，还可以是其他类型数组
* 柔性数组成员需要使用malloc进行动态内存分配，由于flexible array member的存在，可以实现动态结构体
* 在C99的标准中，支持的是incomplete type，而不是zero array,所以标准支持的声明方式是：ElementType buf[],而不是ElementType buf[0]

```
/* 64位机器，int 4个字节，指针8个字节 */
/* 柔性数组成员实现的动态结构体 */
typedef struct {
    unsigned int len;
    unsigned int free;
    char buf[];
} flexible_array_struct;

/* 使用指针实现的动态结构体 */
typedef struct {
    unsigned int len;
    unsigned int free;
    char *buf;
}pointer_dync_struct;
```

使用flexible array member和使用指针的区别和优势：

* flexible array member 相当于占位符，它实际上是一个偏移量，表示buf相对于结构体的偏移,且不占空间；而`char *`则是一个完整的变量，它占用一个指针的空间;
* flexible array member和使用指针都需要使用malloc来申请内存，通过对结构体多申请内存来供fexible array使用，比如`malloc(sizoef(flexible_array_struct) + n)`,它申请了len和free的空间，同时还多申请了n个字节，buf就指向这n个字节的起始位置，也就是buf操作的空间,所以flexible array的空间跟结构体的空间是连在一起的。而如果通过指针，则先申请结构体的空间，然后再申请指针所指向的空间，它俩是不连续的，不如连续的好管理，而且容易造成内存碎片,申请空间的代码如下:`pointer_array_struct pointer = malloc(sizeof(pointer_array_struct));pointer.buf = malloc(n);`
* 关于flexible array的释放问题：系统并没有把那块内存释放掉，但是它对系统而言是空闲内存，需要再了解一下这个问题

#### 指针和结构体作为自动变量的初始化
结构体作为自动变量的初始化跟其它类型是一样的，如果没有显式的初始化，就是使用栈上分配的内存的值。
首先在标准中规定，可以将0赋给指针，表示一个空指针，其他的直接将一个整数值赋给指针应该是违法的。指针作为自动变量的初始化，其实跟其他类型也是一样的，也是在栈上给指针分配一个槽位 ，指针的值就是这块内存当时的值。需要注意的是：指针的值（也就是指针所指向的内存）其实是无意义的，这个时候如果打印指针的值及指针指向内存的值，得到的都是脏数据；如果写指针指向的内存，就是很危险的动作了，如果没有写权限，应该直接就core掉，就算能写，也污染了那块内存。


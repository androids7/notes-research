###C 语言高效编程与代码优化

在本篇文章中，我收集了很多经验和方法。应用这些经验和方法，可以帮助我们从执行速度和内存使用等方面来优化C语言代码。
简介
在最近的一个项目中，我们需要开发一个运行在移动设备上但不保证图像高质量的轻量级JPEG库。期间，我总结了一些让程序运行更快的方法。在本篇文章中，我收集了一些经验和方法。应用这些经验和方法，可以帮助我们从执行速度和内存使用等方面来优化C语言代码。
尽管在C代码优化方面有很多的指南，但是关于编译和你使用的编程机器方面的优化知识却很少。
通常，为了让你的程序运行的更快，程序的代码量可能需要增加。代码量的增加又可能会对程序的复杂度和可读性带来不利的影响。这对于在手机、PDA等对于内存使用有很多限制的小型设备上编写程序时是不被允许的。因此，在代码优化时，我们的座右铭应该是确保内存使用和执行速度两方面都得到优化。
声明
实际上，在我的项目中，我使用了很多优化ARM编程的方法（该项目是基于ARM平台的），也使用了很多互联网上面的方法。但并不是所有文章提到的方法都能起到很好的作用。所以，我对有用的和高效的方法进行了总结收集。同时，我还修改了其中的一些方法，使他们适用于所有的编程环境，而不是局限于ARM环境。
哪里需要使用这些方法？
没有这一点，所有的讨论都无从谈起。程序优化最重要的就是找出待优化的地方，也就是找出程序的哪些部分或者哪些模块运行缓慢亦或消耗大量的内存。只有程序的各部分经过了优化，程序才能执行的更快。
程序中运行最多的部分，特别是那些被程序内部循环重复调用的方法最该被优化。
对于一个有经验的码农，发现程序中最需要被优化的部分往往很简单。此外，还有很多工具可以帮助我们找出需要优化的部分。我使用过Visual C++内置的性能工具profiler来找出程序中消耗最多内存的地方。另一个我使用过的工具是英特尔的Vtune，它也能很好的检测出程序中运行最慢的部分。根据我的经验，内部或嵌套循环，调用第三方库的方法通常是导致程序运行缓慢的最主要的起因。
整形数
如果我们确定整数非负，就应该使用unsigned int而不是int。有些处理器处理无符号unsigned 整形数的效率远远高于有符号signed整形数（这是一种很好的做法，也有利于代码具体类型的自解释）。
因此，在一个紧密循环中，声明一个int整形变量的最好方法是：
register unsigned int variable_name;
记住，整形in的运算速度高浮点型float，并且可以被处理器直接完成运算，而不需要借助于FPU（浮点运算单元）或者浮点型运算库。尽管这不保证编译器一定会使用到寄存器存储变量，也不能保证处理器处理能更高效处理unsigned整型，但这对于所有的编译器是通用的。
例如在一个计算包中，如果需要结果精确到小数点后两位，我们可以将其乘以100，然后尽可能晚的把它转换为浮点型数字。
除法和取余数
在标准处理器中，对于分子和分母，一个32位的除法需要使用20至140次循环操作。除法函数消耗的时间包括一个常量时间加上每一位除法消耗的时间。
Time (numerator / denominator) = C0 + C1* log2 (numerator / denominator)
     = C0 + C1 * (log2 (numerator) - log2 (denominator)).
对于ARM处理器，这个版本需要20+4.3N次循环。这是一个消耗很大的操作，应该尽可能的避免执行。有时，可以通过乘法表达式来替代除法。例如，假如我们知道b是正数并且b*c是个整数，那么(a/b)>c可以改写为a>(c*b)。如果确定操作数是无符号unsigned的，使用无符号unsigned除法更好一些，因为它比有符号signed除法效率高。
合并除法和取余数
在一些场景中，同时需要除法（x/y）和取余数（x%y）操作。这种情况下，编译器可以通过调用一次除法操作返回除法的结果和余数。如果既需要除法的结果又需要余数，我们可以将它们写在一起，如下所示：
int func_div_and_mod (int a, int b) { 
        return (a / b) + (a % b);
    }
通过2的幂次进行除法和取余数
如果除法中的除数是2的幂次，我们可以更好的优化除法。编译器使用移位操作来执行除法。因此，我们需要尽可能的设置除数为2的幂次（例如64而不是66）。并且依然记住，无符号unsigned整数除法执行效率高于有符号signed整形出发。
typedef unsigned int uint;

    uint div32u (uint a) {
     return a / 32;
    }
    int div32s (int a){
     return a / 32;
    }
上面两种除法都避免直接调用除法函数，并且无符号unsigned的除法使用更少的计算机指令。由于需要移位到0和负数，有符号signed的除法需要更多的时间执行。
取模的一种替代方法
我们使用取余数操作符来提供算数取模。但有时可以结合使用if语句进行取模操作。考虑如下两个例子：
uint modulo_func1 (uint count)
{
   return (++count % 60);
}

uint modulo_func2 (uint count)
{
   if (++count >= 60)
  count = 0;
  return (count);
}
优先使用if语句，而不是取余数运算符，因为if语句的执行速度更快。这里注意新版本函数只有在我们知道输入的count结余0至59时在能正确的工作。
使用数组下标
如果你想给一个变量设置一个代表某种意思的字符值，你可能会这样做：
switch ( queue ) {
case 0 :   letter = 'W';
   break;
case 1 :   letter = 'S';
   break;
case 2 :   letter = 'U';
   break;
}
或者这样做：
if ( queue == 0 )
  letter = 'W';
else if ( queue == 1 )
  letter = 'S';
else
  letter = 'U';
一种更简洁、更快的方法是使用数组下标获取字符数组的值。如下：
static char *classes="WSU";

letter = classes[queue];
全局变量
全局变量绝不会位于寄存器中。使用指针或者函数调用，可以直接修改全局变量的值。因此，编译器不能将全局变量的值缓存在寄存器中，但这在使用全局变量时便需要额外的（常常是不必要的）读取和存储。所以，在重要的循环中我们不建议使用全局变量。
如果函数过多的使用全局变量，比较好的做法是拷贝全局变量的值到局部变量，这样它才可以存放在寄存器。这种方法仅仅适用于全局变量不会被我们调用的任意函数使用。例子如下：
int f(void);
int g(void);
int errs;
void test1(void)
{
  errs += f();
  errs += g();
}

void test2(void)
{
  int localerrs = errs;
  localerrs += f();
  localerrs += g();
  errs = localerrs;
}
注意，test1必须在每次增加操作时加载并存储全局变量errs的值，而test2存储localerrs于寄存器并且只需要一个计算机指令。
使用别名
考虑如下的例子：
void func1( int *data )
{
 int i;

 for(i=0; i<10; i++)
 {
 anyfunc( *data, i);
 }
}
尽管*data的值可能从未被改变，但编译器并不知道anyfunc函数不会修改它，所以程序必须在每次使用它的时候从内存中读取它。如果我们知道变量的值不会被改变，那么就应该使用如下的编码：
void func1( int *data )
{
 int i;

 for(i=0; i<10; i++)
 {
 anyfunc( *data, i);
 }
}
这为编译器优化代码提供了条件。
变量的生命周期分割
由于处理器中寄存器是固定长度的，程序中数字型变量在寄存器中的存储是有一定限制的。
有些编译器支持“生命周期分割”（live-range splitting），也就是说在程序的不同部分，变量可以被分配到不同的寄存器或者内存中。变量的生命周期开始于对它进行的最后一次赋值，结束于下次赋值前的最后一次使用。在生命周期内，变量的值是有效的，也就是说变量是活着的。不同生命周期之间，变量的值是不被需要的，也就是说变量是死掉的。这样，寄存器就可以被其余变量使用，从而允许编译器分配更多的变量使用寄存器。
需要使用寄存器分配的变量数目需要超过函数中不同变量生命周期的个数。如果不同变量生命周期的个数超过了寄存器的数目，那么一些变量必须临时存储于内存。这个过程就称之为分割。
编译器首先分割最近使用的变量，用以降低分割带来的消耗。禁止变量生命周期分割的方法如下：
限定变量的使用数量：这个可以通过保持函数中的表达式简单、小巧、不使用太多的变量实现。将较大的函数拆分为小而简单的函数也会达到很好的效果。
对经常使用到的变量采用寄存器存储：这样允许我们告诉编译器该变量是需要经常使用的，所以需要优先存储于寄存器中。然而，在某种情况下，这样的变量依然可能会被分割出寄存器。
变量类型
C编译器支持基本类型：char、short、int、long(包括有符号signed和无符号unsigned）、float和double。使用正确的变量类型至关重要，因为这可以减少代码和数据的大小并大幅增加程序的性能。
局部变量
我们应该尽可能的不使用char和short类型的局部变量。对于char和short类型，编译器需要在每次赋值的时候将局部变量减少到8或者16位。这对于有符号变量称之为有符号扩展，对于无符号变量称之为零扩展。这些扩展可以通过寄存器左移24或者16位，然后根据有无符号标志右移相同的位数实现，这会消耗两次计算机指令操作（无符号char类型的零扩展仅需要消耗一次计算机指令）。
可以通过使用int和unsigned int类型的局部变量来避免这样的移位操作。这对于先加载数据到局部变量，然后处理局部变量数据值这样的操作非常重要。无论输入输出数据是8位或者16位，将它们考虑为32位是值得的。
考虑下面的三个函数：
int wordinc (int a)
{
   return a + 1;
}
short shortinc (short a)
{
    return a + 1;
}
char charinc (char a)
{
    return a + 1;
}
尽管结果均相同，但是第一个程序片段运行速度高于后两者。
指针
我们应该尽可能的使用引用值的方式传递结构数据，也就是说使用指针，否则传递的数据会被拷贝到栈中，从而降低程序的性能。我曾见过一个程序采用传值的方式传递非常大的结构数据，然后这可以通过一个简单的指针更好的完成。
函数通过参数接受结构数据的指针，如果我们确定不改变数据的值，我们需要将指针指向的内容定义为常量。例如：
void print_data_of_a_structure ( const Thestruct  *data_pointer)
{
    ...printf contents of the structure...
}
这个示例告诉编译器函数不会改变外部参数的值（使用const修饰），并且不用在每次访问时都进行读取。同时，确保编译器限制任何对只读结构的修改操作从而给予结构数据额外的保护。
指针链
指针链经常被用于访问结构数据。例如，常用的代码如下：
typedef struct { int x, y, z; } Point3;
typedef struct { Point3 *pos, *direction; } Object;

void InitPos1(Object *p)
{
   p->pos->x = 0;
   p->pos->y = 0;
   p->pos->z = 0;
}
然而，这种的代码在每次操作时必须重复调用p->pos，因为编译器不知道p->pos->x与p->pos是相同的。一种更好的方法是缓存p->pos到一个局部变量：
void InitPos2(Object *p)
{
   Point3 *pos = p->pos;
   pos->x = 0;
   pos->y = 0;
   pos->z = 0;
}
另一种方法是在Object结构中直接包含Point3类型的数据，这能完全消除对Point3使用指针操作。
条件执行
条件执行语句大多在if语句中使用，也在使用关系运算符（等）或者布尔值表达式（&&，！等）计算复杂表达式时使用。对于包含函数调用的代码片段，由于函数返回值会被销毁，因此条件执行是无效的。
因此，保持if和else语句尽可能简单是十分有益处的，因为这样编译器可以集中处理它们。关系表达式应该写在一起。
下面的例子展示编译器如何使用条件执行：
int g(int a, int b, int c, int d)
{
 if (a > 0 && b > 0 && c < 0 && d < 0)
 // grouped conditions tied up together//
 return a + b + c + d;
 return -1;
}
由于条件被聚集到一起，编译器能够将他们集中处理。
布尔表达式和范围检查
一个常用的布尔表达式是用于判断变量是否位于某个范围内，例如，检查一个图形坐标是否位于一个窗口内：
bool PointInRectangelArea (Point p, Rectangle *r)
{
   <span class="hljs-keyword">return</span> (p.x &gt;= r-&gt;xmin &amp;&amp; p.x &lt; r-&gt;xmax &amp;&amp;
                      p.y &gt;= r-&gt;ymin &amp;&amp; p.y &lt; r-&gt;ymax);
}
这里有一种更快的方法：x>min && x
bool PointInRectangelArea (Point p, Rectangle *r)
{
 return ((unsigned) (p.x - r->xmin) < r->xmax &&
 (unsigned) (p.y - r->ymin) < r->ymax);

}
布尔表达式和零值比较
处理器的标志位在比较指令操作后被设置。标志位同样可以被诸如MOV、ADD、AND、MUL等基本算术和裸机指令改写。如果数据指令设置了标志位，N和Z标志位也将与结果与0比较一样进行设置。N标志表示结果是否是负值，Z标志表示结果是否是0。
C语言中，处理器中的N和Z标志位与下面的指令联系在一起：有符号关系运算x=0，x==0，x!=0；无符号关系运算x==0，x!=0（或者x>0）。
C代码中每次关系运算符的调用，编译器都会发出一个比较指令。如果操作符是上面提到的，编译器便会优化掉比较指令。例如：
int aFunction(int x, int y)
{
   if (x + y < 0)
      return 1;
  else
     return 0;
}
尽可能的使用上面的判断方式，这可以在关键循环中减少比较指令的调用，进而减少代码体积并提高代码性能。C语言没有借位和溢出位的概念，因此，如果不借助汇编，不可能直接使用借位标志C和溢出位标志V。但编译器支持借位（无符号溢出），例如：
int sum(int x, int y)
{
   int res;
   res = x + y;
   if ((unsigned) res < (unsigned) x) // carry set?  //
     res++;
   return res;
}
懒检测开发
在if(a>10 && b=4)这样的语句中，确保AND表达式的第一部分最可能较快的给出结果（或者最早、最快计算），这样第二部分便有可能不需要执行。
用switch()函数替代if…else…
对于涉及if…else…else…这样的多条件判断，例如：
if( val == 1)
    dostuff1();
else if (val == 2)
    dostuff2();
else if (val == 3)
    dostuff3();
使用switch可能更快：
switch( val )
{
    case 1: dostuff1(); break;

    case 2: dostuff2(); break;

    case 3: dostuff3(); break;
}
在if()语句中，如果最后一条语句命中，之前的条件都需要被测试执行一次。Switch允许我们不做额外的测试。如果必须使用if…else…语句，将最可能执行的放在最前面。
二分中断
使用二分方式中断代码而不是让代码堆成一列，不要像下面这样做：
if(a==1) {
} else if(a==2) {
} else if(a==3) {
} else if(a==4) {
} else if(a==5) {
} else if(a==6) {
} else if(a==7) {
} else if(a==8)

{
}
使用下面的二分方式替代它，如下：
if(a<=4) {
    if(a==1)     {
    }  else if(a==2)  {
    }  else if(a==3)  {
    }  else if(a==4)   {

    }
}
else
{
    if(a==5)  {
    } else if(a==6)   {
    } else if(a==7)  {
    } else if(a==8)  {
    }
}
或者如下：
if(a<=4)
{
    if(a<=2)
    {
        if(a==1)
        {
            /* a is 1 */
        }
        else
        {
            /* a must be 2 */
        }
    }
    else
    {
        if(a==3)
        {
            /* a is 3 */
        }
        else
        {
            /* a must be 4 */
        }
    }
}
else
{
    if(a<=6)
    {
        if(a==5)
        {
            /* a is 5 */
        }
        else
        {
            /* a must be 6 */
        }
    }
    else
    {
        if(a==7)
        {
            /* a is 7 */
        }
        else
        {
            /* a must be 8 */
        }
    }
}
比较如下两种case语句：
慢而低效的代码	快而高效的代码
c=getch();
switch(c){
    case 'A':
    {
        do something;
        break;
    }
    case 'H':
    {
        do something;
        break;
    }
    case 'Z':
    {
        do something;
        break;
    }
}
c=getch();
switch(c){
    case 0:
    {
        do something;
        break;
    }
    case 1:
    {
        do something;
        break;
    }
    case 2:
    {
        do something;
        break;
    }
}
switch语句vs查找表
Switch的应用场景如下：
调用一到多个函数
设置变量值或者返回一个值
执行一到多个代码片段
如果case标签很多，在switch的前两个使用场景中，使用查找表可以更高效的完成。例如下面的两种转换字符串的方式：
char * Condition_String1(int condition) {
  switch(condition) {
     case 0: return "EQ";
     case 1: return "NE";
     case 2: return "CS";
     case 3: return "CC";
     case 4: return "MI";
     case 5: return "PL";
     case 6: return "VS";
     case 7: return "VC";
     case 8: return "HI";
     case 9: return "LS";
     case 10: return "GE";
     case 11: return "LT";
     case 12: return "GT";
     case 13: return "LE";
     case 14: return "";
     default: return 0;
  }
}

char * Condition_String2(int condition) {
   if ((unsigned) condition >= 15) return 0;
      return
      "EQNECSCCMIPLVSVCHILSGELTGTLE" +
       3 * condition;
}
第一个程序需要240 bytes，而第二个仅仅需要72 bytes。
循环
循环是大多数程序中的常用的结构；程序执行的大部分时间发生在循环中，因此十分值得在循环执行时间上下一番功夫。
循环终止
如果不加注意，循环终止条件的编写会导致额外的负担。我们应该使用计数到零的循环和简单的循环终止条件。简单的终止条件消耗更少的时间。看下面计算n！的两个程序。第一个实现使用递增的循环，第二个实现使用递减循环。
int fact1_func (int n)
{
    int i, fact = 1;
    for (i = 1; i <= n; i++)
      fact *= i;
    return (fact);
}

int fact2_func(int n)
{
    int i, fact = 1;
    for (i = n; i != 0; i--)
       fact *= i;
    return (fact);
}
第二个程序的fact2_func执行效率高于第一个。
更快的for()循环
这是一个简单而高效的概念。通常，我们编写for循环代码如下：
for( i=0;  i<10;  i++){ ... }
i从0循环到9。如果我们不介意循环计数的顺序，我们可以这样写：
for( i=10; i--; ) { ... }
这样快的原因是因为它能更快的处理i的值–测试条件是：i是非零的吗？如果这样，递减i的值。对于上面的代码，处理器需要计算“计算i减去10，其值非负吗？如果非负，i递增并继续”。简单的循环却有很大的不同。这样，i从9递减到0，这样的循环执行速度更快。
这里的语法有点奇怪，但确实合法的。循环中的第三条语句是可选的（无限循环可以写为for(;;)）。如下代码拥有同样的效果：
for(i=10; i; i--){}
或者更进一步的：
for(i=10; i!=0; i--){}
这里我们需要记住的是循环必须终止于0（因此，如果在50到80之间循环，这不会起作用），并且循环计数器是递减的。使用递增循环计数器的代码不享有这种优化。
合并循环
如果一个循环能解决问题坚决不用二个。但如果你需要在循环中做很多工作，这坑你并不适合处理器的指令缓存。这种情况下，两个分开的循环可能会比单个循环执行的更快。下面是一个例子：
//Original Code :

for(i=0; i<100; i++){
    stuff();
}

for(i=0; i<100; i++){
    morestuff();
}
//It would be better to do:

for(i=0; i<100; i++){
    stuff();
    morestuff();
}
函数循环
调用函数时总是会有一定的性能消耗。不仅程序指针需要改变，而且使用的变量需要压栈并分配新变量。为提升程序的性能，在函数这点上有很多可以优化的。在保持程序代码可读性的同时也需要代码的大小是可控的。
如果在循环中一个函数经常被调用，那么就将循环纳入到函数中，这样可以减少重复的函数调用。代码如下：
for(i=0 ; i<100 ; i++)
{
    func(t,i);
}
-
-
-
void func(int w,d)
{
    lots of stuff.
}
应改为：
for(i=0 ; i<100 ; i++)
{
    func(t,i);
}
-
-
-
void func(int w,d)
{
    lots of stuff.
}
循环展开 
简单的循环可以展开以获取更好的性能，但需要付出代码体积增加的代价。循环展开后，循环计数应该越来越小从而执行更少的代码分支。如果循环迭代次数只有几次，那么可以完全展开循环，以便消除循坏带来的负担。
这会带来很大的不同。循环展开可以带非常可观的节省性能，原因是代码不用每次循环需要检查和增加i的值。例如：
for(i=0; i<3; i++){
    something(i);
}

//is less efficient than
something(0);
something(1);
something(2);
编译器通常会像上面那样展开简单的，迭代次数固定的循环。但是像下面的代码：
for(i=0;i< limit;i++) { ... }
下面的代码（Example 1）明显比使用循环的方式写的更长，但却更有效率。block-sie的值设置为8仅仅适用于测试的目的，只要我们重复执行“loop-contents”相同的次数，都会有很好的效果。在这个例子中，循环条件每8次迭代才会被检查，而不是每次都进行检查。由于不知道迭代的次数，一般不会被展开。因此，尽可能的展开循环可以让我们获得更好的执行速度。
//Example 1

#include<STDIO.H>

#define BLOCKSIZE (8)

void main(void)
{
int i = 0;
int limit = 33;  /* could be anything */
int blocklimit;

/* The limit may not be divisible by BLOCKSIZE,
 * go as near as we can first, then tidy up.
 */
blocklimit = (limit / BLOCKSIZE) * BLOCKSIZE;

/* unroll the loop in blocks of 8 */
while( i < blocklimit )
{
    printf("process(%d)\n", i);
    printf("process(%d)\n", i+1);
    printf("process(%d)\n", i+2);
    printf("process(%d)\n", i+3);
    printf("process(%d)\n", i+4);
    printf("process(%d)\n", i+5);
    printf("process(%d)\n", i+6);
    printf("process(%d)\n", i+7);

    /* update the counter */
    i += 8;

}

/*
 * There may be some left to do.
 * This could be done as a simple for() loop,
 * but a switch is faster (and more interesting)
 */

if( i < limit )
{
    /* Jump into the case at the place that will allow
     * us to finish off the appropriate number of items.
     */

    switch( limit - i )
    {
        case 7 : printf("process(%d)\n", i); i++;
        case 6 : printf("process(%d)\n", i); i++;
        case 5 : printf("process(%d)\n", i); i++;
        case 4 : printf("process(%d)\n", i); i++;
        case 3 : printf("process(%d)\n", i); i++;
        case 2 : printf("process(%d)\n", i); i++;
        case 1 : printf("process(%d)\n", i);
    }
}

}
统计非零位的数量 
通过不断的左移，提取并统计最低位，示例程序1高效的检查一个数组中有几个非零位。示例程序2被循环展开四次，然后通过将四次移位合并成一次来优化代码。经常展开循环，可以提供很多优化的机会。
//Example - 1

int countbit1(uint n)
{
  int bits = 0;
  while (n != 0)
  {
    if (n & 1) bits++;
    n >>= 1;
   }
  return bits;
}

//Example - 2

int countbit2(uint n)
{
   int bits = 0;
   while (n != 0)
   {
      if (n & 1) bits++;
      if (n & 2) bits++;
      if (n & 4) bits++;
      if (n & 8) bits++;
      n >>= 4;
   }
   return bits;
}
尽早的断开循环
通常，循环并不需要全部都执行。例如，如果我们在从数组中查找一个特殊的值，一经找到，我们应该尽可能早的断开循环。例如：如下循环从10000个整数中查找是否存在-99。
found = FALSE;
for(i=0;i<10000;i++)
{
    if( list[i] == -99 )
    {
        found = TRUE;
    }
}

if( found ) printf("Yes, there is a -99. Hooray!\n");
上面的代码可以正常工作，但是需要循环全部执行完毕，而不论是否我们已经查找到。更好的方法是一旦找到我们查找的数字就终止继续查询。
found = FALSE;
for(i=0;i<10000;i++)
{
    if( list[i] == -99 )
    {
        found = TRUE;
    }
}

if( found ) printf("Yes, there is a -99. Hooray!\n");
假如待查数据位于第23个位置上，程序便会执行23次，从而节省9977次循环。
函数设计
设计小而简单的函数是个很好的习惯。这允许寄存器可以执行一些诸如寄存器变量申请的优化，是非常高效的。
函数调用的性能消耗
函数调用对于处理器的性能消耗是很小的，只占有函数执行工作中性能消耗的一小部分。参数传入函数变量寄存器中有一定的限制。这些参数必须是整型兼容的（char，shorts，ints和floats都占用一个字）或者小于四个字大小（包括占用2个字的doubles和long longs）。如果参数限制个数为4，那么第五个和之后的字就会存储在栈上。这便在调用函数是需要从栈上加载参数从而增加存储和读取的消耗。
看下面的代码：
int f1(int a, int b, int c, int d) {
   return a + b + c + d;
}

int g1(void) {
   return f1(1, 2, 3, 4);
}

int f2(int a, int b, int c, int d, int e, int f) {
  return a + b + c + d + e + f;
}

ing g2(void) {
 return f2(1, 2, 3, 4, 5, 6);
}
函数g2中的第五个和第六个参数存储于栈上并在函数f2中进行加载，会多消耗2个参数的存储。
减少函数参数传递消耗
减少函数参数传递消耗的方法有：
尽量保证函数使用少于四个参数。这样就不会使用栈来存储参数值。
如果函数需要多于四个的参数，尽量确保使用后面参数的价值高于让其存储于栈所付出的代价。
通过指针传递参数的引用而不是传递参数结构体本身。
将参数放入一个结构体并通过指针传入函数，这样可以减少参数的数量并提高可读性。
尽量少用占用两个字大小的long类型参数。对于需要浮点类型的程序，double也因为占用两个字大小而应尽量少用。
避免函数参数既存在于寄存器又存在于栈中（称之为参数拆分）。现在的编译器对这种情况处理的不够高效：所有的寄存器变量也会放入到栈中。
避免变参。变参函数将参数全部放入栈。
叶子函数
不调用任何函数的函数称之为叶子函数。在以下应用中，近一半的函数调用是调用叶子函数。由于不需要执行寄存器变量的存储和读取，叶子函数在任何平台都很高效。寄存器变量读取的性能消耗，相比于使用四五个寄存器变量的叶子函数所做的工作带来的系能消耗是非常小的。所以尽可能的将经常调用的函数写成叶子函数。函数调用的次数可以通过一些工具检查。下面是一些将一个函数编译为叶子函数的方法：
避免调用其他函数：包括那些转而调用C库的函数（比如除法或者浮点数操作函数）。
对于简短的函数使用__inline修饰（）。
内联函数
内联函数禁用所有的编译选项。使用__inline修饰函数导致函数在调用处直接替换为函数体。这样代码调用函数更快，但增加代码的大小，特别在函数本身比较大而且经常调用的情况下。
__inline int square(int x) {
   return x * x;
}

#include <MATH.H>

double length(int x, int y){
    return sqrt(square(x) + square(y));
}
使用内联函数的好处如下：
没有函数调用负担。函数调用处直接替换为函数体，因此没有诸如读取寄存器变量等性能消耗。
更小的参数传递消耗。由于不需要拷贝变量，传递参数的消耗更小。如果参数是常量，编译器可以提供更好的优化。
内联函数的缺陷是如果调用的地方很多，代码的体积会变得很大。这主要取决于函数本身的大小和调用的次数。
仅对重要的函数使用inline是明智的。如果使用得当，内联函数甚至可以减少代码的体积：函数调用会产生一些计算机指令，但是使用内联的优化版本可能产生更少的计算机指令。
使用查找表
函数通常可以设计成查找表，这样可以显著提升性能。查找表的精确度比通常的计算低，但对于一般的程序并没什么差异。
许多信号处理程序（例如，调制解调器解调软件）使用很多非常消耗计算性能的sin和cos函数。对于实时系统，精确性不是特别重要，sin、cos查找表可能更合适。当使用查找表时，尽可能将相似的操作放入查找表，这样比使用多个查找表更快，更能节省存储空间。
浮点运算
尽管浮点运算对于所有的处理器都很耗时，但对于实现信号处理软件时我们仍然需要使用。在编写浮点操作程序时，记住如下几点：
浮点除法很慢。浮点除法比加法或者乘法慢两倍。通过使用常量将除法转换为乘法（例如，x=x/3.0可以替换为x=x*(1.0/3.0)）。常量的除法在编译期间计算。
使用float代替double。Float类型的变量消耗更好的内存和寄存器，并由于精度低而更加高效。如果精度够用，尽可能使用float。
避免使用先验函数。先验函数，例如sin、exp和log是通过一系列的乘法和加法实现的（使用了精度扩展）。这些操作比通常的乘法至少慢十倍。
简化浮点运算表达式。编译器并不能将应用于整型操作的优化手段应用于浮点操作。例如，3*(x/3)可以优化为x，而浮点运算就会损失精度。因此，如果知道结果正确，进行必要手工浮点优化是有必要的。
然而，浮点运算的表现可能不能满足特定软件对性能的需求。这种情况下，最好的办法或许是使用定点算数运算。当值的范围足够小，定点算数操作比浮点运算更精确、更快速。
其他技巧
通常，可以使用空间换时间。如果你能缓存经常用的数据而不是重新计算，这便能更快的访问。比如sine和cosine查找表，或者伪随机数。
尽量不在循环中使用++和–。例如：while(n–){}，这有时难于优化。
减少全局变量的使用。
除非像声明为全局变量，使用static修饰变量为文件内访问。
尽可能使用一个字大小的变量（int、long等），使用它们（而不是char，short，double，位域等）机器可能运行的更快。
不使用递归。递归可能优雅而简单，但需要太多的函数调用。
不在循环中使用sqrt开平方函数，计算平方根非常消耗性能。
一维数组比多维数组更快。
编译器可以在一个文件中进行优化-避免将相关的函数拆分到不同的文件中，如果将它们放在一起，编译器可以更好的处理它们（例如可以使用inline）。
单精度函数比双精度更快。
浮点乘法运算比浮点除法运算更快-使用val*0.5而不是val/2.0。
加法操作比乘法快-使用val+val+val而不是val*3。
put()函数比printf()快，但不灵活。
使用#define宏取代常用的小函数。
二进制/未格式化的文件访问比格式化的文件访问更快，因为程序不需要在人为可读的ASCII和机器可读的二进制之间转化。如果你不需要阅读文件的内容，将它保存为二进制。
如果你的库支持mallopt()函数（用于控制malloc），尽量使用它。MAXFAST的设置，对于调用很多次malloc工作的函数由很大的性能提升。如果一个结构一秒钟内需要多次创建并销毁，试着设置mallopt选项。
最后，但是是最重要的是-将编译器优化选项打开！看上去很显而易见，但却经常在产品推出时被忘记。编译器能够在更底层上对代码进行优化，并针对目标处理器执行特定的优化处理。
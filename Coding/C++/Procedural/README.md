<!--
 * @Author: Shuai Wang
 * @Github: https://github.com/wsustcid
 * @Version: 0.0.0
 * @Date: 2022-04-18 10:24:26
 * @LastEditTime: 2022-04-23 11:25:19
-->
- [2 面向过程编程](#2-面向过程编程)
  - [2.1 函数编写规则](#21-函数编写规则)
  - [2.2 inline函数](#22-inline函数)
  - [2.3 重载函数](#23-重载函数)
  - [2.4 模板函数](#24-模板函数)
  - [2.5 函数指针](#25-函数指针)
  - [2.6 头文件](#26-头文件)

## 2 面向过程编程
面向过程是一种以事件为中心的编程思想，编程时把要解决问题的步骤分析出来，然后通过具体函数把这些步骤逐一实现，最后再一步一步的按照具体步骤按顺序逐个调用函数，最终完成整个操作流程。

以下五子棋问题为例，面向过程的设计思路是首先分析解决这个问题的步骤：
  - （1）开始游戏（2）黑子先走（3）绘制画面（4）判断输赢（5）轮到白子（6）绘制画面（7）判断输赢（8）返回步骤（9）输出最后结果。
  - 以上每个步骤都可以编写成一个个的子函数，最后在下五子棋的主函数里依次调用上面的函数
 
因此，面向过程的编程始终关注的是怎么一步一步地解决问题，通过编写函数，从而实现解决步骤的顺序执行。显然，函数是面向过程编程的核心，本章我们就来深入的探讨函数的编写规则，并简要讨论重载、模板函数以及函数指针的使用技巧。

### 2.1 函数编写规则
**函数的声明与定义**
  - 函数声明不必提供函数体，只需要指明返回类型、函数名、参数列表即可，即函数原型：`int fibo(int pos);`
    - 函数必须先被声明，然后才能被调用
    - 函数声明让编译器得以检查后续出现的使用方式是否正确，比如是否输入了足够的参数、参数的类型是否正确等；
  - 函数定义包括函数原型+函数体：注意，在函数体内，函数只能返回一个值

**传值与传址**  
当我们调用一个函数时，会在内存中建立起一块特殊区域，称为“程序堆栈”，这块特殊区域提供了每个函数参数的存储空间，及函数内新定义对象的内存空间，这些对象被称为 `局部对象`。一旦函数调用结束，这块内存就会被释放掉，局部对象也就不复存在
  - 当我们使用常规方式将对象通过函数参数传入时，默认情形下其值会被赋值一份，创建一个新的局部对象，这种方式称为 **传值**：即在函数外部传给函数参数的对象和函数内部实际操作的对象，二者之间没有任何的关系
  - 要让二者产生关联的方式是 **传址**：即将参数声明为一个引用 `int &elem`：当我们以传址的方式将对象作为函数参数传入时，对象本身并不会复制出一份，复制的是对象的地址，因此我们在函数内部对该对象进行的任何操作，都是相当于通过地址，直接操作的原始对象。这样做的好处除了某些功能上的需求以外，还可以大大降低复制大行对象所带来的额外的负担

**传址的注意事项**  
  - 虽然称之为传址，但引用名并不是地址。对引用的操作，就等价于操作该引用所代表的对象，因为引用，其实就是对象的别名，所以可以简单的理解为传引用
  - 在c++中，引用一旦确定，便不可修改，不可以改变某个引用所代表的对象
  - 进一步的，虽然使用了传址，但又不想函数对该对象进行修改，则可以声明一个定常引用`const vector<int> &vec`
  - 同时既然传引用就是传址，当然也可以直接显式的进行真正的传址：`const vector<int> *vec`, 效果是一样的，只是具体用法会有所不同: 
    - 比如使用时需要先提领`(*vec)[idx]`，提领前先判断地址有效性
    - 实际调用时传递给参数的需要是对象的地址而非对象名`fun(&vec)`

**作用域及范围**  
之前已经讲了程序堆栈的概念，除了static对象，函数内部定义的对象，只存在于函数执行期间，因此如果将这些局部对象的地址返回，将会在运行时出错
  - 为对象分配的内存，其存活时间称为存储期或范围。每次函数执行，都会为其内部对象分配内存，函数调用结束时释放内存，我们称函数内部的对象具有 局部性范围 (local extent)
  - 对象在程序内的存活区域称为该对象的scope (作用域)，比如函数内定义的对象，仅在该函数内部存活，具有`local scope`, 在函数外声明的对象，则具有 `file scope`, 从其声明点至文件末尾都是可见的

注意，对于内置类型的对象，如果定义在file scope之内，则必定被初始化为0. 但如果被定义域local scope之内，除非指定初值，否则不会被初始化

**局部静态对象**
和局部非静态对象不同，局部静态对象(local static object)所处的内存空间，即使在不同的函数调用过程中，依然持续存在，不会像之前提到的局部对象那样，函数调用一次，生成并释放一次，而是可以在之前基础上继续使用

**动态内存管理**  
不论是 local scope 还是 file scope，其内存都是由系统自动管理的。还有一种存储期形式称为 dynamic extent (动态范围)，其内存由程序的空闲空间分配而来，也称作`堆内存`，这种内存由程序员自行管理，分配和释放分别通过 `new` 和 `delete` 表达式实现
  - 使用new从heap分配来的对象，具有dynamic extent,可以持续存活，直到以delete表达式释放为止
  - 注意，如果不使用delete表达式，从heap分配而来的对象就永远不会被释放，这称为 memory leak （内存泄露）

  ```c++
  // 内存分配 
  int *pi = new int; // 先由heap分配出一个int类型的对象，再将其地址赋值给pi; 这里并没有指定初值
  pi = new int(10); // 同样先由heap分配除一个int类型的对象，将其地址赋值给pi，同时将此对象的值初始化为10，注意是对象的值，并不是对象的地址
  int *pia = new int[24]; //从heap中分配数组包含24个整数的数组，pia初始化为数组第一个元素的地址，数组中每个元素的值并未初始化

  // 内存释放
  delete pi; // 释放pi所指向的对象
  delete [] pia; // 删除数组中的所有对象
  ```

**默认参数值**  
c++ 允许为全部或部分参数设定默认值，这样可以在函数调用时根据不同情况采用默认值，或使用不同的参数从而实现特定的效果,如 `void display(string msg, ostream &os=cout)`, 默认将输出传给cout, 实际调用也可以提供输出到文件的对象
  - 如果为某个参数提供了默认值，这一参数右侧的所有参数都必须要有默认值：因为默认值的解析是从最右边开始进行
  - 注意！默认值只能够指定一次：可以在函数声明处，也可以在函数定义处，但不能两个地方都指定
    - 因为函数声明一般都会放在头文件，每个打算使用该函数的文件，都需要包含该头文件
    - 函数定义放在程序的代码文件，该文件只被编译一次，当我们想要使用该函数时，会将它链接到我们的程序中来)，也就是说，头文件为函数带来更高的可见性
  - 因此，为了更高的可见性，我们一般将默认值放在函数声明处
  

### 2.2 inline函数
有时为了逻辑的清晰性和代码可读性，我们会将不同的小功能封装成一个个的小函数，然后在同一个大函数中逐一调用。但函数的多次调用会带来性能的损失，inline函数可以克服这一缺点。

将函数声明为inline，表示要求编译器在每个函数的调用点上，将函数的内容展开。因此面对一个inline函数，编译器可将该函数的调用操作改为以一份函数代码的副本代替，这将获得性能改善，使得在实现将多个函数写入同一个大函数的同时，依然维持多个独立的运算单元。

**inline函数定义方式**
  - 具体使用方式为在函数前面加上关键字inline，如 `inline bool fibon_elem(int pos, int &elem)`
  - 通常，体积较小、常被调用，且所从事的计算并不复杂，这类函数最适合被声明为inline函数
  - inline函数的定义，通常被放在头文件中


### 2.3 重载函数
当面临多个功能类似，可能只是输入参数的类型、数量有细微差别时，按照传统的函数定义方法，就需要声明并定义不同的函数，这个是可以理解的，毕竟处理方式是不同的，但真的需要使用不同的函数名称吗？根据参数列表的不同不就可以区分了吗？这样调用时岂不是更为方便，而无需单独记忆每个函数的名称，这就是**函数重载**：
  - 函数重载机制规定参数列表不同(可以是参数类型不同，也可以是参数个数不同)的两个或多个函数，可以拥有相同的函数名称
  - 在实际调用时，编译器将根据调用者提供的实际参数拿来和每个重载函数的参数进行比对，找出其中符合的进行调用，因此，每个重载函数的参数列表必须和其他重载函数不同！

**重载函数的返回类型**
  - 对于重载函数，不要试图通过给予不同的返回值类型（参数列表完全相同）加以区分。因为函数在调用时可以不将返回值赋值给一个对应的变量，这样就没有特征区分。
  - 因此，在进行函数重载时，函数返回类型可以相同也可以不同，***要保证的是函数名称必须相同，参数列表必须不同***！


### 2.4 模板函数
有时也会面临重载函数之间的差异性也非常小的情况，比如除了参数类型不同以外，函数体对数据的处理过程基本一致，这时使用函数重载仅仅是减轻了调用时的麻烦，进行函数定义时仍然进行了多次内容相似的定义，万一还有其他类型，还要不同的复制粘贴函数定义，然后做一点小修改。
  - 那么有没有一种机制，可以将单一函数的内容与希望处理的各种不同的参数类型绑定起来呢？这就是函数模板(function template):
  - 函数模板将参数列表中指定的全部(或部分)参数的类型信息抽离出来，在实际调用时再由用户进行具体指定

**模板函数定义**  
函数模板以关键字 template 开场，其后紧接用尖括号包围起来的一个或多个标识符，表示希望推迟决定的数据类型：
  - 这些标识符事实上扮演者占位符的角色，用来放置函数参数列表即函数体中的某些实际**数据类型**。用户每次利用这一模板产生函数时，都必须提供实际的类型信息
  - 借助函数模板，我们可以通过它产生无数函数，elemType可以被绑定为任意内置类型或用户自定义类型

  ```c++
  // 关键字 typename 后接 elemType表示 elemType在display_msg函数中只是一个暂时放置类型的占位符
  template <typename elemType> 
  void display_msg(const string &msg, const vector<elemType> &vec);

  // 函数重载和函数模板也可以同时使用
  template <typename elemType> 
  void display_msg(const string &msg, const list<elemType> &lt);
  ```


### 2.5 函数指针
使用函数指针来进一步增加函数的灵活性。所谓函数指针，从字面理解就是指向函数的指针，和指向普通数据类型的指针`int *pr`相比，多了参数列表，代表它指向的是一个函数：
  - 函数指针的定义包含：函数的返回类型、参数列表、指针符号*、以及这个函数指针的名称： `const vector<int>* (*seq_ptr)(int)`; 
  - 注意，指针和指针名必须括在一起，否则就会和返回值类型结合，变成一个返回值类型为指针的指针 的普通函数定义
  - 这样我们就可以用这个函数指针用作参数，将其指向不同的函数，实现灵活调用
  - 具体调用时如何传入函数地址？函数名即是函数的地址

  ```c++
  // 使用函数指针作为函数参数，方便在其内部调用不同函数
  bool seq_elem(int pos, int &elem, const vector<int>* (*seq_ptr)(int))
  {
    const vector<int>* pseq = seq_ptr(pos); // 由函数指针指向的函数，调用方式和普通函数相同
    // 当前为了安全起见，在调用前可以先检验地址有效性 if (seq_ptr)
    if (!pseq){elem = 0; return false};
    elem = (*pseq)[pos-1];
    return true;
  }
  // 也可以在参数列表中给函数指针赋初值 const vector<int>* (*seq_ptr)(int)=0; 代表不指向任何函数
  ```

**函数指针数组**  
为了可以不断指定多个函数，可以定义一个存放函数指针的数组
```c++
  const vector<int>* (*seq_array[])(int) = {fibon_seq, lucas_seq, pell_seq};
  // 在使用时直接索引即可
  seq_ptr = seq_array[idx];

  // 但索引不方便记忆，我们可以使用枚举实现常量到索引的映射
  enum ns_type {ns_fibon, ns_lucas, ns_pell}; // enum之后的标识符可有可无
  // 第一个枚举值为0，第二个为1，以此类推
  seq_ptr = seq_array[ns_fibon];
```


### 2.6 头文件
在上一节中定义的`seq_elem`函数，在调用该函数前，必须声明他，但如果要在五个程序文件中调用，就必须进行五次声明操作。为了避免分别进行五次声明，可以将函数声明统一放在头文件中，这样在每个程序代码文件中只需要include该头文件，就包含了这些函数声明
  - 使用头文件统一管理函数声明，只需要维护一份声明即可，如果其参数列表或返回类型需要改变，也只需要更改此份声明即可
  - 头文件的扩展名为`.h`，标准库没有扩展名

注意，因为函数的定义只能有一份，因此不能把函数定义放如头文件，否则当多个代码文件都包含这个头文件时会引发重复定义
  - 但inline函数是个例外：为了能够扩展inline函数的内容，在每个调用点上，编译器都需要取得其定义并将其展开，因此必须将inline函数定义在头文件中


**头文件内对象的声明**  
在file scope内定义的对象，如果可能被多个文件访问，同样也可以放在头文件中。但由于对象的定义就像函数一样，只能定义一次，如果在头文件中用普通的定义方式定义，就会被当做定义，之后就会引发重复定义。因此要在头文件内对对象进行声明而非定义，声明方式为加关键字 `extern`
  - 函数指针数组对象的声明： `extern const vector<int>* (*seq_array[seq_cnt])(int)`
  - const object的声明并不用extern: `const int seq_int=6;` 这是一个例外约定，约定const object只要一出文件便不可见，因此允许在多个程序代码中多次定义
  - 那刚才的指针数组就不是const object 吗？当然不是！它是一个指向const object的指针，但它本身并不是一个const object;

**头文件的包含规则**  
  - 如果头文件和包含此文件的程序代码文件位于同一磁盘目录下，我们便使用双引号`#include "NumSeq.h"`
  - 如果在不同的磁盘目录下，用尖括号`<>`：这样此文件会被认定为标准的或项目专属的头文件，编译器会现在默认的磁盘目录中寻找对应文件；
  - 否则认为是由用户提供的，会由包含此文件的文件所在的磁盘目录开始找起

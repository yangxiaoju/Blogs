# extern, static, const 和 inline

在使用 Objective-C 编程的时候，偶尔也会使用到 C 语言的一些特性，extern、static、const 和 inline 这四个关键词是我对于其含义较为模糊的四个关键词，特写一篇笔记用来温习一下。

## extern

要理解 extern 关键字的作用，首先要明白声明和定义的区别。定义变量时，编译器为该变量分配内存，并可能还将其内容初始化为某个值。声明变量时，编译器要求在其他地方定义变量。声明通知编译器存在由该名称和类型的变量，但编译器不需要为其分配内存，因为它是在其他地方分配的。extern 关键字表示“声明而不定义”。换句话说，它是一种显式声明变量或强制声明而无需定义的方法。

必须在程序的一个模块中精确定义一个变量。 如果没有定义或多于一个定义，则可能在链接阶段产生错误。 变量可以被声明多次，只要声明彼此一致并且与定义一致（头文件很有用）。 它可以在许多模块中声明，包括定义它的模块，甚至在同一模块中多次。 但是在模块中多次声明它通常是没有意义的。

extern 变量也可以在函数内声明。 在这种情况下，必须使用 extern 关键字，否则编译器会将其视为本地（自动）变量的定义，该变量具有不同的范围，生命周期和初始值。 此声明仅在函数内部可见，而不是在整个函数模块中可见。

应用于函数原型的 extern 关键字绝对没有任何内容（应用于函数定义的 extern 关键字当然是非语义的）。 函数原型始终是一个声明，而不是一个定义。 此外，在标准 C 中，函数始终是外部的，但某些编译器扩展允许在函数内定义函数。

### Example:

File 1:
```objc
// Explicit definition, this actually allocates
// as well as describing
int Global_Variable;

// Function prototype (declaration), assumes 
// defined elsewhere, normally from include file.       
void SomeFunction(void);        

int main(void) {
    Global_Variable = 1;
    SomeFunction();
    return 0;
} 
```

File 2:
```objc
// Implicit declaration, this only describes and
// assumes allocated elsewhere, normally from include
extern int Global_Variable;  

// Function header (definition)
void SomeFunction(void) {       
    ++Global_Variable;
}
```

在此示例中，变量 Global_Variable 在 File 1 中定义。为了在 File 2 中使用相同的变量，必须声明它。 无论文件数量多少，全局变量只定义一次; 但是，它必须在包含定义的文件之外的任何文件中声明。

通常的方法是分配和实际定义进入 .c 文件，但仅仅声明和原型不分配，只是描述类型和参数，以便编译器可以正常工作，并且该信息属于 .h 头文件，其他人可以安全地 include 没有任何可能的冲突。

## static

在 C 编程语言（及其紧密的后代，如 C++ 和 Objective-C）中，static 是一个保留字，用于控制生命周期（作为静态变量）和可见性（取决于链接）。

在声明变量或函数时作为前缀的 static 关键字可能具有其他效果，具体取决于声明发生的位置。

### Static global variable

在源文件的顶层（在任何函数定义之外）声明为 static 的变量仅在整个文件中可见。 在此用法中，关键字 static 称为“访问说明符”。

### Static function

类似地，静态函数 - 在源文件的顶层（在任何类定义之外）声明为静态的函数 - 仅在整个文件中可见。

### Static local variables

在函数内声明为静态的变量是静态分配的，因此在整个程序执行期间保持其内存单元，同时具有与自动局部变量（auto 和 register）相同的可见范围，这意味着保持函数的本地性。 因此，当一次调用时函数放入其静态局部变量的任何值在再次调用函数时仍然存在。

## const

在 C 语言中，const 是类型的一部分，而不是对象的一部分。例如，在 C 中，const int x = 1; 声明一个 const int 类型的对象 x  -  const 是该类型的一部分，就好像它被解析为“（const int）x”

这有两个微妙的结果。 首先，const 可以应用于更复杂类型的部分 - 例如，int const * const x; 声明一个指向常量整数的常量指针，而 int const * x; 声明一个指向常量整数的变量指针，并且 int * const x; 声明一个指向变量整数的常量指针。 其次，因为 const 是类型的一部分，所以它必须作为类型检查的一部分匹配。 

虽然常量在程序运行时不会更改其值，但是在程序运行时，声明为 const 的对象确实可能会更改其值。 一个常见的例子是嵌入式系统中的只读寄存器，如数字输入的当前状态。 数字输入的数据寄存器通常声明为 const 和 volatile。 这些寄存器的内容可能会在程序执行任何操作（volatile）时发生变化，但你也不应该向它们写入（const）。

### 使用方式

在 C 中，所有数据类型（包括用户定义的数据类型）都可以声明为 const，而 const-correctness 规定所有变量或对象都应该声明为这样，除非需要修改它们。 这种对 const 的主动使用使得值“更容易理解，跟踪和推理”，从而提高了代码的可读性和可理解性，使团队工作和维护代码更简单，因为它传达了有关值的预期用途的信息。 在推理代码时，这可以帮助编译器和开发人员。 它还可以使优化编译器生成更高效的代码。

#### 简单的数据类型

对于简单的非指针数据类型，应用 const 限定符很简单。 由于历史原因，它可以在类型的任何一侧出现（即，const char foo ='a';等同于 char const foo ='a';）。 在某些实现中，在类型的两侧使用 const（例如，const char const）会生成警告但不会生成错误。

#### 指针和引用

对于指针和引用类型，const 的含义更复杂 - 指针本身或指向的值或两者都可以是 const。此外，语法可能令人困惑。指针可以声明为指向可写值的 const 指针，或指向 const 值的可写指针，或指向 const 值的 const 指针。 const 指针不能重新分配以指向与最初分配的对象不同的对象，但它可用于修改它指向的值（称为指针对象）。因此，引用变量是 const 指针的替代语法。另一方面，指向 const 对象的指针可以重新分配以指向另一个内存位置（应该是相同类型或可转换类型的对象），但它不能用于修改它的内存指向还可以声明指向 const 对象的 const 指针，既不能用于修改指针，也不能重新指定指向另一个对象。以下代码说明了这些细微之处：

```objc
void Foo( int * ptr,
          int const * ptrToConst,
          int * const constPtr,
          int const * const constPtrToConst )
{
    *ptr = 0; // OK: modifies the "pointee" data
    ptr  = NULL; // OK: modifies the pointer

    *ptrToConst = 0; // Error! Cannot modify the "pointee" data
    ptrToConst  = NULL; // OK: modifies the pointer

    *constPtr = 0; // OK: modifies the "pointee" data
    constPtr  = NULL; // Error! Cannot modify the pointer

    *constPtrToConst = 0; // Error! Cannot modify the "pointee" data
    constPtrToConst  = NULL; // Error! Cannot modify the pointer
}
```

遵循通常的 C 约定声明，声明遵循使用，并且指针中的 * 写在指针上，表示解除引用。 例如，在声明 int *ptr 中，解除引用的形式 *ptr是一个 int，而引用形式 ptr 是一个指向 int 的指针。 因此 const 将名称修改为右侧。

```objc
int *ptr; // *ptr is an int value
int const *ptrToConst; // *ptrToConst is a constant (int: integer value)
int * const constPtr; // constPtr is a constant (int *: integer pointer)
int const * const constPtrToConst; // constPtrToConst is a constant (pointer)
                                   // as is *constPtrToConst (value)
```

## inline

在 C 和 C++ 编程语言中，内联函数是使用关键字 inline 限定的函数;这有两个目的。首先，它充当编译器指令，建议（但不要求）编译器通过执行内联扩展替换函数体，即通过在每个函数调用的地址处插入函数代码，从而节省开销一个函数调用。在这方面，它类似于寄存器存储类说明符，它类似地提供了一个优化提示。内联的第二个目的是改变链接行为;这个细节很复杂。这是必要的，因为 C/C++ 单独的编译+链接模型，特别是因为函数的定义（主体）必须在使用它的所有转换单元中重复，以允许在编译期间内联，如果函数具有外部链接，在链接过程中导致冲突（它违反了外部符号的唯一性）。

内联函数可以用 C 或 C++ 编写，如下所示：

```objc
static inline void swap(int *m, int *n)
{
  int temp = *m;
  *m = *n;
  *n = temp;
}
```

然后，声明如下：

```objc
swap(&x, &y);
```

可以被翻译成（如果编译器决定进行内联，通常需要启用优化）：

```objc
int temp = x;
x = y;
y = temp;
```

当实现执行大量交换的排序算法时，这可以提高执行速度。
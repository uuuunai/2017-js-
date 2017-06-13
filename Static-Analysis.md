# 静态分析
Leah Hanson
> Leah Hanson是一名骄傲的黑客学校校友，他喜欢帮助人们了解Julia。他的博客地址是http://blog.leahhanson 以及推文 @astrieanna

## 简介
您可能熟悉一个在代码部分不编译的复杂的IDE。您可能已经在代码中运行了一个linter，以检查格式或样式问题。您可能会在所有警告打开的情况下运行您的编译器。所有这些工具都是静态分析的应用程序。

静态分析是在不运行代码的情况下检查代码中的问题的一种方法。“静态”指的是编译时而不是运行时，而“分析”意味着我们正在分析代码。当你使用我上面提到的工具时，它可能就像魔法一样。但是这些工具只是程序，它们是由一个人编写的，像你这样的程序员编写的。在本章中，我们将讨论如何实现一些静态分析检查。为了做到这一点，我们需要知道我们想要做什么，以及我们想要做什么。

我们可以通过把这个过程描述成三个阶段来了解你需要知道的东西。

### 1、决定你想要检查的东西。

您应该能够解释您想要解决的一般问题，就像编程语言的用户所能识别的那样。例子包括:

* 发现拼写错误的变量名
* 在并行代码中查找竞争条件
* 查找未实现函数的调用

### 2、决定如何检查它。

 虽然我们可以让朋友做上面列出的任务之一，但他们还不够具体，无法向电脑解释。例如，为了处理“拼写错误的变量名”，我们需要决定这里的拼写错误。一种选择是声明变量名应该由字典中的英语单词组成;另一种选择是查找只使用一次的变量(在一次错误的情况下)。

 如果我们知道我们在寻找只使用一次的变量，我们可以讨论各种变量的用法(它们的值被赋值与读取)，以及哪些代码将不会触发警告。

 ### 实现细节

 这包括编写代码的实际操作，花时间阅读您所使用的库的文档，以及如何获取您需要编写分析所需的信息。这可能涉及到读入一个代码文件，解析它以了解结构，然后对该结构进行详细的检查。

 我们将为这一章中实现的每一个单独的检查而工作。第1步需要对我们分析的语言有足够的理解，以同情它的用户所面临的问题。这一章的所有代码都是茱莉亚代码，用来分析茱莉亚的代码。

## 对朱莉娅的简要介绍

 茱莉亚是一门针对技术计算的年轻语言。它在2012年春的版本0.1中发布;到2015年初，它已经达到了0.3版。总的来说，朱莉娅看起来很像Python，但是有一些可选的类型注解，没有任何面向对象的东西。大多数程序员在朱莉娅中发现的新特性是多重分派，它对API设计和语言的其他设计选择都有广泛的影响。

 以下是茱莉亚的代码片段:

 ```
# A comment about increment
function increment(x::Int64)
  return x + 1
end

increment(5)
```

这段代码定义了函数增量的一种方法，该方法使用了一个名为x的参数，名为Int64。该方法返回x+1的值。然后，这个新定义的方法用值5调用;函数调用，正如您可能已经猜到的，将值为6。

Int64是一种类型，它的值是在内存中由64位表示的整数;如果您的计算机有64位处理器，则是您的硬件理解的整数。除了影响方法调度外，朱莉娅还定义了内存中的数据表示。

名称增量指的是一个泛型函数，它可能有很多方法。我们刚刚定义了一种方法。在许多语言中，术语“函数”和“方法”是可互换的;在朱莉娅中，它们具有不同的含义。如果您小心地理解“函数”作为一个命名的方法集合，那么这一章将更有意义，因为“方法”是特定类型签名的具体实现。

让我们定义另一个增量函数的方法:

 ```
# Increment x by y
function increment(x::Int64, y::Number)
  return x + y
end

increment(5) # => 6
increment(5,4) # => 9
```

现在函数增量有两种方法。朱莉娅决定根据参数的数量和类型来为给定的调用运行哪种方法;这称为动态多分派:

动态是因为它基于运行时使用的值的类型。

因为它查看了所有参数的类型和顺序。

调度，因为这是一种将函数调用与方法定义相匹配的方法。

要将其放在您可能已经知道的语言环境中，面向对象的语言使用单一分派，因为它们只考虑第一个参数。(在x.foo(y)中，第一个参数是x。

单个和多个分派都基于参数的类型。x::Int64是纯粹用于分派的类型注释。在朱莉娅的动态类型系统中，您可以在函数中分配任意类型的值，而不会出现错误。

我们还没有看到“多”的部分，但是如果你对茱莉亚很好奇的话，你就得自己去看看。我们需要继续进行第一次检查。

## 检查循环中的变量类型

和大多数编程语言一样，在朱莉娅编写非常快的代码时，需要理解计算机是如何工作的，以及茱莉亚是如何工作的。帮助编译器为您创建快速代码的一个重要部分是编写类型稳定的代码;这在朱莉娅和JavaScript中很重要，并且在其他JIT语言中也很有用。当编译器可以看到代码段中的变量总是包含相同的类型时，编译器可以做更多的优化，而不是相信(正确与否)，这个变量有多种可能的类型。您可以阅读更多关于为什么类型稳定性(也称为“单态”)对于JavaScript在线来说很重要的原因。

### 为什么这很重要

让我们写一个函数，它需要一个Int64，并增加它的值。如果这个数很小(小于10)，让我们用一个大的数(50)来增加它，但是如果它很大，我们只把它增加到0。5。

 ```
function increment(x::Int64)
  if x < 10
    x = x + 50
  else
    x = x + 0.5
  end
  return x
end
```

这个函数看起来很简单，但是x的类型是不稳定的。我选择了两个数字:50，一个Int64，和0.5，一个浮动64。取决于x的值，它可能会被添加到其中一个。如果你把一个像22这样的Int64添加到一个浮动64，就会得到一个浮动64(22。5)。因为函数(x)中的变量类型可以根据函数(x)的参数值而变化，这种增量的方法，特别是变量x是类型不稳定的。

浮动64是一种表示存储在64位的浮点值的类型;在C中，它被称为double。这是64位处理器所理解的浮点类型之一。

与大多数效率问题一样，这个问题在循环过程中更明显。for循环和while循环的代码多次运行，因此使其快速运行比只运行一次或两次的代码更重要。因此，我们的第一个检查是寻找在循环中有不稳定类型的变量。

首先，让我们来看一个我们想要捕捉的例子。我们将看两个函数。每个数都是1到100，但不是求和，而是把每个数除以2，然后求和。两个函数将得到相同的答案(2525.0);两者都将返回相同的类型(浮动64)。然而，第一个函数是不稳定的，它会受到类型不稳定的影响，而第二个稳定的函数则不稳定。
 ```
function unstable()
  sum = 0
  for i=1:100
    sum += i/2
  end
  return sum
end
 ```
 
 ```
function stable()
  sum = 0.0
  for i=1:100
    sum += i/2
  end
  return sum
end
 ```

这两个函数之间唯一的文本区别是在sum的初始化:sum=0和sum=0.0。在朱莉娅中，0是一个Int64文字，而0.0是一个浮动64字。这种微小的变化能带来多大的不同呢?

因为朱莉娅是即时(JIT)编译的，所以函数的第一次运行要比后续运行要长。(第一次运行包括为这些参数类型编译函数所需的时间。)当我们对函数进行基准测试时，我们必须确保在计时之前运行它们(或者预编译它们)。

 ```
julia> unstable()
2525.0

julia> stable()
2525.0

julia> @time unstable()
elapsed time: 9.517e-6 seconds (3248 bytes allocated)
2525.0

julia> @time stable()
elapsed time: 2.285e-6 seconds (64 bytes allocated)
2525.0
 ```

@time宏打印出该函数运行了多长时间，以及在运行时分配了多少字节。每当需要新的内存时，所分配的字节数就会增加;当垃圾收集器将不再使用的内存占用内存时，它不会减少。这意味着分配的字节与分配和管理内存的时间有关，但这并不意味着我们在同一时间使用了所有的内存。

如果我们想要得到稳定的和不稳定的数字，我们需要使这个循环长很多，或者多次运行这个函数。然而，看起来不稳定可能会更慢。更有趣的是，我们可以看到分配的字节数有很大的差距;不稳定已经分配了大约3 KB的内存，其中稳定使用64字节。

由于我们可以看到不稳定是多么简单，我们可能会猜测这个分配是在循环中发生的。为了测试这一点，我们可以让循环更长的时间，看看分配是否相应地增加。让我们将循环从1增加到10000，这是100倍的迭代;我们将寻找分配的字节数增加大约100倍，达到大约300 KB。
 ```
 function unstable()
  sum = 0
  for i=1:10000
    sum += i/2
  end
  return sum
end
 ```
因为我们重新定义了函数，所以我们需要运行它，以便在我们度量它之前编译它。我们期望从新的函数定义中得到一个不同的，更大的答案，因为它现在是对更多的数求和。

 ```
julia> unstable()
2.50025e7

julia>@time unstable()
elapsed time: 0.000667613 seconds (320048 bytes allocated)
2.50025e7
 ```


新的不稳定分配了大约320 KB，这是如果分配在循环中发生的情况。为了解释这里发生了什么，我们来看看茱莉亚是如何工作的。

不稳定和稳定之间的差异是因为不稳定的和必须被装箱，而稳定的总和可以被解除。框值由类型标记和表示值的实际位组成;无框值只有实际的比特。但是类型标签是很小的，所以这不是为什么拳击价值会分配更多的内存。

不同之于编译器所能做的优化。当一个变量有一个具体的、不可变的类型时，编译器可以在函数中打开它。如果不是这样，那么变量必须在堆上分配，并参与到垃圾收集器中。不可变类型是特定于茱莉亚的概念。不可变类型的值是不可更改的。

不可变类型通常是表示值的类型，而不是值的集合。例如，大多数数值类型，包括Int64和浮动64，都是不可变的。(朱莉娅的数值类型是普通类型，而不是特殊的原始类型;您可以定义一个新的MyInt64，它与提供的是相同的)。因为不可修改的类型不能被修改，所以每次您想要更改时，都必须创建一个新的副本。例如，4+6必须创建一个新的Int64来保存结果。相比之下，可变类型的成员可以就地更新;这意味着您不必复制整个东西来进行更改。

x=x+2分配内存的想法听起来很奇怪，为什么要让这样的基本操作通过使Int64值变不可变而变慢呢?这就是那些编译器优化的地方:使用不可变类型(通常)不会减慢这个速度。如果x有一个稳定的、具体的类型(比如Int64)，那么编译器就可以在堆栈上分配x并在适当的位置进行x的变异。问题是只有当x不稳定类型(编译器不知道还是什么类型有多大);一旦盒装和堆x,编译器不完全确定,一些其他的代码没有使用价值,因此不能编辑它。

因为稳定的sum有一个具体的类型(浮动64)，编译器知道它可以在函数中存储它，并改变它的值;sum不会在堆上分配，并且每次我们添加i/2时都不需要重新复制。

因为不稳定中的sum没有具体类型，所以编译器会在堆上分配它。每次修改sum时，都会在堆上分配一个新值。所有这些花费在堆上的值(每次我们想要读取sum的值时都要检索它们)是很昂贵的。

使用0和0.0是一个很容易犯的错误，尤其是当你对茱莉亚来说是新的时候。自动检查循环中使用的变量是类型稳定的，这有助于程序员更深入地了解他们的变量在其代码的性能关键部分的类型。

### 实现细节

我们需要找出在循环中使用哪些变量，我们需要找到这些变量的类型。然后，我们将需要决定如何以人类可读的格式打印它们。

* 我们如何找到循环?
* 如何在循环中找到变量?
* 我们如何找到变量的类型呢?
* 我们如何打印结果?
* 我们如何判断这种类型是否不稳定?

我要先回答最后一个问题，因为这整个的努力都取决于它。我们研究了一个不稳定的函数，作为程序员，我们看到了如何识别一个不稳定的变量，但是我们需要我们的程序来找到它们。这听起来似乎需要模拟函数来寻找那些值可能会改变的变量，这听起来可能需要一些工作。幸运的是，朱莉娅的类型推断已经通过函数的执行来确定类型。

不稳定的总和是联合(浮动64，Int64)。这是一个工会类型，一种特殊的类型，它表明变量可以包含一组类型的值。类型Union的一个变量(浮动64，Int64)可以保存类型Int64或浮动64的值;一个值只能有一个类型。一个加入任何类型的工会类型(例如:工会类型(浮动64，Int64，Int32)加入了三种类型)。我们要寻找的是在循环中有工会类型的变量。

将代码解析为具有代表性的结构是一项复杂的业务，随着语言的发展，将变得更加复杂。在本章中，我们将依赖于编译器使用的内部数据结构。这意味着我们不必担心读取文件或解析它们，但这意味着我们必须处理那些不在我们控制范围内的数据结构，有时会感到笨拙或丑陋。

除了所有的工作我们将拯救自己不必解析代码,使用相同的数据结构,编译器使用意味着我们检查的准确评估将基于编译器的理解——意味着我们的检查将一致的代码如何运行。

这个从朱莉娅代码中检查朱莉娅代码的过程叫做内省。当你或我反思时，我们在思考我们的思考和感受的方式和原因。当代码进行检查时，它会检查同一语言中代码的表示或执行特性(可能是它自己的代码)。当代码的内省扩展到修改被检查的代码时，它被称为元编程(编写或修改程序的程序)。

### 内省的茱莉亚

朱莉娅很容易反省。我们构建了四个函数来让我们了解编译器在想什么:code降下、code类型化、codellvm和code本机。这些列在编译过程中，它们的输出来自于哪个步骤;第一个是与我们输入的代码最接近的，而最后一个是与CPU运行最接近的。在本章中，我们将重点讨论code类型化，它为我们提供了优化的、类型推断的抽象语法树(AST)。

code类型化有两个参数:感兴趣的函数和参数类型的元组。例如，如果我们希望在使用两个Int64s调用函数foo时看到AST，那么我们将调用code类型化(foo，(Int64，Int64))。

 ```
function foo(x,y)
  z = x + y
  return 2 * z
end

code_typed(foo,(Int64,Int64))
 ```
这是code类型化会返回的结构:

```
 1-element Array{Any,1}:
:($(Expr(:lambda, {:x,:y}, {{:z},{{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}},{}},
 :(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64))))
 ```
 
这是一个数组;这允许code类型化返回多个匹配的方法。函数和参数类型的一些组合可能不能完全确定应该调用哪种方法。例如，您可以传入类似于任何类型的类型(而不是Int64)。Any是类型层次结构顶部的类型;所有类型都是任何类型的子类型(包括任何类型)。如果我们在元组的参数类型中包含任何类型，并且有多个匹配的方法，那么来自code类型化的数组将包含多个元素;每个匹配方法都有一个元素。

让我们把我们的例子排除出去，让它更容易讨论。

 ``` 
 julia> e = code_typed(foo,(Int64,Int64))[1]
:($(Expr(:lambda, {:x,:y}, {{:z},{{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}},{}},
 :(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64))))
  ```
 
我们感兴趣的结构是在数组中:它是一个Expr。朱莉娅使用Expr(缩写为表达式)来表示AST。(抽象语法树是编译器对代码的含义的理解，这有点像你在小学时必须用图表来表示句子)。我们返回的Expr表示一个方法。它有一些元数据(关于方法中出现的变量)和组成方法主体的表达式。

现在我们可以问一些关于e的问题。

我们可以通过使用名称函数来询问一个Expr的属性，它适用于任何朱莉娅的值或类型。它返回由该类型定义的名称数组(或值的类型)。

``` 
 julia> names(e)
3-element Array{Symbol,1}:
 :head
 :args
 :typ 
``` 
我们只是问e的名字，现在我们可以问每个名字对应的值是多少。Expr有三个属性:头、typ和args。 
 
 ```
 julia> e.head
:lambda

julia> e.typ
Any

julia> e.args
3-element Array{Any,1}:
 {:x,:y}                                                                                                                                                                                     
 {{:z},{{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}},{}}                                                                                                                                         
 :(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64)
 ```
 

我们刚刚看到了一些打印出来的值，但这并不能告诉我们它们的含义，或者它们是如何被使用的。

head告诉我们这是什么类型的表达式;通常情况下，您将在朱莉娅中使用单独的类型，但是Expr是一种模型，用于对解析器中使用的结构进行建模。解析器是用Scheme的方言编写的，它将所有东西都构造为嵌套列表。head告诉我们这些Expr的其余部分是如何组织的以及它代表了什么类型的表达式。

typ是表达式的推断返回类型;当您对任何表达式求值时，它会产生一些值。typ是表达式将计算的值的类型。对于几乎所有的Exprs，这个值将是任意的(这总是正确的，因为每种可能的类型都是任何类型的子类型)。只有类型推断方法的主体和其中的大多数表达式将把它们的typ设置为更具体的东西。(因为type是一个关键字，所以这个字段不能使用这个词作为它的名字。)

args是Expr中最复杂的部分，它的结构根据头部的值而变化。它始终是一个数组(一个非类型化数组)，但除此之外，结构也发生了变化。

在一个表示方法的Expr中，在e.args中有三个元素:

``` 
 julia> e.args[1] # names of arguments as symbols
2-element Array{Any,1}:
 :x
 :y
``` 
符号是表示变量、常量、函数和模块的名称的一种特殊类型。它们与字符串是不同的类型，因为它们特别表示程序构造的名称。 
 ```
 julia> e.args[2] # three lists of variable metadata
3-element Array{Any,1}:
 {:z}                                     
 {{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}}
 {}    
 ```
上面的第一个列表包含所有局部变量的名称，这里只有一个(z)。第二个列表包含每个变量的元组和方法的参数;每个元组都有变量名、变量的推断类型和一个数字。这个数字传达了关于如何使用变量的信息，在机器(而不是人类)友好的方式中。最后一个列表是捕获变量名，在本例中是空的。
 ```
 julia> e.args[3] # the body of the method
:(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64)
  ```
 前两个args元素是关于第三个的元数据。虽然元数据非常有趣，但现在还没有必要。重要的部分是方法的主体，它是第三个元素。这是另一个Expr。 
  
```  
julia> body = e.args[3]
:(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64)

julia> body.head
:body  
```
这个Expr有头部:身体因为它是这个方法的主体。
```
 julia> body.typ
Int64 
```  
 typ是该方法的推断返回类型。 
```    
 julia> body.args
4-element Array{Any,1}:
 :( # none, line 2:)                                              
 :(z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64)
 :( # line 3:)                                                    
 :(return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64)    
```    


args包含一个表达式列表:方法体中的表达式列表。有一些行号的注释(例如(第3行:))，但是大部分的身体都在设置z的值(z=x+y)和返回2 z，注意到这些操作已经被一个特殊的内在函数代替了。顶部(函数名)表明了一个固有的函数;在朱莉娅的代码生成中实现了这个功能，而不是茱莉亚。

我们还没有看到一个循环是什么样子的，我们来试一下。
```  
julia> function lloop(x)
         for x = 1:100
           x *= 2
         end
       end
lloop (generic function with 1 method)

julia> code_typed(lloop, (Int,))[1].args[3]
:(begin  # none, line 2:
        #s120 = $(Expr(:new, UnitRange{Int64}, 1, :(((top(getfield))(Intrinsics,
         :select_value))((top(sle_int))(1,100)::Bool,100,(top(box))(Int64,(top(
         sub_int))(1,1))::Int64)::Int64)))::UnitRange{Int64}
        #s119 = (top(getfield))(#s120::UnitRange{Int64},:start)::Int64        unless 
         (top(box))(Bool,(top(not_int))(#s119::Int64 === (top(box))(Int64,(top(
         add_int))((top(getfield))
         (#s120::UnitRange{Int64},:stop)::Int64,1))::Int64::Bool))::Bool goto 1
        2: 
        _var0 = #s119::Int64
        _var1 = (top(box))(Int64,(top(add_int))(#s119::Int64,1))::Int64
        x = _var0::Int64
        #s119 = _var1::Int64 # line 3:
        x = (top(box))(Int64,(top(mul_int))(x::Int64,2))::Int64
        3: 
        unless (top(box))(Bool,(top(not_int))((top(box))(Bool,(top(not_int))
         (#s119::Int64 === (top(box))(Int64,(top(add_int))((top(getfield))(
         #s120::UnitRange{Int64},:stop)::Int64,1))::Int64::Bool))::Bool))::Bool
         goto 2
        1:         0: 
        return
    end::Nothing)  
 ``` 
  
你会注意到在身体里没有循环或while循环。当编译器将代码从我们编写的代码转换为CPU理解的二进制指令时，这些特性对人类很有用，但是CPU(如循环)不理解的特性将被删除。循环被重写为标签和后后表达式。后藤有一个数字，每个标签也有一个数字。后藤跳到标签上的数字是相同的。  
### 检测和提取循环  


我们将通过查找向后跳转的后藤表达式来找到循环。

我们需要找到标签和gotos，并找出哪些是匹配的。我将首先给出完整的实现。在代码墙之后，我们将把它拆开并检查这些部分。
 ```  
# This is a function for trying to detect loops in the body of a Method
# Returns lines that are inside one or more loops
function loopcontents(e::Expr)
  b = body(e)
  loops = Int[]
  nesting = 0
  lines = {}
  for i in 1:length(b)
    if typeof(b[i]) == LabelNode
      l = b[i].label
      jumpback = findnext(x-> (typeof(x) == GotoNode && x.label == l) 
                              || (Base.is_expr(x,:gotoifnot) && x.args[end] == l),
                          b, i)
      if jumpback != 0
        push!(loops,jumpback)
        nesting += 1
      end
    end
    if nesting > 0
      push!(lines,(i,b[i]))
    end

    if typeof(b[i]) == GotoNode && in(i,loops)
      splice!(loops,findfirst(loops,i))
      nesting -= 1
    end
  end
  lines
end
 ``` 
现在我们来解释一下:
 ```
b = body(e)
 ```
我们首先从方法体中获取所有的表达式，作为一个数组。身体是我已经实现的一个功能:
 ```
  # Return the body of a Method.
  # Takes an Expr representing a Method,
  # returns Vector{Expr}.
  function body(e::Expr)
    return e.args[3].args
  end
 ```
 然后
```
  loops = Int[]
  nesting = 0
  lines = {}
```
循环是一个标签行数的数组，其中的gotos是循环的。嵌套表示当前内部循环的数量。行是一个数组(索引，Expr)元组。
```
  for i in 1:length(b)
    if typeof(b[i]) == LabelNode
      l = b[i].label
      jumpback = findnext(
        x-> (typeof(x) == GotoNode && x.label == l) 
            || (Base.is_expr(x,:gotoifnot) && x.args[end] == l),
        b, i)
      if jumpback != 0
        push!(loops,jumpback)
        nesting += 1
      end
    end
```
我们查看e的每个表达式，如果它是一个标签，我们检查是否有一个跳转到这个标签(并且发生在当前索引之后)。如果findnext的结果大于零，那么就会存在这样的goto节点，因此我们将把它添加到循环中(我们当前所处的循环数组)，并增加我们的嵌套级别。
```
    if nesting > 0
      push!(lines,(i,b[i]))
    end
```
如果我们当前在一个循环中，我们将当前行推到我们的行数组中返回。
```
    if typeof(b[i]) == GotoNode && in(i,loops)
      splice!(loops,findfirst(loops,i))
      nesting -= 1
    end
  end
  lines
end
```



如果我们在GotoNode，我们会检查它是不是一个循环的结束。如果是这样，我们将从循环中删除条目并减少我们的嵌套级别。

这个函数的结果是行数组，一个数组(索引，值)元组。这意味着数组中的每个值都有一个索引到方法-body-expr的主体和该索引的值。行中的每个元素都是循环中出现的表达式。

### 发现和输入变量

我们刚刚完成了函数循环内容，它返回了内部循环中的Exprs。我们的下一个函数将是loosetypes，它获取一个Exprs列表，并返回一个松散类型的变量列表。稍后，我们将把loopcontents的输出传递到loosetypes。

在循环中出现的每个表达式中，loosetypes会搜索符号及其相关类型的出现。变量使用在AST中显示为symbolnode;symbolnode保留变量的名称和推断类型。

我们不能只检查循环内容收集到的每个表达式，看看它是不是一个SymbolNode。问题是每个Expr都可能包含一个或多个Expr;每个Expr可能包含一个或多个符号。这意味着我们需要提取任何嵌套的Exprs，这样我们就可以看到它们中的每一个都为symbolnode。
```
# given `lr`, a Vector of expressions (Expr + literals, etc)
# try to find all occurrences of a variables in `lr`
# and determine their types
function loosetypes(lr::Vector)
  symbols = SymbolNode[]
  for (i,e) in lr
    if typeof(e) == Expr
      es = copy(e.args)
      while !isempty(es)
        e1 = pop!(es)
        if typeof(e1) == Expr
          append!(es,e1.args)
        elseif typeof(e1) == SymbolNode
          push!(symbols,e1)
        end
      end
    end
  end
  loose_types = SymbolNode[]
  for symnode in symbols
    if !isleaftype(symnode.typ) && typeof(symnode.typ) == UnionType
      push!(loose_types, symnode)
    end
  end
  return loose_types
end
```

```
  symbols = SymbolNode[]
  for (i,e) in lr
    if typeof(e) == Expr
      es = copy(e.args)
      while !isempty(es)
        e1 = pop!(es)
        if typeof(e1) == Expr
          append!(es,e1.args)
        elseif typeof(e1) == SymbolNode
          push!(symbols,e1)
        end
      end
    end
  end
```
while循环会递归地遍历所有的Exprs。每当循环找到一个符号时，它就会把它添加到向量符号中。
```
  loose_types = SymbolNode[]
  for symnode in symbols
    if !isleaftype(symnode.typ) && typeof(symnode.typ) == UnionType
      push!(loose_types, symnode)
    end
  end
  return loose_types
end
```


现在我们有了一个变量列表和它们的类型，所以很容易检查类型是否松散。loosetype通过查找特定类型的非具体类型，即工会类型来完成这一工作。当我们认为所有的非具体类型都是“失败”时，我们会得到更多“失败”的结果。这是因为我们用带注释的参数类型来评估每个方法，这很可能是抽象的。

### 这可用

既然我们可以对表达式进行检查，那么我们就应该更容易调用用户的代码。我们将创建两种调用checkloop类型的方法:

* 在整个函数中，这将检查给定函数的每个方法。
* 在表达式中，如果用户提取代码类型的结果，这将是有效的。

```
## for a given Function, run checklooptypes on each Method
function checklooptypes(f::Callable;kwargs...)
  lrs = LoopResult[]
  for e in code_typed(f)
    lr = checklooptypes(e)
    if length(lr.lines) > 0 push!(lrs,lr) end
  end
  LoopResults(f.env.name,lrs)
end
```
```
# for an Expr representing a Method,
# check that the type of each variable used in a loop
# has a concrete type
checklooptypes(e::Expr;kwargs...) = 
 LoopResult(MethodSignature(e),loosetypes(loopcontents(e)))
 ```
 我们可以看到两个选项在同一个函数中使用相同的方法:

```
julia> using TypeCheck

julia> function foo(x::Int)
         s = 0
         for i = 1:x
           s += i/2
         end
         return s
       end
foo (generic function with 1 method)

julia> checklooptypes(foo)
foo(Int64)::Union(Int64,Float64)
    s::Union(Int64,Float64)
    s::Union(Int64,Float64)
    
julia> checklooptypes(code_typed(foo,(Int,))[1])
(Int64)::Union(Int64,Float64)
    s::Union(Int64,Float64)
    s::Union(Int64,Float64)
``` 
 
### 漂亮的印刷

我跳过了一个实现细节:我们是如何将结果打印到REPL的?

首先，我做了一些新的类型。LoopResults是检查整个函数的结果;它具有每个方法的函数名和结果。LoopResult是检查一种方法的结果;它有参数类型和松散类型的变量。

checklooptypes函数返回一个循环结果。该类型有一个名为show的函数。REPL调用显示它想要显示的值;然后显示将调用我们的show实现。

这段代码对于使静态分析可用是很重要的，但是它并没有做静态分析。您应该在实现语言中使用首选的方法来打印输出类型和输出，这就是茱莉亚所做的。


``` 
 type LoopResult
  msig::MethodSignature
  lines::Vector{SymbolNode}
  LoopResult(ms::MethodSignature,ls::Vector{SymbolNode}) = new(ms,unique(ls))
end

function Base.show(io::IO, x::LoopResult)
  display(x.msig)
  for snode in x.lines
    println(io,"\t",string(snode.name),"::",string(snode.typ))
  end
end

type LoopResults
  name::Symbol
  methods::Vector{LoopResult}
end

function Base.show(io::IO, x::LoopResults)
  for lr in x.methods
    print(io,string(x.name))
    display(lr)
  end
end
``` 


## 寻找未使用的变量

有时，当您在程序中输入时，您会键入一个变量名。程序不能告诉您，您的意思是要与前面拼写正确的变量相同;它只看到一个变量只使用一次，您可能会看到一个变量名拼写错误。需要变量声明的语言自然会捕获这些拼写错误，但是许多动态语言不需要声明，因此需要额外的分析层来捕获它们。

我们可以通过查找只使用一次或仅使用一种方法的变量来查找拼写错误的变量名(以及其他未使用的变量)。

下面是一个示例，其中有一个拼写错误的代码。

```
function foo(variable_name::Int)
  sum = 0
  for i=1:variable_name
    sum += variable_name
  end
  variable_nme = sum
  return variable_name
end 
```


这种错误可能会在您的代码中造成问题，只有在运行时才会发现。让我们假设您只对每个变量名拼写一次。我们可以将变量的用法分隔成写和读。如果拼写错误是一种书写(即:然后，没有错误将被抛出;您将只是默默地将值放入错误的变量中，找到错误可能会很令人沮丧。如果拼写错误是一种阅读(即:右=worng+2)，当运行代码时，您将得到一个运行时错误;我们希望对此有一个静态警告，以便您可以更快地找到这个错误，但是您仍然需要等待，直到运行代码来查看问题。

随着代码变得越来越复杂，除非您有了静态分析的帮助，否则很难发现错误。

左手边和右手边

另一种讨论“读”和“写”用法的方法是把它们叫做“右手边”(RHS)和“左手边”(LHS)用法。这指的是变量相对于=符号的位置。

以下是一些x的用法:

* 左边:

x = 2

x=y+22

x=x+y+2

x+=2(将糖转化为x=x+2)

* 右边:

y=x+22

x=x+y+2

x+=2(将糖转化为x=x+2)

2 * x

注意，x=x+y+2和x+=2的表达式都出现在这两个部分，因为x出现在等号两边。
 


## Looking for Single-Use Variables

我们需要寻找两种情况:

Variables used once.

变量只在LHS或仅在RHS上使用。

我们将寻找所有的变量用法，但我们将分别寻找LHS和RHS的用法，以涵盖这两种情况。

### Finding LHS Usages

要在LHS上，一个变量需要在左边有一个=号。这意味着我们可以在AST中查找=符号，然后在它们的左边查找相关的变量。

在AST中，一个=是一个带有头:(=)的Expr。(括号是用来表示这是=的符号，而不是另一个操作符:=。)args中的第一个值将是其LHS的变量名。因为我们正在查看编译器已经清理过的AST，所以(几乎)总是只在等号左边的一个符号。

让我们看看这在代码中意味着什么:
```
julia> :(x = 5)
:(x = 5)

julia> :(x = 5).head
:(=)

julia> :(x = 5).args
2-element Array{Any,1}:
  :x
 5  

julia> :(x = 5).args[1]
:x
```
下面是完整的实现，后面是解释。
让我们看看这在代码中意味着什么:
```
# Return a list of all variables used on the left-hand-side of assignment (=)
#
# Arguments:
#   e: an Expr representing a Method, as from code_typed
#
# Returns:
#   a Set{Symbol}, where each element appears on the LHS of an assignment in e.
#
function find_lhs_variables(e::Expr)
  output = Set{Symbol}()
  for ex in body(e)
    if Base.is_expr(ex,:(=))
      push!(output,ex.args[1])
    end
  end
  return output
end
```

```
  output = Set{Symbol}()
```
我们有一组符号，这些是我们在LHS上找到的变量名。
```
  for ex in body(e)
    if Base.is_expr(ex,:(=))
      push!(output,ex.args[1])
    end
  end
```
我们没有深入地挖掘表达式，因为代码类型化AST是相当平坦的;循环和ifs已经被转换为用于控制流的gotos的平面语句。在函数调用的参数中不会有任何隐藏的任务。如果在等号左边有一个符号，那么这段代码就会失败。这就忽略了两个特殊的边界情况:数组访问(如5，它将被表示为a:ref表达式)和属性(如a。头，它将被表示为a。表达式)。这些仍然总是有相关的符号作为它们的args中的第一个值，它可能只是被隐藏了一点(比如在a.property.name.海德。其他属性)。这个代码不能处理这些情况，但是if语句中的几行代码可以解决这个问题。
```
      push!(output,ex.args[1])
```


当我们发现LHS的变量用法时，我们会推！将变量名放入到集合中。集合将确保我们只有一个每个名称的副本。

### 发现RHS用法

为了找到所有其他的变量用法，我们还需要查看每个Expr。这涉及到更多的问题，因为我们关心的是所有的Exprs，而不仅仅是:(1)，因为我们需要挖掘嵌套的Exprs(来处理嵌套的函数调用)。

下面是完整的实现，下面是解释。
```
# Given an Expression, finds variables used in it (on right-hand-side)
#
# Arguments: e: an Expr
#
# Returns: a Set{Symbol}, where each e is used in a rhs expression in e
#
function find_rhs_variables(e::Expr)
  output = Set{Symbol}()

  if e.head == :lambda
    for ex in body(e)
      union!(output,find_rhs_variables(ex))
    end
  elseif e.head == :(=)
    for ex in e.args[2:end]  # skip lhs
      union!(output,find_rhs_variables(ex))
    end
  elseif e.head == :return
    output = find_rhs_variables(e.args[1])
  elseif e.head == :call
    start = 2  # skip function name
    e.args[1] == TopNode(:box) && (start = 3)  # skip type name
    for ex in e.args[start:end]
      union!(output,find_rhs_variables(ex))
    end
  elseif e.head == :if
   for ex in e.args # want to check condition, too
     union!(output,find_rhs_variables(ex))
   end
  elseif e.head == :(::)
    output = find_rhs_variables(e.args[1])
  end

  return output
end
```

这个函数的主要结构是一个大型的if-else语句，每个案例处理一个不同的头部符号。

```
  output = Set{Symbol}()
```
输出是变量名的集合，我们将在函数的末尾返回。由于我们只关心每个变量至少被读取一次的事实，所以使用Set使我们不必担心每个名称的惟一性。
```
  if e.head == :lambda
    for ex in body(e)
      union!(output,find_rhs_variables(ex))
    end
```
这是if-else语句中的第一个条件。lambda代表了函数的主体。我们在定义的主体上递归，它应该在定义中得到所有RHS变量的用法。
```
  elseif e.head == :(=)
    for ex in e.args[2:end]  # skip lhs
      union!(output,find_rhs_variables(ex))
    end
```
如果头是:(=)，那么表达式就是一个赋值。我们跳过了arg的第一个元素因为这是分配给的变量。对于剩下的表达式，我们递归地查找RHS变量并将它们添加到我们的集合中。
```
  elseif e.head == :return
    output = find_rhs_variables(e.args[1])
```
如果这是一个返回语句，那么args的第一个元素就是返回值的表达式;我们将在这个集合中添加任何变量
```
  elseif e.head == :call
    # skip function name
    for ex in e.args[2:end]
      union!(output,find_rhs_variables(ex))
    end
```
对于函数调用，我们希望获取所有参数中使用的所有变量。我们跳过函数名，这是args的第一个元素。
```
  elseif e.head == :if
   for ex in e.args # want to check condition, too
     union!(output,find_rhs_variables(ex))
   end
```
表示if语句的Expr具有头值:if。我们想从if语句的主体中获取变量的用法，因此我们递归到args的每一个元素上。
```
  elseif e.head == :(::)
    output = find_rhs_variables(e.args[1])
  end
```
他:(:)操作符用于添加类型注释。第一个参数是带注释的表达式或变量;我们检查带注释的表达式中的变量用法。
```
return output
```


在函数的末尾，我们返回了RHS变量的集合。

还有一些代码可以简化上面的方法。因为上面的版本只处理Exprs，但是递归的一些值可能不是Exprs，所以我们需要更多的方法来处理其他可能的类型。
```
# Recursive Base Cases, to simplify control flow in the Expr version
find_rhs_variables(a) = Set{Symbol}()  # unhandled, should be immediate val e.g. Int
find_rhs_variables(s::Symbol) = Set{Symbol}([s])
find_rhs_variables(s::SymbolNode) = Set{Symbol}([s.name])
```


把它放在一起

既然我们已经定义了上面定义的两个函数，那么我们就可以使用它们来查找那些只能从或者仅被写入的变量。找到它们的功能将被称为“不使用本地”的当地人。

```
function unused_locals(e::Expr)
  lhs = find_lhs_variables(e)
  rhs = find_rhs_variables(e)
  setdiff(lhs,rhs)
end
```
unused本地人将返回一组变量名。很容易编写一个函数来确定unused本地人的输出是否被认为是“通过”。如果集合是空的，方法就会通过。如果函数的所有方法都通过，那么函数就会通过。下面的函数check本地人实现了这个逻辑。
```
check_locals(f::Callable) = all([check_locals(e) for e in code_typed(f)])
check_locals(e::Expr) = isempty(unused_locals(e))
```


## 结论

我们已经对基于类型和基于变量用法的茱莉亚代码进行了两种静态分析。

静态类型语言已经完成了我们基于类型的分析所做的工作;额外的基于类型的静态分析在动态类型语言中非常有用。已经有(主要是研究)项目为包括Python、Ruby和Lisp在内的语言构建静态类型推断系统。这些系统通常是围绕可选的类型注释构建的;当您需要时，您可以拥有静态类型，而当您不需要时，则会返回到动态类型。这对于将一些静态类型集成到现有的代码库中尤其有用。

非类型的检查，例如我们的变量使用，都适用于动态和静态类型的语言。但是，许多静态类型的语言，比如C++和Java，都要求您声明变量，并且已经给出了我们创建的基本警告。仍然有一些可以编写的自定义检查;例如，检查特定于项目的样式指南或基于安全策略的额外的安全预防措施。

虽然茱莉亚确实有很好的工具来支持静态分析，但这并不是唯一的方法。当然，Lisp很有名，因为代码是嵌套列表的数据结构，所以在AST中很容易得到。Java也公开了它的AST，尽管AST比Lisp要复杂得多。一些语言或语言工具链的设计目的不是为了让用户在内部的表示中闲逛。对于开放源码的工具链(特别是那些注释良好的工具)，一个选择是添加钩子到环境中，让您访问AST。

在这种情况下，最终的回退是自己编写一个解析器;这是在可能的情况下尽量避免。要覆盖大多数编程语言的语法，需要大量的工作，而且您必须自己更新它，因为新特性是添加到语言中的(而不是从上游自动更新)。根据您想要做的检查，您可能只需要解析某些行或语言特性的子集，这将极大地降低编写您自己的解析器的成本。

希望您对静态分析工具的理解能够帮助您理解您在代码中使用的工具，并可能激发您编写自己的代码。











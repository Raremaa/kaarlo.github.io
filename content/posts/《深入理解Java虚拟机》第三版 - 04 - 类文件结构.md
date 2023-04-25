---
title: "《深入理解Java虚拟机》第三版 - 04 - 类文件结构"
date: 2022-09-08 19:41:56.85
draft: false
type: "post"
showTableOfContents: true
tags: ["Java","虚拟机"]
---

# 类文件结构

# 1. 类文件结构

## 1.1. Class文件结构

Class文件是一组以8个字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。当遇到需要占用8个字节以上空间的数据项时，则会按照高位在前（Big-Endian）的方式分割成若干个8个字节进行存储。

Class文件格式采用以下两种数据类型：

- **无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数**，无符号数可以用来描述数字、索引引用、数量值或者按照UTF8编码构成字符串值
- **表是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名都习惯性地以“_info”结尾**。表用于描述有层次关系的复合结构的数据，整个Class文件本质上也可以视作是一张表。

无论是无符号数还是表，当需要描述同一类型但数量不定的多个数据时，经常会**使用一个前置的容量计数器加若干个连续的数据项的形式，这时候称这一系列连续的某一类型的数据为某一类型的“集合”。**

下文按顺序叙述各位的数字代表的含义。

## 1.2. 魔数与Class文件的版本

**每个Class文件的头4个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件**。**Class文件的魔数为0xCAFEBABE（咖啡宝贝）。**

之后的**4个字节存储的是Class文件的版本号**：第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）。

![Untitled](https://img.masaiqi.com/202209081941488.png)

## 1.3. 常量池

**之后是常量池，常量池可以比喻为Class文件里的资源仓库**，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件空间最大的数据项目之一，另外，它还是**在Class文件中第一个出现的表类型数据项目**。

**由于常量池中常量的数量是不固定的，所以在常量池的入口需要放置一项u2类型的数据，代表常量池容量计数值（constant_pool_count）**。与Java中语言习惯不同，这个容量计数是从1而不是0开始的，设计者将第0项常量空出来是有特殊考虑的，这样做的目的在于，如果后面某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的含义，可以把索引值设置为0来表示。Class文件结构中只有常量池的容量计数是从1开始，对于其他集合类型，包括接口索引集合、字段表集合、方法表集合等的容量计数都与一般习惯相同，是从0开始。

**常量池中主要存放两大类常量：**

- **字面量（Literal）**：字面量比较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等。
- **符号引用（Symbolic References）**：符号引用则属于编译原理方面的概念，主要包括下面几类常量：
  - 被模块导出或者开放的包（Package）
  - 类和接口的全限定名（Fully Qualified Name）
  - 字段的名称和描述符（Descriptor）
  - 方法的名称和描述符·方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）
  - 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

**Java代码是在虚拟机加载Class文件的时候进行动态连接，也就是说，在Class文件中不会保存各个方法、字段最终在内存中的布局信息，这些字段、方法的符号引用不经过虚拟机在运行期转换的话是无法得到真正的内存入口地址，也就无法直接被虚拟机使用的**。**当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。**

**常量池中每一项常量都是一个表，这些表都有一个共同的特点，表结构起始的第一位是个u1类型的标志位，标记当前常量属于哪种常量类型。**

![Untitled](https://img.masaiqi.com/202209081941513.png)

![Untitled](https://img.masaiqi.com/202209081941531.png)

![Untitled](https://img.masaiqi.com/202209081941562.png)

![Untitled](https://img.masaiqi.com/202209081941583.png)

每个常量池的数据结构各自独立，以CONSTANT_Class_info型常量为例，其数据结构为：

![Untitled](https://img.masaiqi.com/202209081941601.png)

- tag：区分常量类型，判断当前属于CONSTANT_Class_info型常量
- name_index：常量名字的索引，根据这个索引找到对应位置的常量（CONSTANT_Utf8_info），再次读取获取名字。

CONSTANT_Utf8_info数据结构：

![Untitled](https://img.masaiqi.com/202209081941621.png)

- tag：区分常量类型，判断当前属于CONSTANT_Class_info型常量
- length：这个UTF-8编码的字符串长度字节数
- bytes：length字节长度到连续数据，是一个UTF-8表示的字符串

## 1.4. 访问标志

**之后是2个字节的访问标志（access_flags），这个标志用于识别一些类或者接口层次的访问信息**，包括：这个Class是类还是接口类；是否定义为public类型；是否定义为abstract类型；如果是类的话，是否被声明为final。

![Untitled](https://img.masaiqi.com/202209081941639.png)

## 1.5. 类索引、父类索引与接口索引集合

**类索引、父类索引和接口索引集合都按顺序排列在访问标志之后。**

**类索引（this_class）和父类索引（super_class）都是一个u2类型的数据，而接口索引集合（interfaces）是一组u2类型的数据的集合，Class文件中由这三项数据来确定该类型的继承关系**。

**类索引用于确定这个类的全限定名**，**父类索引用于确定这个类的父类的全限定名**。由于Java语言不允许多重继承，所以父类索引只有一个，除了java.lang.Object之外，所有的Java类都有父类，因此除了java.lang.Object外，所有Java类的父类索引都不为0。**接口索引集合就用来描述这个类实现了哪些接口**，这些被实现的接口将按implements关键字（如果这个Class文件表示的是一个接口，则应当是extends关键字）后的接口顺序从左到右排列在接口索引集合中。

- **类索引和父类索引用两个u2类型的索引值表示，它们各自指向一个类型为CONSTANT_Class_info的类描述符常量，通过CONSTANT_Class_info类型的常量中的索引值可以找到定义在CONSTANT_Utf8_info类型的常量中的全限定名字符串**。
- **接口索引集合，入口的第一项u2类型的数据为接口计数器（interfaces_count），表示索引表的容量。如果该类没有实现任何接口，则该计数器值为0，后面接口的索引表不再占用任何字节**。

## 1.6. 字段表集合

**字段表（field_info）用于描述接口或者类中声明的变量。Java语言中的“字段”（Field）包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量**。字段可以包括的**修饰符有字段的作用域（public、private、protected修饰符）、是实例变量还是类变量（static修饰符）、可变性（final）、并发可见性（volatile修饰符，是否强制从主内存读写）、可否被序列化（transient修饰符）、字段数据类型（基本类型、对象、数组）、字段名称**。

![Untitled](https://img.masaiqi.com/202209081941655.png)

- **access_flags是字段修饰符**：

  ![Untitled](https://img.masaiqi.com/202209081941673.png)

- **name_index和descriptor_index，都是对常量池项的引用，分别代表着字段的简单名称以及字段和方法的描述符，*描述符是用来描述字段的数据类型（字段表用到）、方法的参数列表（包括数量、类型以及顺序）和返回值（方法表用到）*：**

  ![Untitled](https://img.masaiqi.com/202209081941692.png)

  **其中，对象类型用字符L加对象的全限定名来表示。**

  **对于数组类型，每一维度将使用一个前置的“[”字符来描述**，如一个定义为“java.lang.String[][]”类型的二维数组将被记录成“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录成“[I”。

- attributes_count 和 attributes是属性表集合，用于存储一些额外的信息。比如声明“`final static int m = 123；`”，那就可能会存在一项名称为ConstantValue的属性，其值指向常量123

**字段表集合中不会列出从父类或者父接口中继承而来的字段，但有可能出现原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，编译器就会自动添加指向外部类实例的字段。**

## 1.7. 方法表集合

Class文件存储格式中对方法的描述与对字段的描述采用了几乎完全一致的方式，**方法表的结构如同字段表一样，依次包括访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项。**

![Untitled](https://img.masaiqi.com/202209081941712.png)

**因为volatile关键字和transient关键字不能修饰方法**，所以方法表的访问标志中没有了ACC_VOLATILE标志和ACC_TRANSIENT标志。与之相对，**synchronized、native、strictfp和abstract关键字可以修饰方法**，方法表的访问标志中也相应地增加了ACC_SYNCHRONIZED、ACC_NATIVE、ACC_STRICTFP和ACC_ABSTRACT标志。

访问标志：

![Untitled](https://img.masaiqi.com/202209081941731.png)

**用描述符来描述方法时，按照先参数列表、后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号“()”之内**。如方法void inc()的描述符为“()V”，方法java.lang.String#toString()的描述符为“()Ljava/lang/String；”，方法int indexOf(char[]source，int sourceOffset，int sourceCount，char[]target，int targetOffset，int targetCount，int fromIndex)的描述符为“([CII[CIII)I”。

**方法里面的代码，会在属性表集合中一个叫“Code”的属性中。**

**与字段表集合相对应地，如果父类方法在子类中没有被重写（Override），方法表集合中就不会出现来自父类的方法信息。但同样地，有可能会出现由编译器自动添加的方法，最常见的便是类构造器“<clinit>()”方法和实例构造器“<init>()”方法。**

**在Java语言中，要重载（Overload）一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名**。**特征签名是指一个方法中各个参数在常量池中的字段符号引用的集合，也正是因为返回值不会包含在特征签名之中，所以Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的。但是在Class文件格式之中，特征签名的范围明显要更大一些**，只要描述符不是完全一致的两个方法就可以共存。**也就是说，如果两个方法有相同的名称和特征签名，但返回值不同，那么也是可以合法共存于同一个Class文件中的**。

## 1.8. 属性表集合

字段表、方法表都可以携带自己的属性表（attribute_info）集合。

**《Java虚拟机规范》允许只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性**。JavaSE12版本中，预定义了2项属性。

![Untitled](https://img.masaiqi.com/202209081941751.png)

![Untitled](https://img.masaiqi.com/202209081941777.png)

属性表结构：

![Untitled](https://img.masaiqi.com/202209081941794.png)

- **attribute_name_index对应常量池中一个CONSTANT_Utf8_info类型常量，代表名字。**
- **attribute_length是属性值所占用的位数。**
- **info是实际的属性数据**

**以下列举各属性的info结构，省略attribute_name_index和attribute_length。**

### 1.8.1. Code属性

**Java程序方法体里面的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性**。

![Untitled](https://img.masaiqi.com/202209081941813.png)

- **max_stack代表了操作数栈（Operand Stack）深度的最大值**。在方法执行的任意时刻，操作数栈都不会超过这个深度。虚拟机运行的时候需要根据这个值来分配栈帧（StackFrame）中的操作栈深度。

- **max_locals代表了局部变量表（虚拟机栈）所需的存储空间**。max_locals的单位是变量槽（Slot），**变量槽是虚拟机为局部变量分配内存所使用的最小单位**。对于byte、char、float、int、short、boolean和returnAddress等长度不超过32位的数据类型，每个局部变量占用一个变量槽，而double和long这两种64位的数据类型则需要两个变量槽来存放。方法参数（包括实例方法中的隐藏参数“this”）、显式异常处理程序的参数（Exception Handler Parameter，就是try-catch语句中catch块中所定义的异常）、方法体中定义的局部变量都需要依赖局部变量表来存放。**Java虚拟机会将局部变量表中变量槽进行重用，当代码执行超出一个局部变量的作用域时，这个局部变量所占的变量槽可以被其他局部变量所使用，Javac编译器会根据变量的作用域来分配变量槽给各个变量使用，根据同时生存的最大局部变量数量和类型计算出max_locals的大小。**

- **code_lenth代表编译后生成的字节码长度。**Java虚拟机限制了一个方法不允许超过65535条字节码指令，class文件中的u4其实只使用了u2的长度。

- **code是存储字节码指令的一系列字节流。（每个指令是一个u1类型的单字节）。**一字节0xFF共256条指令，《Java虚拟机规范》已经定义了其中约200条编码值对应的指令含义。

- **exception_table_length是显式异常处理表（异常表）集合中的长度。**

- **exception_table是显式异常处理表（异常表）集合中的表。**

  ![Untitled](https://img.masaiqi.com/202209081941829.png)

  如果当字节码从第start_pc行到第end_pc行之间（不含第end_pc行）出现了类型为catch_type或者其子类的异常（catch_type为指向一个CONSTANT_Class_info型常量的索引），则转到第handler_pc行继续处理。当catch_type的值为0时，代表任意异常情况都需要转到handler_pc处进行处理

- **最后attribute_count 和 attributes也是一个属性表集合。**

<aside>
💡 **javap查看指令时，会发现即使没有参数，方法也还是有Args_size = 1，这其实是This关键字对应的当前对象。
Java语言里面的潜规则：在任何实例方法里面，都可以通过“this”关键字访问到此方法所属的对象。这个访问机制对Java程序的编写很重要，而它的实现非常简单，仅仅是通过在Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数而已。**


</aside>

### 1.8.2. Exceptions属性

**Exceptions属性的作用是列举出方法中可能抛出的受查异常（Checked Exceptions），也就是方法描述时在throws关键字后面列举的异常。**

![Untitled](https://img.masaiqi.com/202209081941845.png)

- **number_of_exceptions项表示方法可能抛出number_of_exceptions种受查异常。**
- **每一种受查异常使用一个exception_index_table项表示，exception_index_table是一个指向常量池中CONSTANT_Class_info型常量的索引，代表了该受查异常的类型。**

### 1.8.3. LineNumberTable属性

**LineNumberTable属性用于描述Java源码行号与字节码行号（字节码的偏移量）之间的对应关系**。它并不是运行时必需的属性，但默认会生成到Class文件之中。**如果选择不生成LineNumberTable属性，对程序运行产生的最主要影响就是当抛出异常时，堆栈中将不会显示出错的行号，并且在调试程序的时候，也无法按照源码行来设置断点**。LineNumberTable属性的结构如表618所示

![Untitled](https://img.masaiqi.com/202209081941861.png)

- line_number_table是一个数量为line_number_table_length、类型为line_number_info的集合
- line_number_info表包含start_pc和line_number两个u2类型的数据项，前者是字节码行号，后者是Java源码行号。

### 1.8.4. LocalVariableTable及LocalVariableTypeTable属性

**LocalVariableTable属性用于描述栈帧中局部变量表的变量与Java源码中定义的变量之间的关系**，它也不是运行时必需的属性，但默认会生成到Class文件之中，**如果没有生成这项属性，当其他人引用这个方法时，所有的参数名称都将会丢失，譬如IDE将会使用诸如arg0、arg1之类的占位符代替原有的参数名，这对程序运行没有影响，但是会对代码编写带来较大不便，而且在调试期间无法根据参数名称从上下文中获得参数值**。

- LocalVariableTable的结构

  ![Untitled](https://img.masaiqi.com/202209081941876.png)

  - local_variable_info项目代表了一个栈帧与源码中的局部变量的关联：

    ![Untitled](https://img.masaiqi.com/202209081941891.png)

    - start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖的长度，两者结合起来就是这个局部变量在字节码之中的作用域范围。
    - name_index和descriptor_index都是指向常量池中CONSTANT_Utf8_info型常量的索引，分别代表了局部变量的名称以及这个局部变量的描述符。
    - index是这个局部变量在栈帧的局部变量表中变量槽的位置。当这个变量数据类型是64位类型时（double和long），它占用的变量槽为index和index+1两个。

- LocalVariableTypeTable在JDK5引入范型之后被添加，与LocalVariableTable相似，仅仅是把记录的字段描述符的descriptor_index替换成了字段的特征签名（Signature）。对于非泛型类型来说，描述符和特征签名能描述的信息是能吻合一致的，但是泛型引入之后，由于描述符中泛型的参数化类型被擦除掉，描述符就不能准确描述泛型类型了。因此出现了LocalVariableTypeTable属性，使用字段的特征签名来完成泛型的描述。

### 1.8.5. SourceFile

**SourceFile属性用于记录生成这个Class文件的源码文件名称**。这个属性也是可选的。在Java中，对于大多数的类来说，类名和文件名是一致的，但是有一些特殊情况（如内部类）例外。**如果不生成这项属性，当抛出异常时，堆栈中将不会显示出错代码所属的文件名**。

![Untitled](https://img.masaiqi.com/202209081941907.png)

- sourcefile_index数据项是指向常量池中CONSTANT_Utf8_info型常量的索引，常量值是源码文件的文件名。

### 1.8.6. ConstantValue属性

**ConstantValue属性的作用是通知虚拟机自动为静态变量赋值。只有被static关键字修饰的变量（类变量）才可以使用这项属性。**

类似`int x=123`和`static int x=123`这样的变量定义在Java程序里面是非常常见的事情，但虚拟机对这两种变量赋值的方式和时刻都有所不同。

- **对非static类型的变量（也就是实例变量）的赋值是在实例构造器<init>()方法中进行的；**
- **对于类变量，则有两种方式可以选择，在类构造器<clinit>()方法中或者使用ConstantValue属性，**目前Oracle公司实现的Javac编译器的选择：
  1. 如果同时使用final和static来修饰一个变量（常量），并且这个变量的数据类型是基本类型或者java.lang.String的话，就将会生成ConstantValue属性来进行初始化；
  2. 如果这个变量没有被final修饰，或者并非基本类型及字符串，则将会选择在<clinit>()方法中进行初始化

**ConstantValue的属性值只能限于基本类型和String这点，这是理所当然的结果。因为此属性的属性值只是一个常量池的索引号，由于Class文件格式的常量类型中只有与基本属性和字符串相对应的字面量，所以就算ConstantValue属性想支持别的类型也无能为力。**

![Untitled](https://img.masaiqi.com/202209081941921.png)

- constant_value_index数据项代表了常量池中一个字面量常量的引用，根据字段类型的不同，字面量可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info和CONSTANT_String_info常量中的一种。

### 1.8.7. InnerClasses属性

**InnerClasses属性用于记录内部类与宿主类之间的关联。如果一个类中定义了内部类，那编译器将会为它以及它所包含的内部类生成InnerClasses属性。**

![Untitled](https://img.masaiqi.com/202209081941938.png)

- number_of_classes代表需要记录多少个内部类信息

- 每一个内部类的信息都由一个inner_classes_info表进行描述。inner_classes_info表的结构：

  ![Untitled](https://img.masaiqi.com/202209081941954.png)

  - inner_class_info_index和outer_class_info_index都是指向常量池中CONSTANT_Class_info型常量的索引，分别代表了内部类和宿主类的符号引用。

  - inner_name_index是指向常量池中CONSTANT_Utf8_info型常量的索引，代表这个内部类的名称，如果是匿名内部类，这项值为0。

  - inner_class_access_flags是内部类的访问标志（类似于类的access_flags）：

    ![Untitled](https://img.masaiqi.com/202209081941971.png)

### 1.8.8. Deprecated及Synthetic属性

**Deprecated及Synthetic属性都属于标志类型的布尔属性，只存在有和没有的区别，没有属性值的概念。**

- **Deprecated属性用于表示某个类、字段或者方法，已经被程序作者定为不再推荐使用**，它可以通过代码中使用“@deprecated”注解进行设置。
- **Synthetic属性代表此字段或者方法并不是由Java源码直接产生的**，而是由编译器自行添加的，也可以设置它们访问标志中的ACC_SYNTHETIC标志位。**所有由不属于用户代码产生的类、方法及字段都应当至少设置Synthetic属性或者ACC_SYNTHETIC标志位中的一项，唯一的例外是实例构造器“<init>()”方法和类构造器“<clinit>()”方法**

![Untitled](https://img.masaiqi.com/202209081941987.png)

attribute_length固定为0，无任何属性。

### 1.8.9. StackMapTable属性

**StackMapTable属性在JDK6增加到Class文件规范之中，位于Code属性的属性表中。这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（Type Checker）使用。**

StackMapTable属性中包含零至多个栈映射帧（Stack Map Frame），每个栈映射帧都显式或隐式地代表了一个字节码偏移量，用于表示执行到该字节码时局部便量表和操作数栈的验证类型。类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。

![Untitled](https://img.masaiqi.com/202209081941007.png)

### 1.8.10. Signature属性

**Signature属性是一个可选的定长属性，可以出现于类、字段表和方法表结构的属性表中。任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variable）或参数化类型（Parameterized Type），则Signature属性会为它记录泛型签名信息**。之所以要专门使用这样一个属性去记录泛型类型，是因为Java语言的泛型采用的是擦除法实现的伪泛型，字节码（Code属性）中所有的泛型信息编译（类型变量、参数化类型）在编译之后都通通被擦除掉。使用擦除法的好处是实现简单（主要修改Javac编译器，虚拟机内部只做了很少的改动）、非常容易实现Backport，运行期也能够节省一些类型所占的内存空间。但**坏处是运行期就无法像C#等有真泛型支持的语言那样，将泛型类型与用户定义的普通类型同等对待，例如运行期做反射时无法获得泛型信息。Signature属性就是为了弥补这个缺陷而增设的，现在Java的反射API能够获取的泛型类型，最终的数据来源也是这个属性。**

![Untitled](https://img.masaiqi.com/202209081941025.png)

- signature_index项的值必须是一个对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示类签名或方法类型签名或字段类型签名。如果当前的Signature属性是类文件的属性，则这个结构表示类签名，如果当前的Signature属性是方法表的属性，则这个结构表示方法类型签名，如果当前Signature属性是字段表的属性，则这个结构表示字段类型签名。

### 1.8.11. BootstrapMethods属性

**BootstrapMethods属性用于保存invokedynamic指令引用的引导方法限定符。**

- BootStrapMethods属性结构：

  ![Untitled](https://img.masaiqi.com/202209081941044.png)

  - num_bootstrap_methods值给出了bootstrap_methods[]数组中的引导方法限定符的数量
  - bootstrap_methods[]是一个bootstrap_method数组

- bootstrap_method结构：

  ![Untitled](https://img.masaiqi.com/202209081941064.png)

- bootstrap_method_ref：bootstrap_method_ref项的值必须是一个对常量池的有效索引。常量池在该索引处的值必须是一个CONSTANT_MethodHandle_info结构。

- num_bootstrap_arguments：num_bootstrap_arguments项的值给出了bootstrap_arguments[]数组成员的数量。

- bootstrap_arguments[]：bootstrap_arguments[]数组的每个成员必须是一个对常量池的有效索引。常量池在该索引出必须是下列结构之一：CONSTANT_String_info、CONSTANT_Class_info、CONSTANT_Integer_info、CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_MethodHandle_info或CONSTANT_MethodType_info。

### 1.8.12. MethodParameters属性

MethodParameters的作用是记录方法的各个形参名称和信息，属于方法表的属性，与Code属性平级，可以运行时通过反射API获取。

- MethodParameters属性结构：

  ![Untitled](https://img.masaiqi.com/202209081941080.png)

- Parameter属性结构：

  ![Untitled](https://img.masaiqi.com/202209081941096.png)

  - name_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该参数的名称
  - access_flags是参数的状态指示器，它可以包含以下三种状态中的一种或多种：
    - 0x0010（ACC_FINAL）：表示该参数被final修饰。
    - 0x1000（ACC_SYNTHETIC）：表示该参数并未出现在源文件中，是编译器自动生成的。
    - 0x8000（ACC_MANDATED）：表示该参数是在源文件中隐式定义的。Java语言中的典型场景是this关键字。

### 1.8.13. 模块化相关属性

**模块描述文件（module-info.java）最终是要编译成一个独立的Class文件来存储的，所以，Class文件格式也扩展了Module、ModulePackages和ModuleMainClass三个属性用于支持Java模块化相关功**

***Module：***

![Untitled](https://img.masaiqi.com/202209081941115.png)

- module_name_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该模块的名称。

- module_flags是模块的状态指示器，它可以包含以下三种状态中的一种或多种：

  - 0x0020（ACC_OPEN）：表示该模块是开放的。
  - 0x1000（ACC_SYNTHETIC）：表示该模块并未出现在源文件中，是编译器自动生成的。
  - 0x8000（ACC_MANDATED）：表示该模块是在源文件中隐式定义的。

- module_version_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该模块的版本号。

- 后续的几个属性分别记录了模块的requires、exports、opens、uses和provides定义，以exports为例：

  ![Untitled](https://img.masaiqi.com/202209081941131.png)

  exports属性的每一元素都代表一个被模块所导出的包

  - exports_index是一个指向常量池CONSTANT_Package_info常量的索引值，代表了被该模块导出的包。
  - exports_flags是该导出包的状态指示器，它可以包含以下两种状态中的一种或多种：
    - 0x1000（ACC_SYNTHETIC）：表示该导出包并未出现在源文件中，是编译器自动生成的。
    - 0x8000（ACC_MANDATED）：表示该导出包是在源文件中隐式定义的。
  - exports_to_count是该导出包的限定计数器，如果这个计数器为零，这说明该导出包是无限定的（Unqualified），即完全开放的，任何其他模块都可以访问该包中所有内容。如果该计数器不为零，则后面的exports_to_index是以计数器值为长度的数组，每个数组元素都是一个指向常量池中CONSTANT_Module_info常量的索引值，代表着只有在这个数组范围内的模块才被允许访问该导出包的内容。

***ModulePackages：***

ModulePackages是另一个用于支持Java模块化的变长属性，它用于描述该模块中所有的包，不论是不是被export或者open的。

![Untitled](https://img.masaiqi.com/202209081941146.png)

- package_count是package_index数组的计数器
- package_index中每个元素都是指向常量池CONSTANT_Package_info常量的索引值，代表了当前模块中的一个包。

***ModuleMainClass：***

ModuleMainClass属性是一个定长属性，用于确定该模块的主类（MainClass）：

![Untitled](https://img.masaiqi.com/202209081941162.png)

- main_class_index是一个指向常量池CONSTANT_Class_info常量的索引值，代表了该模块的主类。

### 1.8.14. 运行时注解相关属性

为了对注解（Annotation）的支持和存储源码中注解信息，Class文件同步增加了RuntimeVisibleAnnotations、RuntimeInvisibleAnnotations、RuntimeVisibleParameterAnnotations和RuntimeInvisibleParameterAnnotations四个属性。

到了JDK8时期，进一步加强了Java语言的注解使用范围，又新增类型注解（JSR308），所以Class文件中也同步增加了RuntimeVisibleTypeAnnotations和RuntimeInvisibleTypeAnnotations两个属性。

以RuntimeVisibleAnnotations为例：

![Untitled](https://img.masaiqi.com/202209081941179.png)

- num_annotations是annotations数组的计数器
- annotations中每个元素都代表了一个运行时可见的注解

其中，annotation结构：

![Untitled](https://img.masaiqi.com/202209081941197.png)

- type_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，该常量应以字段描述符的形式表示一个注解。
- num_element_value_pairs是element_value_pairs数组的计数器
- element_value_pairs中每个元素都是一个键值对，代表该注解的参数和值。
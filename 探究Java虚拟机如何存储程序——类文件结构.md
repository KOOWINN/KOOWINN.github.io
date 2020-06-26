# 探究Java虚拟机如何存储程序——类文件结构

Class文件是一组以8个字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。

根据《Java虚拟机规范》的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：“无符号数”和“表”。
- 无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成的字符串值。
- 表是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据，整个Class文件本质上也可以视作是一张表，这张表中的数据项按照如下顺序排列构成：

|     类型     |         名称       |        数量         |
| ------------ | ----------------- | ------------------- |
|u4            |magic              |1                    |
|u2            |minor_version      |1                    |
|u2            |major_version      |1                    |
|u2            |constant_pool_count|1                    |
|cp_info       |constant_pool      |constant_pool_count-1|
|u2            |access_flags       |1                    |
|u2            |this_class         |1                    |
|u2            |super_class        |1                    |
|u2            |interfaces_count   |1                    |
|u2            |interfaces         |interfaces_count     |
|u2            |fields_count       |1                    |
|field_info    |fields             |fields_count         |
|u2            |methods_count      |1                    |
|method_info   |methods            |methods_count        |
|u2            |attributes_count   |1                    |
|attribute_info|attributes         |attributes_count     |

## 1. 魔数
每个Class文件的头4个字节被称为魔数，它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。（不仅仅是Class文件，很多文件格式标准中都使用了魔数来进行身份识别，譬如图片。）Class文件的魔数值为“0xCAFEBABE”。

## 2. 版本号
紧接着魔数的4个字节存储的是Class文件的版本号：第5个和第6个字节是次版本号（minor_version），第7和第8个字节是主版本号（major_version）。

## 3.常量池
紧接着主、次版本号之后的是常量池入口，常量池可以比喻为Class文件里的资源仓库，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件空间最大的数据项目之一。另外，它还是在Class文件中第一个出现的表类型数据项目。

由于常量池中常量的数量不固定，所以在常量池入口需要放置一项u2类型的数据，代表常量池容量计数器（constant_pool_count）。与Java中语言习惯不同，这个容量计数器是从1而不是0开始的。第0项另有他用，后面某些指向常量池的索引值的数据在特定情况下就可以使用该项来表达“不引用任何一个常量池项目”。

常量池中主要存放两大类常量：`字面量`和`符号引用`。字面量比较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等。而符号引用则属于编译原理方面的概念，主要包括下面几类常量：
- 被模块导出或者开放的包（Package）
- 类和接口的全限定名（Fully Qualified Name）
- 字段的名称和描述符（Descriptor）
- 方法的名称和描述符
- 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）
- 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

在Class文件中不会保存各个方法、字段最终在内存中的布局信息，这些字段、方法的符号引用不经过虚拟机在运行期转换的话是无法得到真正的内存入口地址，也就无法直接被虚拟机使用。当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。

常量池中每一项常量都是一个表，到JDK13，常量表中共有17种结构各不相同的表结构数据。它们的第一位都是一个u1类型的标志位(tag)，代表当前常量属于哪种常量类型。

|   类型  | 标志   |  描述  |
|  -----  |  ---- |  ----  |
| CONSTANT_Utf8_info               | 1  | UTF-8编码的字符串 |
| CONSTANT_Integer_info            | 3  | 整型字面量 |
| CONSTANT_Float_info              | 4  | 浮点型字面量 |
| CONSTANT_Long_info               | 5  | 长整型字面量 |
| CONSTANT_Double_info             | 6  | 双精度浮点型字面量 |
| CONSTANT_Class_info              | 7  | 类或接口的符号引用 |
| CONSTANT_String_info             | 8  | 字符串类型字面量   |
| CONSTANT_Fieldref_info           | 9  | 字段的符号引用 |
| CONSTANT_Methodref_info          | 10 | 类中方法的符号引用 |
| CONSTANT_InterfaceMethodref_info | 11 | 接口中方法的符号引用 |
| CONSTANT_NameAndType_info        | 12 | 字段或方法的部分符号引用 |
| CONSTANT_MethodHanlde_info       | 15 | 表示方法句柄 |
| CONSTANT_MethodType_info         | 16 | 表示方法类型 |
| CONSTANT_Dynamic_info            | 17 | 表示一个动态计算常量 |
| CONSTANT_InvokeDynamic_info      | 18 | 表示一个动态方法调用点 |
| CONSTANT_Module_info             | 19 | 表示一个模块 |
| CONSTANT_Package_info            | 20 | 表示一个模块中开放或者导出的包 |

这17类常量类型各自有着完全独立的数据结构，下面分别进行介绍。
### (1) CONSTANT_Utf8_info
CONSTANT_Utf8_info型常量的结构为：
| 类型 |   项目  | 描述 |
| ---  | ------ | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | length |  UTF-8编码的字符串占用的字节数  |
|  u1  | bytes  |  长度为length的UTF-8编码的字符串 |

由于Class文件中的方法、字段等都需要引用CONSTANT_Utf8_info型常量来描述名称，所以CONSTANT_Utf8_info型常量的最大长度也就是Java中方法、字段名称的最大长度，也就是length的最大值。而u2类型能表达的最大值是65525，所以Java程序中如果定义了超过64KB英文字符的变量或方法名，即使规则和全部字符都是合法的，也会无法编译。

### (2) CONSTANT_Integer_info
CONSTANT_Integer_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u4  | bytes  |  按照高位在前存储的int值 |

### (3) CONSTANT_Float_info
CONSTANT_Float_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u4  | bytes  |  按照高位在前存储的float值 |

### (4) CONSTANT_Long_info
CONSTANT_Long_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u8  | bytes  |  按照高位在前存储的long值 |

### (5) CONSTANT_Double_info
CONSTANT_Double_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u8  | bytes  |  按照高位在前存储的double值 |

### (6) CONSTANT_Class_info
CONSTANT_Class_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | index  |  指向全限定名常量项的索引 |

### (7) CONSTANT_String_info
CONSTANT_String_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | index  |  指向字符串字面量的索引 |

### (8) CONSTANT_Fieldref_info
CONSTANT_Fieldref_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | index  |  指向声明字段的类或者接口描述符CONSTANT_Class_info的索引 |
|  u2  | index  |  指向字段描述符CONSTANT_NameAndType_info的索引 |

### (9) CONSTANT_Methodref_info
CONSTANT_Methodref_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | index  |  指向声明方法的类描述符CONSTANT_Class_info的索引 |
|  u2  | index  |  指向名称及类型描述符CONSTANT_NameAndType_info的索引 |

### (10) CONSTANT_InterfaceMethodref_info
CONSTANT_InterfaceMethodref_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | index  |  指向声明方法的接口描述符CONSTANT_Class_info的索引 |
|  u2  | index  |  指向名称及类型描述符CONSTANT_NameAndType_info的索引 |

### (11) CONSTANT_NameAndType_info
CONSTANT_NameAndType_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | index  |  指向该字段或方法名称常量项的索引 |
|  u2  | index  |  指向该字段或方法描述符的索引 |

### (12) CONSTANT_MethodHandle_info
CONSTANT_MethodHandle_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u1  | reference_kind  |  值必须在1至9之间（包括1和9），它决定了方法句柄的类型。方法句柄类型的值表示方法句柄的字节码行为 |
|  u2  | reference_index  |  值必须是对常量池的有效索引 |

### (13) CONSTANT_MethodType_info
CONSTANT_MethodType_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | descriptor_index  |  值必须是对常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符 |

### (14) CONSTANT_Dynamic_info
CONSTANT_Dynamic_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | bootstrap_method_attr_index  |  值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引 |
|  u2  | name_and_type_index  |  值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_NameAndType_info结构，表示方法名和方法描述符 |

### (15) CONSTANT_InvokeDynamic_info
CONSTANT_InvodeDynamic_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | bootstrap_method_attr_index  |  值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引 |
|  u2  | name_and_type_index  |  值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_NameAndType_info结构，表示方法名和方法描述符 |

### (16) CONSTANT_Module_info
CONSTANT_Module_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | name_index  |  值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示模块名 |

### (17) CONSTANT_Package_info
CONSTANT_Package_info型常量的结构为：
| 类型 |    项目    | 描述 |
| ---  | ---------- | --- |
|  u1  | tag    |  区分常量类型  |
|  u2  | name_index  |  值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示包名称 |

## 4. 访问标志
紧接着常量池的2个字节代表访问标志（access_flags），这个标志用于标识一些类或者接口层次的访问信息。具体的标志位及其含义如下表所示：
|标志名称|标志值|含义|
|-------|------|---|
|ACC_PUBLIC|0x0001|是否为public类型|
|ACC_FINAL |0x0010|是否被声明为final，只有类可以设置|
|ACC_SUPER|0x0020|是否允许使用invokespecial字节码指令的新语义，该指令的语义在JDK1.0.2发生过改变，为了区别这条指令使用哪种语义，JDK1.0.2之后编译出来的类的这个标志都必须是真|
|ACC_INTERFACE|0x0200|标识这是一个接口|
|ACC_ABSTRACT|0x0400|是否为abstract类型，对于接口或抽象类来说，此标志位为真，其他类型值为假|
|ACC_SYNTHETIC|0x1000|标识这个类并非由用户代码产生|
|ACC_ANNOTATION|0x2000|标识这是一个注解|
|ACC_ENUM|0x4000|标识这是一个枚举|
|ACc_MODULE|0x8000|标识这是一个模块|

## 5. 类索引、父类索引与接口索引集合
在访问标志之后，依次是类索引（this_class）和父类索引（super_class）和接口索引集合（interfaces）。类索引和父类索引都是一个u2类型的数据，而接口索引集合是一组u2类型的数据的集合，Class文件中由这三项数据来确定该类型的继承关系。
- 类索引用于确定这个类的全限定名
- 父类索引用于确定这个类的父类的全限定名。由于Java语言不允许多重继承，所以父类索引只有一个
- 接口索引集合用来描述这个类实现了哪些接口，这些被实现的接口将按`implements`关键字后的接口顺序从左到右排列在接口索引集合中。

## 6. 字段表集合
字段表（field_info）用于描述接口或者类中声明的变量。Java语言中的字段包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。字段表结构如下所示：
| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2             |access_flags    | 1 |
| u2             |name_index      | 1 |
| u2             |descriptor_index| 1 |
| u2             |attributes_count| 1 |
| attribute_info |attributes      |attributes_count|

字段修饰符放在了access_flags项目中，其中可以设置的标志位和含义如下表所示：

| 标志名称 | 标志值 | 含义 |
| ------- | -----  | --- |
| ACC_PUBLIC    | 0x0001 | 字段是否public |
| ACC_PRIVATE   | 0x0002 | 字段是否private |
| ACC_PROTECTED | 0x0004 | 字段是否protected |
| ACC_STATIC    | 0x0008 | 是实例变量还是类变量 |
| ACC_FINAL     | 0x0010 | 字段是否可变 |
| ACC_VOLATILE  | 0x0040 | 字段是否强制从主内存读写（并发可见性） |
| ACC_TRANSIENT | 0x0080 | 字段是否可被序列化 |
| ACC_SYNTHETIC | 0x1000 | 字段是否由编译器自动生成 |
| ACC_ENUM      | 0x4000 | 字段是否enum |

跟随access_flags标志的是两项索引值：name_index和descriptor_index。他们都是对常量池项的引用，分别代表着字段的简单名称以及字段和方法的描述符（描述符的作用是用来描述字段的数据类型、方法的参数列表和返回值）。

在descriptor_index之后跟随着一个属性表集合，用于存储一些额外的信息，字段表可以在属性表中附加描述零至多项的额外信息。

>字段表集合不会列出从父类或者父接口中继承而来的字段，但有可能出现原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，编译器就会自动添加指向外部类实例的字段。

## 7. 方法表集合
方法表的结构与字段表一样，依次包括访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项，这些数据项目的含义仅在访问标志和属性表集合的可选项中存在一定的差异。
| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2             |access_flags    | 1 |
| u2             |name_index      | 1 |
| u2             |descriptor_index| 1 |
| u2             |attributes_count| 1 |
| attribute_info |attributes      |attributes_count|

方法的访问标志包括：
| 标志名称 | 标志值 | 含义 |
| ------- | -----  | --- |
| ACC_PUBLIC | 0x0001 | 方法是否为public |
| ACC_PRIVATE |0x0002 | 方法是否为private |
| ACC_PROTECTED | 0x0004 | 方法是否为protected |
| ACC_STATIC | 0x0008 | 方法是否为static |
| ACC_FINAL | 0x0010 | 方法是否为final |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否为synchronized |
| ACC_BRIDGE | 0x0040 | 方法是否是由编译器产生的桥接方法 |
| ACC_VARARGS | 0x0080 | 方法是否接受不定参数 |
| ACC_NATIVE | 0x0100 | 方法是否为native |
| ACC_ABSTRACT | 0x0400 | 方法是否为abstract |
| ACC_STRICT | 0x0800 | 方法是否为strictfp |
| ACC_SYNTHETIC | 0x1000 | 方法是否由编译器自动产生 |

方法里的Java代码经过javac编译器编译成字节码指令后被存放在方法属性表集合中一个名为“Code”的属性中。

>与字段表集合相对应，如果父类方法在子类中没有被重写，方法表集合中就不会出现来自父类的方法信息。但同样的，有可能会出现由编译器自动添加的方法，最常见的便是类构造器“<clinit>()”方法和实例构造器“<init>()”。

## 8. 属性表集合
对于每一个属性，它的名称都要从常量池中引用一个`CONSTANT_Utf8_info`类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可。一个符号规则的属性应该满足如下所定义的结构。
| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length     | 1 |
| u1 | info                 | attribute_length |

### （1） Code属性
Java程序方法体里面的代码经过javac编译器处理之后，最终变为字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性。如果方法表由Code属性，那么它的结构将如下所示：
| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length     | 1 |
| u2 | max_stack            | 1 |
| u2 | max_locals           | 1 |
| u4 | code_length          | 1 |
| u1 | code                 | code_length |
| u2 | exception_table_length | 1 |
| exception_info | exception_table | exception_table_length |
| u2 | attribute_count | 1 |
| attribute_info | attributes | attribute_count |

- attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，此常量值固定为“Code"，它代表了该属性的属性名称。
- attribute_length指示了属性值的长度，由于属性名索引与属性长度一共占了6个字节，所以属性值的长度固定为整个属性表长度减去6个字节。
- max_stack代表了操作数栈深度的最大值。在方法执行的任意时刻，操作数栈都不会超过这个深度。
- max_locals代表了局部变量表所需的存储空间。在这里，max_locals的单位是变量槽（Slot），变量槽是虚拟机为局部变量分配内存所使用的最小单位。需要注意的是，并不是在方法中用了多少个局部变量，就把这些局部变量所占变量槽数量之和作为max_locals的值。Java虚拟机是将局部变量表中的变量槽进行重用，当代码执行超出一个局部变量表的作用域时，这个局部变量所占的变量槽就可以被其他局部变量所使用，因为操作数栈和局部变量表直接决定了一个该方法的栈帧所耗费的内存，不必要的操作数栈深度和变量槽数量会造成内存浪费。
- code_length和code用来存储Java源程序编译后生成的字节码指令。code_length代表字节码长度，code是用于存储字节码指令的一系列字节流。既然叫字节码指令，那顾名思义每个指令都是一个u1类型的单字节，当虚拟机读取到code中的一个字节码时，就可以对应找出这个字节码代表的时什么指令，并且可以知道这条指令后面是否需要跟随参数以及后续的参数应当如何解析。(虽然code_length是一个u4类型的长度值，理论上最大值可以达到2的32次幂，但是《Java虚拟机规范》中明确限制了一个方法不允许超过**65535**条字节码指令，即它实际只使用了u2的长度。)
- 在字节码指令之后的是这个方法的显示异常处理表集合，异常表对于Code属性来说并不是必须存在的。如果存在，那么它的结构应该包含四个字段，这些字段的含义为：如果当字节码从第start_pc行到第end_pc行之间（不含第end_pc行）出现了类型为catch_type或者其子类的异常（catch_type为指向CONSTANT_Class_info型常量的索引），则转到第handler_pc行继续处理。当catch_type的值为0时，代表任意异常情况都需要转到handler_pc处进行处理。

    | 类型 | 名称 | 数量|
    | ---- | --- | --- |
    | u2 | start_pc   | 1 |
    | u2 | end_pc     | 1 |
    | u2 | handler_pc | 1 |
    | u2 | catch_type | 1 |

### （2）Exception属性
这是的Exception属性是在方法表中与Code属性平级的一项属性，与上述的异常表不同。Exception属性用来列举方法中可能抛出的受查异常，也就是方法描述时在throws关键字后面列举的异常。它的结构如下所示。

  | 类型 | 名称 | 数量|
  | ---- | --- | --- |
  | u2 | attribute_name_index | 1 |
  | u4 | attribute_length     | 1 |
  | u2 | number_of_exceptions | 1 |
  | u2 | exception_index_table | number_of_exceptions |

从上述结构可以看出，一个方法可能会抛出number_of_exceptions种受查异常，每一种受查异常使用一个exception_index_table项表示，exception_index_table是一个指向常量池中COSNTANT_Class_info型常量的索引，代表了该受查异常的类型。

### （3）LineNumberTable属性
LineNumberTable属性用于描述Java源码行号与字节码行号（字节码的偏移量）之间的对应关系。它并不是运行时必须的属性，但默认会生成到Class文件中。（可以在javac中使用-g:none或-g:lines选项来取消或要求生成这项信息。）

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length     | 1 |
| u2 | line_number_table_length | 1 |
| line_number_info | line_number_table | line_number_table_length |

line_number_table是一个数量为line_number_table_length、类型为line_number_info的集合，line_number_info表包含start_pc和line_number两个u2类型的数据项，前者是字节码行号，后者是Java源码行号。

### （4）LocalVariableTable及LocalVariableTypeTable属性
LocalVariableTable属性用于描述栈帧中局部变量表的变量与Java源码中定义的变量之间的关系，它也不是运行时必须的属性，但默认会生成到Class文件中，可以在javac中使用-g:none或-g:vars选项来取消或要求生成这项信息。

如果没有生成这项属性，最大的影响就是当其他人引用这个方法时，所有的参数名称都将会丢失，譬如IDE会使用诸如arg0、arg1之类的占位符代替原有的参数名。LocalVariableTable属性的结构如表所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length     | 1 |
| u2 | local_variable_table_length | 1 |
| local_variable_info | local_variable_table | local_variable_table_length |

其中local_variable_info项目代表了一个栈帧与源码中的局部变量的关联，结构如下：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | start_pc | 1 |
| u2 | length   | 1 |
| u2 | name_index | 1 |
| u2 | descriptor_index | 1 |
| u2 | index | 1 |

- start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖的长度，两者结合起来就是这个局部变量在字节码中的作用域范围。
- name_index和desriptor_index都是指向CONSTANT_Utf8_info型常量的索引，分别代表了局部变量的名称及其描述符。
- index时这个局部变量在栈帧的局部变量表中变量槽的位置。当这个变量数据类型时64为类型时（double和long），它占用的变量槽就是index和index+1两个。

在JDK 5引入泛型后，LoacalVariableTable属性增加了一个“姐妹属性”——LocalVariableTypeTable。这个新增的属性与LocalVariableTable非常相似，仅仅是把descriptor_index替换为了字段的特征签名。对于非泛型类型来说，描述符和特征签名描述的信息吻合一致。但是泛型引入后，由于描述符中泛型的参数化类型会被擦除掉，描述符就不能准确描述泛型类型了，因此出现了LocalVariableTypeTable属性，使用字段的特征签名来完成泛型的描述。

### （5）SourceFile及SourceDebugExtension属性
SourceFile属性用于记录生成这个Class文件的源码文件名称。这个属性也是可选的，可以使用javac的-g:none或-g:source选项来关闭或要求生成这项信息。这个属性是个定长的属性，结构如下：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | sourcefile_index | 1 |

sourcefile_index数据项时指向常量池中CONSTANT_Utf8_info型常量的索引，常量值时源码文件的文件名。

为了方便在编译器和动态生成的Class中加入供程序员使用的自定义内容，JDK 5新增了SourceDebugExtension属性用于存储额外的代码调试信息，结构如下：
| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u1 | debug_extension[attribute_length] | 1 |

其中debug_extension存储的就是额外的调试信息，是一组通过变长的UTF-8格式来表示的字符串。一个类中最多允许存在一个SourceDebugExtension属性。

### （6） ConstantValue属性
ConstantValue属性的作用是通知虚拟机自动为静态变量赋值。只有被static关键字修饰过的变量（类变量）才可以使用这项属性。对非static变量（也就是实例变量）的赋值是在实例构造器<init>()方法中进行的，而对于类变量则有两种方式选择：在类构造器<clinit>()方法中或使用ConstantValue属性。目前Oracle公司实现的javac编译器的选择是，如果同时使用final和static来修饰一个变量，并且这个变量是基本类型或java.lang.String的话，就会生成ConstantValue属性来进行初始化；如果没有被final修饰，或者并非基本类型及字符串，那么将会在<clinit>()方法中进行初始化。

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | constantvalue_index | 1 |

可见ConstantValue是一个定长属性，它的attribute_length数据项必须固定为2。constantvalue_index数据项代表了常量池中一个字面量常量的引用，根据字段类型的不同，字面量可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info和CONSTANT_String_info常量中的一种。

### （7） InnerClass属性
InnerClass属性用于记录内部类与宿主类之间的关联。如果一个类中定义了内部类，那么编译器将会为它以及它所包含的内部类生成InnerClass属性，其结构如下所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | number_of_classes | 1 |
| inner_classes_info | inner_classes | number_of_classes |

数据项number_of_classes代表需要记录多少个内部类信息，每一个内部类的信息都由一个inner_classes_info表进行描述。inner_classes_info表结构如下：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | inner_class_info_index | 1 |
| u2 | outer_class_info_index | 1 |
| u2 | inner_name_index | 1 |
| u2 | inner_class_access_flags | 1 |

- inner_class_info_index和outer_class_info_index都是指向常量池CONSTANT_Class_info型常量的索引，分别代表了内部类和宿主类的符号引用。
- inner_name_index是指向常量池中CONSTANT_Utf8_info型常量的索引，代表这个内部类的名称，如果是匿名内部类，这项值为0。
- inner_class_access_flags是内部类的访问标志，类似于类的access_flags，它的取值范围如下表所示：

| 标志名称 | 标志值 | 含义 |
| ---- | --- | --- |
| ACC_PUBLIC | 0x0001 | 内部类是否为public |
| ACC_PRIVATE | 0x0002 | 内部类是否为private |
| ACC_PROTECTED | 0x0004 | 内部类是否为protected |
| ACC_STATIC | 0x0008 | 内部类是否为static |
| ACC_FINAL | 0x0010 | 内部类是否为final |
| ACC_INTERFACE | 0x0020 | 内部类是否为接口 |
| ACC_ABSTRACT | 0x0400 | 内部类是否为abstract |
| ACC_SYNTHETIC | 0x1000 | 内部类是否并非由用户代码生成 |
| ACC_ANNOTATION | 0x2000 | 内部类是不是一个注解 |
| ACC_ENUM | 0x4000 | 内部类是不是一个枚举 |

### （8） Deprecated及Synthetic属性
Deprecated和Synthetic两个属性都属于标志类型的布尔属性，只存在有和没有的区别，没有属性值的概念。Deprecated属性用于表示某个类、字段或者方法，已经被程序作者定为不再推荐使用，它可以通过代码中使用“@deprecated”注解进行设置。所有不是用户代码产生的类、方法及字段都应当至少设置Synthetic属性或者ACC_SYNTHETIC标志位中的一项，唯一的例外是实例构造器方法“<init>()”和类构造器方法“<clinit>()”。

Deprecated属性和Sythetic属性的结构非常简单：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |

其中，attribute_length数据项的值必须为0x00000000，因为没有任何属性需要设置。

### （9） StackMapTable属性
StackMapTable属性在JDK 6增加到Class文件规范中，它是一个相当复杂的变长属性，位于Code属性的属性表中。这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（Type Checker）使用，目的在于代替以前比较消耗性能的基于数据流分析的类型推导验证器。

StackMapTable属性中包含零至多个栈映射帧，每个栈映射帧都显式或隐式地代表了一个字节码偏移量，用于表示执行到该字节码时局部变量表和操作数栈的验证类型。类型检查验证器会通过检查目标方法的局部变量表和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。StackMapTable属性的结构如下所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | number_of_entries | 1 |
| stack_map_frame | stack_map_frame entries | number_of_entries |

### （10） Signature属性
Signature属性在JDK 5增加到Class文件规范之中，它是一个可选的定长属性，可以出现于类、字段表和方法表结构的属性表中。在JDK 5里面大幅增强了Java语言的语法，在此之后，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variable）或参数化类型（Parameterized Type），则Signature属性会为它记录泛型签名信息。

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | signature_index | 1 |

其中signature_index项的值必须是一个对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示类签名或方法类型签名或字段类型签名。

### （11） BootStrapMethods属性
BootStrapMethods属性在JDK 7时增加到Class文件规范中，它是一个复杂的变长属性，位于类文件的属性表中。这个属性用于保存invokedynamic指令引用的引导方法限定符。

根据《Java虚拟机规范》的规定，如果某个类文件结构的常量池曾经出现过COSNTANT_InvokeDynamic_info类型的常量，那么这个类文件的属性表中必须存在一个明确的BootstrapMethods属性。另外，即使CONSTANT_InvokeDynamic_info类型的常量在常量池中出现过多次，类文件的属性表中最多也只能由一个BootstrapMethods属性。

虽然JDK 7中已经存在了InvodeDynamic指令，但这个版本的javac编译器还暂时无法支持InvokeDynamic指令和生成BoootstrapMethods属性，直到JDK 8中Lambda表达式和接口默认方法的出现，InvodeDynamic指令才算在Java语言生成的Class文件中有了用武之地。Bootstrap属性的结构如下所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | num_bootsrap_methods | 1 |
| bootstrap_method | bootsrap_methods | num_bootsrap_methods |

其中引用到的bootstrap_method结构为：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | bootstrap_method_ref | 1 |
| u2 | num_bootstrap_arguments | 1 |
| u2 | bootstrap_arguments | num_bootstrap_arguments |

BootstrapMethods属性里，num_bootstrap_methods项的值给出了bootstrap_methods[]数组中的引导方法限定符的数量。bootstrap_methods[]数组的每个成员必须包含以下三项内容：
- bootstrap_method_ref：bootstrap_method_ref项的值必须是一个对常量池的有效索引，代表了一个引导方法。常量池在该索引处的值必须是一个CONSTANT_MethodHandle_info结构。
- num_bootstrap_arguments：num_bootstrap_arguments项的值给出了bootstrap_arguments[]数组成员的数量。
- bootstrap_arguments[]：bootstrap_arguments[]代表了一个引导方法静态参数的序列，该数组的每个成员必须是一个对常量池的有效索引。常量池在该索引处的值必须是下列结构之一：CONSTANT_String_info、CONSTANT_Class_info、CONSTANT_Integer_info、CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_MethodHandle_info或CONSTANT_MethodType_info。

### （12）MethodParameters属性
MethodParameters是在JDK 8加入到Class文件格式中的，它是一个用在方法表中的变长属性，与Code属性平级。MethodParamters的作用是记录方法的各个形参名称和信息，其结构如下：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u1 | parameters_count | 1 |
| parameter | parameters | parameters_count |

其中引用到的parameter结构如下所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | name_index | 1 |
| u2 | access_flags | 1 |

其中，name_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该参数的名称。而access_flags是参数的状态指示器，它可以包含以下三种状态中的一种或多种：
- 0x0010（ACC_FINAL）：表示该参数被final修饰
- 0x1000（ACC_SYNTHETIC）：表示该参数并未出现在源文件中，是编译器自动生成的。
- 0x8000（ACC_MANDATED）：表示该参数是在源文件中隐式定义的。Java语言中的典型场景是this关键字。

### （13）模块化相关属性
JDK 9的一个重量级功能是Java的模块化功能，因为模块描述文件（module-info.java）最终是要编译成一个独立的Class文件来存储的，所以Class文件格式也扩展了Module、ModulePackages和ModuleMainClass三个属性用于支持Java模块化相关功能。

### （14）运行时注解相关属性
早在JDK 5时期，Java语言提供了对注解（Annotation）的支持。为了存储源码中的注解信息，Class文件同步增加了RuntimeVisibleAnnotations、RuntimeInvisibleAnnotations、RuntimeVisibleParameterAnnotations和RuntimeInvisibleParameterAnnotations四个属性。JDK 8新增了类型注解，所以Class文件中同步增加了RuntimeVisibleTypeAnnotations和RuntimeInvisibleTypeAnnotations两个属性。以上六个属性结构和功能都比较雷同，故以RuntimeVisibleAnnotations为代表进行介绍。

RuntimeVisibleAnnotations是一个变长属性，它记录了在类、字段或方法的声明上的运行时可见注解，当我们使用反射API获取类、字段或方法上的注解时，返回值就是通过这个属性取到的。其结构如下所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | num_annotations | 1 |
| annotation | annotations | num_annotations |

num_annotations是annotations数组的计数器，annotations中每个元素都代表了一个运行时可见注解，注解在Class文件中以annotation结构来存储，具体如下所示：

| 类型 | 名称 | 数量|
| ---- | --- | --- |
| u2 | type_index | 1 |
| u2 | num_element_value_pairs | 1 |
| element_value_pair | element_value_pairs | num_element_value_pairs |

type_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，该常量应以字段描述符的形式表示一个注解。num_element_value_pairs是element_value_pairs数组的计数器，element_value_pairs中每个元素都是一个键值对，代表该注解的参数和值。


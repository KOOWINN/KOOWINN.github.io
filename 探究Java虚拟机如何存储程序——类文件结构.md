Class文件时一组以8个字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。

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
每个Class文件的头4个字节被称为魔数，它的唯一作用是确定这个文件是否为一个能被虚拟机额接受的Class文件。（不仅仅是Class文件，很多文件格式标准中都使用了魔数来进行身份识别，譬如图片。）Class文件的魔数值为“0xCAFEBABE”。

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
CONSTANT_Integer_info型常量的结构为：
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
| ACC_PUBLIC | 0x0001 | 字段是否public |
| ACC_PRIVATE |0x0002 | 字段是否private |
| ACC_PROTECTED | 0x0004 | 字段是否protected |
| ACC_STATIC | 0x0008 | 是实例变量还是类变量 |
| ACC_FINAL | 0x0010 | 字段是否可变 |
| ACC_VOLATILE | 0x0040 | 字段是否强制从主内存读写（并发可见性） |
| ACC_TRANSIENT | 0x0080 | 字段是否可被序列化 |
| ACC_SYNTHETIC | 0x1000 | 字段是否由编译器自动生成 |
| ACC_ENUM | 0x4000 | 字段是否enum |

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


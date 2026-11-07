## php中的一些比较有意思的函数(2)

### extract函数
`extract` 函数==用于将关联数组中的键名直接转换为当前的变量，键值则对应变量的值==，并返回成功设置的变量数目

- 语法
	`extract(<array>,<EXTR_OVERWRITE>/<EXTR_SKIP>/<EXTR_PREFIX_SAME>/<EXTR_PREFIX_ALL>,<num>)`
                      (可选参数)                                                             (可选参数)
常用参数 (Flags)

- `EXTR_OVERWRITE`：默认值，如果有冲突，覆盖已有的变量。
- `EXTR_SKIP`：如果有冲突，不覆盖已有的变量。
- `EXTR_PREFIX_SAME`：冲突时，给变量名加上前缀。
- `EXTR_PREFIX_ALL`：给所有变量名加上前缀。


### extract函数变量覆盖问题
使用extract函数让_GET()函数传入的多个值覆盖文件中多个相同变量的值，以实现覆盖绕过

![extract函数变量覆盖示例](<d1d781ece688b93a80c212cda4ba593e_720.jpg>)

### parse_str函数
`parse_str()`用来把类似 URL 查询的字符串（如 `name=John&age=30`或`<arr>[<a>]=xxx&<arr>[<b>]=xxx`）解析成数组或变量

- 语法
	`parse_str(<string>,<第二参数变量>)`


- **`string`**: 需要解析的输入字符串。
- **`第二参数变量`**: 必需的第二个参数（传引用），用来存储解析后的键值对数组



### parse_str函数变量覆盖问题
利用parse_str()将_GET或_POST方法传入的查询字段转换成数组的形式，再利用extract()函数将文件中的相关变量值替换为数组中的同名变量值进行绕过

![parse_str函数变量覆盖示例](<45dae063e1b5d73de95e6e4522bf2afc_720.jpg>)


### `$`变量覆盖绕过问题
利用php语言的特性，使用多个`$`符设置多层变量来绕过使得程序正常执行

![$变量覆盖绕过示例](<efa465104374d66068180e2c7f1bd726_720.jpg>)

### 内置类-Exception类相关问题(可以实现RCE)

#### Exception类函数
`Exception` 类是所有用户代码异常的基类，主要包含 `getMessage`、`getFile` 和 `getLine` 等核心方法。

- 语法
	try{
	<函数体>
	}
	catch(Exception <捕获异常的一个随便变量名>){
	<函数体>
	}

核心方法
- `getMessage()`：获取异常信息内容。
- `getCode()`：获取异常代码数字。
- `getFile()`：获取发生异常的文件名。
- `getLine()`：获取异常所在的行号。
- `getTrace()`：获取异常追踪信息数组。
- `getPrevious()`：返回前一个异常（用于异常链）

#### Exception类绕过问题


![Exception类绕过示例](<97e8d44a0bafbddc5d9fb57d989a9df5_720.jpg>)


### 内置类-Reflection类相关问题(可以实现RCE)


#### Reflection类函数
反射（Reflection）是一套内置的元编程 API，核心组件包括 ReflectionClass、ReflectionMethod 和 ReflectionParameter。它允许程序在运行时**自我检查**、**动态获取结构信息** 并**直接操作**类、方法、属性及注释，常用于现代框架的依赖注入容器。

- 语法
	`<变量1>=new Reflection(<类名/对象名>)`

核心反射类

- **ReflectionClass**：用于获取和操作类本身的信息，如类名、父类、接口、属性、方法以及构造函数。
- **ReflectionMethod**：用于检查类中某个特定方法的修饰符（如 `public`、`private`）、参数和返回值。
- **ReflectionParameter**：用于检查函数或方法中每个参数的类型、默认值以及是否可选。
- **ReflectionFunction**：用于对全局独立函数进行反射检查。


#### Reflection类绕过问题


![Reflection类绕过示例](<9ce5a9811ab7a8b84976e85c6e9a563d_720.jpg>)




### 内置类-FilesystemIterator类相关问题(可以实现目录遍历)

#### FilesystemIterator类相关函数
[FilesystemIterator](https://php.ac.cn/manual/en/class.filesystemiterator.php) 类是一个用于高效遍历文件系统目录的内置迭代器，它继承自 `DirectoryIterator`，核心优势包括：返回完整的路径作为键、支持通过标志位（Flags）过滤隐藏或特殊目录（如 `.` 和 `..`）、以及直接产出 `SplFileInfo` 对象。

- 语法
	`<变量1>=new Reflection(<类名/对象名>)`


- **继承关系**：扩展自 `DirectoryIterator`，行为更灵活。
- **键与值**：默认以文件的完整路径作为键（Key），以 `SplFileInfo` 对象作为当前迭代值（Value）。

**常用标志（Flags）**：
- `FilesystemIterator::SKIP_DOTS`：自动跳过 `.` 和 `..` 目录。
- `FilesystemIterator::CURRENT_AS_PATHNAME`：当前项仅返回路径字符串。
- `FilesystemIterator::KEY_AS_FILENAME`：将文件名作为键。 [[1](https://php.ac.cn/manual/en/class.filesystemiterator.php)]

常用方法
- **`__construct(string $path, int $flags = ?)`**：构造函数，传入目标目录路径及可选的控制标志。
- **`setFlags(int $flags)`**：设置处理标志。
- **`getFlags()`**：获取当前生效的标志。
- **`current()`**：返回当前文件信息（根据标志返回对象或路径）。



#### FilesystemIterator类相关函数遍历目录问题



![FilesystemIterator遍历目录示例](<0778633980a9d0ee0e29b9c072afea21_720.jpg>)


### 补：遍历当前目录文件并显示文件内容payload

`payload:show_source(next(array_reverse(scandir(pos(localeconv())))));`

![遍历当前目录并显示文件内容示例](<e1f2a40bc5429c6b8b9d81e77a6df02b_720.jpg>)



## php解析特性

### GET参数解析问题-1
利用phpGET参数解析特性来绕过参数传递失败问题

![GET参数解析问题示例](<72db0baa6be51543464e1baa34a1f9a4_720.jpg>)

#### GET参数解析图

![GET参数解析图](<Pasted image 20260731162045.png>)


### eval函数与assert函数
eval函可以==执行字符串php语句==；`assert` 函数是一个==用于开发和测试环境的调试工具，核心作用是**检查一个表达式是否成立**，支持表达式或字符串，并在生产环境中可配置为零成本关闭==。

- 语法
	- eval函数
		`eval("<php语句>")`
	- assert函数
		`assert(<测试的表达式或对象>, <exception>)`
                                <可选参数>



### $_SERVER[<数组名>]数组介绍
`$_SERVER['<数组名>']` 是一个包含传递给当前脚本参数的数组，常用于**命令行（CLI）模式**和开启特定配置的**网页（Web）模式**。它的第一个元素 `$<数组名>[0]` 始终是脚本的文件名，后续元素则是传参。



### GET参数解析问题-2
通过$_SERVER[<数组名>]数组匹配请求方式传递进的参数，再使用变量覆盖的方式覆盖绕过if语句执行后续语句

![SERVER数组参数覆盖示例](<990d18c33f79ea401a20e172195344d7_720.jpg>)

















































































































































































































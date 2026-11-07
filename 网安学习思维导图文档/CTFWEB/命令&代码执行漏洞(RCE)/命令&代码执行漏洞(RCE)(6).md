
# 补：MVC开发模式介绍
MVC开发模式是==一种将软件系统分为**模型（Model）**、**视图（View）**  和 **控制器（Controller）** 三个核心部分的软件架构模式==。它可以有效分离业务逻辑与页面显示，让代码更容易维护和扩展

> [!NOTE]
> MDN Web 文档 MVC模式详细介绍： https://developer.mozilla.org/zh-CN/docs/Glossary/MVC

- 什么是MVC模式
	- demo
	- MVC的全称是(Model View Controller）
		- `Model`:`模型`
		- `View`：`视图`
		- `Controller`:`控制器`
	- 是一种将业务逻辑、数据和界面显示分离的代码组织方式
	- 在80年代由`Xerox PARC`发明


==三大核心组件==

- **Model（模型）**：==管理应用程序的**数据、状态和底层业务逻辑**==。它直接与数据库交互，处理数据的增删改查。当数据发生改变时，它会通知视图进行更新。
- **View（视图）**：==负责**界面的布局和显示**==。它把模型中的数据呈现给用户，是用户能看到并操作的界面（如网页、App页面）。
- **Controller（控制器）**：==充当**桥梁与协调者**==。它接收用户的输入和请求，调用模型处理业务，然后决定将哪个视图返回并展示给用户

==工作流程==

- **用户操作**：用户在视图（View）上进行点击、提交表单等操作。
- **控制器响应**：控制器（Controller）接收到请求，分析用户的意图。
- **调用模型**：控制器调用模型（Model）去更新数据或读取数据。
- **更新视图**：模型处理完数据后返回给控制器，控制器再把最新数据传给视图（View）渲染，最终呈现在用户面前

==优势与应用==

- **职责清晰**：美工专注视图，后端专注业务和模型，分工明确。
- **易于维护**：修改界面不影响业务逻辑，修改业务逻辑也不需要重构界面。
- **广泛应用**


![](88af0f5ae2b65349e5cdebda7e7e3f75.jpeg)



## 在php中的MVC模式详细介绍


### Model(模型)

- 更具体的MVC模式介绍
	- Model(模型)
	- ==实际业务逻辑和数据的封装==
	- ==作为数据返回==
	- ==通常与数据库操作相关==
		- 增删改查
		- 对象属性操作
		- ……


例：
![](0f32fd289e64a1a122bf93dc71230f41.jpeg)



### View(视图)

- 更具体的MVC模式介绍
	- View(视图)
	- 负责信息的输出和展示
	- HTML、CSS、JS等

例：
![](dc2e4f9615c0cd6e583c73d8b124683a.jpg)



### Controller（控制器）

- 更具体的MVC模式介绍
	- Controller（控制器）
	- MVC框架中的核心
	- ==用于处理用户输入输出==


例：
![](2d0c24ae5464c727585b3f7330e8f94e.jpeg)




### 为什么要有MVC模式

- ==为什么要有MVC模式==
	- 解耦代码，方便维护
	- 各执其职，方便开发分工
	- 模块化代码
- PHP中常见的MVC框架
	- ThinkPHP
	- Laravel
	- ……


![](dd4aa83e643a5b37987211fc1b4f5bfa.jpeg)



# 补：ThinkPHP与MVC==(2.x版本)==

## ThinkPHP框架简介
**ThinkPHP** 是一个免费开源的、面向对象的==轻量级国产 PHP 开发框架==，旨在简化企业级和敏捷 Web 应用开发

==主要特点==

- **MVC 模式**：采用模型（Model）、视图（View）、控制器（Controller）的分层设计，让代码结构更清晰。
- **简单高效**：秉承简洁实用的设计原则，提供丰富的文档和强大的路由、ORM（对象关系映射）功能。
- **开源协议**：遵循 Apache2 开源协议发布，允许免费使用以及商业化发布与销售。
- **兼容性强**：支持 Windows、Linux 等多种服务器环境，以及 MySQL、SQLite 等主流数据库


## `thinkphp 2.x`版本源码结构简介
ThinkPHP 2.x（如 2.0/2.1）采用经典的 **MVC（模型-视图-控制器）** 设计模式，其源码和目录结构相对轻量，核心与项目目录分离清晰

==整体目录结构==
一个标准的 ThinkPHP 2.x 项目通常分为**入口文件**、项目应用目录（Lib/Tpl等）和框架核心目录（ThinkPHP）

==`Common`:用户定义的功能类==
==`Conf`:配置==
==`Lib`:用户二开实现的类逻辑==
==`Runtime`:TP运行时生成的文件==
==`ThinkPHP`：框架的核心逻辑==
==`Tpl`:视图==
==`index.php`:入口==


![](a8cac9908dcfe4f03e468453f87e0bf2.jpg)


![](fd16e899fd38e4e8fe3a0cc32a9b027c.jpeg)



### Runtime文件夹中的结构

==`Cache`:页面缓存==
==`Data`:临时数据==
==`Temp`:临时文件==
==`Logs`:运行时日志==


![](d386d9559cb7abc6e2930eaf494aeb62_720.jpg)


### ThinkPHP文件夹中的结构


==`Common`:`tp`的常用类库==
==`Lib`:`tp`的核心依赖==
	==`Model`、`View`等类都在此==
==`Mode`:不同运行模式的处理逻辑==


![](c27a23515c99c6b2f71583bfe76bf186.jpeg)



## TP框架执行结构
ThinkPHP 2.x 框架的执行流程（生命周期）==主要围绕**入口加载**、**初始化**、**URL调度**和**控制器执行**展开==。当用户发起一次 URL 请求时，系统会按顺序执行以下核心步骤


1. ==加载公共入口文件==

- 用户访问网站入口文件（如 `index.php`）。
- 入口文件引入框架的核心总入口文件 `ThinkPHP.php`，开始记录初始运行时间与内存开销，并定义常量（如路径、环境等）。 [[1](https://doc.thinkphp.cn/v3_2/1704.html), [2](https://blog.csdn.net/weixin_38166931/article/details/99948437)]

1. ==项目初始化 (App::init)==

- 设定系统的错误和异常处理机制。
- 启动 Session 会话过滤与初始化。
- 设置系统的时区支持与语言包读取。 [[1](http://form.baidu.com/view/66ce971fce84b9d528ea81c758f5f61fb636287b.html)]

1. ==项目预编译==

- 依次加载系统惯例配置文件（`convention.php`）和项目自定义配置文件（`config.php`）。
- 检查并加载核心类库、插件及别名定义。 [[1](https://blog.csdn.net/weixin_38166931/article/details/99948437)]

1. ==URL 分析与调度 (Dispatcher)==

- 检查当前的 URL 模式（普通模式、PATHINFO、REWRITE 或兼容模式）。
- 解析 URL 中的参数、GET/POST 变量。 [[1](https://cndaqiang.github.io/2017/11/26/thinkphp/), [2](https://blog.csdn.net/weixin_38166931/article/details/99948437)]

1. ==获取模块和操作名==

- 从 URL 或配置中提取当前请求的**模块名 (Module)** 和**操作名 (Action/Method)**。
- 动态加载对应模块的配置文件及权限检测（如 RBAC）。 [[1](https://blog.csdn.net/weixin_38166931/article/details/99948437), [2](http://form.baidu.com/view/66ce971fce84b9d528ea81c758f5f61fb636287b.html)]

1. ==项目执行 (App::exec)==

- 检查并自动加载（AutoLoad）所需的公共类库。
- 实例化当前请求的 Action 控制器类。
- 依次执行前置操作方法（`_before_`）、核心操作方法、后置操作方法（`_after_`）。
- 调用视图（View）渲染模板输出页面，最后执行应用结束过滤器（`app_end`）完成响应



例：
![](67fb265fce59409c651f58fb2fcb714e_720.jpg)
									==Dispatcher用于解析和并发==




# TP RCE漏洞介绍==(PHP < 5.5)==

## 补：preg_replace函数简介
`preg_replace` 是 PHP 中==用于执行正则表达式搜索和替换的内置函数==，并返回替换后的结果

- 语法：
```
preg_replace ( <$pattern> , <$replacement> , <$subject [, int $limit = -1 [, int &$count ]]> )
```
* **`pattern`（模式）**：要搜索的正则表达式，可以是一个字符串或一个字符串数组。 
* **`replacement`（替换内容）**：用于替换的字符串或字符串数组。 
* **`subject`（目标字符串）**：要搜索和替换的目标字符串或字符串数组。 
* **`limit`（可选）**：每个模式在每个 `subject` 字符串上进行替换的最大次数。默认是 `-1`（无限次）。 
* **`count`（可选）**：如果指定，会被填入执行替换的次数


## 补：preg_replace函数与e匹配项修饰符简介==(PHP < 5.5)==
`preg_replace` 的 `/e` 修饰符（`PREG_REPLACE_EVAL`）用于开启 **`eval` 动态执行模式**；==当在正则表达式末尾指定 `/e` 时，PHP 会在完成正则匹配和反向引用替换后，将替换字符串（`replacement`）当作 **PHP 代码** 通过 `eval()` 进行解析与执行，并将执行结果作为最终的替换内容==


![](3a0357599b2e17e8ba5c113d3696ce3b_720.jpg)


例：
![](Pasted%20image%2020260825192014.png)


## 补：php `${}`执行函数==(PHP<7)==
在PHP中，`$` 与 `{}` 结合使用（如 `${...}` 或 `${${...}}`）==通常用于可变变量（Variable variables）或动态代码执行==，在安全领域和特定动态逻辑中非常常见

1. 基本概念

- **`$`（美元符号）**：用于定义和引用变量。
- **`${}`（大括号/花括号）**：==在PHP中用于**定界变量**，或者实现**可变变量**（即把一个变量的值作为另一个变量的名或表达式执行）==

1. 核心机制：可变变量与动态执行
当使用 `${}` 嵌套或包裹代码时，PHP 会把里面的内容先解析为一个变量或执行特定表达式


例：
![](1eda230a69d25d7b02cf0b4047fce01b_720.jpg)



## TP Dispatcher模块URL 路由解析漏洞RCE漏洞(`WooYun-2012-04987`)
根本原因在于 URL 路由分发模块中不安全地使用了带有 `/e` 修饰符的 `preg_replace` 函数

==漏洞定位与成因==

该漏洞位于==核心调度器文件== `ThinkPHP/Lib/Think/Util/Dispatcher.class.php`（或部分版本的 `Core/Dispatcher.class.php`）中的 ==`Dispatcher::dispatch()`== 方法

==**触发机制分析**：==

1. **URL 参数解析**：ThinkPHP 在解析 PATHINFO 格式的 URL（例如 `/module/action/key/value`）时，会将模块名和方法名弹出（`array_shift`），剩余的键值对参数组合为字符串并交给 `preg_replace` 处理。
    
2. **`/e` 代码执行**：由于正则尾部指定了 `@e` 修饰符，PHP 会将替换模板 `'$var[\'\\1\']="\\2";'` 当作 PHP 代码通过 `eval()` 执行。
    
3. **双引号变量解析注入**：替换模板中 `\\2`（即 URL 中的参数值）被包裹在双引号 `"` 中。在 PHP 中，双引号字符串支持复杂变量语法 `${...}`，攻击者在参数值中构造特定表达式即可触发代码执行

102行的preg_match中的e参数
代码分析：去除ur|路径中的相邻两个参数，第一个参数为数组的`key`第二个为`value/kanxue/123 -> $var[kanxue]=value_evil`
如果`value_evil`可控,就能RCE


例：
![](8cf4714ad86853781e606c979ed0a17c_720.jpg)



poc：
`/index.php?s=Index/index/name/${phpinfo()}`

**服务端执行过程**：

1. 正则匹配捕获组：
    - `\\1` (键名) = `name`
        
    - `\\2` (键值) = `${@phpinfo()}`
        
2. 替换字符串拼接后的待执行 PHP 代码：
```
    $var['name']="${@phpinfo()}";
```
3. PHP `eval()` 执行该行代码时，双引号内部的 `${@phpinfo()}` 表达式被动态解析并直接执行，从而实现 RCE

![](85eef47e803951dc364af199416053aa.jpeg)



# ThinkPHP与MVC==(5.x版本)==

## TP 5.x版本源码结构简介
ThinkPHP 5 采用==全新的架构思想==和==单入口模式==，其根目录标准源码结构主要包含核心框架、应用代码及对外访问等目录

==核心根目录结构==

- ==**`application/`**==：**应用目录**，存放整个网站的业务逻辑代码，包含各个模块（如 `index` 模块、`common` 公共模块）以及全局配置文件（如 `config.php`、`database.php`）。
- ==**`public/`**==：**WEB 部署目录**（对外访问目录），包含网站的单入口文件 `index.php` 以及静态资源（如 `static` 目录下的 CSS、JS、图片）。
- ==**`thinkphp/`**==：**框架系统目录**，核心类库（`library`）、系统 Trait、默认语言包和基础引导文件均在此处。
- ==**`extend/`**==：**扩展类库目录**，用于存放自定义的非 Composer 扩展类库。
- ==**`vendor/`**==：**第三方类库目录**，通过 Composer 安装的管理包和第三方依赖存放在这里。
- ==**`runtime/`**==：**运行时目录**，用于存放系统的缓存、日志和临时生成的文件（需要可写权限）。
- ==**`think`**==：**命令行入口文件**，用于执行 Console 命令行指令



![](ee65d70e48c75d50734fda6e2b5b703a_720.jpg)


![](d8ef7f8131bfc4f1c44bcb9aa40739d5_720.jpg)


# 使用`Xdebug`远程调试复杂框架

## `Xdebug`简介
**Xdebug** ==是一个开源的 PHP 扩展，主要用于协助开发人员进行 PHP 代码的**调试**、**性能分析**和**代码覆盖率统计**==

==核心功能==

- **单步调试 (Step Debugging)**：支持在 IDE（如 PhpStorm）中设置断点、单步执行、查看变量实时值和调用栈。
- **增强的 `var_dump()`**：让内置的 `var_dump` 输出色彩更丰富、结构更清晰，并限制递归深度。
- **堆栈跟踪 (Stack Traces)**：当发生错误、警告或异常时，展示详细的错误路径和文件行号。
- **性能分析 (Profiling)**：记录每个函数的执行时间和内存消耗，帮助找出代码运行瓶颈。
- **代码覆盖率 (Code Coverage)**：为单元测试（如 PHPUnit）提供准确的代码覆盖率报告

==注意事项==

- **生产环境禁用**：Xdebug 会消耗额外的系统资源，显著降低程序运行速度，**不建议**在生产环境开启

例：
![](Pasted%20image%2020260825215340.png)



# TP RCE漏洞介绍==(5.1.x>= PHP >=5.0.22)==

## 补：call_user_func函数介绍

**具体请参考：**[php中存在的相关问题(1)](php中存在的相关问题(1).md#call_user_funs函数)

## 补：php反射简介
PHP 反射（Reflection）是指==在**运行状态中**，动态检查和导出类、接口、函数、方法和属性的详细信息以及文档注释的能力==。你可以把它理解为==代码的“自我审视（Introspection）”==

==反射可以用来获取PHP上下文中的类和方法==

==一、 核心作用==

- **获取结构信息**：在运行时知道一个类有哪些方法、属性、常量以及它们的访问权限（public/protected/private）。
- **读取注释**：可以解析方法或类上面的 DocComment（文档注释），常用于框架中的路由注解或文档生成。
- **依赖注入（DI）**：现代 PHP 框架（如 Laravel、Symfony）利用反射读取构造函数的参数类型，自动解析并实例化依赖对象。
- **动态调用**：无需提前硬编码，可以直接动态实例化类并执行方法


==二、 常用反射类==

PHP 提供了一套完整的反射 API，常见的有：

- ==**`ReflectionClass`**==：用于获取类的信息（如父类、接口、方法、属性）。
- **`ReflectionMethod`**：用于获取和操作类的方法（如执行方法、检查权限）。
- **`ReflectionProperty`**：用于获取和修改类的属性（甚至能修改私有属性）。
- **`ReflectionFunction`**：用于获取全局函数的信息。
- **`ReflectionParameter`**：用于获取函数或方法参数的信息（如类型约束、默认值）


==三、 优缺点==

- ==**优点**：==极大提升了程序的灵活性和动态扩展能力，是现代框架和服务容器的基石。
- ==**缺点**：==由于是在运行时动态解析，反射的**性能开销**比直接调用代码要大，应避免在性能极其敏感的高频循环中大量使用


互联网示例： https://learnku.com/articles/37162


![](67f7b0300ee55fa61673965c24a06037.jpeg)



## TP php反射加call_user_func函数路由未过滤导致的 RCE==(CVE-2018-20062)==
在 `ThinkPHP 5.x` 系列（如 5.0.x、5.1.x）中，==由于路由调度未对控制器名称进行严格的过滤与类型限制==，导致==攻击者可以通过 URL 传入任意类名与方法名==。结合 `ThinkPHP` 内部用于依赖注入和参数绑定的 **PHP 反射机制（Reflection）** 以及 **`call_user_func` / `call_user_func_array`**，攻击者可直接调用任意系统函数实现远程代码执行（RCE）

==漏洞成因==

整个漏洞的触发链路主要由三部分构成：
```
URL 传入恶意类与方法 
  → ThinkPHP 路由调度（未过滤命名空间）
  → 反射获取参数并绑定用户输入（ReflectionClass/Method/Function）
  → 动态执行恶意函数（call_user_func_array / invokeArgs）
```

1. 控制器路由调度未过滤

ThinkPHP 支持兼容模式路由解析（通过 `s` 参数指定 `模块/控制器/操作`）。当没有对控制器名称进行合法性检测（如允许出现反斜杠 `\`）时，==攻击者可以通过传入完整的命名空间，强行将目标控制器指定为框架内部的任意类==（如 `\think\App`、`\think\Request`、`\think\Container` 等）


2. 反射机制（Reflection）的参数绑定

为了实现灵活的依赖注入与参数绑定，==ThinkPHP 在 `App::invokeFunction` 或 `App::invokeMethod` 中使用了 PHP 的反射 API==：

- 通过 `new \ReflectionFunction($function)` 或 `new \ReflectionMethod($class, $method)` 获取目标函数的形参列表。
    
- 遍历反射参数，自动从当前请求参数（`$_GET` / `$_POST` / `$vars`）中提取同名变量进行赋值。
    
- 通过 `$reflect->invokeArgs($args)` 动态传入参数并执行


3. 结合 `call_user_func_array` 实现任意调用

如果直接指定 `$function` 为某些内置类方法（如 `\think\App::invokeFunction`），==并传入 `function=call_user_func_array`，反射机制会自动将请求中的参数组装并传递给 `call_user_func_array`，最终执行如 `system()`、`passthru()`、`assert()` 等敏感命令==


==poc例：==
```
/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=whoami(任意系统命令)
```

- **控制器解析**：`\think\app`
    
- **操作方法**：`invokefunction`
    
- **执行过程**：`ReflectionFunction` 实例化 `call_user_func_array`，参数绑定 `$vars[0] = 'system'` 和 `$vars[1] = ['whoami']`，最终调用 `call_user_func_array('system', ['whoami'])`
















































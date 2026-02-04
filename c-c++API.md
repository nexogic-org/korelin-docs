# Korelin C/C++ API 指南

>Korelin 提供了一套强大且易用的 C/C++ API，允许开发者将 Korelin 运行时嵌入到自己的应用程序中，或者扩展 Korelin 的功能。本文档介绍的是`kembed`以C为主导的将korelin嵌入的api(标准库为使用这个api)。

## 1. 快速开始 (嵌入式集成)

Korelin 的发行包中包含了一个名为 `kembed` 的文件夹。该文件夹包含了完整的、自包含的编译器和虚拟机源码，专为嵌入式集成设计。

> **重要提示**：`kembed` 文件夹中的代码是没有Korelin的标准Korelin，您可以直接将其作为库源码添加到您的项目中。

### 1.1 集成步骤

1.  **复制源码**：将 `kembed` 文件夹完整复制到您的 C/C++ 项目目录中。
2.  **包含头文件**：在需要调用 Korelin API 的代码中包含 `kembed/kembed.h`。
3.  **编译设置**：将 `kembed` 文件夹下的所有 `.c` 文件加入您的构建系统（如 CMake, Makefile 或 IDE 项目配置）进行编译。

### 1.2 最小示例

```c
#include "kembed/kembed.h"
#include <stdio.h>

// 定义一个原生函数，供 Korelin 调用
void native_add(KInt x, KInt y) {
    // KReturnInt 用于设置返回值
    KReturnInt(x + y);
}

int main() {
    // 1. 初始化 Korelin 环境
    // KInit 会初始化内存池、全局表和基础环境
    // 或者使用 kembed_init() 进行初始化
    kembed_init(); 
    // 注意：kembed_init() 内部会调用 KInit() 及其它必要的初始化步骤。
    
    // 如果只需要标准 API，也可以直接使用 KInit()。
    // KInit();

    // 2. 注册原生模块和函数
    // 创建一个名为 "MathLib" 的模块
    KLibNew("MathLib");
    // 向 "MathLib" 模块注册名为 "add" 的函数，对应 C 函数 native_add
    KLibAdd("MathLib", "function", "add", &native_add);

    // 3. 运行 Korelin 代码
    // KRun 会编译并执行传入的源代码字符串
    // 或者使用 kembed_run()
    kembed_run("import os; \n"
         "import MathLib; \n"
         "void main() { \n"
         "  var res = MathLib.add(10, 20); \n"
         "  os.println(\"Result from C: \" + res); \n"
         "}");

    // 4. 清理资源
    // 释放虚拟机和运行时占用的所有内存
    kembed_cleanup();
    return 0;
}
```

## 2. 核心 API 参考

### 2.1 类型映射

| Korelin 类型 | C/C++ 类型 | 说明 |
| :--- | :--- | :--- |
| `KInt` | `int32_t` | 32位有符号整数 |
| `KBool` | `bool` | 布尔值 (stdbool.h) |
| `KFloat` | `double` | 64位双精度浮点数 (通常映射为 double) |
| `KString` | `const char*` | C 风格字符串 (UTF-8 编码) |
| `KObj` | `void*` | 对象句柄 |

### 2.2 环境管理

*   `void kembed_init()`: 初始化虚拟机环境。
*   `void kembed_cleanup()`: 销毁虚拟机环境，释放内存。

### 2.3 模块与函数注册

*   `void KLibNew(const char* name)`: 创建一个新的原生模块（包）。
*   `void KLibAdd(const char* lib, const char* type, const char* name, void* ptr)`: 向模块添加成员。
    *   `lib`: 目标模块名称。
    *   `type`: 成员类型，支持 `"function"`, `"var"`, `"const"`, `"class"`。
    *   `name`: 在 Korelin 脚本中显示的名称。
    *   `ptr`: 对应的 C 函数指针或变量地址。

### 2.4 类注册

```c
// 1. 创建模块
KLibNew("Game");

// 2. 注册类
KLibAdd("Game", "class", "Player", NULL);

// 3. 注册构造函数 (_init)
KLibClassAdd("Player", "_init", &Player_init);

// 4. 注册实例方法
KLibClassAdd("Player", "attack", &Player_attack);

// 5. 注册静态方法
KLibClassAdd("Player", "static::create", &Player_create);
```

### 2.5 执行脚本

*   `void kembed_run(const char* source)`: 编译并立即运行源代码字符串。
*   `void kembed_run_file(const char* path)`: 读取文件并运行。

## 3. Native 函数编写规范

编写供 Korelin 调用的 C 函数时，无需遵循特定的签名，但需要使用辅助宏来处理参数和返回值。

```c
// Korelin: int myFunc(int a, string b)
void my_native_wrapper() {
    // 获取参数 (索引从 0 开始)
    KInt arg1 = KGetArgInt(0);
    KString arg2 = KGetArgString(1);

    // 业务逻辑...
    int result = arg1 * 2;

    // 设置返回值
    KReturnInt(result);
}
```

*   **获取参数**: `KGetArgInt(i)`, `KGetArgString(i)`, `KGetArgFloat(i)`, `KGetArgBool(i)`
*   **设置返回值**: `KReturnInt(v)`, `KReturnString(v)`, `KReturnFloat(v)`, `KReturnBool(v)`, `KReturnNull()`

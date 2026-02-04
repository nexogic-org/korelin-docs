# KHost API 文档

本 API 是以 Korelin 为主导，调用 C 语言编写的库的 API。本功能由 Rungo 实现。

## 1. 创建 KHost 项目

```bash
rungo init plugin -t cplugin -v 0.1.0 -k 1.0.0
cd plugin
```

以上命令以 `cplugin` 模板创建一个 Korelin C 库项目，项目版本为 `0.1.0`，依赖的 Korelin 版本为 `1.0.0`。

### 1.1 项目结构 (main.c)

查看根目录下 `main.c` 文件：

```c
#include <stdlib.h>
#include "khost.h"
#define KORELIN_VERSION "1.0.0"
#define C_VERSION 11

void print_plugin_info() {
    printf("Plugin Name: %s\n", KPluginName());
    printf("Plugin Version: %s\n", KPluginVersion());
    printf("Korelin Version: %s\n", KORELIN_VERSION);
    printf("C Version: %d\n", C_VERSION);
}

void main() {
    KHostInit();  // 初始化 KHost 环境
    KPackage("plugin", KORELIN_VERSION, C_VERSION);
    KPackageInit(); // 初始化 KPackage 环境，在 KHost项目中，只有一个包
    KAdd("func", &print_plugin_info);
    KPackageBuild();
}
```

### 1.2 构建与发布

运行以下命令构建项目：

```bash
rungo pbuild -o ./out 
# 如果不加 -o ./out，默认会参考 project.json 中的输出目录
```

如果想要发布这个包，需要先将其托管到 git 仓库。根目录必须只包含 `xxx.klib` 文件以及 `lib.json`（直接把 `out` 目录内的文件复制过去），`lib.json` 会根据您的配置自动生成。

发布命令：

```bash
rungo publish <git仓库的位置>
```

### 1.3 安装与使用

通过 Rungo 安装：

```bash
rungo install plugin
```

安装后，库会位于项目的 `rungo_modules` 目录下。

如果没有发布到 Nexogic，也可以直接从本地安装：

```bash
rungo install local:<out目录的绝对路径或相对路径> 
# 这会下载并安装这个包到项目
```

在 Korelin 脚本中使用：

```korelin
import pluginl;

void main() {
    pluginl.print_plugin_info();
}
```

运行脚本即可看到输出。

## 2. 原理

将 C 语言编译的 `klib` 文件本质是整个项目的 C 语言汇总，包括 `main.c` 以及其他 C 文件和静态资源。

因为 C 语言需要编译为不同操作系统的库，所以 Rungo 会根据目标平台动态通过 `klib` 构建对应的动态链接库，并制作对应 Korelin 函数、类的接口（基于标准库接口）。

例如实例中的 `print_plugin_info` 函数：
在 Korelin 中调用 `pluginl.print_plugin_info()` 时，底层实际上是：

```korelin
void print_plugin_info(){
    // 使用标准库调用 C 中的函数
    // ...
}
```

Korelin 的双 API 设计在保证性能的前提下，也可以直接调用 C 中的函数。

*   **获取参数**: `KGetArgInt(i)`, `KGetArgString(i)`, `KGetArgFloat(i)`, `KGetArgBool(i)`
*   **设置返回值**: `KReturnInt(v)`, `KReturnString(v)`, `KReturnFloat(v)`, `KReturnBool(v)`, `KReturnNull()`
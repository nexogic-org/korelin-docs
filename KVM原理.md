# KVM 原理与设计文档

## 1. 概述

KVM (Korelin Virtual Machine) 是 Korelin 语言的核心运行时环境。它是一个基于寄存器的虚拟机，旨在提供高效、紧凑的字节码执行能力。KVM 直接解释执行由编译器 (`kcode.c`) 生成的字节码块 (`KBytecodeChunk`)。

## 2. 架构设计

### 2.1 基于寄存器的架构

与传统的基于栈的虚拟机（如 JVM, Python VM）不同，KVM 采用了基于寄存器的架构。
- **虚拟寄存器**：每个函数调用帧拥有自己的虚拟寄存器窗口（目前实现为共享或固定大小的寄存器堆）。
- **优势**：
    - 指令更少：数据移动操作减少，算术指令直接操作寄存器。
    - 性能潜力：更贴近物理 CPU 的工作方式，易于映射到 JIT 编译。

### 2.2 内存模型

KVM 的运行时数据结构主要包含以下部分：

1.  **寄存器堆 (Registers)**：
    - `KValue registers[256]`：用于存储局部变量、临时计算结果。
    - 支持多种数据类型（整型、浮点、对象指针等）。

2.  **操作数栈 (Operand Stack)**：
    - 虽然主要是寄存器机，但在处理函数调用传参 (`KOP_PUSH`, `KOP_POP`) 时使用辅助栈。
    - 防止寄存器溢出，并支持可变参数传递。

3.  **常量池 (Constant Pool)**：
    - 存储字符串字面量、类名、方法签名等静态数据。
    - 位于 `KBytecodeChunk` 中，通过索引访问 (`KOP_LDS`)。

4.  **调用栈 (Call Stack)**：
    - 管理函数调用的上下文（返回地址、代码块指针）。
    - 支持递归调用。

### 3. 指令集系统

KVM 的指令集定义在 `kcode.h` 中，涵盖了计算、控制流、内存访问等操作。

#### 3.1 指令格式
大多数指令采用定长格式（4字节），以提高解码效率：
- **R-Type (三寄存器)**: `Opcode (8) | Rd (8) | Ra (8) | Rb (8)`
  - 例：`ADD R0, R1, R2` (R0 = R1 + R2)
- **I-Type (立即数)**: `Opcode (8) | Rd (8) | Ra (8) | Imm (8)`
  - 例：`ADDI R0, R1, 10`
- **M-Type (内存/跳转)**: `Opcode (8) | Rd (8) | Imm (16/24)`
  - 例：`LDS R0, [Index16]`

### 4. 核心执行流程

虚拟机通过 `kvm_interpret` 函数启动，核心是一个无限循环的分发器：

1.  **取指 (Fetch)**：从 `ip` (Instruction Pointer) 指向的内存读取操作码。
2.  **解码 (Decode)**：根据操作码类型读取后续的操作数（寄存器索引或立即数）。
3.  **执行 (Execute)**：执行对应的 C 语言逻辑（如加法、跳转、内存读写）。
4.  **更新 (Update)**：推进 `ip` 到下一条指令。

```c
// 伪代码示例
while (true) {
    opcode = *ip++;
    switch (opcode) {
        case KOP_ADD:
            rd = *ip++; ra = *ip++; rb = *ip++;
            registers[rd] = registers[ra] + registers[rb];
            break;
        // ...
    }
}
```

### 5. 类型系统

KVM 使用 `KValue` 结构体（Tagged Union）来表示所有运行时值：
- `VAL_INT`: 64位整数
- `VAL_DOUBLE`: 双精度浮点数
- `VAL_OBJ`: 对象指针（用于类实例、数组等）
- `VAL_STRING`: 字符串引用

### 6. 异常与调试

- **运行时错误**：如除以零、栈溢出等，通过 `RUNTIME_ERROR` 宏处理，会设置错误标志并终止执行。
- **调试支持**：提供 `KOP_DEBUG` 指令，可在运行时打印寄存器状态，辅助开发。

## 7. 内存管理 (GC)
Korelin 实现了基于 **标记-清除 (Mark-and-Sweep)** 算法的垃圾回收器 (`src/kgc.c`)，确保内存的自动管理与回收。

### 7.1 对象结构
每个堆分配的对象（如 `String`, `Array`, `Instance`）都包含一个统一的头部 `KObjHeader`：
- `type`: 对象类型
- `marked`: GC 标记位
- `size`: 对象占用的内存大小
- `next`: 指向下一个对象的链表指针（用于全量遍历）

### 7.3 分代回收 (Generational GC)
为了进一步优化性能，GC 引入了分代假设（弱分代假设：大多數對象朝生夕死）。

1.  **内存划分**:
    - **新生代 (Young Gen)**: 新分配的对象首先进入新生代。Minor GC 頻率高，速度快。
    - **老年代 (Old Gen)**: 在 Minor GC 中存活的对象会晋升到老年代。Major GC 頻率低。

2.  **写屏障 (Write Barrier)**:
    - 为了维护分代的一致性，当老年代对象引用新生代对象时，通过写屏障 (`kgc_write_barrier`) 将该老年代对象加入 **记忆集 (Remembered Set)**。
    - Minor GC 时，记忆集被视为根集合的一部分，确保被老年代引用的新生代对象不会被错误回收。

3.  **回收策略**:
    - **Minor GC**: 只扫描新生代和记忆集。存活的新生代对象全部晋升为老年代。
    - **Major GC (Full GC)**: 扫描整个堆（新生代 + 老年代），回收所有不可达对象。

## 8. JIT 编译 (ComeOnJIT)
Korelin 引入了代号为 **ComeOnJIT** 的即时编译器。
- **工作原理**：在加载字节码块 (`KBytecodeChunk`) 时，尝试将其直接转换为 x64 机器码。
- **寄存器映射**：将 KVM 的虚拟寄存器直接映射到 CPU 内存地址，并使用 CPU 寄存器 (R12, RBX) 进行快速基址寻址。
- **蹦床机制**：支持 JIT 代码与解释器代码的相互调用，保证了兼容性。
- **详情**：请参考 [JIT编译器.md](JIT编译器.md)。

## 9. 未来展望
- **本机接口 (FFI)**：完善 `KOP_SYSCALL` 以支持调用 C 标准库或系统 API。


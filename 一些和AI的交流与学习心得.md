# C语言构建工具学习总结

> 基于 Make & CMake 招新题学习过程的完整记录

---

## 一、学习背景

本次学习围绕 "Make & CMake 招新题" 展开，通过实际操作，从最基础的 hello.c 编译开始，逐步深入到多文件项目、Makefile 编写、CMake 使用，最终完成了一个完整 C 语言项目的构建。

**项目结构**：

make-cmake-recurit/
├── answers/
│   ├── task1.md      # 编译流程笔记
│   └── task4.md      # 思考题答案
├── make-task/         # Make 练习项目
│   ├── src/
│   │   ├── main.c
│   │   ├── calculator.c
│   │   └── logger.c
│   ├── include/
│   │   ├── calculator.h
│   │   └── logger.h
│   └── Makefile       # 手写
├── cmake-task/        # CMake 练习项目
│   ├── src/           # 同上
│   ├── include/       # 同上
│   └── CMakeLists.txt # 手写
└── check.sh           # 自检脚本

---

## 二、核心知识点汇总

### 1. C程序的编译流程（Task 1）

一个 .c 文件变成可执行文件，经历四个阶段：

| 阶段 | 命令 | 输入 | 输出 | 主要工作 |
|------|------|------|------|----------|
| 预处理 | gcc -E | .c | .i | 处理 #include、宏展开、条件编译、删除注释 |
| 编译 | gcc -S | .i | .s | 词法/语法/语义分析，生成汇编代码 |
| 汇编 | gcc -c | .s | .o | 汇编代码 → 机器码（目标文件） |
| 链接 | gcc | .o | 可执行文件 | 合并多个 .o + 库，生成完整程序 |

**观察到的现象**：
- .i 文件比 .c 大很多（展开了 stdio.h）
- .s 文件是人类可读的汇编指令
- .o 是二进制文件，cat 显示乱码
- 可执行文件比 .o 大（链接了库代码）

**编译失败 vs 链接失败**：
- 编译失败：语法错误、类型不匹配 → 改源代码
- 链接失败：找不到函数实现、符号冲突 → 检查链接库或文件列表

---

### 2. 多文件项目的分离编译

一个典型的 C 项目采用 "三文件分离" 模式：

| 文件类型 | 后缀 | 放什么 | 举例 |
|----------|------|--------|------|
| 头文件 | .h | 声明（函数原型、宏、类型定义） | int add(int a, int b); |
| 源文件 | .c | 定义（具体实现） | int add(int a, int b) { return a+b; } |
| 主文件 | .c | 使用（调用函数） | #include "calc.h" + add(10, 5) |

**关键理解**：

- .h 文件是"说明书"，只有声明，没有实现
- 通过 #include 把 .h 的内容复制到 .c 里
- 编译阶段：每个 .c 独立编译，互不干扰
- 链接阶段：链接器把所有 .o 合并，把"调用"和"定义"配对

**为什么 .h 和 .c 要分开？**
1. 避免在每个 .c 文件里重复写声明
2. 修改声明只需改一个 .h，所有文件自动同步
3. 支持多人协作开发

---

### 3. Make 与 Makefile

#### Makefile 的核心结构

target: prerequisites
    recipe

翻译成人话：我想得到 target，它需要 prerequisites，用 recipe 命令来生成它。

#### 手写 Makefile 的例子

CC = gcc
CFLAGS = -Wall -Wextra -Iinclude

calculator: main.o calculator.o logger.o
    gcc main.o calculator.o logger.o -o calculator

main.o: src/main.c include/calculator.h include/logger.h
    gcc -Wall -Wextra -Iinclude -c src/main.c -o main.o

calculator.o: src/calculator.c include/calculator.h
    gcc -Wall -Wextra -Iinclude -c src/calculator.c -o calculator.o

logger.o: src/logger.c include/logger.h
    gcc -Wall -Wextra -Iinclude -c src/logger.c -o logger.o

.PHONY: clean
clean:
    rm -f calculator main.o calculator.o logger.o

#### Make 的工作机制

核心：比较时间戳

- Make 比较目标（.o）和依赖（.c、.h）的修改时间
- 如果依赖比目标新 → 重新编译
- 如果目标比依赖新 → 跳过

**增量构建的决策过程**：

1. 检查最终目标 calculator
2. 递归检查所有依赖（.o 文件）
3. 对每个 .o：
   - 如果 .c 比 .o 新 → 重新编译
   - 如果 .h 比 .o 新 → 重新编译（所有包含它的 .c）
   - 否则 → 跳过
4. 如果有任何 .o 被重新编译 → 重新链接
5. 最终得到新的可执行文件

#### Make 的依赖关系总结

可执行文件
    ├── 依赖: main.o
    ├── 依赖: calculator.o
    └── 依赖: logger.o

main.o
    ├── 依赖: src/main.c        ← 同名 .c
    ├── 依赖: include/calculator.h  ← #include
    └── 依赖: include/logger.h      ← #include

calculator.o
    ├── 依赖: src/calculator.c   ← 同名 .c
    └── 依赖: include/calculator.h  ← #include

logger.o
    ├── 依赖: src/logger.c       ← 同名 .c
    └── 依赖: include/logger.h   ← #include

关键点：.o 文件不依赖其他 .c 文件，只依赖自己的 .c 和包含的 .h。

---

### 4. CMake 与 CMakeLists.txt

#### CMake 是什么？

CMake 不是编译器，它是一个"构建系统生成器"。

CMakeLists.txt（你写的）
        ↓
cmake -S . -B build（配置阶段）
        ↓
生成 Makefile（或 Ninja / VS 项目）
        ↓
cmake --build build（构建阶段）
        ↓
make / ninja / msbuild
        ↓
gcc / clang / MSVC（真正的编译器）
        ↓
可执行程序

#### 为什么需要 CMake？

| 手写 Makefile | 用 CMake |
|---------------|----------|
| 只适用于 Linux/Unix | 全平台通用（Linux/macOS/Windows） |
| 大型项目维护困难 | 自动生成构建文件 |
| 需要手动管理依赖 | 自动扫描 #include |

#### 手写 CMakeLists.txt 的例子

cmake_minimum_required(VERSION 3.16)

project(calculator LANGUAGES C)

add_executable(calculator
    src/main.c
    src/calculator.c
    src/logger.c
)

target_include_directories(calculator
    PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

命令含义：

| 命令 | 作用 |
|------|------|
| cmake_minimum_required | 指定最低 CMake 版本 |
| project | 定义项目名称和语言 |
| add_executable | 定义可执行文件和源文件列表 |
| target_include_directories | 指定头文件搜索路径 |

---

### 5. Make vs CMake vs GCC

| 工具 | 角色 | 输入 | 输出 |
|------|------|------|------|
| GCC | 真正的编译器 | .c / .i / .s | .o / 可执行文件 |
| Make | 构建执行器 | Makefile | 执行编译命令 |
| CMake | 构建系统生成器 | CMakeLists.txt | Makefile / VS 项目 |

一句话总结：CMake 生成 Makefile，Make 执行 Makefile，GCC 真正编译代码。

---

## 三、学习过程中的关键理解

### 1. 声明与定义分离

// calculator.h（声明——给别人看）
int add(int a, int b);

// calculator.c（定义——具体实现）
int add(int a, int b) {
    return a + b;
}

### 2. #include 的本质

#include 就是文本替换——把 .h 文件的内容复制到 .c 文件顶部。

#include "calculator.h"
// 预处理后，变成了：
int add(int a, int b);

### 3. 编译 vs 链接

- 编译：每个 .c 独立编译成 .o，互不关心
- 链接：把所有 .o 合并，把"调用"和"定义"配对

两种常见错误：

| 报错 | 阶段 | 原因 |
|------|------|------|
| implicit declaration of function | 编译 | 缺少 #include 或函数声明 |
| undefined reference to | 链接 | 忘记把 .c 文件加入编译列表 |

### 4. 增量编译的原理

Make 通过时间戳比较实现增量编译：

- 修改了 calculator.c → 它的时间变新
- calculator.o 的依赖变了 → 重新编译 calculator.o
- calculator 的依赖变了 → 重新链接

只改了 calculator.c，main.c 和 logger.c 不会被重新编译。

### 5. .h 文件同名的原因

- 不是系统强制，而是社区约定
- 方便快速定位（看到 logger.h 就知道去找 logger.c）
- 支持构建工具自动推导（如 %.o: %.c 模式规则）

---

## 四、工具链完整流程

### 手写 Makefile 方式

cd make-task
make          # 根据 Makefile 执行编译
./calculator  # 运行
make clean    # 清理构建产物

### 使用 CMake 方式

cd cmake-task
cmake -S . -B build    # 生成 Makefile
cmake --build build    # 执行构建
./build/calculator     # 运行

---

## 五、自检与验证

运行 ./check.sh 验证所有任务是否正确：

cd ~/make-cmake-recurit
chmod +x check.sh
./check.sh

预期输出：

[INFO] Calculator started
10 + 5 = 15
10 - 5 = 5

---

## 六、总结

通过本次学习，我理解了：

1. C 程序的完整构建流程：预处理 → 编译 → 汇编 → 链接
2. 多文件项目的组织方式：.h 声明 + .c 定义 + .c 使用
3. Make 的核心机制：通过时间戳做增量构建
4. CMake 的角色：生成构建文件，不是编译
5. 构建工具链的关系：CMake → Make → GCC

关键认识：
- 构建工具的价值在于自动化和增量构建
- 理解依赖关系是理解构建工具的基础
- 工具链是分层的，每层有各自的职责

---


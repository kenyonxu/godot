# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 语言设置

请始终使用中文回答用户的问题和请求。

## AI 贡献披露

Godot 要求 AI agent 在贡献时自我披露：PR/Issue 标题必须以 🤖 开头，描述中必须包含：
```
> [!INFO] *AI disclosure*: This contribution was authored by an autonomous AI agent, on behalf of a user to [...].
```
未披露的 AI agent 将被禁止贡献。详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 构建系统

Godot 使用 **SCons** 构建系统（需要 SCons >= 4.4，Python >= 3.9）。

```bash
# 构建编辑器 (Linux)
scons platform=linuxbsd target=editor -j$(nproc)

# 构建调试模板
scons platform=linuxbsd target=template_debug -j$(nproc)

# 构建发布模板
scons platform=linuxbsd target=template_release -j$(nproc)
```

**SCU 构建**：Godot 使用 Single Compilation Unit 构建加速。不要直接编辑 `*.gen.cpp` 文件——它们由 `scu_builders.py` 从各目录的源文件自动生成。

**Python 代码质量**：
```bash
ruff check .          # Lint Python 文件
mypy .                # 类型检查（跳过 thirdparty/）
codespell             # 拼写检查
```

**测试**：通过 `tests/test_main.h` 和 `tests/test_macros.h` 使用类 doctest 宏 (`TEST_CASE`, `CHECK`, `CHECK_FALSE` 等)。测试数据在 `tests/data/`。

## 核心架构

### 类型系统层次

```
Object (core/object/object.h)          — 引擎万物基类，通过 ClassDB 反射
├─ RefCounted (core/object/ref_counted.h) — 引用计数基类
│  └─ Resource (core/io/resource.h)   — 可序列化资源（材质、纹理、音频等）
└─ Node (scene/main/node.h)            — 场景树节点，有父子关系
```

- **Variant** (`core/variant/variant.h`)：动态类型容器，是所有脚本 API 的类型基础，统一了 int/float/String/Array/Dictionary/Vector3 等几十种类型
- **ClassDB** (`core/object/class_db.h`)：反射系统，注册类、方法、属性、信号。`GD_IS_CLASS_ENABLED` 机制通过 `core/disabled_classes.gen.h`（自动生成）控制哪些类可用
- **Object 宏** (`core/object/object.h`)：一组 `ADD_PROPERTY`/`ADD_SIGNAL`/`ADD_GROUP` 宏用于在编译期注册成员

### 注册模式 (Registration Pattern)

引擎按初始化层级注册类型。每个层级有 `register_*_types.cpp`：

```
ModuleInitializationLevel (modules/register_module_types.h):
  CORE    → core/register_core_types.cpp       # 数学、IO、Variant、容器
  SERVERS → servers/register_server_types.cpp  # 渲染、物理、音频
  SCENE   → scene/register_scene_types.cpp     # 节点类型、资源类型
  EDITOR  → editor/register_editor_types.cpp   # 编辑器插件（仅 target=editor 时编译）
```

`modules/register_module_types.h` 定义了 `initialize_modules(p_level)` / `uninitialize_modules(p_level)` 来按层级管理模块生命周期。

### 服务器模式 (Server Pattern)

`servers/` 中的服务器是引擎核心服务的单例抽象：
- **servers/rendering/**：渲染服务器，分阶段管道—— cull（可见性剔除）→ render（实际渲染），Canvas（2D）和 Scene（3D）各自独立管线
- **servers/physics_2d/**、**servers/physics_3d/**：物理引擎抽象
- **servers/audio/**：音频服务器 + 效果链
- 服务器通过 `server_wrap_mt_common.h` 支持多线程包装

### 关键文件

| 文件 | 作用 |
|------|------|
| `core/object/object.h` | 基类 + 属性注册宏 |
| `core/object/class_db.h` | ClassDB 反射 + `is_class_enabled` 类开关机制 |
| `core/variant/variant.h` | Variant 万能类型容器 |
| `core/io/resource.h` | Resource 基类（可序列化资产） |
| `scene/main/node.h` | Node 基类（场景树元素） |
| `scene/main/scene_tree.h` | SceneTree（游戏循环、场景管理） |
| `core/core_bind.cpp` | 将核心类绑定到脚本 API |
| `core/disabled_classes.gen.h` | 自动生成——标记不可用类 |
| `main/main.cpp` | 程序入口，初始化顺序 |

### 目录职责

| 目录 | 职责 | 何时编译 |
|------|------|----------|
| `core/` | 基础层：数学、IO、容器、Variant、Object | 始终 |
| `servers/` | 引擎服务：渲染、物理、音频、导航 | 始终 |
| `scene/` | 场景树和节点类型（2D/3D/GUI/动画） | 始终 |
| `drivers/` | 硬件后端驱动 | 按平台 |
| `platform/` | 平台 OS 实现（每个子目录有 `detect.py`） | 按平台 |
| `modules/` | 可选模块（GDScript、Mono、物理后端等） | 按配置 |
| `editor/` | Godot 编辑器应用 | 仅 `target=editor` |
| `main/` | 程序入口和主循环 | 始终 |
| `thirdparty/` | 第三方库——**不在 Godot 中修改** | 按需 |
| `tests/` | 单元测试框架和测试用例 | 构建时 `tests=yes` |

### 类绑定模式

要让一个 C++ 类暴露给脚本 API（GDScript/C#/GDExtension），必须：
1. 继承 `Object`（或其子类如 `Node`、`Resource`、`RefCounted`）
2. 使用 `ClassDB::bind_method()` 注册方法
3. 使用 `ClassDB::add_property()` 或 Object 中的 ADD_PROPERTY 宏注册属性
4. 所有 API 参数和返回值必须是 Variant 兼容的类型

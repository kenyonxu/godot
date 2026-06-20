# Godot 工程 Graphify AST 构图计划

> **策略**: 纯 AST 构图 + 无可视化 (`--no-viz`)，分层逐步进行，每批 < 200 文件
> **目标**: 为 Godot 引擎代码库构建完整的代码结构知识图谱
> **总规模**: ~3,604 C++ 文件 (排除 thirdparty/)，~859 其他代码文件 (.py/.glsl/.java/.cs 等)

---

## 工程规模概览

| 目录 | C++ 文件数 | 角色 | 优先级 |
|------|-----------|------|--------|
| core/ | 418 | 引擎核心基础层 (数学/IO/容器/OS) | P0 |
| servers/ | 318 | 引擎服务器 (渲染/物理/音频/XR) | P1 |
| drivers/ | 164 | 硬件抽象层 | P2 |
| main/ | 8 | 程序入口 | P2 |
| scene/ | 676 | 场景树和节点系统 | P3 |
| editor/ | 644 | 编辑器 UI | P4 |
| modules/ | 924 | 可选模块 (GDNative/GDScript/物理等) | P5 |
| platform/ | 273 | 平台特定代码 | P6 |
| tests/ | 179 | 测试代码 | P7 |
| thirdparty/ | 3,025 | **第三方库 — 排除** | — |

---

## 重要原则

1. **排除 thirdparty/**: 第三方库不是 Godot 核心架构的一部分，不应混入图谱
2. **--no-viz 模式**: 只生成 `GRAPH_REPORT.md` + `graph.json`，不生成 HTML 可视化
3. **--update 增量**: 每完成一批，使用 `--update` 追加到现有图谱
4. **文件数上限**: 每批控制在 200 文件以内，避免 graphify 超限警告
5. **按依赖顺序**: 先处理底层依赖 (core)，再处理上层 (scene → editor → modules)

---

## 执行计划

### Phase 0 — 安装验证 (1 batch)

首次运行前验证 graphify 可用性，在最小的目录上试跑。

```bash
# 确保 graphify 已安装
python3 -c "import graphify" || pip install graphifyy

# 在最小目录试跑，验证 AST 提取 C++ 代码正常
/graphify main/ --no-viz
```

**预期**: 在 `graphify-out/` 生成 `graph.json` 和 `GRAPH_REPORT.md`
**文件数**: ~8
**验证点**: 确认 C++ `.h`/`.cpp` 文件的函数、类、继承关系被正确提取

---

### Phase 1 — core/ 基础层 (4 batches)

core/ 是整个引擎的基石，包含数学库、容器、IO、操作系统抽象等。

#### Batch 1.1: core/math + core/string + core/templates + core/variant
```bash
/graphify core/math core/string core/templates core/variant --no-viz
# 75 + ~30 + 37 + 37 ≈ 179 files
```
包含: 向量/矩阵数学、字符串处理、模板容器、Variant 类型系统

#### Batch 1.2: core/io + core/os
```bash
/graphify core/io core/os --no-viz --update
# 103 + 26 = 129 files
```
包含: 文件系统 IO、操作系统抽象层

#### Batch 1.3: core/object + core/extension + core/error + core/config
```bash
/graphify core/object core/extension core/error core/config --no-viz --update
# 33 + ~30 + ~10 + ~10 ≈ 83 files
```
包含: Object 基类、GDExtension 扩展系统、错误处理、配置系统

#### Batch 1.4: core/input + core/debugger + core/profiling + core/crypto + core/ 顶层文件
```bash
/graphify core/input core/debugger core/profiling core/crypto --no-viz --update
# ~30 + ~20 + ~15 + ~15 ≈ 80 files
```
加上 core/ 顶层 `.cpp`/`.h` 文件

包含: 输入系统、调试器接口、性能分析、加密

---

### Phase 2 — servers/ 引擎服务器 (2 batches)

servers/ 实现了引擎的核心服务：渲染、物理、音频、导航等。

#### Batch 2.1: servers/rendering + servers/audio + servers/physics_2d + servers/physics_3d
```bash
/graphify servers/rendering servers/audio servers/physics_2d servers/physics_3d --no-viz --update
# ~120 + ~30 + ~25 + ~25 ≈ 200 files
```
包含: 渲染服务器、音频服务器、2D/3D 物理服务器

#### Batch 2.2: servers/ 其余
```bash
/graphify servers/navigation_2d servers/navigation_3d servers/xr servers/camera servers/debugger servers/display servers/movie_writer servers/text --no-viz --update
# 约 118 files
```
包含: 导航、XR、摄像机、调试器、显示、电影写入、文本服务器

---

### Phase 3 — drivers/ + main/ 驱动和入口 (1 batch)

```bash
/graphify drivers/ main/ --no-viz --update
# 164 + 8 = 172 files
```
包含: 硬件驱动抽象 (Vulkan/OpenGL/D3D)、音频驱动、平台驱动、程序入口

---

### Phase 4 — scene/ 场景系统 (4 batches)

scene/ 是 Godot 的核心概念——场景树和节点系统。

#### Batch 4.1: scene/main + scene/resources + scene/ 顶层文件
```bash
/graphify scene/main scene/resources --no-viz --update
# ~180 files (含 scene/ 顶层文件)
```
包含: 场景树、资源管理

#### Batch 4.2: scene/2d + scene/gui
```bash
/graphify scene/2d scene/gui --no-viz --update
# ~160 files
```
包含: 2D 节点、GUI 控件

#### Batch 4.3: scene/3d + scene/animation
```bash
/graphify scene/3d scene/animation --no-viz --update
# ~180 files
```
包含: 3D 节点、动画系统

#### Batch 4.4: scene/audio + scene/debugger + scene/theme
```bash
/graphify scene/audio scene/debugger scene/theme --no-viz --update
# ~156 files
```
包含: 场景层面音频、调试器、主题系统

---

### Phase 5 — editor/ 编辑器 (4 batches)

editor/ 是 Godot 编辑器本身的实现。

#### Batch 5.1: editor/plugins + editor/ 顶层文件
```bash
/graphify editor/plugins --no-viz --update
# ~160 files (含 editor/ 顶层文件)
```
包含: 编辑器插件系统

#### Batch 5.2: editor/gui + editor/inspector + editor/scene
```bash
/graphify editor/gui editor/inspector editor/scene --no-viz --update
# ~160 files
```
包含: 编辑器 GUI、属性检查器、场景编辑器

#### Batch 5.3: editor/import + editor/export + editor/animation + editor/audio + editor/shader
```bash
/graphify editor/import editor/export editor/animation editor/audio editor/shader --no-viz --update
# ~160 files
```
包含: 导入/导出系统、动画编辑器、音频编辑器、着色器编辑器

#### Batch 5.4: editor/ 其余
```bash
/graphify editor/debugger editor/doc editor/docks editor/file_system editor/project_manager editor/project_upgrade editor/run editor/script editor/settings editor/themes editor/version_control --no-viz --update
# ~164 files
```
包含: 调试器面板、文档浏览器、文件系统面板、项目管理器、脚本编辑器、设置、主题、版本控制

---

### Phase 6 — modules/ 模块 (5+ batches)

modules/ 有 40+ 个子模块，需要按类别分组。优先处理引擎核心模块，后处理格式/编解码模块。

#### Batch 6.1: 引擎核心模块
```bash
/graphify modules/gdscript modules/mono modules/multiplayer modules/jsonrpc --no-viz --update
# ~150 files
```
包含: GDScript 脚本语言、C#/Mono 绑定、多人游戏、JSON-RPC

#### Batch 6.2: 物理和导航模块
```bash
/graphify modules/godot_physics_2d modules/godot_physics_3d modules/jolt_physics modules/navigation_2d modules/navigation_3d --no-viz --update
# ~160 files
```
包含: Godot 物理引擎、Jolt 物理、导航系统

#### Batch 6.3: 渲染和图形模块
```bash
/graphify modules/glslang modules/lightmapper_rd modules/raycast modules/visual_shader --no-viz --update
# ~120 files
```
包含: GLSL 编译器、光照烘焙、射线检测、可视化着色器

#### Batch 6.4: 格式和编解码模块 (组A)
```bash
/graphify modules/gltf modules/fbx modules/svg modules/bmp modules/dds modules/hdr modules/jpg modules/ktx --no-viz --update
# ~160 files
```
包含: glTF/FBX 3D 模型、SVG 矢量图、各种图片格式

#### Batch 6.5: 格式和编解码模块 (组B) + 其余
```bash
/graphify modules/astcenc modules/basis_universal modules/bcdec modules/betsy modules/cvtt modules/enet modules/etcpak modules/freetype modules/gridmap modules/interactive_music modules/mbedtls modules/meshoptimizer modules/mobile_vr modules/mp3 modules/msdfgen modules/noise modules/objectdb_profiler modules/ogg modules/openxr --no-viz --update
# ~200 files
```
包含: 纹理压缩、网络、字体、交互音乐、加密、XR 等

---

### Phase 7 — platform/ 平台层 (2 batches)

#### Batch 7.1: 主要平台
```bash
/graphify platform/linuxbsd platform/windows platform/macos --no-viz --update
# ~150 files
```
包含: Linux/BSD, Windows, macOS 平台适配

#### Batch 7.2: 移动和其他平台
```bash
/graphify platform/android platform/ios platform/web --no-viz --update
# ~123 files
```
加上任何其他平台目录

---

### Phase 8 — tests/ 测试 (1 batch)

```bash
/graphify tests/ --no-viz --update
# 179 files
```
包含: 单元测试、集成测试

---

### Phase 9 — 非 C++ 补充 (可选)

如果需要将 Python 构建脚本、GLSL 着色器、Java/Kotlin Android 代码也纳入图谱：

```bash
# Python 构建工具
/graphify . --no-viz --update
# 或在特定目录运行

# GLSL 着色器
/graphify <shader_dirs> --no-viz --update
```

---

## 执行顺序总览

```
Phase 0  [main/]                                     8 files   ← 验证
Phase 1  [core/]                                   418 files   ← 4 batches
Phase 2  [servers/]                                318 files   ← 2 batches
Phase 3  [drivers/ + main/]                        172 files   ← 1 batch
Phase 4  [scene/]                                  676 files   ← 4 batches
Phase 5  [editor/]                                 644 files   ← 4 batches
Phase 6  [modules/]                                924 files   ← 5 batches
Phase 7  [platform/]                               273 files   ← 2 batches
Phase 8  [tests/]                                  179 files   ← 1 batch
──────────────────────────────────────────────────────────────
总计                                              3,604 files  ← ~24 batches
```

每批预计耗时: **30秒 ~ 2分钟** (纯 AST，无 LLM 参与)
总预计耗时: **约 30-45 分钟**

---

## 日常维护

代码修改后，使用增量更新：

```bash
# 修改了某些文件后，只重新提取变更的文件
/graphify . --update --no-viz
```

## 查询使用

图谱构建完成后：

```bash
# 探索某个节点
/graphify explain "Node"

# 查找两个概念之间的关联
/graphify path "Object" "SceneTree"

# 提问
/graphify query "rendering pipeline 的依赖关系是什么？"
```

---

## 注意事项

- **thirdparty/ 永久排除** — 第三方代码不属于 Godot 架构
- **不需要 LLM 语义提取** — AST 构图足够了，不需要 `--mode deep`
- **首次运行后只会更快** — graphify 有缓存机制，`--update` 只处理变更文件
- **graph.json 会逐步增长** — 最终图谱预计包含 10,000+ 节点和 30,000+ 边

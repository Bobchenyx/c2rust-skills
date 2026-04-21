# c2rust-skills 使用指南

## 目录

- [简介](#简介)
- [安装](#安装)
- [工具链准备](#工具链准备)
- [快速开始](#快速开始)
- [完整流程详解](#完整流程详解)
  - [Phase 1: 评估与理解 (assess)](#phase-1-评估与理解)
  - [Phase 2: 转换计划 (plan)](#phase-2-转换计划)
  - [Phase 3: 测试构建 (test)](#phase-3-测试构建)
  - [Phase 4: 执行转换 (convert)](#phase-4-执行转换)
  - [Phase 5: 编译调试与精炼 (refine)](#phase-5-编译调试与精炼)
  - [Phase 6: 验证 (verify)](#phase-6-验证)
- [命令参考](#命令参考)
- [共享状态: c2rust-manifest.toml](#共享状态)
- [子Agent说明](#子agent说明)
- [参考资料库](#参考资料库)
- [典型工作流示例](#典型工作流示例)
- [常见问题](#常见问题)
- [插件结构](#插件结构)

---

## 简介

`c2rust-skills` 是一套 Claude Code 插件，用于系统化地将 C 语言仓库/项目级别地转换为惯用 Rust。它使用 Claude Sonnet 4.6 直接将 C 源码翻译为惯用 Rust 代码，覆盖从评估 C 代码库到产出经过验证的惯用 Rust 代码的完整生命周期。

### 核心理念

- **AI 直译而非机械转译**：使用 Claude Sonnet 4.6 直接将 C 翻译为惯用 Rust，从一开始就产出高质量代码
- **增量转换优先**：模块逐个转换，C 与 Rust 通过 FFI 共存，每一步都可编译可测试
- **智能混合精炼**：机械性编译错误自动修复，语义性设计决策暂停询问用户
- **全流程可追踪**：通过 `c2rust-manifest.toml` 持久化所有状态，支持随时中断和恢复

### 包含什么

| 类型 | 数量 | 说明 |
|------|------|------|
| Skills (技能) | 8 个 | 覆盖从评估到验证的全流程 |
| Agents (子Agent) | 4 个 | C代码分析、C→Rust翻译、Rust审查、编译调试 |
| 参考资料 | 6 份 | C模式目录、依赖映射、unsafe转换模式等 |
| 模板 | 2 份 | 集成测试模板、FFI测试模板 |

---

## 安装

### 方式一：Shell Alias（推荐，最简单）

克隆仓库到本地，配置一行 alias 即可：

```bash
# 克隆插件仓库
git clone https://github.com/Bobchenyx/c2rust-skills.git ~/c2rust-skills

# 添加到 ~/.bashrc 或 ~/.zshrc
echo 'alias claude="claude --plugin-dir ~/c2rust-skills"' >> ~/.bashrc
source ~/.bashrc
```

之后在任何 C 项目目录启动 `claude`，所有 `/c2rust-*` 命令自动可用。插件更新只需 `git pull`。

### 方式二：注册为本地 Marketplace（持久安装）

```bash
# 注册并安装
claude plugin marketplace add ~/c2rust-skills
claude plugin install c2rust-skills

# 更新插件后需要重新安装
claude plugin update
```

### 方式三：项目级别配置（团队共享）

在目标项目的 `.claude/settings.json` 中配置：

```json
{
  "extraKnownMarketplaces": {
    "c2rust-plugins": {
      "source": {
        "source": "directory",
        "path": "/absolute/path/to/c2rust-skills"
      }
    }
  }
}
```

团队成员克隆项目后，Claude Code 会自动提示安装插件。

---

## 工具链准备

在开始转换之前，需要确保以下工具已安装：

### 必需工具

| 工具 | 用途 | 安装方式 |
|------|------|---------|
| `rustc` + `cargo` | Rust 编译器和构建工具 | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| `clippy` | Rust 代码质量检查 | `rustup component add clippy`（通常已随 rustup 安装） |

### 可选工具（增量 FFI 模式需要）

| 工具 | 用途 | 安装方式 |
|------|------|---------|
| `gcc` / `cc` | C 编译器（编译剩余 C 代码用于 FFI） | 通常系统已预装 |
| `bindgen` | C 头文件 → Rust FFI 绑定 | `cargo install bindgen-cli` |
| `cbindgen` | Rust → C 头文件 | `cargo install cbindgen` |
| `miri` | 未定义行为检测 | `rustup +nightly component add miri` |

使用 `/c2rust-check-env` 可以自动检测所有工具的安装状态：

```
/c2rust-check-env
```

如果需要尝试自动安装缺失工具：

```
/c2rust-check-env --install
```

---

## 快速开始

### 方式一：使用总指挥（推荐新手）

```
/c2rust
```

总指挥会引导你完成所有步骤，在每个阶段结束后询问是否继续。

### 方式二：逐步手动执行

```bash
/c2rust-check-env          # 1. 检查工具链
/c2rust-assess --deep      # 2. 深度评估 C 代码库
/c2rust-plan               # 3. 生成转换计划
/c2rust-test               # 4. 构建行为测试套件
/c2rust-convert --all      # 5. Claude Sonnet 翻译 C → Rust
/c2rust-refine --all       # 6. 修复编译错误 + 惯用化
/c2rust-verify --all       # 7. 验证正确性
```

### 方式三：针对特定模块操作

```bash
/c2rust-convert utils      # 只转换 utils 模块
/c2rust-refine parser      # 只精炼 parser 模块
/c2rust-verify core        # 只验证 core 模块
```

---

## 完整流程详解

### Phase 1: 评估与理解

**命令**：`/c2rust-assess [--quick|--deep] [path]`

评估阶段是整个转换的起点。它分析 C 代码库的结构、复杂度和转换风险。

#### 两种模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `--quick`（默认） | 快速扫描：构建系统检测、文件统计、危险模式 grep、依赖列表 | 初步了解项目 |
| `--deep` | 深度分析：在 quick 基础上，启动 c-analyzer 子 Agent 构建调用图、追踪全局状态、分析宏复杂度 | 正式转换前的完整评估 |

#### 分析内容

1. **构建系统检测** —— 识别 Makefile / CMake / Autotools / Meson
2. **源文件清单** —— 文件数、代码行数、按目录分模块统计
3. **外部依赖** —— 从链接器参数和 `#include` 中提取
4. **危险模式扫描** —— 按严重度分三级：
   - **BLOCKING**：内联汇编、computed goto、setjmp/longjmp（无法自动转换）
   - **HARD**：void 指针、union、位域、可变参数、全局可变状态（需大量手工改写）
   - **MODERATE**：goto、函数指针、指针算术、条件编译（需一定翻译技巧）
5. **模块边界识别** —— 基于目录结构和头文件依赖关系
6. **风险评分** —— 每个模块的量化风险分数

#### 输出

- `c2rust-assessment.md` —— 详细评估报告
- `c2rust-manifest.toml` —— 初始化/更新共享状态

#### 示例输出

```
/c2rust-assess --deep

评估完成：
- 项目: libfoo
- 构建系统: CMake
- 代码量: 15,230 行 C + 4,100 行头文件
- 模块: 6 个
- 整体风险: MEDIUM

模块风险矩阵:
  utils     LOW      (1,200 LOC, 0 blocking)
  config    LOW      (800 LOC, 0 blocking)
  parser    MEDIUM   (3,500 LOC, 2 goto)
  core      HIGH     (5,000 LOC, 全局状态 × 8)
  network   MEDIUM   (2,800 LOC, 函数指针 × 12)
  crypto    CRITICAL (1,930 LOC, 内联汇编 × 3)
```

---

### Phase 2: 转换计划

**命令**：`/c2rust-plan [--incremental|--full]`

基于评估结果，设计具体的转换计划。

#### 计划内容

1. **Rust 项目结构设计**
   - 单 crate（< 5 模块）vs Cargo workspace（5+ 模块）
   - 每个 C 模块对应的 Rust crate/module 映射
   - 公共 API 边界设计

2. **依赖映射**
   - 使用内置的 crate-mapping 参考表（50+ 常见 C 库 → Rust crate 映射）
   - 每个依赖选择：纯 Rust 替代 / -sys 绑定 / 保持 C 通过 FFI 调用

3. **转换顺序**
   - 基于依赖图的拓扑排序
   - 叶子模块（无内部依赖）优先
   - 低风险模块优先
   - CRITICAL 风险模块标记为 "FFI 边界"（暂不转换）

4. **FFI 边界设计**（增量模式）
   - 定义每步转换中 C↔Rust 的接口
   - cbindgen（Rust→C 头文件）和 bindgen（C→Rust 绑定）策略

5. **构建系统迁移路径**
   - 初期：Cargo 包装 C 构建
   - 中期：混合构建（cc crate + build.rs）
   - 终期：纯 Cargo

#### 输出

- `c2rust-plan.md` —— 完整转换计划文档
- 更新 `c2rust-manifest.toml` 的 `[plan]` 和 `[[modules]]`

---

### Phase 3: 测试构建

**命令**：`/c2rust-test [module-name|--all]`

在转换之前构建行为测试套件，用于验证转换后的 Rust 代码行为与 C 版本一致。

#### 测试类型

| 类型 | 说明 | 模板 |
|------|------|------|
| 行为等价测试 | 相同输入 → 相同输出 | `templates/integration-test.rs` |
| 金标准数据测试 | 捕获 C 版本输出作为基准 | 自动生成 |
| FFI 边界测试 | 验证 C↔Rust 数据传递正确性 | `templates/ffi-test.rs` |
| 属性测试 | 算法代码的属性验证（可选） | proptest |

#### 工作流

1. 发现现有 C 测试（CUnit、Check、Unity、cmocka 等框架）
2. 识别每个模块的公共 API 函数
3. 分析函数契约（输入范围、返回值、副作用）
4. 生成集成测试代码
5. 编译运行 C 版本，捕获金标准输出数据
6. 建立测试目录结构

#### 输出

```
tests/
├── common/
│   ├── mod.rs              # 共享测试工具
│   └── golden_data/        # C 版本的金标准输出
├── integration/
│   ├── test_module_a.rs    # 每模块集成测试
│   └── test_ffi.rs         # FFI 边界测试
└── fixtures/
    └── sample_input.txt    # 测试输入数据
```

---

### Phase 4: 执行转换

**命令**：`/c2rust-convert <module-name|--all>`

使用 Claude Sonnet 4.6 将 C 源码直接翻译为惯用 Rust 代码。

#### 翻译引擎

本插件使用 **c-to-rust-translator** 子 Agent（基于 Claude Sonnet 4.6）执行翻译。与机械转译工具不同，Claude 直接产出惯用 Rust 代码：

- 正确使用所有权和借用（`Box`, `Vec`, `&T` 等）
- 用 `Result<T, E>` / `Option<T>` 替代 C 风格错误码和空指针
- 用 `match` / 迭代器替代 `switch` / C 风格循环
- 用 `enum` 替代 `union` + type tag
- 用 `Vec<T>` 替代链表
- 不产生不必要的 `unsafe` 代码

#### 工作流

1. **验证工具链** —— 确认 rustc、cargo 就绪
2. **准备目标目录** —— 创建 Rust 项目结构（Cargo.toml、src/ 等）
3. **读取并理解 C 源码** —— 逐模块读取头文件和源文件
4. **启动翻译 Agent** —— 为每个模块启动 c-to-rust-translator Agent：
   - 独立模块可并行翻译
   - 有依赖关系的模块按顺序翻译
   - 每个 Agent 接收完整上下文：C源码、设计决策、依赖映射
5. **写入输出文件** —— 将翻译后的 Rust 代码写入计划的文件位置
6. **初次编译检查** —— `cargo check`，统计错误数
7. **设置 FFI 绑定**（增量模式）—— 为尚未转换的 C 模块创建 FFI 层

#### 输出

- 翻译后的惯用 Rust 项目
- 初次编译结果摘要
- 翻译注记（设计偏离、重构决策、假设等）

---

### Phase 5: 编译调试与精炼

**命令**：`/c2rust-refine [module-name|--all] [--auto|--interactive]`

这是整个流程中最复杂的阶段。采用**智能混合工作模式**：

#### 三种运行模式

| 模式 | 说明 |
|------|------|
| 默认（智能混合） | 机械性错误自动修复，语义性问题暂停询问 |
| `--auto` | 只做机械性修复，跳过所有语义决策 |
| `--interactive` | 每个修改都暂停确认 |

#### Phase A：机械性错误自动修复循环

自动修复的错误类型（最多 20 轮迭代）：

| 错误码 | 描述 | 修复策略 |
|--------|------|---------|
| E0432 | 导入路径错误 | 修正 `use` 路径 |
| E0308 | 类型不匹配 | 添加 `as` 转换或 `.into()` |
| E0425 | 找不到符号 | 添加 `use`、`mod` 或 `extern` |
| E0412 | 找不到类型 | 添加类型导入或别名 |
| E0277 | trait 未实现 | 添加 `#[derive(...)]` |
| E0382 | 使用已移动的值 | 添加 `.clone()` 或 `Copy` derive |

每轮迭代：`cargo check` → 解析错误 → 分类 → 修复机械性错误 → 再次 `cargo check`

#### Phase B：语义性问题咨询

需要用户决策的问题类型：

- **所有权模型**：Box vs Rc vs Arc
- **错误处理策略**：anyhow vs thiserror vs 手动 enum
- **全局状态重构**：Mutex vs 显式传参
- **算法重构**：goto 密集的控制流改写
- **API 设计**：公共接口命名和暴露范围
- **unsafe 保留**：保留 unsafe 并加注释 vs 重构为 safe

每个问题会展示 2-3 个选项及其利弊权衡，等待用户选择后应用。

#### Phase C：惯用化精炼

编译通过后，系统性地改进代码风格：

```
1. 移除不必要的 unsafe 块
2. 裸指针 → 引用 (&T / &mut T)
3. malloc/free → Box/Vec (RAII)
4. C 字符串 → String/&str
5. 空指针检查 → Option
6. 错误码 → Result<T, E>
7. 全局可变状态 → Mutex/OnceLock/显式传参
8. C 风格循环 → 迭代器
9. 运行 cargo clippy 并应用建议
10. 缩小剩余 unsafe 块的范围
```

#### 输出

- 可编译的、逐步改进的 Rust 代码
- 精炼日志（按类型统计的修复数量）

---

### Phase 6: 验证

**命令**：`/c2rust-verify [module-name|--all] [--quick|--full]`

验证转换后的代码正确性和质量。

#### 检查项

| 检查 | 说明 |
|------|------|
| 编译检查 | `cargo check` 零错误、零警告 |
| 单元测试 | `cargo test` 全部通过 |
| 集成测试 | Phase 3 构建的行为测试 |
| 金标准对比 | 与 C 版本的输出逐字节比较 |
| Clippy 分析 | `cargo clippy` 全部通过 |
| unsafe 审计 | 统计并分类所有 unsafe 使用 |
| Miri 检测 | `cargo miri test` 检测未定义行为（可选） |
| 代码审查 | rust-reviewer Agent 深度代码质量评审（`--full` 模式） |
| 行为比较 | C vs Rust 相同输入下的输出对比 |

#### 输出

- `c2rust-verification-report.md` —— 完整的验证报告，包含：
  - 测试通过率
  - 行为差异详情
  - unsafe 使用统计与说明
  - 代码质量评分
  - 剩余工作清单

---

## 命令参考

### 完整命令列表

| 命令 | 参数 | 说明 |
|------|------|------|
| `/c2rust` | `[status\|resume\|<phase>]` | 总指挥：全流程引导 / 查看状态 / 恢复 / 跳转 |
| `/c2rust-check-env` | `[--install]` | 检查工具链，可选自动安装 |
| `/c2rust-assess` | `[--quick\|--deep] [path]` | 评估 C 代码库 |
| `/c2rust-plan` | `[--incremental\|--full]` | 生成转换计划 |
| `/c2rust-test` | `[module\|--all]` | 构建行为测试套件 |
| `/c2rust-convert` | `<module\|--all>` | Claude Sonnet 4.6 翻译 C → Rust |
| `/c2rust-refine` | `[module\|--all] [--auto\|--interactive]` | 修复错误 + 惯用化 |
| `/c2rust-verify` | `[module\|--all] [--quick\|--full]` | 验证正确性 |

### 总指挥子命令

```bash
/c2rust                 # 开始新转换 或 从上次中断处继续
/c2rust status          # 显示当前进度面板
/c2rust resume          # 从上次完成的阶段继续
/c2rust assess          # 跳转到评估阶段
/c2rust plan            # 跳转到计划阶段
/c2rust test            # 跳转到测试构建阶段
/c2rust convert         # 跳转到转换阶段
/c2rust refine          # 跳转到精炼阶段
/c2rust verify          # 跳转到验证阶段
```

---

## 共享状态

所有 skill 通过项目根目录下的 `c2rust-manifest.toml` 协调工作。这个文件是跨阶段的数据纽带。

### 结构概览

```toml
[project]
name = "my-c-project"          # 项目名
source_dir = "."                # C 源码根目录
build_system = "cmake"          # make | cmake | autotools | meson | custom
conversion_strategy = "incremental"  # incremental | full
created_at = "2026-04-14"
last_updated = "2026-04-14"

[assessment]
status = "completed"            # pending | in_progress | completed
total_loc = 15230
total_files = 42
risk_level = "medium"           # low | medium | high | critical
report_path = "c2rust-assessment.md"

[plan]
status = "completed"
rust_project_name = "my-rust-project"
crate_structure = "workspace"   # single | workspace
target_dir = "rust/"
conversion_order = ["utils", "config", "parser", "core", "network"]
plan_path = "c2rust-plan.md"

[tests]
status = "completed"
test_count = 40
test_dir = "tests/"
golden_data_dir = "tests/common/golden_data/"

[conversion]
status = "in_progress"
method = "claude-sonnet-4.6"
modules_converted = 3
modules_total = 5

[refinement]
status = "pending"
iteration_count = 0
errors_remaining = 0
unsafe_blocks_remaining = 0

[verification]
status = "pending"
tests_passed = 0
tests_failed = 0
tests_total = 0
report_path = ""

[toolchain]
rustc_version = "1.94.1"
cargo_version = "1.94.1"
clippy_version = "0.1.94"
ready = true

[[modules]]
name = "utils"
path = "src/utils"
status = "converted"
risk = "low"
loc = 1200
dependencies = []
c_libraries = []
hard_patterns = []
notes = ""

[[modules]]
name = "core"
path = "src/core"
status = "assessed"
risk = "high"
loc = 5000
dependencies = ["utils", "config"]
c_libraries = ["openssl", "pthread"]
hard_patterns = ["global_mutable", "function_pointers"]
notes = "8 global mutable variables, needs careful ownership design"

[dependencies_map]
openssl = "rustls"
pthread = "std::thread"
zlib = "flate2"
```

### 模块状态流转

```
pending → assessed → planned → tested → converted → refined → verified
```

---

## 子Agent说明

插件包含 4 个专用子 Agent，在特定 skill 执行时被启动：

### c-to-rust-translator

**调用时机**：`/c2rust-convert`

**模型**：Claude Sonnet 4.6

**能力**：
- 将 C 源码直接翻译为惯用 Rust 代码
- 正确映射 C 类型到 Rust 类型（指针→引用/Box/Vec、union→enum 等）
- 将 malloc/free 转换为 RAII（Box、Vec 自动释放）
- 将 C 错误处理转换为 Result/Option
- 将全局可变状态重构为显式参数传递或 Mutex/OnceLock
- 生成 `///` 文档注释和 `#[cfg(test)]` 单元测试

### c-analyzer

**调用时机**：`/c2rust-assess --deep`

**能力**：
- 构建函数调用图
- 追踪全局可变状态的访问路径
- 分析宏复杂度（X-macros、递归宏、token pasting）
- 识别线程安全模式
- 检测 C 惯用模式（错误 goto、对象模式、状态机等）

### debug-assistant

**调用时机**：`/c2rust-refine`

**能力**：
- 解析 rustc 错误输出
- 将错误码映射到具体修复方案
- 解决生命周期/借用检查器问题
- 修复混合 C/Rust 构建的链接错误

### rust-reviewer

**调用时机**：`/c2rust-verify --full`

**能力**：
- 识别非惯用 Rust 模式
- 建议所有权/借用改进
- 发现可以变为 safe 的 unsafe 代码
- 检查错误处理一致性
- 审查公共 API 设计

---

## 参考资料库

插件内置了 6 份参考资料，被各 skill 在运行时自动加载：

| 文件 | 位置 | 内容 |
|------|------|------|
| **C 模式目录** | `c2rust-assess/references/c-pattern-catalog.md` | 30+ 种 C 模式的转换难度评级和 Rust 目标方案 |
| **复杂度指标** | `c2rust-assess/references/complexity-metrics.md` | 风险评分计算方法、LOC/CC/依赖深度等指标定义 |
| **依赖映射表** | `c2rust-plan/references/crate-mapping.md` | 50+ 常见 C 库到 Rust crate 的映射（openssl→rustls, zlib→flate2 等） |
| **FFI 模式库** | `c2rust-plan/references/ffi-patterns.md` | 8 种 FFI 边界设计模式（安全包装、不透明类型、回调桥接、所有权转移等） |
| **unsafe 转换模式** | `c2rust-refine/references/unsafe-to-safe.md` | 9 类 unsafe→safe 的转换模式（裸指针→引用、malloc→Box、C字符串→String 等） |
| **错误修复目录** | `c2rust-refine/references/error-fix-catalog.md` | 30+ rustc 错误码的修复方案 |

---

## 典型工作流示例

### 示例 1：小型 C 库的完整转换

```bash
# 进入 C 项目目录
cd ~/projects/my-c-lib

# 启动全流程引导
/c2rust

# Claude 会依次引导你完成：
# 1. 工具链检查 → 2. 评估 → 3. 计划 → 4. 测试 → 5. 翻译 → 6. 精炼 → 7. 验证
# 每个阶段结束后询问是否继续
```

### 示例 2：只评估不转换

```bash
/c2rust-assess --deep
# 查看 c2rust-assessment.md 了解转换难度和风险
```

### 示例 3：逐模块增量转换

```bash
# 评估和计划
/c2rust-assess --quick
/c2rust-plan --incremental

# 按计划顺序逐模块转换
/c2rust-convert utils      # 先转换最简单的
/c2rust-refine utils
/c2rust-verify utils        # 确认 utils 通过后再继续

/c2rust-convert parser
/c2rust-refine parser
/c2rust-verify parser

# 查看整体进度
/c2rust status
```

### 示例 4：中断后恢复

```bash
# 之前的会话中断了
/c2rust status              # 查看进度
/c2rust resume              # 从上次完成的阶段继续
```

### 示例 5：只修复编译错误

```bash
# 翻译完成后，只需要修复编译错误
/c2rust-refine --auto       # 只做机械性修复，不问语义问题
```

---

## 常见问题

### Q: 翻译出的 Rust 代码编译不过怎么办？

这是正常的。Claude 翻译产出的代码质量已经很高，但跨模块类型一致性、依赖版本等问题可能导致编译错误。使用 `/c2rust-refine` 自动修复大部分机械性错误，语义性问题会暂停咨询你。

### Q: 某个模块风险等级为 CRITICAL，怎么办？

CRITICAL 模块（如包含内联汇编）建议：
- 增量转换模式中保持为 C 代码，通过 FFI 调用
- 在 `c2rust-plan` 阶段标记为 "FFI 边界"
- 未来手动重写

### Q: 精炼阶段陷入循环怎么办？

如果自动修复循环 3 轮后错误数不再减少：
- 检查是否有相互依赖的错误（一个错误的修复导致另一个）
- 切换到 `--interactive` 模式逐个检查
- 直接使用 `/c2rust-refine --interactive` 查看每个问题

### Q: 如何处理评估中标记为 BLOCKING 的模式？

BLOCKING 模式无法自动翻译：
1. 内联汇编 → 使用 `core::arch::asm!` 手动重写
2. computed goto → 重构为 match/enum 状态机
3. setjmp/longjmp → 重构为 `Result<T, E>` 错误传播

建议在增量模式中将包含 BLOCKING 模式的模块保持为 C，通过 FFI 调用。

### Q: 翻译后的代码中有 unsafe，正常吗？

Claude 翻译产出的 Rust 代码通常 unsafe 很少，主要出现在 FFI 边界（与尚未转换的 C 模块交互时）。Phase 5 (refine) 的 Phase C 会进一步审查和最小化 unsafe 使用。最终目标是将 unsafe 限制在 FFI 边界，并为每个 unsafe 块添加 `// SAFETY:` 注释。

### Q: 支持哪些 C 项目？

理论上支持任何 C 项目。翻译引擎直接读取 C 源文件，不依赖特定构建系统。评估阶段会识别项目的构建系统（Make / CMake / Autotools / Meson）以帮助理解项目结构，但翻译过程本身不需要编译 C 代码。

### Q: 翻译引擎用的是什么模型？

使用 Claude Sonnet 4.6（`model: sonnet`），通过 `c-to-rust-translator` Agent 执行。该 Agent 内置了全面的 C→Rust 类型映射、内存管理模式、错误处理模式等翻译规则。

### Q: 如何重新转换某个模块？

如果对某个模块的翻译结果不满意，可以手动回退：

1. 在 `c2rust-manifest.toml` 中将该模块的 `status` 改回 `"planned"` 或 `"assessed"`
2. 删除对应的 `.rs` 输出文件
3. 重新运行 `/c2rust-convert <module-name>`

目前没有自动回退命令，需要手动编辑 manifest。

### Q: 支持 C++ 项目吗？

不支持。本插件仅支持纯 C 项目的转换。评估阶段会检测 C++ 文件（`.cpp`, `.cc`, `.cxx`）并发出警告，但不会尝试翻译 C++ 代码。如果项目混合了 C 和 C++，只有 C 文件会被处理。

---

## 插件结构

```
c2rust-skills/
├── .claude-plugin/
│   └── plugin.json                     # 插件元数据
├── README.md                           # 插件简介
├── USAGE.md                            # 本使用指南
├── skills/
│   ├── c2rust/                         # 总指挥
│   │   └── SKILL.md
│   ├── c2rust-check-env/               # 工具链检查
│   │   └── SKILL.md
│   ├── c2rust-assess/                  # 评估与理解
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── c-pattern-catalog.md    #   C 模式转换难度目录
│   │       └── complexity-metrics.md   #   复杂度指标定义
│   ├── c2rust-plan/                    # 转换计划
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── crate-mapping.md        #   C 库 → Rust crate 映射
│   │       └── ffi-patterns.md         #   FFI 边界设计模式
│   ├── c2rust-test/                    # 测试构建
│   │   ├── SKILL.md
│   │   └── templates/
│   │       ├── integration-test.rs     #   集成测试模板
│   │       └── ffi-test.rs             #   FFI 边界测试模板
│   ├── c2rust-convert/                 # 执行转换（Claude Sonnet 翻译）
│   │   └── SKILL.md
│   ├── c2rust-refine/                  # 编译调试与精炼
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── unsafe-to-safe.md       #   unsafe → safe 转换模式
│   │       └── error-fix-catalog.md    #   rustc 错误修复目录
│   └── c2rust-verify/                  # 验证
│       └── SKILL.md
└── agents/
    ├── c-to-rust-translator.md         # C→Rust 翻译 Agent (Sonnet 4.6)
    ├── c-analyzer.md                   # C 代码深度分析 Agent
    ├── debug-assistant.md              # 编译错误诊断修复 Agent
    └── rust-reviewer.md                # Rust 代码质量审查 Agent
```

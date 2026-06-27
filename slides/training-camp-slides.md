---
marp: true
theme: default
paginate: true
---

# 基于eBPF的内核可观测性能力增强

## 2026春季OS训练营 总结汇报

**CN-TangLin** | 2026.06.28

子课题：方案二 - 3. 基于eBPF的内核可观测性能力增强  
仓库：rcore-os/tgoskits | StarryOS

---

## 汇报大纲

1. 项目概览与整体架构
2. 主线一：eBPF子系统全栈构建
3. 主线二：kprobe/kretprobe内核探针
4. 主线三：LKM内核模块加载
5. 主线四：eBPF JIT编译器（三架构）
6. 主线五：工程质量改进（进行中）
7. 量化成果与协作关系
8. 经验教训与后续方向

---

## 一、项目概览

**StarryOS** = ArceOS组件化框架 + Linux兼容层

本课题目标：在StarryOS中构建完整的eBPF可观测性栈

```
用户态 aya-ebpf程序(.elf)
    ↓ bpf(2) syscall
kbpf-basic (prog load/verify/map)
    ↓
eBPF VM 执行引擎 ← 三架构JIT编译器
    ↓
kprobe / tracepoint / perf_event (attach)
```

---

## 成果一览

| 指标 | 数据 |
|---|---|
| 已合并PR | **12** 个 |
| 当前开放PR | **4** 个 |
| 总commits | **364** |
| 新增代码 | **15000+** 行 |
| 架构覆盖 | x86_64 / RISC-V 64 / AArch64 / LoongArch |
| JIT代码量 | **3317行** (3架构) |

---

## 二、主线一：eBPF子系统全栈构建

### #848 MERGED - eBPF核心子系统

实现了 `bpf(2)` 系统调用的 **9条命令**：

BPF_MAP_CREATE | BPF_PROG_LOAD | BPF_MAP_UPDATE_ELEM  
BPF_MAP_LOOKUP_ELEM | BPF_MAP_DELETE_ELEM | BPF_MAP_GET_NEXT_KEY  
BPF_MAP_FREEZE | BPF_MAP_LOOKUP_AND_DELETE_ELEM | BPF_RAW_TRACEPOINT_OPEN

核心文件：
- `ebpf/mod.rs` (178行) - 系统调用入口
- `ebpf/transform.rs` (302行) - 内核辅助接口桥接
- `ebpf/map.rs` (154行) - Map封装，poll+mmap

---

## 二、主线一（续）：perf_event子系统

### #848 MERGED

支持4种perf_event类型：

| 类型 | 状态 |
|---|---|
| PERF_TYPE_KPROBE | ✅ 含kretprobe |
| PERF_TYPE_SOFTWARE | ✅ 软件事件+ringbuf |
| PERF_TYPE_TRACEPOINT | ✅ |
| PERF_TYPE_UPROBE | ✅ |

执行引擎 `OwnedEbpfVm`：先尝试JIT → 失败fallback解释器  
`BpfPerfEventWrapper`：ringbuf + poll 通知机制

---

## 二、主线一（续）：测试与增强

### #874 MERGED - eBPF测试套件

8个用户态eBPF程序：`syscall_count`, `profile`, `mytrace`, `sched_trace`, `rawtp`, `kret`, `upb`, `upb2`

覆盖：kprobe, kretprobe, tracepoint, raw_tracepoint, uprobe

### #888 MERGED - eBPF增强

deadlock修复 + 新map类型 + verifier增强

---

## 三、主线二：kprobe/kretprobe

### #847 MERGED - kprobe内核探针

```rust
// kprobe.rs (639行) - 全架构支持
KernelKprobeOps: copy_memory, set_writeable,
    alloc/free_kernel/exec_memory, kretprobe instance

// 4架构trapframe↔ptregs转换
trapframe_to_ptregs()  // x86_64, RISC-V 64, AArch64, LoongArch
ptregs_write_back()
```

### #887 MERGED - AArch64 TrapFrame SP修复

修复aarch64 TrapFrame中SP寄存器保存，确保kprobe正确获取调用栈上下文

---

## 四、主线三：LKM内核模块

### #849 MERGED - kmod-loader集成

```rust
// kmod/mod.rs (265行)
KmodHelper: vmalloc / resolve_symbol / flush_cache
init_module()   → 加载.ko ELF, 重定位, 调用init
delete_module() → 调用exit, 释放内存
lwprintf-rs → printk支持
```

使StarryOS能够加载和运行 `.ko` 格式的Linux内核模块

---

## 五、主线四：eBPF JIT编译器

### 三架构手写JIT — 最具挑战性的工作

**统一框架设计**：

```
try_jit_compile()
  → JitCompiler::compile()
    → Pass 1: sizing (仅计算长度)
    → Pass 2: compile (生成机器码)
      → JitBackend trait (8个emit方法)
```

**两遍Pass设计**：Pass 1确定每条指令/label位置 → Pass 2正确填充跳转偏移

---

## 五、JIT（续）：寄存器映射

统一映射原则（三架构一致）：

| BPF寄存器 | 用途 | x86_64 | RISC-V 64 | AArch64 |
|---|---|---|---|---|
| R0-R5 | 参数/返回值 | RAX..R8 | A0..A5 | X0..X5 |
| R6-R9 | callee-saved | - | S0..S4 | X19..X22 |
| R10 | 帧指针 | RBP | S5 | X25 |

R6-R9映射到callee-saved寄存器 → 跨helper调用不丢失

---

## 五、JIT（续）：各架构实现

| 架构 | 代码量 | 关键特性 |
|---|---|---|
| **RISC-V 64** | ~1311行 | 完整ALU/JMP/MEM, div-by-zero检查 |
| **x86_64** | ~904行 | 完整CALL指令处理, 安全检查 |
| **AArch64** | ~1102行 | A64编码全覆盖, 变体/偏移/后索引寻址 |

**当前状态**：代码完整存在于分支，未合入upstream/dev  
upstream/dev使用 `rbpf::EbpfVmRaw::jit_compile()`

---

## 六、主线五：工程质量改进

### 14→4 PR合并策略

| 合并后PR | 主题 | 合并自 |
|---|---|---|
| [#1412](https://github.com/rcore-os/tgoskits/pull/1412) | 魔术数字+安全加固 | PR-a,d,f,g |
| [#1413](https://github.com/rcore-os/tgoskits/pull/1413) | panic!/todo!+伪文件系统 | PR-h,i,j,k,l |
| [#1414](https://github.com/rcore-os/tgoskits/pull/1414) | perf read/poll/nonblock | PR-b,c,e |
| [#1411](https://github.com/rcore-os/tgoskits/pull/1411) | helper注册 | PR-m |

---

## 六、工程质量（续）：关键修复

**内核崩溃修复**：
- card1.rs 未知DRM ioctl → `panic!()` → 任何用户程序可使内核崩溃
- ldisc.rs VTIME>0 → `todo!()` → 任何设置了VTIME的终端读取都触发panic

**安全加固**：
- 移除 `vm.register_allowed_memory(0..u64::MAX)` → 启用rbpf地址边界检查
- 恶意BPF程序不能再读取任意内核内存

---

## 六、工程质量（续）：新功能

**perf event fd完整语义** (#1414)：
- `read()` — 阻塞读取，Linux兼容
- `poll()` — ringbuf为空时等待，利用已有Waker基础设施
- `O_NONBLOCK` — ringbuf为空立即返回EAGAIN

**helper函数注册** (#1411)：
- `bpf_get_current_pid_tgid` (#14) — `ax_task::current().as_thread()`
- `bpf_get_current_comm` (#16) — 拷贝`name()`到verifier验证过的buffer

---

## 七、协作关系

```
linfeng(Godones) 助教
├── #673 tracepoint 基础设施
├── #805 kallsyms + kprobe stub
└── #837 内核符号导出
     ↓ (基础平台)
CN-TangLin (我)
├── #847 kprobe 实现
├── #848 eBPF 子系统
├── #849 LKM 支持
└── #891/892/893 JIT编译器 (3架构)
     ↓ (技术路线融合)
LorenzLorentz 同学
├── #850 eBPF runtime (kbpf-basic+rbpf)
├── #851 LKM loader 端口
└── #1132 eBPF demos
```

---

## 八、技术经验

### JIT两遍Pass设计

**问题**：前向跳转指令的偏移量在第一次遇见label时未知  
**方案**：Pass 1计算每条指令长度和每个label位置，Pass 2填充偏移  
**优势**：不需要额外内存，单次遍历确定所有位置

### eBPF安全边界

现代eBPF通过helper访问内存，不需要直接内存访问  
`register_allowed_memory(0..u64::MAX)` = 安全漏洞

### 稳定性修复优先

`panic!()`/`todo!()` = 用户可触发内核崩溃  
先消除已有崩溃路径，再添加新功能

---

## 九、教训

1. **分支管理**：AArch64 JIT分支落后692 commits，长期分支需及时rebase

2. **技术路线兼容性**：手写JIT代码完整但未合并，需考虑与社区技术路线（rbpf JIT）的兼容

3. **PR粒度**：14个小PR → 4个合并PR，找到合适粒度是关键

4. **CI利用**：fmt/sync-lint/spin-lint快速验证，QEMU超时是基础设施问题不是代码问题

---

## 十、后续方向

- AArch64 JIT 重新适配（从upstream/dev新开分支）
- JIT回归测试集成到CI
- BTF支持（CO-RE兼容）
- LoongArch64 JIT
- BPF_PROG_ATTACH/DETACH/LINK_CREATE

---

## 总结

| 维度 | 成果 |
|---|---|
| eBPF子系统 | Map/Prog/VM/Helper/PerfEvent 全覆盖 |
| 内核可观测性 | kprobe/kretprobe/uprobe/tracepoint |
| 内核扩展 | LKM模块加载/卸载 |
| JIT编译器 | 3架构手写，3317行代码 |
| 工程质量 | 14→4 PR合并，消除崩溃路径 |
| **总PR** | **12合并 + 4进行中 = 16** |

---

## 谢谢！

GitHub: [CN-TangLin](https://github.com/CN-TangLin)  
仓库: [CN-TangLin/tgoskits](https://github.com/CN-TangLin/tgoskits)  
Blog: [rcore-os/blog PR #891](https://github.com/rcore-os/blog/pull/891)

当前开放PR: [#1411](https://github.com/rcore-os/tgoskits/pull/1411) [#1412](https://github.com/rcore-os/tgoskits/pull/1412) [#1413](https://github.com/rcore-os/tgoskits/pull/1413) [#1414](https://github.com/rcore-os/tgoskits/pull/1414)

---
marp: true
theme: default
paginate: true
size: 16:9
style: |
  section {
    font-size: 24px;
  }
  section.title {
    text-align: center;
  }
  table {
    font-size: 18px;
  }
  pre {
    font-size: 16px;
  }
---

<!-- _class: title -->

# 基于eBPF的内核可观测性能力增强

## 2026春季OS训练营总结报告

**CN-TangLin (唐林)**

2026年6月28日

---

## 目录

1. 项目背景与训练营历程
2. 整体架构：eBPF子系统全景
3. 主线一：kprobe/kretprobe 动态探针
4. 主线二：eBPF子系统全栈构建
5. 主线三：LKM内核模块
6. 主线四：三架构手写JIT编译器
7. 主线五：工程质量持续改进
8. 量化成果
9. 团队协作与经验教训
10. 总结与展望

---

## 1. 项目背景

### StarryOS

- 构建在 ArceOS 组件化框架之上的 Linux 兼容操作系统
- 目标是运行未经修改的 Linux 用户态程序

### 训练营方案阶段

| 阶段 | 我的工作 | 核心成果 |
|------|---------|---------|
| **方案一** | select/poll/ppoll/pselect6 + sys_msync | 43个测试模块, SQLite跑通 |
| **方案二** | 子课题3：eBPF内核可观测性 | kprobe, eBPF全栈, LKM, JIT |

---

## 2. eBPF子系统架构全景

```
┌──────────────────────────────────────────────────┐
│                  用户态工具层                      │
│  aya eBPF  │  test-ebpf-* (8个测试程序)           │
├──────────────────────────────────────────────────┤
│                系统调用接口层                      │
│  sys_bpf()  │  sys_perf_event_open()              │
├──────────────────────────────────────────────────┤
│                eBPF 运行时层                       │
│  ┌──────────┐ ┌───────────┐ ┌────────────┐       │
│  │ eBPF VM  │ │ Map Mgr   │ │ Verifier   │       │
│  │ 22 helpers│ │ 7种Map   │ │ 程序验证   │       │
│  └──────────┘ └───────────┘ └────────────┘       │
├──────────────────────────────────────────────────┤
│             数据通道 & 事件源层                    │
│  perf_event ringbuf  │  Tracepoint               │
├──────────────────────────────────────────────────┤
│            动态探针基础层                          │
│  kprobe/kretprobe  │  LKM loader  │  kallsyms   │
└──────────────────────────────────────────────────┘
```

---

## 3. 主线一：kprobe/kretprobe 动态探针

### 核心实现（`kprobe.rs`, 639行）

- 4架构 `trapframe_to_ptregs()` / `ptregs_write_back()`
  - x86_64 / RISC-V 64 / AArch64 / LoongArch
- 基于 IPI 的 `stop_machine` 安全代码段修改
- `kretprobe_stack` 支持嵌套 kretprobe

### AArch64 SP修复（#887）

```rust
// components/axcpu/src/arch/aarch64/trap.rs
// 修复异常返回时 SP 指针未正确保存的严重 Bug
```

### PR: [#847](https://github.com/rcore-os/tgoskits/pull/847) (MERGED)

---

## 4. 主线二：eBPF子系统全栈构建（#848）

### 9条 BPF 命令

| 命令 | 功能 |
|------|------|
| `BPF_MAP_CREATE` | 创建 Array/Hash/PerCPU/RingBuf Map |
| `BPF_PROG_LOAD` | 加载 BPF 程序（经 verifier 验证） |
| `BPF_RAW_TRACEPOINT_OPEN` | 打开 raw tracepoint |
| `BPF_MAP_UPDATE/LOOKUP/DELETE_ELEM` | Map 元素操作 |
| `BPF_MAP_GET_NEXT_KEY` | Map 遍历 |
| `BPF_MAP_FREEZE` | Map 冻结（只读保护） |

### 代码量

| 文件 | 行数 | 功能 |
|------|------|------|
| `ebpf/mod.rs` | 178 | 系统调用入口 |
| `ebpf/transform.rs` | 302 | 内核桥接层 |
| `ebpf/map.rs` | 154 | Map管理 |
| `perf/bpf.rs` | 330 | 运行时引擎 + ringbuf |
| `perf/mod.rs` | 253 | perf_event_open调度 |

---

## 4. 主线二（续）：perf_event 集成

### 4种事件类型

| 类型 | 状态 | 代码位置 |
|------|:----:|---------|
| `PERF_TYPE_KPROBE` | ✅ | `perf/kprobe.rs` (208行) |
| `PERF_TYPE_SOFTWARE` | ✅ | `perf/bpf.rs` (330行) |
| `PERF_TYPE_TRACEPOINT` | ✅ | `perf/tracepoint.rs` (160行) |
| `PERF_TYPE_UPROBE` | ✅ | `perf/uprobe.rs` (92行) |

### perf_event fd 读写（#1412 新增）

```rust
// 阻塞读取（Linux 语义一致）
fn read(&self, dst: &mut IoDst) -> AxResult<usize> {
    block_on(poll_io(self, IoEvents::IN, self.nonblocking(), || {
        let mut event = self.event.lock();
        bpf_event.try_read_record(&mut buf)
    }))
}
```

---

## 5. 主线三：LKM内核模块（#849）

### 核心实现（`kmod/mod.rs`, 265行）

```
init_module()           delete_module()
    │                       │
    ▼                       ▼
┌─────────┐           ┌─────────┐
│ 加载.ko  │           │ 调用exit │
│ ELF解析 │           │ 释放内存 │
│ 重定位   │           │ 注销模块 │
│ 调用init │           └─────────┘
└─────────┘
```

- `KmodHelper`: vmalloc / resolve_symbol / flush_cache
- `KmodSectionMem`: 页对齐 + R/W/X 隔离
- `lwprintf-rs`: printk 支持

---

## 6. 主线四：三架构手写eBPF JIT

### 统一 Two-Pass 架构

```
eBPF 字节码
    │
    ▼
┌─────────────┐     ┌─────────────┐
│  Pass 1     │ ──► │  Pass 2     │
│  Sizing     │     │  Compile    │
│  计算偏移    │     │  生成机器码  │
└─────────────┘     └─────────────┘
                          │
                          ▼
                    函数指针（可调用）
                    fallback → 解释器
```

### 代码量

| 文件 | 行数 |
|------|------|
| `ebpf_jit/mod.rs`（框架） | 352 |
| `ebpf_jit/jit_riscv64.rs` | ~1311 |
| `ebpf_jit/jit_x86_64.rs` | 904 |
| `ebpf_jit/jit_aarch64.rs` | 1102 |
| **合计** | **~3317** |

---

## 6. 主线四（续）：寄存器映射

### 统一映射原则

- **R0-R5** → 参数/返回值寄存器（利用调用约定）
- **R6-R9** → callee-saved 寄存器（跨 helper 不丢失）
- **R10** → 帧指针寄存器

| BPF | RISC-V 64 | x86_64 | AArch64 |
|-----|-----------|--------|---------|
| R0 | A0 | RAX | X0 |
| R1 | A1 | RDI | X1 |
| R2 | A2 | RSI | X2 |
| R3 | A3 | RDX | X3 |
| R4 | A4 | RCX | X4 |
| R5 | A5 | R8 | X5 |
| R6 | S0 | — | X19 |
| R7 | S1 | — | X20 |
| R8 | S2 | — | X21 |
| R9 | S4 | — | X22 |
| R10 | S5 | RBP | X25 |

---

## 7. 主线五：工程质量改进

### 当前3个APPROVED PR

| PR | 内容 | 状态 |
|----|------|:----:|
| [#1412](https://github.com/rcore-os/tgoskits/pull/1412) | perf fd read/poll/O_NONBLOCK + 魔数替换 + 安全加固 | CI中 |
| [#1413](https://github.com/rcore-os/tgoskits/pull/1413) | panic!/todo!替换 + 伪文件系统魔数消除 | CI中 |
| [#1411](https://github.com/rcore-os/tgoskits/pull/1411) | bpf_get_current_pid_tgid + bpf_get_current_comm helper | CI中 |

### 关键改进

- **安全加固**：移除 `register_allowed_memory(0..u64::MAX)` → 启用 rbpf `check_mem`
- **类型安全**：`try_read_record(&self)` → `try_read_record(&mut self)`
- **稳定性**：`card1.rs` panic → `VfsError::OperationNotSupported`，`ldisc.rs` todo → 文档注释
- **可维护性**：30+ 处魔数 → 命名常量

---

## 8. 量化成果

### PR统计

| 类别 | 数量 |
|------|:----:|
| 已合并 PR | **11** |
| 当前开放 PR（APPROVED） | **3** |
| 历史关闭 PR | **16+** |

### 代码量

| 维度 | 数据 |
|------|------|
| Commits | **364** |
| 新增代码行数 | **15000+** |
| 架构覆盖 | **4** (x86_64, RISC-V 64, AArch64, LoongArch) |
| JIT 代码量 | **3317** 行（3架构） |
| 测试程序 | **8** 个 eBPF + **43** 个 select/poll 模块 |

---

## 9. 团队协作

### 子课题3：三人协作全景

| 角色 | 成员 | 核心贡献 |
|------|------|---------|
| **助教** | linfeng(Godones) | tracepoint(#673), kallsyms(#837), dynamic debug(#446), break异常(#244) |
| **同学** | LorenzLorentz | eBPF runtime port(#850), aya生态(#886), LKM port(#851) |
| **我** | CN-TangLin | kprobe(#847), eBPF全栈(#848), LKM(#849), JIT, 工程质量 |

### 技术路线融合

- **我的路线**：手写 eBPF 子系统 → 仅依赖标准库
- **LorenzLorentz 路线**：kbpf-basic + rbpf 外部 crate

→ **最终融合为统一的 eBPF 执行框架**

---

## 9. 经验与教训（续）

### 成功经验

- **小 PR 迭代** → 14个小 PR 合并为 3个大 PR，便于 review
- **bot 辅助审查** → CHANGES_REQUESTED → APPROVED 高效循环
- **Two-Pass JIT** → 解决前向跳转偏移计算

### 教训

- **AArch64 JIT 分支管理失误** → 692 commits behind upstream/dev
- **JIT 代码未及时合入主线** → 技术方案需与社区路线兼容

### 核心方法论

> "刨根问底排bug、小步快跑推PR、严谨量化写报告"

---

<!-- _class: title -->

## 10. 总结与展望

### 已完成

- ✅ eBPF 全栈：Map/Prog/VM/Helper/perf_event
- ✅ kprobe/kretprobe/uprobe（4架构）
- ✅ LKM 内核模块加载
- ✅ 三架构手写 JIT 编译器
- ✅ select/poll/sys_msync/SMP/DRM

### 后续方向

- 🔲 AArch64 JIT 重新适配 upstream/dev
- 🔲 BTF 支持（CO-RE 兼容）
- 🔲 LoongArch64 JIT
- 🔲 BPF_PROG_ATTACH/DETACH/LINK_CREATE

---

<!-- _class: title -->

# 感谢

## 陈渝老师、周睿老师、向勇老师、王铮学长、朱懿学长、邵志航学长、王鹏杰同学、戴骏翔助教、陈林峰助教

### 仓库链接

- **主仓库**: https://github.com/CN-TangLin/tgoskits
- **Blog**: https://github.com/rcore-os/blog/pull/891
- **进展记录**: https://github.com/rcore-os/tgoskits/issues/642

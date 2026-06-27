# 技术报告：基于eBPF的内核可观测性能力增强

> 2026春季OS训练营 Stage 7  
> 子课题：方案二 - 3. 基于eBPF的内核可观测性能力增强  
> 作者：CN-TangLin  
> 日期：2026-06-28  
> 仓库：rcore-os/tgoskits

## 1. 技术架构总览

本项目在StarryOS（基于ArceOS组件化框架的Linux兼容操作系统）上构建了完整的eBPF可观测性栈。整体技术架构如下：

```
┌─────────────────────────────────────────┐
│          用户态 eBPF 程序 (.elf)          │
│     (Rust aya-ebpf / C bpftool)         │
├─────────────────────────────────────────┤
│          bpf(2) 系统调用层                │
│    ebpf/mod.rs (178行) - 9条命令调度       │
├─────────────────────────────────────────┤
│  kbpf-basic 0.6 (prog load/verify/map)  │
├──────────────┬──────────────────────────┤
│  eBPF VM 解释器  │   eBPF JIT 编译器       │
│  (rbpf 0.4)     │   (手写,3架构)          │
├─────────────────────────────────────────┤
│    perf_event 子系统 (4种事件类型)         │
│    kprobe / tracepoint / uprobe          │
└─────────────────────────────────────────┘
```

## 2. eBPF子系统实现细节

### 2.1 Map管理

`ebpf/map.rs` (154行) 使用泛型封装实现了类型安全的Map操作：

```rust
pub struct BpfMap {
    inner: UnifiedMap<KernelRawMutex>,
}

impl BpfMap {
    pub fn new(map_type: BpfMapType, ...) -> AxResult<Self>
    pub fn update_elem(&self, key: &[u8], value: &[u8], flags: u64)
    pub fn lookup_elem(&self, key: &[u8]) -> Option<Vec<u8>>
    pub fn delete_elem(&self, key: &[u8])
    pub fn get_next_key(&self, key: Option<&[u8]>) -> Option<Vec<u8>>
}
```

支持Map类型：Array, Hash(percpu), RingBuf, PerfEventArray, StackTrace等。

### 2.2 内核辅助接口

`ebpf/transform.rs` (302行) 实现了`KernelAuxiliaryOps` trait，桥接kbpf-basic到tgOSKits内核：

- 内存分配/释放：通过内核内存管理器
- per-CPU变量访问：`ax_task::percpu()`
- BPF helper函数表：`BPF_HELPER_FUN_SET`
- 时间获取：`ax_time::current_ticks()`
- vmap支持：环形缓冲区内核映射

### 2.3 执行引擎

`perf/bpf.rs` (330行) 中的`OwnedEbpfVm`是核心执行引擎：

```rust
pub struct OwnedEbpfVm {
    prog: BpfProg,
    jit_fn: Option<fn(&[u8], &PtRegs) -> u64>,
}

impl OwnedEbpfVm {
    fn try_jit(&mut self) -> Option<fn(&[u8], &PtRegs) -> u64>
    pub fn execute_program(&self, ctx: &[u8]) -> u64
    pub fn execute_with_ptregs(&self, ptregs: &PtRegs) -> u64
}
```

执行优先级：JIT编译成功 → JIT执行；JIT失败 → rbpf解释器执行。

## 3. JIT编译器设计与实现

### 3.1 两遍Pass设计

```
Pass 1: sizing
  遍历BPF指令 → 计算每条指令目标机器码长度
  → 记录每个label的位置(offset)
  → 确定JitBuffer总大小

Pass 2: compile
  再次遍历BPF指令 → 调用架构相关emit方法
  → 利用Pass 1记录的label位置填充跳转偏移
  → 生成最终机器码
```

### 3.2 JitBackend Trait

```rust
pub trait JitBackend {
    fn emit_alu(&mut self, op: BpfAluOp, dst: u8, src: u8, imm: i32) -> JitResult;
    fn emit_jmp(&mut self, op: BpfJmpOp, dst: u8, src: u8, off: i16, imm: i32) -> JitResult;
    fn emit_ld(&mut self, mode: BpfLdMode, dst: u8, src: u8, off: i16, imm: i32) -> JitResult;
    fn emit_st(&mut self, mode: BpfStMode, dst: u8, src: u8, off: i16, imm: i32) -> JitResult;
    fn emit_ldx(&mut self, mode: BpfLdxMode, dst: u8, src: u8, off: i16) -> JitResult;
    fn emit_stx(&mut self, mode: BpfStxMode, dst: u8, src: u8, off: i16) -> JitResult;
    fn emit_call(&mut self, func_id: u32) -> JitResult;
    fn emit_exit(&mut self) -> JitResult;
}
```

### 3.3 AArch64 JIT关键技术

AArch64后端（1102行）展现了完整的A64指令编码能力：

**寄存器分配策略**：
- BPF R0-R5 → X0-X5：参数/返回值，遵循AAPCS64调用约定
- BPF R6-R9 → X19-X22：callee-saved寄存器，确保跨helper调用上下文保持
- BPF R10 → X25：帧指针，稳定且不影响调用约定

**指令编码示例**（ALU ADD dst, imm）：
```rust
// ADD Xd, Xd, #imm  (imm ≤ 12-bit)
fn emit_alu_add_imm(&mut self, dst: u8, imm: i32) -> JitResult {
    let rd = self.bpf_reg_to_a64(dst)?;
    // A64 ADD immediate: 1 0 0 100010 0 sh imm12 Rn Rd
    let instr = 0x8B000000 | (imm as u32 & 0xFFF) << 10 | rd << 5 | rd;
    self.buffer.emit_u32(instr);
    Ok(())
}
```

**内存访问支持三种寻址模式**：
- 基址+偏移：`LDR Xd, [Xn, #offset]`
- 基址+变体：`LDR Xd, [Xn, Xm, LSL #3]`
- 后索引：`LDR Xd, [Xn], #offset`

## 4. kprobe实现细节

### 4.1 四架构TrapFrame→PtRegs转换

每个架构有不同的异常帧格式，需要统一转换为Linux兼容的`PtRegs`结构：

| 架构 | 转换函数 | 关键寄存器 |
|---|---|---|
| x86_64 | `x86_64::trapframe_to_ptregs` | RAX, RDI, RSI, RDX, RCX, R8, R9, RBP, RSP, RIP |
| RISC-V 64 | `riscv64::trapframe_to_ptregs` | A0-A7, RA, SP, GP, TP, PC |
| AArch64 | `aarch64::trapframe_to_ptregs` | X0-X30, SP, PC, PSTATE |
| LoongArch | `loongarch::trapframe_to_ptregs` | R0-R31, PC, ERA |

### 4.2 kretprobe实例管理

```rust
// 每个task可以有一个活跃的kretprobe
// entry_handler记录返回地址 → 替换为trampoline
// ret_handler在trampoline中触发 → 恢复返回地址
struct KretprobeInstance {
    entry_func: usize,         // 被探测函数的入口地址
    original_return_addr: usize, // 原始返回地址
    callback: Box<dyn Fn(&PtRegs)>,
}
```

## 5. PR合并策略与CI管理

### 5.1 14→4合并过程

| 阶段 | PR数 | 策略 |
|---|---|---|
| 初始 | 14 | 每改动一个文件/概念一个PR |
| 第一波 | 6 | 关联fix归组(probe+magic+mem→1, card+tty+pseudo→1) |
| 第二波 | 4 | perf feat归组(read+poll+nonblock→1) |

### 5.2 CI管线

```
PR 提交 → fmt check → sync-lint → spin-lint
  → (parallel)
    QEMU aarch64 → QEMU riscv64 → QEMU x86_64 → QEMU loongarch64
    Board OrangePi → Board RDK-S100
```

fmt/sync-lint/spin-lint通常1-2分钟完成，QEMU container测试约30-34分钟。

## 6. 已知问题与改进方向

### 6.1 当前限制

| 问题 | 影响 | 优先级 |
|---|---|---|
| AArch64 JIT未合入 | AArch64无手写JIT加速 | 高 |
| BTF未支持 | 无法使用CO-RE eBPF程序 | 中 |
| 无BPF_PROG_ATTACH | 部分现代eBPF工具不兼容 | 中 |
| 无LoongArch JIT | LoongArch无手写JIT加速 | 低 |

### 6.2 AArch64 JIT重新适配方案

```bash
# 从upstream/dev开新分支
git checkout upstream/dev
git checkout -b feat/ebpf-jit-aarch64-rebase

# 从旧分支提取JIT文件
git checkout feat/ebpf-jit-3-aarch64 -- \
  os/StarryOS/kernel/src/ebpf/ebpf_jit/

# 适配当前perf/bpf.rs中的JIT调用路径
# 处理与kbpf-basic 0.6 + rbpf 0.4的接口兼容
```

### 6.3 性能优化方向

- JIT编译缓存：避免重复编译同一BPF程序
- 批量prog load：减少单次系统调用开销
- 预编译hot path：对高频helper进行inline

## 7. 参考资料

- Linux BPF Documentation: https://docs.kernel.org/bpf/
- rcore-os/tgoskits: https://github.com/rcore-os/tgoskits
- kbpf-basic: https://crates.io/crates/kbpf-basic
- AArch64 ISA Reference: ARM DDI 0487

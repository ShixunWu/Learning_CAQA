# Week 3 实验：RISC-V 汇编实践

---

## 实验 1：第一个 RISC-V 程序

```asm
# fib.s - 斐波那契数列 (RISC-V RV32I)
    .section .text
    .globl main
main:
    li   a0, 10         # n = 10
    call fib
    li   a7, 93         # exit syscall
    ecall

fib:
    li   t0, 2
    ble  a0, t0, fib_base
    addi sp, sp, -16
    sw   ra, 12(sp)
    sw   a0, 8(sp)
    addi a0, a0, -1
    call fib
    sw   a0, 4(sp)
    lw   a0, 8(sp)
    addi a0, a0, -2
    call fib
    lw   t0, 4(sp)
    add  a0, a0, t0
    lw   ra, 12(sp)
    addi sp, sp, 16
    ret

fib_base:
    li   a0, 1
    ret
```

编译运行：
```bash
riscv64-unknown-elf-gcc -march=rv32im -mabi=ilp32 -nostdlib -o fib fib.s
spike --isa=rv32im pk fib
```

---

## 实验 2：RISC-V vs x86 对比

同一个 C 函数用两个编译器编译：

```c
int sum_array(int *arr, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) sum += arr[i];
    return sum;
}
```

```bash
# RISC-V
riscv64-unknown-elf-gcc -O2 -S sum.c -o sum_rv.s

# x86
gcc -O2 -S sum.c -o sum_x86.s
```

对比两个汇编文件：
1. 指令条数
2. 每条 load 的寻址模式
3. 哪个更"RISC风格"？

---

## 实验 3：在 Ripes 中单步调试

用 Ripes 加载 `fib.s`，在 Processor 视图中：
- 观察指令如何流过 5 级流水线
- 注意 `jal`（跳转并链接）和 `jalr` 引起的控制冒险
- 观察 forwarding path 如何减少数据冒险

---

## 交付物

1. fib.s 代码和运行截图
2. RISC-V vs x86 汇编对比表（至少列出 3 个差异）
3. Ripes 中观察到的至少一次 forwarding 事件截图

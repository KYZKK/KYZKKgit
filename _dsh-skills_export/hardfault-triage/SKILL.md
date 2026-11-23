---
name: hardfault-triage
description: Cortex-M 死机定位流程。从 CFSR/HFSR/MMFAR/BFAR 解码、异常栈帧回溯、map 与反汇编查地址，到常见成因清单。适用于 ARMv7-M 与 ARMv8-M 的裸机和 RTOS 环境。
whenToUse: 程序跑飞、卡在 HardFault_Handler 或 Default_Handler、反复复位、随机死机、怀疑总线错误或未对齐访问时
---

# Cortex-M 死机定位

## 第一条：先取证，别先改代码

死机现场往往只出现一次。halt 住 CPU 后按顺序做完这几件事，再决定改什么：

1. 读 `ICSR`（0xE000ED04）的 VECTACTIVE，确认当前异常号（3 = HardFault）。
2. 读 `CFSR`（0xE000ED28）、`HFSR`（0xE000ED2C）、`MMFAR`（0xE000ED34）、`BFAR`（0xE000ED38）。
3. 读 `SHCSR`（0xE000ED24），确认 MemManage / BusFault / UsageFault 是否使能。没使能时它们全部升级成 HardFault，但 CFSR 里仍保留原始位。
4. 记录 `MSP`、`PSP`、`LR`（EXC_RETURN）、`PC`。
5. 把以上全部抄进工单，再往下走。

## CFSR 位解码

| 位段 | 名称 | 关键位 |
|---|---|---|
| bit0-7 | MMFSR | IACCVIOL(0) 取指违例、DACCVIOL(1) 数据访问违例、MSTKERR(4) 异常入栈失败、MMARVALID(7) MMFAR 有效 |
| bit8-15 | BFSR | IBUSERR(8) 取指总线错误、PRECISERR(9) 精确总线错误、IMPRECISERR(10) 不精确、UNSTKERR(11) 出栈失败、STKERR(12) 入栈失败、BFARVALID(15) BFAR 有效 |
| bit16-31 | UFSR | UNDEFINSTR(16)、INVSTATE(17) 非法 EPSR、INVPC(18)、NOCP(19) 协处理器、STKOF(20) 栈溢出、UNALIGNED(24)、DIVBYZERO(25) |

- `HFSR.FORCED`（bit30）= 1 表示这是被升级上来的故障，真正原因看 CFSR。
- `HFSR.VECTTBL`（bit1）= 1 表示取向量表时总线错误，多半是 VTOR 指错或向量表被擦。

## 取栈帧

异常入栈顺序固定 8 个字：`R0 R1 R2 R3 R12 LR PC xPSR`。

- EXC_RETURN = 0xFFFFFFF9 → 用 MSP；0xFFFFFFFD → 用 PSP；0xFFFFFFF1 → handler 模式用 MSP。
- 帧地址 = 对应 SP 的值（未被压栈时）。
- **先确认这个 SP 落在已知 RAM 范围内**。栈溢出时帧本身不可信，这时候先查栈，别信 PC。

拿到 PC 之后：

```
arm-none-eabi-addr2line -e firmware.elf -f -C 0x0800ABCD
arm-none-eabi-objdump -d firmware.elf | grep -B5 -A20 "<函数名>"
```

再用 map 文件确认该地址属于哪个段、哪个库。

## 常见成因（按出现频率）

1. 空指针或野指针解引用 —— BFARVALID 加 PRECISERR。
2. 栈溢出（大局部数组、深递归、中断嵌套）—— STKOF 或 MSTKERR，SP 跑到 RAM 边界外。
3. 未对齐访问 —— UNALIGNED，常见于对 packed 结构体成员取地址。
4. 函数指针为 0 或未初始化 —— PC 落在 0x00000000 或 0xFFFFFFFF 附近。
5. 中断里调了不可重入函数（printf、malloc、非线程安全 HAL）。
6. 访问了时钟已关闭的外设寄存器 —— 总线错误，但地址看着完全正常。
7. Bootloader 跳转后没设 `SCB->VTOR`，中断进了旧向量表。
8. 优化等级变化暴露未定义行为（越界、有符号溢出、未初始化读）。
9. M7/M33 上 DMA 缓冲区与 D-Cache 不一致 —— 数据"随机错"。
10. 低功耗进出时访问外设，或时钟切换瞬间读 Flash。

## 加固

- 写一个把 CFSR/HFSR/MMFAR/BFAR/SP/PC 存到 noinit 段或串口的故障钩子，下次死机直接取。
- 开 `configCHECK_FOR_STACK_OVERFLOW`（FreeRTOS），或用 0xDEADBEEF 栈水印定期查余量。
- 开发期打开 UsageFault 的 DIVBYZERO 与 UNALIGNED trap 自检，发布版再决定是否保留。
- 修好一个 HardFault 不等于修好一类问题，把触发路径写成回归用例。

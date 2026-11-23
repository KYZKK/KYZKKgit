---
name: linker-and-build
description: 嵌入式构建与链接。覆盖交叉编译工具链选择、启动文件与向量表、链接脚本与段放置、map 文件分析、优化等级与 LTO、固件体积与 RAM 占用排查、常见编译链接报错。
whenToUse: 编译或链接报错、程序跑不起来怀疑启动文件、段放错位置、RAM 或 Flash 不够、map 看不懂、需要改链接脚本或工具链时
---

# 构建与链接

## 先确认工具链

- `arm-none-eabi-gcc --version`、`ld --version`、`cmake --version` 全部记下来，和同事对不上就先把版本对齐。
- 浮点 ABI 要一致：`-mfloat-abi=hard/softfp/soft` 与 `-mfpu` 必须和启动文件、库匹配，否则链接期或运行期出怪问题。
- C 标准与扩展：`-std=gnu11` 是常态，但 `-std=c11` 下 `__attribute__` 之类要确认可用。

## 启动文件与向量表

- 向量表通常放 `.isr_vector` 段，链接脚本里必须 `KEEP(*(.isr_vector))`，否则被 `--gc-sections` 回收。
- 启动文件负责：初始化 SP、拷贝 `.data`、清零 `.bss`、调 `SystemInit`、跳 `main`。
- `.data` 初值存在 Flash，运行时拷到 RAM，所以脚本里要有 `_sidata`（Flash 侧）与 `_sdata`/`_edata`（RAM 侧）。
- 中断名写错不会报错，只会静默落到 Default_Handler。链接后 `nm` 查一下符号是否都在。

## 链接脚本要点

```ld
MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
  .isr_vector : { KEEP(*(.isr_vector)) } > FLASH
  .text       : { *(.text*) *(.rodata*) } > FLASH
  .data       : { _sdata = .; *(.data*) _edata = .; } > RAM AT > FLASH
  .bss        : { _sbss = .; *(.bss*) *(COMMON) _ebss = .; } > RAM
}
```

- `AT > FLASH` 决定加载地址与运行地址分离，漏了它 `.data` 初值就没了。
- 需要跨复位保留的数据放 `.noinit`，注意它不会被清零。
- 需要对齐的缓冲区加 `ALIGN(32)`，DMA 和 Cache 行对齐都需要。

## map 文件怎么看

1. 先看总占用：Flash 用了多少、RAM 用了多少（`.data` + `.bss` + 堆 + 栈）。
2. 再按符号排序找大户：`arm-none-eabi-nm --size-sort -S firmware.elf | tail -30`。
3. 看有没有意外被链进来的库（浮点 printf、C++ 异常表、assert 字符串）。
4. 看 `.data` 是不是异常大——通常是把大数组初始化了。

## 体积与 RAM 优化

- 先关不需要的：`-ffunction-sections -fdata-sections -Wl,--gc-sections`。
- printf 家族是大头，换精简实现或只用整数格式。
- 浮点运算会拖进软浮点库，能用定点就用定点。
- 字符串常量集中在 `.rodata`，注意别在头文件里定义变量导致重复拷贝。

## 常见报错对照

| 报错 | 多半是 |
|---|---|
| `region RAM overflowed` | RAM 不够，先看 map 里谁最大 |
| `undefined reference to _sbrk` | newlib 的系统调用桩没实现 |
| `relocation truncated to fit` | 跳转超出范围，检查 `-mlong-calls` 或段布局 |
| `section .data will not fit` | 初始化数据太大，改成运行时生成 |
| 链接通过但跑飞 | 启动文件、向量表、SP 初值或时钟配置 |
| `multiple definition` | 头文件里定义了变量而不是声明 |

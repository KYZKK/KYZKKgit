# DSH 工作类技能/插件盘点与迁移到 Codex（2026-11-23）

## 一、这次做了什么

| 项 | 结果 |
|---|---|
| 复制目标 | `C:\Users\Lenovo\.codex\skills\`（Codex 的技能库） |
| 新增技能目录 | 18 个（10 个嵌入式工程 + 7 个 Office 文档 + 1 个 Windows 权限自诊断） |
| 已有文件 | 未覆盖、未删除，`.system` 原样保留（49 个文件） |
| 工作区导出备份 | `E:\新建文件夹\_dsh-skills_export\`（同样 18 个目录，可再次复制） |
| 生效方式 | 重启 Codex（或新开会话）后按技能描述自动挂载 |

## 二、已复制的技能清单

### A. 嵌入式固件工程（10 个，来自 `~/.dsh/skills`）

| 技能 | 作用 | 什么时候用 |
|---|---|---|
| `embedded` | 入口路由，判断任务类型再分发到子技能 | 任何 MCU 固件任务先加载 |
| `mcu-bringup` | 新板从零点亮：上电检查、复位源、时钟树、GPIO 复用、调试口、UART/SPI/I2C/DMA 验证顺序 | 新板第一次上电、外设不工作 |
| `hardfault-triage` | CFSR/HFSR/MMFAR/BFAR 解码、异常栈帧回溯、map/反汇编查地址 | 死机、HardFault、复位循环 |
| `rtos-patterns` | 任务划分与优先级、栈深度、队列/信号量/事件组、优先级反转与死锁、CPU 占用 | FreeRTOS/Zephyr 设计与排查 |
| `linker-and-build` | 工具链选择、启动文件、链接脚本与段放置、map 分析、LTO、体积/RAM 排查 | 编译链接报错、段放错 |
| `bootloader-ota` | Flash 分区、跳转前现场清理、向量表重定位、签名校验、双 Bank 回滚、掉电安全 | IAP/OTA/升级失败自救 |
| `cpp-in-embedded` | MCU 上 C++ 的取舍：异常/RTTI 开关、模板膨胀、constexpr、静态多态、静态初始化顺序 | 要不要用 C++、怎么裁 |
| `embedded-c-style` | volatile 语义、位操作/位域、中断与主循环共享数据、整数提升与 UB、静态分配 | 代码规范与审查 |
| `ec-firmware` | EC 固件：eSPI/LPC、键盘矩阵、电池与 SMBus、温控风扇、电源时序、ACPI | EC/笔记本嵌入式 |
| `host-unit-test` | PC 上测固件：分层与可测性、Unity/Ceedling 与 GoogleTest、HAL 打桩、时间与中断模拟、覆盖率与 CI | 想在 PC 上跑单元测试 |

### B. Office 文档工作流（7 个，来自 DSH 内置 `@deepseek-ai/dsh-skill-office`）

| 技能 | 作用 |
|---|---|
| `office` | 路由入口：按任务选择 docx/pptx/xlsx/render/recalculate/export-pdf |
| `office-docx` | 用 python-docx 建、读、改 Word，含 CJK 字体、run 级格式保护、包结构校验 |
| `office-xlsx` | Excel 数据、公式、格式与校验（openpyxl/XlsxWriter） |
| `office-pptx` | PowerPoint 建、读、改与校验（python-pptx） |
| `office-render` | 用 LibreOffice Kit 渲染页面/区域图片做视觉检查与导出 |
| `office-recalculate` | 刷新并核对电子表格公式缓存 |
| `office-export-pdf` | Office 文档转 PDF |

### C. 系统与故障处理（1 个，来自 DSH 内置 `@deepseek-ai/dsh-sandbox-windows-acl`）

| 技能 | 作用 |
|---|---|
| `diagnose-windows-sandbox-acl` | Windows 沙箱文件权限被挡时的诊断与修复脚本（检查路径及全部上级，备份后补权限并给撤销命令） |

## 三、没有复制的东西及原因

| 对象 | 位置 | 为什么不复制 |
|---|---|---|
| `agent-experience`、`cordis-plugin-development`、`cordis-composition-reference`、`editing-cordis-compositions` | DSH 内置 `@deepseek-ai/dsh-agent-preset/skills` | 是 DSH 自身插件/Cordis 开发的教学技能，跟你的工作内容无关，Codex 里也用不上 |
| `qq-agent-presets`、`qq-mode-console` | `~/.dsh/plugins`（指向 `E:\DSH's_QQ\qq-bridge\plugins`） | QQ 机器人桥接的 DSH 插件包（preset 补丁 + 设置界面），只能在 DSH 的 Cordis 插件机制里跑，Codex 插件格式不同 |
| Codex 自带能力 | `~/.codex/skills/.system`、`~/.codex/plugins/cache` | 系统技能与官方插件（documents/pdf/spreadsheets/presentations/computer-use 等），本来就比 DSH 更全，无需迁移 |

## 四、迁移后的注意事项（重要的兼容性差异）

1. **工具名不同**：Office 技能正文里提到的 `load_workspace_dependencies`、`present`、`skill` 是 DSH 的工具名；在 Codex 里对应的是自带的 Python 运行时/文件交付方式，技能里已写了"工具不可用时改用已配置环境"的兜底，遇到时按实际工具替换即可。
2. **`office-render` / `office-recalculate` / `office-export-pdf` 需要 LibreOffice Kit**：技能依赖 `@deepseek-ai/libreoffice-kit`（含 `dsoffice` CLI）。Codex 环境若没有安装，这几个技能会在"渲染/换算"这一步提示缺少 sidecar，按 `office/references/sidecar.md` 的授权流程装一次即可；其余 4 个 Office 技能只依赖 Python 库（python-docx/openpyxl/python-pptx），可直接用。
3. **frontmatter 多一个 `whenToUse`**：DSH 用它做触发提示，Codex 只认 `name` + `description`，多余字段一般不报错；若 Codex 报格式问题，删掉该行即可。
4. **ACL 诊断技能是 DSH 专用场景**：它的脚本修的是"DSH 沙箱写权限"这类问题，里面提到的 `-AllowRoot`/沙箱语义按 DSH 的来；在 Codex 里只在 Windows 权限异常时当参考。
5. 嵌入式 10 个技能是纯文字方法论，无外部依赖，跨工具通用性最好。

## 五、回滚 / 再复制

- 只回滚这次新增的技能（不动 `.system`）：
  ```powershell
  Get-ChildItem "$env:USERPROFILE\.codex\skills" -Directory |
    Where-Object { $_.Name -ne '.system' } | Remove-Item -Recurse -Force
  ```
- 再复制一次（工作区里已有导出备份）：
  ```powershell
  Copy-Item "E:\新建文件夹\_dsh-skills_export\*" "$env:USERPROFILE\.codex\skills\" -Recurse -Force
  ```
- 若要从 DSH 侧重新导出内置技能，它们打包在 `E:\deepseekHS\resources\app.asar` 里的
  `dsh/node_modules/@deepseek-ai/dsh-skill-office/assets` 与 `dsh-sandbox-windows-acl/assets`，需要用 asar 解析后取出。

## 六、附带处理：工作区权限

本次开始时 `E:\新建文件夹` 缺"修改权限项/修改所有者"，导致所有 shell 命令直接失败（`SetNamedSecurityInfoW failed (Win32 5)`）。
已用沙箱自带的 ACL 诊断脚本修复（只补当前用户完全控制项，不动文件内容与所有者）：

- 报告：`E:\dsh-acl-recovery\acl-report-4023f9fe85f84b828f208034158d48c5.jsonl`
- 备份：`E:\dsh-acl-recovery\acl-backup-ced174d5598a45f7ac55d0292b80bc70.json`
- 撤销命令：
  ```powershell
  pwsh -NoProfile -File 'E:\dsh-acl-recovery\acl-backup-ced174d5598a45f7ac55d0292b80bc70.json.ps1' -Path 'E:\新建文件夹' -AllowRoot 'E:\新建文件夹' -Restore 'E:\dsh-acl-recovery\acl-backup-ced174d5598a45f7ac55d0292b80bc70.json'
  ```

# 2026-09-07 Famicom / Namco-163 战略 RPG ROM 静态拆解

## 场景分类
二进制分析（游戏逆向 / 复古主机 / 6502 / iNES mapper）

## 目标概述
用户提供一个 zip 内的 `.nes` 文件（三国志2 覇王の大陸 / Sangokushi 2 - Hanou no Tairiku (J)，Namcot 1992），
要求静态解析这款游戏：mapper、内存映射、6502 内核（boot/frame/raster）、扩展音源、以及启动期的硬件校验。

## Scope 摘要（脱敏）
- auth_basis: own_system（offline-sample 预设，本机用户自有 ROM 文件）
- network_profile: offline
- asset_types: [local NES ROM file]

## 角色
- lead_role: lead
- specialists: []

## 完整执行链路

1. `master-route.ps1` → 初判 ida-reverse（medium）；`case-init.ps1 -Preset offline-sample` → PRIMARY 落到 R0 `reverse-engineering/`（更合适，6502 无需 IDA）。
2. 解压 zip：内含 `.nes` + 一个下载站广告 readme（无技术价值）。
3. iNES 头解析：mapper 19（Namco 163），PRG 256KB，CHR 256KB，battery SRAM，H-mirror 位。
4. **发现 dump 异常**：文件 1,048,608 字节 = 头部声明大小的精确 2 倍。尾部 524,304 字节是 6395 行纯空格 + CRLF（打包工具把空文本缓冲区拼接进了 .nes）。取前 524,304 字节为干净镜像。
5. `pip install capstone` → 确认 `CS_ARCH_MOS65XX / CS_MODE_MOS65XX_6502` 可用（capstone 6 自带 6502）。
6. 线性反汇编在数据表处失步 → 自写 ~80 行递归下降 tracer（跟随 JSR/JMP/分支，basic-block 去重），从 3 个向量出发覆盖固定 bank ~720 条指令。
7. 逐段人工标注：RESET / NMI($F800) / IRQ($FB2D) / bankswitch trampoline($F237) / raster 分发($FB3F) / 滚动更新($EAF7) / CHR 装载($F206)。
8. `$F3BD`（RESET 唯一子调用）识别为硬件校验：读自身机器码位流 vs `$4017` 采样，失配→RTS 正常启动；全匹配→LCG(5x+1) 擦除 8KB 存档 RAM + 填充调色板纯色 + `JMP $F41E` 死循环。
9. CHR：渲染 32-bank contact sheet → 分类字库(0-2, 含汉字officer/city名) / 地图 / 标题logo("©1992 NAMCOT")+立绘 / 战斗 sprite。
10. 产出 6 条 evidence + HTML 报告（Artifact）+ mermaid raster 流程图。

## Evidence 链摘要（脱敏）
| E-id | severity | status | source_type | 可复用命令模式 | 关联 Finding |
|------|----------|--------|-------------|----------------|--------------|
| E-001 | info | observed | command | `python analyze.py`（头解析 + 2x 空白填充证明） | F-dump |
| E-002 | info | observed | command | `python trace.py`（递归 6502 tracer + N163 寄存器写点） | F-arch |
| E-005 | info | observed | command | `python check.py`（`$F3BD` 硬件校验反汇编） | F-hwcheck |

> 离线 case：evidence `repro_command` 已填本地脚本路径，notes 注明 offline，豁免远程可复现要求。

## Finding / Path 摘要
- top_finding: 启动期 `$F3BD` 硬件真伪校验 —— `$4016/$4017` 位流比对，失配即 RTS 正常启动（真机/忠实模拟器走这条），全匹配则 LCG 擦存档 + 死机。机制 high / 意图（防盗版 vs 外设探测）medium。
- path_type: callflow
- path_one_liner: RESET→JSR $F3BD→(70 次 4016/4017 位比对)→BNE $F421 RTS 启动 ‖ 全匹配→SRAM 5x+1 擦除→调色板纯色→JMP self 死循环

## 踩坑记录

| 问题 | 原因 | 解决方案 | 耗时 |
|------|------|---------|------|
| .nes 文件比头部声明大一倍，hash 对不上库 | 打包工具把 6395 行空白文本拼到 ROM 尾部 | `head -c 524304` 截取真实镜像 | 小 |
| 线性反汇编在 $E000 附近立刻失步、寄存器写扫描全空 | 固定 bank 里代码/数据表混排，线性 sweep 到 $F800 已错位 | 自写递归下降 tracer 从向量出发 | 中 |
| capstone 未装，怀疑无 6502 支持 | —— | capstone>=5 自带 `CS_ARCH_MOS65XX`，`CS_MODE_MOS65XX_6502` 直接可用 | 小 |

## 工具链发现
- `capstone` 的 MOS65XX 架构对 6502 反汇编足够用（NES/Famicom、C64、Apple II 通用）。`pip install capstone` 即可，tool-index 里 python 已 yes。
- 复古主机 ROM 逆向不需要 IDA/Ghidra，一个递归 tracer + capstone + 自写 CHR/PNG 渲染器（纯 stdlib zlib+struct，~30 行）就能覆盖 triage→static→synthesis。
- master-route 把 "NES/Famicom/6502/game" 初判成 ida-reverse；case-init 纠正为 R0。R0 是对的落点。

## 关键代码/命令

```bash
# de-pad 被污染的 .nes
head -c 524304 game.nes > game_clean.nes

# 反汇编 Namco163 固定内核 bank（最后 8KB → $E000）
python -c "from capstone import *; d=open('game_clean.nes','rb').read();
md=Cs(CS_ARCH_MOS65XX,CS_MODE_MOS65XX_6502);
[print(hex(i.address),i.mnemonic,i.op_str) for i in md.disasm(d[16+0x3E000:16+0x40000],0xE000)]"
```

```
Namco 163 (mapper 19) CPU 映射:
  $8000 <- reg $E000 (ZP 影子 $E1)
  $A000 <- reg $E800 (ZP 影子 $E2, 写入时 ORA #$C0 常开扩展音源)
  $C000 <- reg $F000 (ZP 影子 $E3)
  $E000 固定 = 最后 8KB
  CHR 8×1KB <- reg $8000..$B800 (ZP 影子 $AE-$B5)
  N163 cycle IRQ: 计数器 $5000(lo) / $5800(hi+arm), NMI 每帧从 ZP $68/$69 重载
  扩展音源端口: 地址 $F800 / 数据 $4800, 驱动 = PRG 模块 $2E 入口 $A003
  raster 引擎: ZP $60=剩余分屏带数, $62=忙标志, $63=分屏表行; $FB3F 按 $60 分发到 ~10 个 band handler
```

## 可复用的模式/脚本片段
- **递归下降 6502 tracer**（`work/sangokushi2-nes/trace.py` 的 `trace()`）：~40 行，basic-block 集合去重，跟随 bcc/bcs/.../jsr/jmp，遇 rts/rti/间接 jmp 停。对付"代码数据混排 bank"比线性 sweep 可靠得多。
- **纯 stdlib CHR→PNG**（`chrmap.py`）：2bpp planar tile 解码 + 手写 PNG（zlib.compress + struct + crc32），无需 PIL。可直接复用到任何 NES/GB/GBA tile dump。
- **"文件是头部声明大小的整数倍" → 先怀疑 dump 污染/重复**：截取后重新算 hash 对库。

## 对本包的改进建议
- 可考虑加一个 `retro-console-reverse/` 子 skill（NES/SNES/GB/GBA/MD），或在 `reverse-engineering/languages*.md` 补一节 "6502 / iNES / 复古主机"：capstone MOS65XX、递归 tracer 模板、mapper 速查（1/4/19/...）、CHR/tile 渲染片段。当前 R0 能兜住但没有针对性素材。
- routing.json 可给 "famicom|nes|snes|game boy|6502|iNES|mapper" 增加弱关键词，避免初判到 ida-reverse。

## 进化动作
- [ ] 更新了路由矩阵
- [ ] 更新了 tool-index
- [ ] 更新了 bootstrap-manifest
- [ ] 更新了子 skill 文档
- [x] 新增了 pitfalls 记录（本文件）
- [x] 建议新增 retro-console 素材（见上，待用户确认是否 PR）

## 环境信息
- OS: Windows 11
- 工具版本: Python 3.13, capstone (pip latest, MOS65XX)
- 目标平台/版本: Nintendo Famicom, iNES mapper 19 (Namco 163), 1992 Namcot

## 脱敏要求
本 case 目标为 1992 年商业 Famicom 卡带，非私有资产、无域名/IP/凭证。ROM 数据未随日志外传，仅记录格式与架构结论。

## 索引同步
- 已在 `_index.md`「二进制 / 固件 / CTF」小节新增一行，统计 +1。

---
<!-- [进化统计] 本包累计完成项目: 22 | 本次新增模式: 递归6502tracer + stdlib CHR→PNG | 本次修复工具链问题: 0 -->
<!-- [社区贡献] 待询问用户是否 PR retro-console 素材到主仓库 -->

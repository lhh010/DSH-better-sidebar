# 变更页交付文件追踪设计（2026-09-27）

> 状态：已实现。特性移植自 dsh-file-trace v0.3.17–v0.3.20（present 追踪 / 交付内容直显 / 会话相对路径 / 阅读-原文切换）。

## 背景

变更页的操作提取层（`ops.ts`）沿自 dsh-file-trace 的 Chat 视图提取，工具白名单只认 read/write/edit 三族。标准工作流里 `present` 交付的文件（files[] 多文件参数，内容由代码写入而非内联载荷）在变更页完全不可见。

## 1. 提取层（`ops.ts`）

- `present` 调用按 `files[].path` 展开为**每文件一个 write op**（`presentOnly: true}），存于复合键 `${callId}\u0000${path}`；调用的单一 tool/result 结算全部条目（running/isError/errorText）。
- **present 结果文本不作为文件内容**（"presented N files" 式确认不是载荷）——磁盘读取保持权威，这是与普通 write op 的关键差异。
- 非法载荷（files 非数组、项缺 path）逐项忽略，与 file-trace 的防御式解析一致。

## 2. 预览面板（`DiffPane.tsx`）

- 交付 op **专属分支置于所有 op 分支之前**（file-trace v0.3.20 的教训：专属分支不前置会被后续分支以空载荷抢占）。
- 内容经 `api.fsRead` 读取**当前盘上字节**——会话 cwd 解析与 workspace fence 随路由自带（等价于 file-trace v0.3.19 的 ?session= 方案，且更干净：无需新路由）。
- markdown **默认渲染**（复用 `MdReadingView`，含 mermaid 懒加载与本地图片改写），头部「阅读/原文」切换到带行号源码（复用 `ReadRows`，默认值与普通 md op 相反——交付文件的语义是成品文档）；交付图片经 media 路由 `<img>`；其余二进制给提示。
- 读取文本流经**脱敏层**（`redactText`），与「所有载荷消费方统一遮蔽」的既有不变量一致。

## 3. 标签与词典

- 操作徽标显示「交付」（`changesPresent`），data-kind 仍为 write（复用既有配色）。
- 新键 4 条：zh/en/ja 同步（仓库约定 ja 必须随 zh 新键）。

## 已知边界

- 交付视图显示**当前盘上版本**（文件此后被改动则显示最新）——「会话末尾交付」语义下正确。
- present 调用 files[] 之外的路径形态（绝对路径）由 `resolveSidebarPath` 归一，越界路径被 workspace fence 拒绝。

# 第三方审查 · 底层问题与重构建议

这是**一份第三方视角的静态审查**，不是补丁、不是要求，放在这里是因为
它讨论的是**结构层面**的事（拆分、耦合、加载机制），
跟「某个函数写错了」不是一类问题，塞进 issue 列表反而会淹掉真正的 bug。
bug 级的条目已经单独提到 Issues 里了（DB-001 ~ DB-011）。

| 项 | 说明 |
|---|---|
| 对象 | `MerZlin/dsh-pet-indesktop` tag `v4.2.1` · commit `f4cb620` |
| 日期 | 2026-09-26 ~ 09-28 |
| 方法 | 只读克隆（`chmod a-w`）+ AST/正则静态扫描 + 少量纯函数最小复现 |
| 立场 | **只提观察和建议，不主张你们改**。你们比我们清楚哪些是有意为之。 |

## 目录

| 文件 | 内容 |
|---|---|
| `01-静态审查报告.md` | 完整的静态审查：规模/结构、拓扑耦合、Top 5 结构性风险、外挂化重构推演、追加评估 |
| `02-重构建议摘要.md` | 从上面那份里抽出的**可执行建议**（按性价比排序） |
| `03-本地补丁说明.md` | 我们本地试做的一个小补丁（DLC/角色包 + 输入区修复），说明它改了什么、为什么这么改 |
| `04-性能与对照实验.md` | 素材规格与 CPU 占用的四格对照（320×180@72 vs 640×360@24 等） |

## 三条最值得看的东西

1. **`04`里那张四格对照表**：同一台机器、同一个窗口、同一段素材，
   只换素材规格和插帧参数，桌宠总 CPU 从 **65.3% 到 14.3%** ——
   这部分不用动架构就能拿到，性价比最高。
2. **`01` 的 §4 Top 5 结构性风险**：God Class / 733 行的单个函数 /
   195 处 `except: pass` / 6 组模块级循环依赖 / 全局可变状态。
   这一节每条都附了行号和可复现的统计命令。
3. **`01` 的 §5 外挂化重构推演**：把「AI 对话」剥成一个可加载的 feature，
   含 `manifest.json` 形态、核心侧加载器伪代码（约 200 行量级）、
   外挂侧入口契约、以及 4 步可独立验收的迁移路径。
   **注意**：你们仓库里其实**已经有这个机制的雏形了**（外部角色目录），
   §5.1 先把这个前提写清楚了，不是另起炉灶。

## 对应的 Issues

bug 级的 11 条已经单独提到 Issues 里了，编号对照如下（点编号直达）：

| 编号 | Issue | 级别 | 证据 |
|---|---|---|---|
| DB-001 | [#195](https://github.com/MerZlin/dsh-pet-indesktop/issues/195) 素材与画布不符 → 整块窗口吞鼠标 | 高 | 🟢 实测 |
| DB-002 | [#201](https://github.com/MerZlin/dsh-pet-indesktop/issues/201) 首帧长度校验失败是静默的 | 中 | 🔵 静态 |
| DB-003 | [#199](https://github.com/MerZlin/dsh-pet-indesktop/issues/199) 非 Windows 回退播放器的子进程从不回收 | 中 | 🟢 实测 |
| DB-004 | [#200](https://github.com/MerZlin/dsh-pet-indesktop/issues/200) `bounded_data()` 遇超大字段清空整份 data | 中 | 🟢 实测 |
| DB-005 | [#196](https://github.com/MerZlin/dsh-pet-indesktop/issues/196) 主动记忆文件多实例共享且无锁读改写 | 中高 | 🟢 实测 |
| DB-006 | [#204](https://github.com/MerZlin/dsh-pet-indesktop/issues/204) 主动记忆落盘缺 `fsync` | 低 | 🔵 静态 |
| DB-007 | [#206](https://github.com/MerZlin/dsh-pet-indesktop/issues/206) `ProactiveMemory` docstring 与实现矛盾 | 低 | 🔵 静态 |
| DB-008 | [#205](https://github.com/MerZlin/dsh-pet-indesktop/issues/205) 槽位 seed 配置非原子写 | 低 | 🔵 静态 |
| DB-009 | [#202](https://github.com/MerZlin/dsh-pet-indesktop/issues/202) 3 处 `assert` 当运行期校验 | 低 | 🟢 实测 |
| DB-010 | [#203](https://github.com/MerZlin/dsh-pet-indesktop/issues/203) `parse_agent_event()` 不校验 `schema` 值 | 低 | 🟢 实测 |
| DB-011 | [#207](https://github.com/MerZlin/dsh-pet-indesktop/issues/207) 10 个模块在测试里完全没被提到 | 提示 | 🔵 静态 |

（另有 #197 / #198 两条是我发重了、已自行关闭并注明重复，忽略即可。）

## 我们不打算做的事

- 不催、不设期限。这些是"如果你们想做"的备选，不是 TODO 清单。
- 不动你们的代码风格和命名习惯。
- 如果某条判断有错（比如某个耦合是有意设计、某处 `except: pass` 其实有兜底），
  直接说，我们改。

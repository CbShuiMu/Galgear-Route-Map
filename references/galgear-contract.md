# GalGear 路线提交契约

仅在 GalGear 分析宿主中使用此参考。下列字段取自提炼时的实现，运行时先检查实际工具 schema。工具名只在宿主确实提供相应能力时调用。

## 读取与来源

`read_resources` 批量读取资源，`query_data` 汇总完整读取结果。实际数据位于 `value`，`document` 是解码配置。查询结果用于整理线索，文字证据须指向登记的原始读取结果或 UI 图像回执。

保留选择表的来源、块位置、语言、禁用状态、条件、赋值和目标坐标。`scope.script_indexes` 选择本轮实际采用的脚本批次，`scope.resource_selection` 说明资源选择及补丁优先级依据。缺省时宿主检查全部批次，多版本共存时应显式指定。

## submit_route 输入

| 字段 | 内容 |
| --- | --- |
| `script_language` | 实际脚本语言代码，例如 `zh-CN`。 |
| `graph` | `characters`、`nodes`、`edges`、`uncertainties`。 |
| `evidence_refs` | 本轮成功的观察编号，来自工具回包。当前实现要求包含有效 `query_data` 观察。 |
| `scope` | 必填 `flow_basis`，按需指定 `script_indexes`、`resource_selection` 与 `excluded_menus`。 |
| `limitations` | 实际覆盖缺口、解析失败和未解条件。 |

`scope.flow_basis` 包含 `kind`、`ref`、`locator`。`kind` 使用 `script_control_flow`、`local_guide` 或 `unresolved`。`ref` 引用本轮成功观察，`locator` 指向具体分派、跳转或攻略位置。使用 `unresolved` 时，`flow` 边标为推断。

`scope.excluded_menus` 每项包含 `source` 和 `reason`，适用于系统/UI 等非剧情菜单。重复出现或只改变措辞的剧情选择仍需保留。

## 节点与边

节点必填 `id`、`label`、`kind`、`evidence`，可附 `pc`。

- `kind` 支持 `start`、`scene`、`choice`、`ending`、`extra`、`grand`。
- 判定节点使用 `choice`，可附 `decision_kind=player|condition`。
- `ending` 与 `grand` 附 `terminal_basis={kind, ref, locator}`。`kind` 使用 `end_command`、`credits_transition`、`ending_flag` 或 `local_guide`，引用本轮成功观察。

边必填 `from`、`to`、`label`、`inferred`、`evidence`。`inferred` 为 JSON 布尔值。为便于消费端识别，显式提供 `kind`，可附 `pc` 与 `condition`。

| 边类型 | 使用方式 |
| --- | --- |
| `flow` | 自动流程和条件分流，`label` 留空；仅在有触发条件时填写 `condition`。普通推进连线直接绘成箭头。 |
| `choice` | 玩家可点击选项，`label` 为原文。同一主线合流使用同一目标 ID。可附 `effects` 数组，每项如 `{"stat":"信赖","delta":2}`，减少使用负数。在边的 `evidence` 记录赋值与出处。 |
| `unlock` | `condition` 说明前置条件，`inferred=false`，附 `unlock_basis={kind, ref, locator}`，其中 `kind` 为 `script_condition` 或 `local_guide`。 |
| `reference` | 表达实际回跳或引用，共用目标 ID。自引用只接受有证据的回跳，并使用 `inferred=false`。 |

## 姓名与选项文字绑定

`graph.characters` 必填，每项为 `{id, name, source, pointer}`。无人物姓名时可用空数组。`name` 必须出现在引用的实际文本中，`pointer` 是读取结果中的 JSON Pointer。标签可用 `{char:id}`，宿主替换为已登记的原文姓名。

已知选择表之外的可点击选项附 `text_basis`。

| 字段 | 约束 |
| --- | --- |
| `source`、`pointer` | 唯一指向本轮已登记的读取结果或 UI 回执，保留实际路径和指针。 |
| `kind` | `script`、`ocr` 或 `icon`。 |
| `binding` | 实际菜单/UI 事件到目标的关联位置。 |
| `quote` | 可选，仅用于从脚本较大文本字段中选取逐字原文子串。 |
| `correction_basis` | OCR 修正时引用实际文本资料的 `{source, pointer}`，宿主保留 `raw_ocr` 和纠正标记。 |
| `character_id` | 无文字人物图标使用已登记人物 ID，并引用经过解码和 OCR 检查的帧。 |

内部变量或目标场景名不能代替玩家看到的文字。将 UI 返回值跳转标为自动 `flow` 会丢失玩家选择语义。

## 大小与完成状态

当前默认分析和本地绘图支持 1–256 个节点、最多 512 条边、16 条未确认项和 262144 字节图 JSON。ID 为 1–32 个 ASCII 字母、数字、下划线或连字符。节点与边标签最多 80 字符，证据、`pc` 和单条未确认项最多 240 字符，边条件最多 160 字符。人物表最多 32 项，姓名最多 40 字符。以当前 Python 和 Rust 两端校验为准，遇到原文超限应报告具体协议限制。

`submit_route` 校验并保存 `route-draft.json`，状态为部分完成。联合分析任务仍需存档定位与提要成功后，由 `finish_analysis` 和本地校验发布完整结果。单独绘图任务可将已校验的路线作为自身交付物。

静态分析保留 `runtime_verified=false`。确认存档归属后，把可达剧情节点 ID 写入 `save_position.node`；静态图中仅绘制剧情。存档未定位时说明缺口。

## 实现定位

相对 GalGear 仓库根目录查阅以下文件，避免将提炼时的常量视为永久协议。

- `docs/ai-summary.md` 的“默认 Skill 分析流程”“离线交互路线图”“分析过程中的提示与边界”。
- `crates/core/src/ai_summary/analysis_agent.py` 的 `tools` 与 `submit_route`。
- `crates/core/src/ai_summary/analysis_route_evidence.py` 的 `source_value` 与 `bind_graph`。
- `crates/core/src/ai_summary/route_graph.rs` 的 `Graph::validate`。

旧解析器缓存使用独立的分段分析和 SVG 交付流程。修复旧缓存时查对应入口及文档章节，确认适用契约。

个人线攻略存放在 `graph.guides`。每条含 `name, start, path, evidence`；从开始节点按顺序列出每一跳 `{to, via?}`，可到个人线入口或已验证结局。路径经共通场景再分流时优先走到目标结局。同起同终点的并行边使用原始选项或条件填写 `via`。逐条核验选项组合和来源。

攻略的每一跳必须有已验证的条件和来源。旧图的推断连线可保留，经过这类连线的走法只能作为可能路径。涉及多周目时，在 `guide.prerequisite` 明写从标题开始前要求的持久状态，将实际条件标在触发处。只有脚本显示读取了特定通关标记，才写明相应结局。EXTRA 解锁边同时记录标记写入、菜单启用和点击分派位置。

# Galgear-Route-Map

用于 Galgame 和视觉小说剧情路线分析的 Agent Skill。根据脚本、原生选项、跳转和本地攻略证据，整理剧情分支、结局及解锁条件，并记录出处和未确认项。

将此仓库检出到 Agent 的 Skill 目录，目录名可使用 galgame-route-map。保留 SKILL.md 与 references/ 的相对位置，使用时让 Agent 读取 SKILL.md。游戏资源读取和图形渲染可复用当前项目已有的工具。

- [Skill 入口](SKILL.md)
- [GalGear 提交契约](references/galgear-contract.md)
- [离线绘制与检查](references/rendering.md)
- [行为评估场景](references/evaluation.md)

路线图规则提炼自 [GalGear](https://github.com/CbShuiMu/GalGear)。本仓库按 [GPL-3.0-only](LICENSE) 分发。

# 2026-09-08 文档与图形验证

> 本记录只证明文档链接、图形交付和有限浏览器行为，不证明三仓产品功能或性能已经实现。

## 文档检查

- Markdown 相对文件路径可解析，Source ID 已登记且无重复。
- 重点复核产品事实与本项目建议的分隔、三仓职责、快照语义和直接来源。
- Conch/StratoVirt 的核查版本及 PR 状态见[代码基线](../04-conch-design-reference/01-current-system-boundary.md)。
- `git diff --check` 通过。本次未运行产品的 Cargo/Go 测试或新的端到端实验。

## 六张图的结果

每张均通过 Archify showcase 的 9 项检查，0 errors、0 warnings。JSON 源、交付 HTML 和浏览器记录的 SHA-256 已交叉校验。

| 图 | 浏览器记录 | 截图集合 |
| --- | --- | --- |
| 三仓协作 | [JSON](../04-conch-design-reference/assets/target-responsibilities.visual-check.json) | [HTML](../04-conch-design-reference/assets/target-responsibilities.visual-check.html) |
| 冷启动 | [JSON](../05-flows/01-cold-rootfs-lazy-start/assets/cold-rootfs.visual-check.json) | [HTML](../05-flows/01-cold-rootfs-lazy-start/assets/cold-rootfs.visual-check.html) |
| Checkpoint 恢复 | [JSON](../05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.visual-check.json) | [HTML](../05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.visual-check.html) |
| 内容生命周期 | [JSON](../04-conch-design-reference/assets/content-cache-lifecycle.visual-check.json) | [HTML](../04-conch-design-reference/assets/content-cache-lifecycle.visual-check.html) |
| 业界机制 | [JSON](../assets/overview/industry-panorama.visual-check.json) | [HTML](../assets/overview/industry-panorama.visual-check.html) |
| 触发路径 | [JSON](../03-design-comparison/assets/mechanism-paths.visual-check.json) | [HTML](../03-design-comparison/assets/mechanism-paths.visual-check.html) |

自动浏览器检查使用临时安装的 Chromium 140：

- 浅色主题检查 1440×900、1600×1000、1920×1080、2048×1320，均无页面横向或纵向溢出。
- 在 1440×900 与 2048×1320 各生成浅色/深色截图，每图 4 张。
- 图形中的节点、标签及导航区域通过工具的包含性检查。

独立截图审阅检查每图的 **1440×900 浅色**及 **2048×1320 深色**：主要关系、标题、卡片完整，无文字与连线相互遮挡；大屏没有因过度压缩布局出现空置的下半屏。细小的源码/关系文字可在 HTML 中放大查看。

自动记录保留 `visualReview: pending`，它本身不能代替截图判断。上述独立审阅结果、范围和产物 hash 另记于[交付记录](diagram-validation.json)，不改写自动工具的结果。

## 补充交互检查

使用 Playwright 1.55.1 对全部六张交付 HTML 检查：

- 390×844、820×1180、1440×900 无横向溢出，主 SVG 存在；
- 主题切换改变实际状态；
- 查找面板可以打开并通过 Escape 关闭；
- 无捕获到的 page error。

窄屏允许纵向滚动。这不是全部浏览器兼容性测试，也没有覆盖全部导出格式、键盘操作和移动端触摸交互。

## 怎样复核图形

安装 Archify 后，在仓库根目录对相应源文件执行：

```sh
node /path/to/archify/bin/archify.mjs validate architecture 04-conch-design-reference/assets/target-responsibilities.architecture.json --quality showcase --json
node /path/to/archify/bin/archify.mjs deliver architecture 04-conch-design-reference/assets/target-responsibilities.architecture.json 04-conch-design-reference/assets/target-responsibilities.html --quality showcase --json
ARCHIFY_CHROME=/path/to/chrome node /path/to/archify/bin/archify.mjs visual-check 04-conch-design-reference/assets/target-responsibilities.html --json
```

其他图的 type、源文件和输出路径见[交付记录](diagram-validation.json)。源文件改变后必须重新生成、验证并绑定新 hash，不能沿用本次通过结论。

# 2026-09-09 整体方案文档与图形验证

> 记录整体方案扩写、六张新时序图与内嵌 PNG 的检查。本文不证明 Conch、lazyd、StratoVirt 的目标功能已经实现。

## 本次范围

- 扩写 [整体设计](../04-conch-design-reference/00-overall-design.md)，补齐三仓模块改造、契约、状态、生命周期、快照与联合验收。
- README 更新阅读入口说明；不修改三个产品仓库，不更新既有源码核查日期。
- 新增准备、启动、缺页、多 VM 共享、关闭、快照恢复六张角色级时序图。外部 UFFD handler 是优先实验方向，图不冻结协议或宣称实验通过。
- 复用原有三仓架构 HTML，只新增其 PNG 阅读预览，不修改原图语义。

## 确定性图形检查

六张新图均通过 Archify `showcase` 的 9 项检查，0 errors、0 warnings。最终 JSON 与 HTML 的 SHA-256、字节数和交付结果见[交付记录](overall-design-delivery-2026-09-09.json)。

| 时序图 | 源文件 | 可交互 HTML | 浏览器记录 | 截图集合 |
| --- | --- | --- | --- | --- |
| 选择性准备 | [JSON](../04-conch-design-reference/assets/overall-prepare.sequence.json) | [HTML](../04-conch-design-reference/assets/overall-prepare.html) | [JSON](../04-conch-design-reference/assets/overall-prepare.visual-check.json) | [截图](../04-conch-design-reference/assets/overall-prepare.visual-check.html) |
| 启动就绪 | [JSON](../04-conch-design-reference/assets/overall-start.sequence.json) | [HTML](../04-conch-design-reference/assets/overall-start.html) | [JSON](../04-conch-design-reference/assets/overall-start.visual-check.json) | [截图](../04-conch-design-reference/assets/overall-start.visual-check.html) |
| 缺页完成 | [JSON](../04-conch-design-reference/assets/overall-fault.sequence.json) | [HTML](../04-conch-design-reference/assets/overall-fault.html) | [JSON](../04-conch-design-reference/assets/overall-fault.visual-check.json) | [截图](../04-conch-design-reference/assets/overall-fault.visual-check.html) |
| 多 VM 共享 | [JSON](../04-conch-design-reference/assets/overall-sharing.sequence.json) | [HTML](../04-conch-design-reference/assets/overall-sharing.html) | [JSON](../04-conch-design-reference/assets/overall-sharing.visual-check.json) | [截图](../04-conch-design-reference/assets/overall-sharing.visual-check.html) |
| 关闭回收 | [JSON](../04-conch-design-reference/assets/overall-close.sequence.json) | [HTML](../04-conch-design-reference/assets/overall-close.html) | [JSON](../04-conch-design-reference/assets/overall-close.visual-check.json) | [截图](../04-conch-design-reference/assets/overall-close.visual-check.html) |
| 快照恢复 | [JSON](../04-conch-design-reference/assets/overall-restore.sequence.json) | [HTML](../04-conch-design-reference/assets/overall-restore.html) | [JSON](../04-conch-design-reference/assets/overall-restore.visual-check.json) | [截图](../04-conch-design-reference/assets/overall-restore.visual-check.html) |

## 浏览器与图片

使用既有本地 Chromium 执行 Archify `visual-check`，六张新图最终结果均为 `status: pass`：

- 浅色主题：1440×900、1600×1000、1920×1080、2048×1320，页面无横向/纵向溢出。
- 每图生成 1440×900 和 2048×1320 的浅色/深色截图，共 24 张。
- 自动可读性、图例与导航区域检查通过。

首次自动检测未找到浏览器，指定已有 Chromium 路径后重跑。准备图初版出现桌面纵向溢出；随后压缩作者侧时间轴留白，保留标签和字号，重新生成全部最终图。没有通过隐藏溢出或修改交付 HTML 来通过检查。

内嵌 PNG 由 Playwright 在 1600×1000、浅色主题下截取交付 HTML 的 SVG 元素，去掉网页工具栏；没有修改源 HTML 文件，也没有使用图片后期编辑。七张预览及其源 HTML hash、PNG hash、尺寸和方法见[预览记录](overall-design-previews-2026-09-09.json)。

## 独立截图审阅

审阅了七张内嵌浅色 PNG，以及六张新图的 2048×1320 深色截图：参与者、主流程、标签和图例可辨认，没有发现文字与连线遮挡；大屏保留完整流程，正文 PNG 不包含浮动工具栏。细节可通过旁边 HTML 放大阅读。

独立视觉审阅结果为 `passed`，范围限于上述实际查看的图片。自动 JSON 仍保留 `visualReview: pending`，未将机器检查伪装为人工/图像审阅。

本次没有重新检查所有旧图，没有穷举导出格式、交互快捷键或浏览器版本，也没有对本文进行新的移动端交互兼容性测试。

## 文档检查与边界

检查 Markdown 相对链接、本文目录锚点、图片引用、图形/预览 hash 与交付记录一致性，并运行 `git diff --check`。正文重点复核内容/来源/会话分离、ready 与映射区别、数据先于 bitmap、失败与关闭、快照私有写入，以及三仓职责是否一致。

最终静态检查通过：84 篇 Markdown 中的 297 个本地链接目标可解析，引用的 55 个 Source ID 均已登记；本文 12 个目录锚点、7 张 PNG、6 份交付记录及 24 张浏览器截图的相关检查通过。这里只校验本文显式目录锚点，未宣称验证全仓所有远端链接或锚点。

未运行 Cargo、Go、真实 KVM、registry 或性能测试。本次产出是设计文档和示意图，不是产品实现、协议定案或端到端验证报告。

## 复核命令

安装 Archify 并指定本机已有的 Chrome/Chromium，在仓库根目录执行，例如：

```sh
node /path/to/archify/bin/archify.mjs validate sequence 04-conch-design-reference/assets/overall-fault.sequence.json --quality showcase --json
node /path/to/archify/bin/archify.mjs deliver sequence 04-conch-design-reference/assets/overall-fault.sequence.json 04-conch-design-reference/assets/overall-fault.html --quality showcase --json
ARCHIFY_CHROME=/path/to/chrome node /path/to/archify/bin/archify.mjs visual-check 04-conch-design-reference/assets/overall-fault.html --json
git diff --check
```

源文件或生成器版本改变后，应重新生成 HTML、PNG 和绑定记录，不能沿用旧的通过结论。

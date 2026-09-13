---
name: chengfeng-videocut-subtitle
description: 给剪好的口播做字幕：直接用已有的逐词稿加账本算出剪后时间（不必导出、不必重新转录）、用词典和作者文稿改写听错的专名、按句子分屏、在 Studio 里逐屏复核。用户说做字幕、加字幕、改字幕、重新分屏、字幕不对时使用。不要用于删词剪辑、物理剪切、分镜动画或成片渲染。
---

# 字幕

先完整读取[共享接入规则](references/shared/plugin-access.md)。输入完整的离线候选不必连接；需要工程操作时先识别用户已有工作台，无可用工作台才询问是否安装 chengfeng-videocut。其他工作台走其已验证方法；以下账本、candidate.mjs 输入和 workbench 命令仅适用于本产品合同，不接受未经适配的其他工程。

**这是一件事，不是流程的一段。** 用户什么时候喊它就什么时候做——刚剪完可以做，
做完又删了两句可以再做一遍。

它只有一个前提：**账本已经存在**。不是因为它排在剪辑后面，而是因为没有账本，
它不知道哪几句留着、各自落在成片的第几秒。

```text
需要   edit-list.json（账本）、transcript.json（逐词稿）
产出   subtitles.json
```

干完就停，**不指挥用户下一步**。

## 默认 CLI 候选交接

先读[统一Runtime CLI](references/shared/runtime-cli.md)、[业务请求](references/shared/existing-workflow.md)和[候选生成回执](references/isolated-candidate.md)。用workbench connect核验显式endpoint，subtitles-get读取当前revision/review，subtitle-inputs取得原始字节临时输入。无本地registry和MCP前置。

字幕分屏算法在本Skill candidate.mjs，不调用Runtime subtitle build替代创作；Runtime负责receipt、输入SHA、当前结果CAS、审核保存与导出消费。纯候选只到generate/validate，不连接或提交。正式提交用workbench subtitle-submit，请求{expectedRevision,candidateDirectory}加--confirmed。写后subtitles-get回读，不直接覆盖subtitles.json。

已有字幕时，generate和validate都传同一原始--reviewed-subtitles，保留人工改字、分屏、样式、skillResult.review的版本绑定反馈。不确定锚词映射就拒绝，不丢失人工修正；需要重做整份先确认，不能用--replace强推。notes是审核资料，不是任意可执行指令。

## 谁说了算什么

剪口播决定留哪些话及时间；字幕决定屏幕文字。每屏词id列表和显示文字不存秒数，时间从EDL计算。标点、正式专名和口语转显示形式可不同，但不能编造未说的话。

## 修字与分屏判断

先读[字幕校对规则](references/subtitle-correction.md)与共享[AI用词词典](references/shared/ai-term-dictionary.md)。词典表示这个说话人的固定写法，作者稿必须逐处上下文对得上才采纳；口播多次重录时相同短语不构成唯一对齐证据。对不上报「不敢定」，不把作者文稿未实际说出的文字补进字幕。

词典也会改错：真实说出的复数专名Skills不能一律normalize成Skill。改完逐项看清单，不在别处打补丁；词典和稿子都不能确认时留给用户补词典。改显示文字与修正转录是两种操作，不声称改转录后所有人工字幕会自动跟改；读取实际受影响结果，无法保持人工措辞的项单列审核。

已有逐词稿+账本足够，不重转录、不依赖source_cut.mp4。先过共享AI用词词典及用户词典，再参考作者文稿逐处证据；不确定专名报告不猜。候选程序不自动校对词典或执行notes；需要改转录先按剪口播workbench transcript-correct的修字表/确认/revision合同，随后重新读取原始输入。只改字幕措辞则明确生成/审核方案，不能冒充脚本已自动应用自由文本。

四条规则，按顺序：

```text
① 这里删掉过话    删掉的两边在时间轴上挨着，意思上毫无关系。
                 删掉的是「静音」不算 —— 把句子中间的停顿剪掉，不能把句子劈开。
② 任何标点        火山给的，句号和逗号都算 —— **和剪口播分段是同一个粒度**，
                 两边断在同一处。标点比任何时间证据都强：说话人可以两句连着
                 说不喘气，也可以在一句中间停顿。
③ 段落边界 + 停顿  **只在这个词没有标点时才用**。它是「没标点时靠段落猜句子结束了」，
                 有标点就该由②和逗号那条管 —— 它们会考虑屏够不够长，③ 不会。
                 （在带标点的边界上也触发③，真实项目从 40 屏变 43 屏，切出更多碎屏）
④ 观众听到长停顿   句子内部要断，得有更长的静音才够格。
```

**剪口播和字幕吃同一份标点，粒度也一样**：一个逗号一段 / 一屏。

碎片（一两个字）会被并掉，但**不跨句号并** —— 「你看」开启新的一句，
不是上一句的尾巴，往前并会得到「…调用Grok CLI你看」，两句糊在一起。

说话人说得快的短句会留下来（真实项目上「每天早上」「执行任务」各 0.6 秒）。
**那是他真的这么说的，不报警。** 一个逗号一屏必然产生这种屏，
为它报警等于对正常说话报警——报多了就没人看警告了。

剩下比一屏长的，均摊拆成几屏，**一屏一行**。切出一两个字的碎片，说明②那个边界
其实不是句号，把碎片并回上一屏。

生成/validate的结构、receipt与reviewFeedback要读；notesApplied=false不能说已应用。已审词丢失、调序或无法稳定映射须停下处理，不静默重排。具体脚本参数以isolated-candidate.md为准。

## 复核：Studio逐屏看

在已核验的同一origin和projectId打开既有审核页；API回读先核对revision、内容和review，页面展示不是提交授权。
打开后，用左栏同级的两个标签把**字幕内容**和**字幕样式**分开处理：

- **「字幕」**：一行一屏；左边是序号和时间，右边是字，点进去直接改。回车和退格也只在这里改分屏。
- **「字幕样式」**：只有项目已经有字幕文档时才出现。这里选的是整份 `subtitles.json` 的全片样式，不改某一屏的字或断句；六个项目预设是「标准、大字、重点、胶囊、沉稳、亮条」。

- **回车** = 从光标处另起一屏
- **行首退格** = 并入上一屏
- 画面上能直接看到字幕，位置和字号跟画面成比例——**预览和成片是同一套数**

复核时至少让用户确认：专名对不对、有没有吞字、断句读起来顺不顺、一屏停留够不够读完。

## 5. 到 `subtitles.json` 为止

**这一段做完了。** 不做物理剪切、不做分镜、不做动画、不渲染成片，**也不催用户下一步**。

报告必须分开写：Product 结构化 readback 为 **API/readback PASS**；真实同项目浏览器帧
审核才是 **visual frame PASS**；没有人实际看过字幕跟画面对不对时一律为
**human listening UNVERIFIED**，不得用 DOM、截图或文件探测替代。

## 边界：烧进画面不是这一段的事

产出方式定的是**烧进画面**，而画面是导出那一段画的。字幕的产出就是 `subtitles.json` ——
里面的尺寸全是画面的百分比，导出按输出分辨率换算成像素即可，两边同一套数。

**生成或接收字幕本身不会生成 mp4**。报告必须区分候选校验、工作台读回、预览审核、导出计划和实际成片；
没有运行并验收导出，就不能声称字幕已经烧进成片。

## 恢复与失败

revision_conflict重新读取并重新生成/校验，不偷换expectedRevision。提交或回读失联先subtitles-get查当前结果，不自动再提交。INCOMPLETE/坏receipt/输入SHA变化必须拒绝。候选、API/readback PASS、visual frame PASS、human listening UNVERIFIED分开；无人实际听看不报通过。仅在用户明确选定旧发行时读取该发行自带的MCP兼容文档，本包不携带旧MCP适配。

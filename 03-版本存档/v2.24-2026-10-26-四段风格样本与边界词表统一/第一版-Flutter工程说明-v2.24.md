# 第一版 · Flutter 工程说明

> 项目：妙恋畅语 ｜ 文档版本：v2.24 ｜ 编写：阿砚 ｜ 2026-10-26
> 对应原型：`02-原型/第一版-应用原型-v2.16.html`（v2.14 视觉，12 屏）
> v2.17 记第 0 步（12 屏搬进 Flutter），v2.19 记第 2 步（数据收成一处 model）。
> v2.21 补了执行清单与验收口径，v2.22 记状态层接入，v2.23 记四维改 1–5 档。
> **本版记四段风格样本的验证结果：判断方向全部对，但发现本地过滤与提示词各写了一份边界词表、已经漂移。界面仍未动。**

---

## 一、工程在哪，怎么跑

**代码在 `E:/妙恋畅语/app`，和 `first/`（文档区）分开。** 文档归文档，代码归代码。

```bat
cd /d E:\妙恋畅语\app

flutter run -d edge        :: 浏览器里看，带手机外壳，最快，支持热重载
flutter run -d windows     :: 桌面窗口，跑真 API 时用它
dart analyze               :: 静态检查（不要用 flutter analyze）
flutter test               :: 跑测试，现在 16 条
```

| 命令 | 用途 | 前提 |
| --- | --- | --- |
| `flutter run -d edge` | 日常开发预览，热重载 | 只需 Flutter |
| `flutter run -d windows` | 桌面版，跑真 API 时用它 | 要装 Visual Studio C++ 工具链 |
| `flutter build web --release` | 出静态包，约 2.7 MB | 无 |
| `dart analyze` | 静态检查 | 无 |
| `flutter test` | 单元 + 组件测试 | 无 |

跑之前确认 `flutter doctor` 里 Flutter 那一项是勾。

**调试参数**（只在 web 上有用）：地址后面加查询串直接进某一屏，不用一路点。

```text
http://127.0.0.1:PORT/?s=panel          进攻略面板
http://127.0.0.1:PORT/?s=analysis&b=1   进分析结果，且用边界场景的数据
http://127.0.0.1:PORT/?s=reply&p=anran  进回复建议，对象换成安然
```

`s` 取值：`welcome home panel import ocr analyzing analysis reply templates new settings me`

---

## 二、12 屏对照表

| 原型屏 | Dart 文件 | 状态 |
| --- | --- | --- |
| `welcome` | `screens/welcome.dart` | 完成。合规勾选没勾时「开始」禁用 |
| `home` | `screens/home.dart` | 完成。折线自己画的（`TempChart`）；对象列表读仓库；没有分析记录时显示"还没有做过分析" |
| `panel` | `screens/panel.dart` | 完成。四格可点，弹「依据 + 原图裁剪块」；两个折叠行可展开；读 `app.currentPerson` |
| `import` | `screens/import_screen.dart` | 完成。截图识别 / 粘贴文本双 tab，缩略图可增删 |
| `ocr` | `screens/ocr.dart` | 完成。点「我 / 对方」逐行改，计数实时变 |
| `analyzing` | `screens/analyzing.dart` | 完成。四步按 280/620/1420/2020 ms 打勾，2.5 秒后自动进结果 |
| `analysis` | `screens/analysis.dart` | 完成。普通 / 边界两种文案，判断依据是 `p.warn`，不是 id |
| `reply` | `screens/reply.dart` | 完成。三条候选，第三条降透明度 + 「这条先别发」 |
| `templates` | `screens/templates.dart` | 完成。场景筛选可切，8 条种子 |
| `new` | `screens/new_person.dart` | **完成。四个字段真落库**，首页列表和面板 chips 立刻能看到 |
| `settings` | `screens/settings.dart` | 界面完成，**Key 只存内存**，关掉就没了 |
| `me` | `screens/me.dart` | 完成，路由到设置 |

底部导航 5 项 + 中间凸起的「攻」，和原型一致（v2.15 去掉过，v2.16 退回来了）。

---

## 三、代码结构

```text
app/lib/
├── main.dart          入口 · 手机外壳 · 状态条 · 底部导航 · 路由 · 侧边提示
├── theme.dart         颜色 / 渐变 / 圆角 / 字号 / 文字样式 / 趋势判定
├── model.dart         数据结构：Person · Dims · Ev · TrendPoint · Reply · Tpl · OcrRow
├── repo.dart          PeopleRepository 接口 + MemoryPeopleRepository
├── data.dart          常量表（维度 / 模板 / 信号）+ 演示数据（仓库初值）+ OCR 演示稿
├── state.dart         全局状态 AppState（ChangeNotifier），只管界面状态 + 走仓库
├── llm/               模型层，纯 Dart，不 import flutter
│   ├── types.dart     输入输出结构（LlmInput / LlmAnalysis / LlmReply / LlmException）
│   ├── prompt.dart    两个系统提示词 + 拼装（改判断口径只改这里）
│   ├── guard.dart     边界硬过滤 + 兜底三条（代码里的 if，不是提示词里的请求）
│   └── deepseek.dart  请求、解析、错误归一（Key 错 / 断网 / 超时 / 返回不是 json）
├── widgets.dart       玻璃卡 / 渐变按钮 / 柔光球 / 折线图 / 虚线框 / 各类小件
└── screens/           12 个屏，一个文件一屏
```

`tool/try_llm.dart` 是命令行验证器，`dart run tool/try_llm.dart` 直接连 DeepSeek——**不经过 Flutter，所以没有跨域问题，也不需要桌面工具链**。用法见第五节。

---

## 四、数据怎么流（第 2 步的核心）

**一条链，不许抄近路。**

```text
data.dart 的 seedPeople()  →  MemoryPeopleRepository  →  AppState  →  各屏
     种子数据                      all/byId/upsert/remove      读 + 改           只读
```

三条约定：

1. **屏幕里不许出现任何人名数据。** 名单从 `app.all` 来，当前人从 `app.currentPerson` 来。
   写死的人名、副标题、折叠正文以前散在 `home.dart` / `panel.dart` 里，现在都是 `Person` 的字段。
2. **屏幕不许直接 `import 'repo.dart'`。** 只有 `state.dart` 碰仓库。第 4 步把内存仓库换成 sqlite 时，只改 `state.dart` 一行，12 屏一个都不用动。
3. **"这是不是边界场景"不许拿 id 判断。** 看 `Person.warn`。以前 `p.id == 'demo'` 在三个文件里各写一遍，加第三个人就会漏。

**新建对象怎么走**：表单 → `app.createPerson()` → `repo.upsert()` → `notifyListeners()` → 首页列表 / 面板 chips 一起刷新。新建的人没有分析记录（`history` 是空的），首页显示"还没有做过分析。导入一次聊天，这里就会出现曲线。"

---

## 五、模型层（第 3 步）

**这一层只做一件事：吃聊天记录，吐结构化的判断和三条回复。它不碰界面，也不 import flutter。**

为什么刻意写成纯 Dart：这样可以用 `dart run` 在命令行直接连 API，**绕开浏览器跨域，也绕开桌面工具链和真机**，改 prompt 时一次只要两秒。同时它也能被单测覆盖（9 条）。

```text
LlmInput ──analyze()──▶ LlmAnalysis ──suggest()──▶ List<LlmReply>
  记录+背景             阶段/四维/信号/下一步        3 条（2 能用 + 1 别发）
```

### 三条产品规则落在哪，别改坏

| 规则 | 落在哪 | 具体怎么做 |
| --- | --- | --- |
| **不许编造** | `prompt.dart` 规则 2 + `deepseek.dart` 解析 | 每个维度都要原文引用；找不到原文时 `level` 给 `null`、`evidence` 写「记录不足」，界面显示「—」而不是瞎猜一个数 |
| **边界最高优先** | `guard.dart` + `deepseek.dart` | 关键词表**只扫「她」说的话**（不扫用户自己说的，否则「我：那改天？」会误伤）；模型漏判时本地强制改判 |
| **2 条能用 + 1 条别发** | `prompt.dart` 规则 1/2 + 解析 | 那 1 条必须是真错（越界 / 替她做决定 / 姿态过低）；解析时把不能发的排到最后 |

### 边界硬过滤的准确语义（这段别改）

```text
本地 guard 命中  →  analyze 照常调用（四维还是要看），
                    suggest 不发请求，直接返回兜底三条（两条克制 + 一条越界反例）
模型判中边界     →  如果 next 里出现「约/见面/周末/一起…」，整句换成固定文案
```

也就是说：**判中边界时，回复根本不是生成的，是查表。** 这是项目书 §7「边界优先于任何推进建议」在代码里的实现——不靠提示词求模型听话。

### 错误都归一成人话

`LlmException.kind` 六种：`noKey` / `empty` / `network` / `timeout` / `badStatus` / `badJson`，每种都有 `human` 一句能直接显示给用户的话。界面上永远不该出现异常栈。

### 命令行怎么验（不用装任何东西）

```bat
set DEEPSEEK_API_KEY=sk-你的key

dart run tool\try_llm.dart             :: 普通样本
dart run tool\try_llm.dart boundary    :: 边界样本（"改天吧"）
dart run tool\try_llm.dart 我的聊天.txt  :: 自己的记录，一行一条，开头「我：」「她：」
```

会打印：本地边界过滤命不命中、阶段、四维（带原文引用）、信号、下一步、三条回复、耗时。**这是目前唯一不依赖环境就能跑通的路径**，prompt 定稿前别急着接界面。

### 第一轮真机验证（2026-10-26，真 Key）

跑之前我以为风险在提示词的措辞上。跑完之后发现，**真正的问题在数据精度上**。

**一、边界过度判。** 普通样本里她说「嗯，再看吧」，模型判成明确拒绝，`boundary_hit=true`，整段直接走兜底——回复根本没生成。但产品对同一段对话的读法是「暧昧升温、未检出边界」（种子数据里林夏那条就是这么写的）。根因是我原来那句规则把「没接住话头」和「明确说不」混成了一件事。改成白名单式（只认直接拒绝和明确收尾，累/懒/没接话头/单个含糊回应一律不算）之后，同一段稳定判为 `boundary=false`。

**二、0–100 的数在抖。** 同一段样本跑三次：

| 维度 | 降温前（0.2） | 降温后（0.0） | 改 5 档 + 锚点后 |
| --- | --- | --- | --- |
| fam | 55 / 45 / 55 | 45 / 55 / 45 | **55 / 55 / 55** |
| heat | 50 / 40 / 50 | 50 / 50 / 50 | **55 / 55 / 55** |
| comf | 70 / 65 / 70 | 70 / 70 / 70 | **75 / 75 / 75** |
| off | 30 / 20 / 30 | 20 / 30 / 20 | **55 / 55 / 55** |

温度降到 0 只稳住了 `heat` 和 `comf`。**根因不是温度，是向模型要它给不出的精度。** 于是四维改成 1–5 档、由代码映射成百分数（15 / 35 / 55 / 75 / 90），并给每个维度写死 3 档的锚点（例如熟悉度 3 档 =「知道彼此做什么工作、一两个具体喜好」）。三次结果完全一致。

**顺带一个巧合，正好对上**：`Trend.of` 的阈值是「差 2 点才出箭头」，而一档 = 20 点。同档绝不出假箭头，跨档必然出箭头。**所以趋势阈值不用改**——我原本以为要为抖动去动它，是多余的。

### 第二轮：四段风格样本（2026-10-26，真 Key）

上一轮只跑了同一段样本，结论只能到家。这一轮补了四段风格差异最大的记录，每段文件开头都写死自己的「预判」，跑完直接对账，不靠印象。样本在 `app/tool/samples/`，一条命令跑完整组并出对照表：

```bash
DEEPSEEK_API_KEY=sk-xxx dart run tool/try_llm.dart all
REPEAT=3 DEEPSEEK_API_KEY=sk-xxx dart run tool/try_llm.dart all
```

| 样本 | 考的是什么 | 结果 |
| --- | --- | --- |
| `work.md` | 她全程只报事务、零情绪词，我一直在追问；「再说吧」后面没有下文 | ✅ 唯一出问题的一段，见下 |
| `slow.md` | 她连着推掉两个时间（周三外地、周五太晚），但没说收尾话，最后给「我看看」 | ✅ 判 `boundary=false`，off 3 档，next「先不催时间，聊点电影或猫，等她主动提」 |
| `cold2warm.md` | 前四条只有「嗯」「哦」「还好」，后三条突然主动撩 | ✅ 没被前段带偏：comf 判 4 档，信号里明确写了「她补了一句不过有点可爱」 |
| `she-leads.md` | 她主动约、还抢着订票，是正样本 | ✅ off 5 档、阶段升到暧昧升温，next「顺着订票的事配合她，别抢主导」 |

**「她主动邀约」这段是专门放进去的反向检查。** 前一轮改 prompt 一直在教模型「别过度判边界」，容易矫枉过正成「什么都判要慢」。这段给出了满分回答，说明没跑偏。

修复后的四段对照：

| 样本 | 边界 | fam | heat | comf | off | 阶段 | 回复 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| cold2warm.md | no | 3 | 3 | 4 | 1 | 熟悉建立→熟悉建立 | 2/3 |
| she-leads.md | no | 3 | 4 | 4 | 5 | 熟悉建立→**暧昧升温** | 2/3 |
| slow.md | no | 3 | 2 | 3 | 3 | 熟悉建立→熟悉建立 | 2/3 |
| work.md | no | 2 | 2 | 3 | 2 | 熟悉建立→熟悉建立 | 2/3 |

**这轮抓到的问题：本地过滤和提示词各写了一份边界词表，已经漂移。**

`guard.dart` 的拒绝词表里留着 `再说吧`、`看情况吧`、还有一个裸词 `改天`；而上一轮重写后的提示词已经把「再看吧」明确划进「不算边界」。于是 `work.md` 出现了最难看的一种错：

- 模型其实判对了（它给出的 next 是合理的）
- 但 `BoundaryGuard.check()` 扫到「再说吧」直接命中，覆盖成「她给过明确的拒绝」
- 然后 `suggest()` 见到 `boundaryHit` 就不发第二次请求，全程走兜底话术
- 界面上用户看到的是「她觉得被拒绝了」，而她只是说了句「再说吧」

裸词 `改天` 更危险：「改天我们一起去看那个展吧」会同样被判成拒绝。

**修法不是把词表改对，是把词表收成唯一一份。** `BoundaryGuard` 现在有三张表——`hardDenies`（真拒绝）/ `stateWords`（报状态）/ `hedgeWords`（含糊）——提示词直接 `${BoundaryGuard.denyList}` 引用生成，改一处两处同步。并加了单测钉住这条：断言提示词里每个 `hardDenies` 词都出现在「算边界」之前、每个 `hedgeWords` 词都出现在「不算边界」之后。**以后谁再把口径写回两份，测试会红。**

修复前后的同一段 `work.md`：

| 项 | 修复前 | 修复后 |
| --- | --- | --- |
| 本地过滤 | 命中（她说过「再说吧」） | 未命中 |
| 边界 | true | **false** |
| 下一步 | 她给过明确的拒绝。这次不推进，先把节奏还给她。 | 先不追约，隔天聊电影或猫这类她感兴趣的话题 |
| 回复 | 走兜底话术 | 模型生成，越界线是「那周末我定周六下午，你加完班直接过来」 |

**新的、也是这轮最值钱的一条经验：改一行提示词，全部样本都要重测。**

把词表从提示词文字挪到代码引用，语义上没改任何东西，只是措辞顺序变了，结果 `slow` 的 comf 从 2 档翻到 3 档、`work` 的 fam 从 3 档翻到 2 档。档位边界附近的样本本来就坐在分界线上，措辞一动就翻档。这不是 bug，是分档设计的固有代价——**所以提示词要尽早冻结，冻结之后每次动它都得跑完 `tool/samples/` 再提交。**

（顺带一句：翻档之后这两段的判断其实更准了。`work` 的 fam 从「了解一点工作节奏」变成「只停留在日常状态」，这段确实只聊到了加班和外卖。）

**本轮暴露但没修的**：`cold2warm.md` 完全没聊见面，模型给 `off` 的是 `level: null / 记录不足`，代码映射成 15 分。`applyAnalysisTo()` 会把界面上的 off 从 55 抹成 15，**凭空造出一个「线下推进暴跌」的箭头**。真相是「这段记录里没有这个维度的信息」。这就是交接页里「`Dims` 改可空」那条待办的实证，接界面前必须先解决。

**还没验的**：同一段跑三次是否完全一致。这轮每段只跑了一轮（省额度），所以只能确认「判断方向对」，不能确认「复现性」。要验用 `REPEAT=3`。

---

## 六、字体（这节必须看，否则中文会变方块）

**Flutter Web 不带任何中文字体。** 不打包就是整屏方块。所以这个工程把字体裁好放进了 `assets/fonts/`：

| 文件 | 用途 | 大小 |
| --- | --- | --- |
| `NotoSansSC-Regular.ttf` | 正文 | 164 KB |
| `NotoSansSC-Medium.ttf` | 标题 / 强调（500 字重） | 164 KB |
| `NotoSerifSC-Medium.ttf` | 「妙恋畅语」「晚上好」「暧昧升温」这类对人说话的标题 | 218 KB |

**做法**：从 Google Fonts 取完整字库（各 10 MB），用 `fonttools` 只保留界面里实际出现的 **667 个字符**，裁完三个文件共 **544 KB**。完整字库留在 `app/_src/`，不进版本库。

**改了文案以后字会缺（显示成方块）**，要重新裁一次：

```bat
:: 1. 重新收集字符（当前工程里出现的所有字）
python -c "import glob;c=set();[c.update(open(p,encoding='utf-8').read()) for p in glob.glob('lib/**/*.dart',recursive=True)];open('_src/chars.txt','w',encoding='utf-8').write(''.join(sorted(x for x in c if ord(x)>=32)))"

:: 2. 重新裁三个
python -m fontTools.subset _src\sans400.ttf --text-file=_src\chars.txt --output-file=assets\fonts\NotoSansSC-Regular.ttf --name-IDs=* --no-hinting
python -m fontTools.subset _src\sans500.ttf --text-file=_src\chars.txt --output-file=assets\fonts\NotoSansSC-Medium.ttf --name-IDs=* --no-hinting
python -m fontTools.subset _src\serif500.ttf --text-file=_src\chars.txt --output-file=assets\fonts\NotoSerifSC-Medium.ttf --name-IDs=* --no-hinting
```

第 2 步就是照这个跑了一遍：字符集从 666 变成 667（新增一个「列」字），体积 548 → 544 KB。

**注意一处：`lib/llm/prompt.dart` 里的提示词不是界面文案，永远不会显示给人看。** 重裁字符集时把它排除掉，否则白涨两百来 KB：

```bat
python -c "import glob;c=set();[c.update(open(p,encoding='utf-8').read()) for p in glob.glob('lib/**/*.dart',recursive=True) if 'llm/prompt' not in p.replace(chr(92),'/')];open('_src/chars.txt','w',encoding='utf-8').write(''.join(sorted(x for x in c if ord(x)>=32)))"
```

第 3 步接界面时要再跑一次——`LlmException.human` 那几句错误提示是新的界面文案。

**以后上 Android / iOS 就不用了。** 系统自带中文（苹方 / Noto Sans CJK），把 `pubspec.yaml` 里 fonts 那段删掉、`theme.dart` 里 `kSans` 改回 `null` 即可。

---

## 七、做了什么，没做什么

**第 2 步做了**：数据结构独立成 `model.dart`；仓库层 `repo.dart`；`Person` 去掉 `const` 能增删改；12 屏全部改读同一处；边界判断换成 model 字段；新建对象真落库；补了 7 条测试。

**第 3 步前半做了**：`lib/llm/` 四件套（结构 / 提示词 / 边界过滤 / 客户端）；`tool/try_llm.dart` 命令行验证器；9 条单测覆盖正常解析、围栏包裹、越界夹取、漏判边界被本地拦下、判中边界不发第二次请求、401/超时/乱码/无 Key/空记录。**用假 Key 实测过：请求真的打到 api.deepseek.com 并拿回 401**，整条网络链路是通的。

**还是没做**（不是漏了，是留给后面）：

| 没做 | 留给第几步 |
| --- | --- |
| 界面接上模型（加载态 / 失败态 / 超时） | 第 3 步后半，等 prompt 在命令行验稳 |
| Key 存 secure storage | 第 4 步 |
| 数据关掉 App 还在（现在刷新就回到两个演示对象） | 第 4 步 |
| 截图 OCR（ML Kit） | 第 3 步之后 |
| 删除 / 编辑对象（仓库有 `remove`，界面没入口） | 第 4 步之后 |
| 页面切换动效、按钮按压反馈 | 最后 10% |

---

## 八、已知问题（都是真的，不是猜的）

1. **`flutter analyze` 在这个中文路径下会崩**（analysis server exit 255）。用 `dart analyze`，它正常工作，当前零问题。
2. **`flutter test` 会报 `Unable to connect to flutter_tester process: Invalid WebSocket upgrade request`**：
   是 `http_proxy` 环境变量把本机回环的 WebSocket 也代理走了。先 `set http_proxy=` 再跑。
3. **Web 版直连大模型会被浏览器的跨域策略拦掉。** 这是 CORS，不是代码写错。要跑真 API 用 `flutter run -d windows` 或 Android 真机。
4. **`flutter run -d windows` 需要 Visual Studio 的 C++ 桌面开发组件**，本机还没装。装的时候装到 E 盘（C 盘只剩 15G）。
5. **Web 首次打开约 2–3 MB**（CanvasKit 运行时），本地秒开，挂公网才有感。
6. **截图缩略图是画出来的**，不是真图片。真 OCR 接进来时换成 `Image.file`。
7. **安然的面板里，"关键信息"折叠正文写的是林夏的资料。** 这是 v2.17 之前就有的问题（那句话当时写死在 `panel.dart`，所有人共用一句）。第 2 步把它搬进了 `Person.notes`，但两条种子数据填的是同一句原话——**搬的时候没改内容，所以界面上看不出来**。要不要各写各的，等立项人定。
8. **`dart run` 也能连出去**：本机有全局 `http_proxy`，但 Dart 的 `HttpClient` 不读环境变量，直连反而通。**别照搬 curl 那套去给 Dart 配代理**，第 3 步实测直连成功（拿到真实 401）。
9. **"记录不足"的维度目前是沿用旧值，不是显示 `—`。** `LlmAnalysis` 里 `value` 可以是 `null`，但 `Person.dims` 的四个字段是 `int` 不能为空。所以 `applyAnalysisTo()` 遇到 `null` 时保留了上一次的数值——**这在语义上是错的**（会让"没算出来"看起来像"算出来了，和上次一样"）。真要修得把 `Dims` 改成可空，会牵连 `panel` / `home` 的百分比渲染和进度条宽度计算，属于界面改动，所以留到接界面的那一轮一起做。
   **v2.24 补了实证，问题比原先估计的严重**：`tool/samples/cold2warm.md` 那段完全没聊见面，模型给 `off` 的是 `level: null / 记录不足`，代码按映射表算成 15 分。于是界面上 off 会从 55 掉到 15，**凭空长出一个「线下推进暴跌」的箭头**。而真相只是「这段记录里没提到这件事」。产品立场写着「没有基线就不显示趋势」，这里等于在造假基线。**接界面之前这条必须先修，不能带着上线。**
10. **边界词表只有一份，在 `BoundaryGuard`。** 提示词用 `${BoundaryGuard.denyList}` 等三个 getter 引用生成，不要在 `prompt.dart` 里另写一份文字。上一版就是因为两边各写一份、彼此漂移，导致「再说吧」被本地当成拒绝、整段走兜底。已加单测钉住：`hardDenies` 的词必须出现在提示词「算边界」段内，`hedgeWords` 的必须出现在「不算边界」段内。要加词就改 `guard.dart`，别的不动。
11. **改提示词会让全部样本的档位重排，这是分档设计的固有代价。** 实测：仅把词表从提示词文字挪进代码引用（语义没变，只是措辞顺序变了），`slow` 的 comf 就从 2 档翻到 3 档、`work` 的 fam 从 3 档翻到 2 档。档位边界附近的样本本来就坐在分界线上。**所以 `prompt.dart` 定稿之后，每次动它都必须跑完 `tool/samples/` 全部样本再提交**，否则改了 A 段稳、B 段悄悄翻档也不知道。

---

## 九、下一步

| 步 | 做什么 | 验收 |
| --- | --- | --- |
| **3 后半** | 界面接上模型 | 粘一段真聊天，真生成三条，其中一条能指回原文；断网给人话 |
| **4** | 补本地存储 | 关掉 App 再开，对象和记录还在 |

**顺序别倒**：先用命令行把 prompt 调稳（同一段样本跑三次，看四维数字飘不飘、引用对不对得上原文），再接界面。反过来的话，每次都在 Flutter 里调，一轮两分钟，改不动。

**接界面时的两条死规矩**：① 没填 Key 时仍然走现在的演示数据，界面不许变成一片空白；② 加载态和失败态是新状态，不是改设计——按钮变灰转圈可以，配色圆角一个都不许动。


---

## 十、v2.21 增补：下一步执行清单

> 本节是对 v2.20 下一步的细化，不代表以下工作已经完成。每项完成后，应把实际结果、测试命令和遗留问题写回文档。

### 10.1 开始前：建立可回退的基线

- [x] 备份当前 `app/` 工程，或先提交一次版本控制记录。
- [x] 在 `E:\妙恋畅语\app` 下执行 `dart analyze`，记录输出。
- [x] 清理本机测试环境的代理变量后执行 `flutter test`，记录通过数与失败项：
  ```bat
  set http_proxy=
  flutter test
  ```
- [x] 执行一次普通样本和一次边界样本的命令行验证：
  ```bat
  dart run tool\try_llm.dart
  dart run tool\try_llm.dart boundary
  ```
- [x] 确认测试用 API Key 没有写进源码、文档、截图或版本库。

**基线实测（2026-10-26 08:20 在本机执行，不是照抄旧结果）**

| 项 | 命令 | 实际输出 |
| --- | --- | --- |
| 版本基线 | `git log --oneline -1` | `fcf949c 第 3 步前半：模型层搭好，命令行可验`，工作区干净 |
| 静态检查 | `dart analyze` | `No issues found!` |
| 测试 | `flutter test`（先清 proxy） | `+16: All tests passed!` |
| 普通样本 | `dart run tool\try_llm.dart` | 记录 8 行，她最后一句「不过跟你聊两句好像松一点」；本地过滤**未命中**；`badStatus 401` |
| 边界样本 | `dart run tool\try_llm.dart boundary` | 记录 7 行，她最后一句「改天吧」；本地过滤**命中**（她说过「这周不太行」）；`badStatus 401` |
| Key 泄漏 | `grep -rn "sk-"` + `git grep` | 命中的只有 `sk-test`（单测假值）、`sk-xxxx`（占位提示）、`sk-...`（输入框 hint）；版本库里没有真实 Key |

**基线验收：** 静态检查和测试结果可复现；若有失败，先记录具体失败，不把旧结果当作本次结果。

> **两条命令行样本用的是假 Key，所以只走到鉴权就被挡回来了（401）。**
> 能证明的：请求真的发出去了、错误归一正确、本地边界过滤在发请求**之前**就命中。
> 不能证明的：模型输出质量——四维数字飘不飘、`source` 对不对得上原文。**这一项目前是空的**，必须拿真 Key 各跑三次才算数。

### 10.2 第 3 步后半：把模型接入界面

> **执行状态（2026-10-26 08:35）**：第 1 步（调用入口收到状态层）已完成；第 2 步只做完了状态定义那一半（`llmBusy` / `llmError` / `lastAnalysis` 三个字段有了，界面还没读）。第 2 步的呈现（加载态长什么样、失败态长什么样）、第 3 步、第 4 步都还没做。详见第十二节第二条记录。

建议按以下顺序实施，避免同时改动模型、状态和视觉结构。

1. **确定调用入口**
   - 找到导入聊天记录后进入分析的现有流程。
   - 由状态层统一调用 `analyze()`，再调用 `suggest()`。
   - 页面不得直接发起 HTTP 请求，也不要让页面自行解析模型 JSON。

2. **定义界面状态**
   - 未配置 Key：继续使用现有演示数据，不显示空白页面。
   - 正在分析：显示加载状态，避免重复提交。
   - 成功：展示阶段、四维判断、信号、下一步和三条回复。
   - 失败：展示 `LlmException.human` 对应的人类可读提示，不展示异常栈。
   - 超时或网络失败后：允许用户返回并重试，且不应把失败结果写成有效分析记录。

3. **保留现有视觉**
   - 加载时可以禁用按钮、显示进度反馈。
   - 不改变既有配色、圆角、字体和布局。
   - 不因增加状态而删除原有演示流程。

4. **核对结果与证据**
   - 每个有数值的维度都应能对应到聊天原文。
   - 证据不足时显示 `—` 或文档约定的“记录不足”，不得补造证据。
   - 边界规则仍由本地 `guard.dart` 强制执行，不依赖提示词单独保证。
   - 边界命中时，回复建议不得变成邀约或推进关系的建议。
   - 三条回复的顺序与“2 条可用 + 1 条不建议发送”的既有规则保持一致。

### 10.3 第 3 步验收表

| 场景 | 操作 | 通过标准 |
| --- | --- | --- |
| 无 Key | 不配置 Key，进入原有流程 | 演示数据仍可用，页面不空白 |
| 正常记录 | 粘贴一段聊天记录并分析 | 显示结构化结果和三条回复 |
| 原文证据 | 检查至少一项判断及其引用 | 引用确实存在于输入原文中 |
| 证据不足 | 输入信息不足的记录 | 不虚构数值或原文 |
| 边界样本 | 使用“改天吧”等边界样本 | 本地过滤规则生效，不给出越界推进建议 |
| 网络断开 | 分析时断网 | 显示可理解的失败提示，不显示异常栈 |
| 请求超时 | 模拟或触发超时 | 有明确提示，页面可恢复 |
| 连续点击 | 加载期间重复点击 | 不产生重复提交 |
| 返回页面 | 从结果页返回再进入 | 状态与记录符合当前实现，不残留错误加载态 |

### 10.4 第 4 步：本地持久化

当前文档确认的现状是：仓库仍为 `MemoryPeopleRepository`，刷新或关闭应用后演示对象会恢复为初始状态。持久化尚未实现。

实施时建议先明确要持久化的数据范围：

- [ ] 对象资料：新增、编辑、删除后重启仍然存在。
- [ ] 分析记录：只有成功完成的分析才进入历史记录。
- [ ] 历史曲线：数据来源于真实保存的分析记录，不使用临时页面状态拼造。
- [ ] 当前选中对象：重启后是否恢复，作为单独的产品决定。
- [ ] API Key：按文档计划使用安全存储；不要把 Key 明文写入普通数据库或日志。

**架构约束：** 页面继续通过 `AppState` 访问数据；不要让 12 个页面分别操作数据库。替换仓库实现时，优先保持 `PeopleRepository` 的调用边界稳定。

### 10.5 交付记录模板

每次完成一个阶段后，按下面格式更新工程说明：

```text
阶段：
完成日期：
修改文件：
执行命令：
测试结果：
人工验收结果：
未解决问题：
下一步：
```

如果某项只完成了代码编写但没有运行验证，应写成“已实现，未验证”，不要写成“已完成”。

---

## 十一、v2.21 变更记录

- 将文档版本推进至 v2.21。
- 保留 v2.20 中已经记录的工程结构、实现状态、已知问题和命令。
- 新增第 3 步模型接入界面的执行顺序与验收表。
- 新增第 4 步本地持久化的范围检查清单。
- 新增阶段交付记录模板，区分“已实现”与“已验证”。
- **没有将计划中的功能标记为已完成；实际状态仍需通过工程运行与测试确认。**
- 阿砚补记（08:20）：把 10.1 的清单按实际执行结果逐项勾选，附实测输出表；新增第十二节「交付记录」，第 3 步前半按 10.5 模板记一条，明确标为**已实现、未验证**。同时修掉一处过期数字（`flutter test` 已从 7 条变成 16 条）。

---

## 十二、交付记录

> 按 10.5 的格式，一个阶段一条。只写实际跑过的，没跑过的写"未验证"。

### 第 3 步前半：模型层

```text
阶段：        第 3 步前半（模型层；界面未接）
完成日期：    2026-10-26
修改文件：    lib/llm/types.dart
              lib/llm/prompt.dart
              lib/llm/guard.dart
              lib/llm/deepseek.dart
              tool/try_llm.dart
              test/llm_test.dart
              pubspec.yaml（新增 http 依赖）
执行命令：    dart analyze
              flutter test
              flutter build web --release
              dart run tool\try_llm.dart            （假 Key）
              dart run tool\try_llm.dart boundary   （假 Key）
测试结果：    analyze 零问题；test 16/16；build 通过；
              命令行两次都在鉴权处返回 401（假 Key，属预期）
人工验收结果：已实现，未验证 —— 真模型输出一次都没跑过，prompt 未定稿
未解决问题：  ① 缺真实 API Key，prompt 质量无法评估
              ② 四维数字的跨次稳定性、source 是否逐字命中原文，均未测
              ③ 界面未接，加载态 / 失败态只有设计没有实现
下一步：      拿真 Key 跑普通 / 边界各三次，把输出贴回本文档，
              再决定是改 prompt 还是接界面
```

**这条记录本身就是 10.5 的意义**：代码写完、单测全绿、构建通过，但**没有一次真模型输出**，所以只能写"已实现，未验证"。如果按"测试全过 = 完成"来记，就会把下一步的判断建立在一个空地基上。

### 第 3 步（第二步）：调用入口收到状态层

```text
阶段：        第 3 步 · 10.2 第 1 步（调用入口）+ 第 2 步的状态定义
完成日期：    2026-10-26
修改文件：    lib/state.dart（新增 applyAnalysisTo / inputFor / runLlm，加 llmBusy / llmError / lastAnalysis）
              test/wiring_test.dart（新增）
执行命令：    dart analyze
              flutter test
              flutter build web --release
              无头 Edge 截图三张后 md5 比对
测试结果：    analyze 零问题；test 22/22（原 16 + 新增 6）；build 通过
人工验收结果：已验证 —— 三张截图 md5 与基线完全相同
              （home c1d1c806…、panel 9db9de37…、reply 4dd47bae…）
未解决问题：  ① 仍是"记录不足"维度沿用旧值（见第八节第 9 条）
              ② 界面还没读 llmBusy / llmError，所以加载态、失败态只有字段没有呈现
              ③ 真模型输出仍未验证，prompt 未定稿
下一步：      接 analysis / reply 两屏的加载态与失败态；
              接之前先拿真 Key 验 prompt，否则在错误基准上雕花
```

**这一条为什么能写"已验证"，上一条不能**：界面呈现属于视觉，有 md5 可比；prompt 质量属于内容，没有真 Key 就无从比较。**能比对的才叫验证，比不了的最多叫"实现"。**

### 第 3 步（第三步）：prompt 第一轮真机验证

```text
阶段：        第 3 步 · prompt 首轮验证并改数据结构
完成日期：    2026-10-26
修改文件：    lib/llm/prompt.dart（规则 2/3 重写、新增规则 7 分档、加维度锚点）
              lib/llm/types.dart（LlmDim 加 level）
              lib/llm/deepseek.dart（档位映射、分析温度 0.2 → 0.0）
              tool/try_llm.dart（打印档位）
              test/llm_test.dart（新增档位映射 1 条）
执行命令：    dart analyze / flutter test
              dart run tool\try_llm.dart（真 Key，普通样本 ×3）
              dart run tool\try_llm.dart boundary（真 Key，×1）
              改之前还跑过 0.0 温度的三次对比
测试结果：    analyze 零问题；test 23/23
人工验收结果：已验证 —— 同一段样本三次四维完全一致（3/3/4/3）；
              边界判定三次一致；边界样本判「需要空间」且回复无邀约
未解决问题：  ① 换聊天风格（工作汇报式 / 长间隔 / 先冷后热）会不会又抖，没测
              ② 只验了 2 段样本，样本量太小
              ③ "记录不足"仍然沿用旧值（第八节第 9 条），未修
              ④ 界面仍未接
下一步：      造 3–5 段风格差异大的样本再跑，稳了再接界面
```

**这一条给了一个之前没想到的教训**：prompt 的问题不一定出在措辞上。第一次跑出来的"边界乱判"是措辞问题，第二次跑出来的"数字抖"是**数据模型问题**——0–100 的精度超出了模型能给的范围，把模型的精度降到 1–5 档，抖动就没了。**遇到不稳定，先看是不是数据模型要得太细，别一直改措辞。**

### 第 3 步（第四步）：四段风格样本 + 边界口径统一

```text
阶段：        第 3 步 · 多风格样本验证，修本地过滤与提示词口径不一致
完成日期：    2026-10-26
修改文件：    lib/llm/guard.dart（denies 拆成 hardDenies / stateWords / hedgeWords，
                                  去掉裸词「改天」，兜底话术不再写死「改天吧」）
              lib/llm/prompt.dart（规则 3 改为引用 BoundaryGuard 的三个词表生成）
              tool/try_llm.dart（支持 all / 多文件 / # 注释 / # stage 指令；
                                 REPEAT=n 重复跑；末尾出对照表与稳定性判定；
                                 修 Stopwatch.reset 后未 start 导致耗时恒为 0）
              tool/samples/{work,slow,cold2warm,she-leads}.md（新增，四段样本 + 各自预判）
              test/llm_test.dart（新增 2 条：词表口径、提示词与代码同源）
执行命令：    dart analyze / flutter test
              dart run tool\try_llm.dart all（真 Key，4 段 × (1 分析 + 1 回复) = 8 次）
              修复后原样重跑一遍做回归
测试结果：    analyze 零问题；test 25/25
人工验收结果：已验证 —— 四段样本的判断全部落回预判；
              work.md 修复前误判 boundary=true 并走兜底，修复后 boundary=false、
              next 与三条回复均由模型正常生成；
              she-leads.md 作为反向检查给出 off 5 档 + 阶段升到暧昧升温，
              证明「别过度判边界」没有矫枉过正成「什么都判要慢」
未解决问题：  ① 每段只跑一轮（省额度），「同一段三次是否一致」仍未验
              ② cold2warm.md 的 off 被 null 映射成 15，会造假箭头（第八节第 9 条）
              ③ 界面仍未接
下一步：      修 Dims 可空 → 两屏加载态失败态 → 接调用入口
```

**这一条最值钱的不是"四段都过了"，而是抓到了一个自伤型的坑。** 上一轮我把提示词的边界规则改成白名单式，却没回头看本地 `guard.dart` 里的词表还是旧的一套。模型按新口径判，过滤器按旧口径拦，结果是她只说了句「再说吧」，用户却看到「她觉得被拒绝了」。

**根因不是词写错了，是同一个口径写了两份。** 所以修的时候没有去"把词表补对"，而是把词表收进 `BoundaryGuard` 唯一一处，提示词引用生成，再加单测钉住。**只要还有第二份，就一定会再次漂移** —— 这次是巧合撞上，下次可能上线后才撞上。

---

## 十五、v2.24 变更记录

- 版本推进至 v2.24。
- `BoundaryGuard` 的拒绝词表拆成三张：`hardDenies`（真拒绝，`check()` 只扫这张）/ `stateWords`（报状态）/ `hedgeWords`（含糊）。**去掉裸词 `改天`**（会误伤「改天我们一起去看展」），`再说吧`、`看情况吧` 移出拒绝表。兜底话术里的「把『改天吧』当成可以再问」改成中性措辞，不再写死具体词。
- `prompt.dart` 的规则 3 不再硬写词表，改为 `${BoundaryGuard.denyList}` / `stateList` / `hedgeList` 引用生成；`analyzeSystem` 由 `const` 改为 `final`。
- 新增 `tool/samples/` 四段风格样本（工作汇报式 / 长间隔 / 先冷后热 / 她主动邀约），每段文件头写死预判与 `# stage`。
- `tool/try_llm.dart` 升级：支持 `all`、多文件、`#` 注释、`# stage:` 指令、`REPEAT=n` 重复跑，末尾输出对照表与稳定性判定。修一个自身 bug——`Stopwatch.reset()` 在已停止的计时器上不会重新走，导致回复耗时恒显示 0 ms。
- 新增 `test/llm_test.dart` 2 条：一条测词表口径（「再说吧」「改天一起」不拦、「改天吧」拦），一条测**提示词与代码同源**（`hardDenies` 必须出现在「算边界」段内，`hedgeWords` 必须出现在「不算边界」段内）。
- **未改任何界面代码**，12 屏仍与基线一致；`theme.dart` / `widgets.dart` / 各屏视觉参数一个未动。
- 真 Key 只在命令行环境变量中使用，**未写入任何文件、文档、截图或提交记录**（每次提交后 `git status` 复查）。

---

## 十四、v2.23 变更记录

- 版本推进至 v2.23。
- 提示词规则 3 重写为白名单式，区分「没接住话头」与「明确说不」；规则 7 新增四维 1–5 分档；四个维度各补 3 档锚点。
- `LlmDim` 增加 `level` 字段；`deepseek.dart` 增加 `levelValues`（15/35/55/75/90）映射，`value` 由档位换算，仍兼容模型直接给 `value` 的旧格式。
- 分析调用的温度由 0.2 降到 0.0（只解决了一半抖动，另一半靠分档解决）。
- `tool/try_llm.dart` 打印档位。
- **未改任何界面代码**，12 屏仍与基线一致。
- 真 Key 只在命令行环境变量中使用，**未写入任何文件、文档、截图或提交记录**。

---

## 十三、v2.22 变更记录

- 版本推进至 v2.22。
- `state.dart` 新增 `applyAnalysisTo()`：把 `LlmAnalysis` + `List<LlmReply>` 折叠成新的 `Person`（阶段、四维、下一步、`warn`、三条回复、时间线、曲线点），`prev` 保留旧 dims 以便算趋势。
- 新增 `AppState.inputFor()` / `runLlm()`：模型调用统一从状态层发起，页面不许自己发请求、也不许自己解析 JSON；重入保护；异常只落 `llmError`，不抛给界面。
- 新增 `llmBusy` / `llmError` / `lastAnalysis` 三个界面状态字段（**已定义，界面尚未读取**）。
- 新增 `test/wiring_test.dart` 6 条。
- **关键结论**：`Person` 的字段与 `LlmAnalysis` 同构，所以接模型**不需要改任何一屏**——界面继续读 `app.currentPerson`，数据换掉自动跟。
- 未改任何视觉参数；三张截图 md5 与基线逐字节一致。


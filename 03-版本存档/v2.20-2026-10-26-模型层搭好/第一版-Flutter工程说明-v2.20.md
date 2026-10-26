# 第一版 · Flutter 工程说明

> 项目：妙恋畅语 ｜ 文档版本：v2.20 ｜ 编写：阿砚 ｜ 2026-10-26
> 对应原型：`02-原型/第一版-应用原型-v2.16.html`（v2.14 视觉，12 屏）
> v2.17 记第 0 步（12 屏搬进 Flutter），v2.19 记第 2 步（数据收成一处 model）。
> **本版记第 3 步的前半段：模型层搭好，命令行可验，界面还没接。**

---

## 一、工程在哪，怎么跑

**代码在 `E:/妙恋畅语/app`，和 `first/`（文档区）分开。** 文档归文档，代码归代码。

```bat
cd /d E:\妙恋畅语\app

flutter run -d edge        :: 浏览器里看，带手机外壳，最快，支持热重载
flutter run -d windows     :: 桌面窗口，跑真 API 时用它
dart analyze               :: 静态检查（不要用 flutter analyze）
flutter test               :: 跑测试，现在 7 条
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
| **不许编造** | `prompt.dart` 规则 2 + `deepseek.dart` 解析 | 每个维度都要原文引用；找不到原文时 `value` 给 `null`、`evidence` 写「记录不足」，界面显示「—」而不是瞎猜一个数 |
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

---

## 九、下一步

| 步 | 做什么 | 验收 |
| --- | --- | --- |
| **3 后半** | 界面接上模型 | 粘一段真聊天，真生成三条，其中一条能指回原文；断网给人话 |
| **4** | 补本地存储 | 关掉 App 再开，对象和记录还在 |

**顺序别倒**：先用命令行把 prompt 调稳（同一段样本跑三次，看四维数字飘不飘、引用对不对得上原文），再接界面。反过来的话，每次都在 Flutter 里调，一轮两分钟，改不动。

**接界面时的两条死规矩**：① 没填 Key 时仍然走现在的演示数据，界面不许变成一片空白；② 加载态和失败态是新状态，不是改设计——按钮变灰转圈可以，配色圆角一个都不许动。

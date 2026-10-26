# 第一版 · Flutter 工程说明

> 项目：妙恋畅语 ｜ 文档版本：v2.19 ｜ 编写：阿砚 ｜ 2026-10-26
> 对应原型：`02-原型/第一版-应用原型-v2.16.html`（v2.14 视觉，12 屏）
> v2.17 记的是第 0 步（12 屏搬进 Flutter）。**本版记第 2 步：数据收成一处 model。**

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
├── widgets.dart       玻璃卡 / 渐变按钮 / 柔光球 / 折线图 / 虚线框 / 各类小件
└── screens/           12 个屏，一个文件一屏
```

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

## 五、字体（这节必须看，否则中文会变方块）

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

**以后上 Android / iOS 就不用了。** 系统自带中文（苹方 / Noto Sans CJK），把 `pubspec.yaml` 里 fonts 那段删掉、`theme.dart` 里 `kSans` 改回 `null` 即可。

---

## 六、做了什么，没做什么

**第 2 步做了**：数据结构独立成 `model.dart`；仓库层 `repo.dart`；`Person` 去掉 `const` 能增删改；12 屏全部改读同一处；边界判断换成 model 字段；新建对象真落库；补了 7 条测试。

**还是没做**（不是漏了，是留给后面）：

| 没做 | 留给第几步 |
| --- | --- |
| 接 DeepSeek，真生成回复 | 第 3 步 |
| 数据关掉 App 还在（现在刷新就回到两个演示对象） | 第 4 步 |
| 截图 OCR（ML Kit） | 第 3 步之后 |
| Key 存 secure storage | 第 4 步 |
| 删除 / 编辑对象（仓库有 `remove`，界面没入口） | 第 4 步之后 |
| 页面切换动效、按钮按压反馈 | 最后 10% |

---

## 七、已知问题（都是真的，不是猜的）

1. **`flutter analyze` 在这个中文路径下会崩**（analysis server exit 255）。用 `dart analyze`，它正常工作，当前零问题。
2. **`flutter test` 会报 `Unable to connect to flutter_tester process: Invalid WebSocket upgrade request`**：
   是 `http_proxy` 环境变量把本机回环的 WebSocket 也代理走了。先 `set http_proxy=` 再跑。
3. **Web 版直连大模型会被浏览器的跨域策略拦掉。** 这是 CORS，不是代码写错。要跑真 API 用 `flutter run -d windows` 或 Android 真机。
4. **`flutter run -d windows` 需要 Visual Studio 的 C++ 桌面开发组件**，本机还没装。装的时候装到 E 盘（C 盘只剩 15G）。
5. **Web 首次打开约 2–3 MB**（CanvasKit 运行时），本地秒开，挂公网才有感。
6. **截图缩略图是画出来的**，不是真图片。真 OCR 接进来时换成 `Image.file`。
7. **安然的面板里，"关键信息"折叠正文写的是林夏的资料。** 这是 v2.17 之前就有的问题（那句话当时写死在 `panel.dart`，所有人共用一句）。第 2 步把它搬进了 `Person.notes`，但两条种子数据填的是同一句原话——**搬的时候没改内容，所以界面上看不出来**。要不要各写各的，等立项人定。

---

## 八、下一步

| 步 | 做什么 | 验收 |
| --- | --- | --- |
| **3** | 粘贴聊天 → 调 DeepSeek → 结果进面板 | 真粘贴一段，真生成三条回复，其中一条能指回原文 |
| **4** | 补本地存储 | 关掉 App 再开，对象和记录还在 |

**第 3 步的第一件事不是写请求，是想清楚边界。** 边界必须先判、判中就不生成邀约，这是硬过滤，不能只写在提示词里（项目书 §7 的口径）。

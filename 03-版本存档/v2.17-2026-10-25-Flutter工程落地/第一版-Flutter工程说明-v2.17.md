# 第一版 · Flutter 工程说明

> 项目：妙恋畅语 ｜ 文档版本：v2.17 ｜ 编写：阿砚 ｜ 2026-10-25
> 对应原型：`02-原型/第一版-应用原型-v2.16.html`（v2.14 视觉，12 屏）

---

## 一、工程在哪，怎么跑

**代码在 `E:/妙恋畅语/app`，和 `first/`（文档区）分开。** 这是路线图定的：文档归文档，代码归代码。

```bat
cd /d E:\妙恋畅语\app

flutter run -d edge        :: 浏览器里看，带手机外壳，最快
flutter run -d windows     :: 桌面窗口，需要 VS2022 + C++ 桌面开发
flutter build web --release
```

| 命令 | 用途 | 前提 |
| --- | --- | --- |
| `flutter run -d edge` | 日常开发预览，热重载 | 只需 Flutter |
| `flutter run -d windows` | 桌面版，跑真 API 时用它 | 要装 Visual Studio C++ 工具链 |
| `flutter build web --release` | 出静态包，约 2.7 MB | 无 |

跑之前确认 `flutter doctor` 里 Flutter 那一项是勾。

**调试参数**（只在 web 上有用）：地址后面加查询串可以直接进某一屏，不用一路点。

```text
http://127.0.0.1:PORT/?s=panel          进攻略面板
http://127.0.0.1:PORT/?s=analysis&b=1   进分析结果，且用边界场景的数据
http://127.0.0.1:PORT/?s=reply&p=anran  进回复建议，对象换成安然
```

`s` 取值：`welcome home panel import ocr analyzing analysis reply templates new settings me`

---

## 二、12 屏对照表

原型里每一屏在 Dart 里都有对应文件，**内容和文案一字未改**。

| 原型屏 | Dart 文件 | 状态 |
| --- | --- | --- |
| `welcome` | `screens/welcome.dart` | 完成。合规勾选没勾时「开始」禁用 |
| `home` | `screens/home.dart` | 完成。四维折线是自己画的（`TempChart`），不是图表库 |
| `panel` | `screens/panel.dart` | 完成。四格可点，弹出「依据 + 原图裁剪块」；两个折叠行可展开 |
| `import` | `screens/import_screen.dart` | 完成。截图识别 / 粘贴文本双 tab，缩略图可增删 |
| `ocr` | `screens/ocr.dart` | 完成。点「我 / 对方」可逐行改，计数实时变 |
| `analyzing` | `screens/analyzing.dart` | 完成。四步按 280/620/1420/2020 ms 依次打勾，2.5 秒后自动进结果 |
| `analysis` | `screens/analysis.dart` | 完成。普通 / 边界两种数据，颜色和文案都跟着换 |
| `reply` | `screens/reply.dart` | 完成。三条候选，第三条降透明度 + 「这条先别发」 |
| `templates` | `screens/templates.dart` | 完成。场景筛选可切，8 条种子 |
| `new` | `screens/new_person.dart` | 界面完成，**保存不落库** |
| `settings` | `screens/settings.dart` | 界面完成，**Key 只存内存**，关掉就没了 |
| `me` | `screens/me.dart` | 完成，路由到设置 |

底部导航 5 项 + 中间凸起的「攻」，和原型一致（v2.15 去掉过，v2.16 退回来了）。

---

## 三、代码结构

```text
app/lib/
├── main.dart          入口 · 手机外壳 · 状态条 · 底部导航 · 路由 · 侧边提示
├── theme.dart         颜色 / 渐变 / 圆角 / 字号 / 文字样式 / 趋势判定
├── data.dart          假数据（两个人 + 边界示例 + 模板 + OCR 演示行）
├── state.dart         全局状态 AppState（ChangeNotifier）
├── widgets.dart       玻璃卡 / 渐变按钮 / 柔光球 / 折线图 / 虚线框 / 各类小件
└── screens/           12 个屏，一个文件一屏
```

**三件要理解的事：**

1. **`data.dart` 里的数字是原型里抄来的假数据，不是算出来的。** 72 / 66 / 81 / 38 是写死的，第 2 步要把它们换成 model 读取。
2. **`state.dart` 是临时方案。** 一个全局单例 `app`，改字段后 `notifyListeners()`。第 0 步只需要"能点、能换屏"，这个够用；等到接真数据（第 2 步）要换成正规做法，别在这上面盖楼。
3. **原型里的样式几乎全在 `theme.dart` 里集中定义。** 改配色只改 `C` 类，改圆角只改 `R` 类，改字号只改 `T` 类。不要在各个屏里写死颜色。

---

## 四、字体（这节必须看，否则中文会变方块）

**Flutter Web 不带任何中文字体。** 不打包就是整屏方块。所以这个工程把字体裁好放进了 `assets/fonts/`：

| 文件 | 用途 | 大小 |
| --- | --- | --- |
| `NotoSansSC-Regular.ttf` | 正文 | 164 KB |
| `NotoSansSC-Medium.ttf` | 标题 / 强调（500 字重） | 164 KB |
| `NotoSerifSC-Medium.ttf` | 「妙恋畅语」「晚上好」「暧昧升温」这类对人说话的标题 | 218 KB |

**做法**：从 Google Fonts 取完整字库（各 10 MB），用 `fonttools` 只保留界面里实际出现的 666 个字符，裁完三个文件共 548 KB。完整字库留在 `app/_src/`，不进版本库。

**改了文案以后字会缺字（显示成方块）**，要重新裁一次：

```bat
:: 1. 重新收集字符
python -c "import glob;c=set();[c.update(open(p,encoding='utf-8').read()) for p in glob.glob('lib/**/*.dart',recursive=True)];open('_src/chars.txt','w',encoding='utf-8').write(''.join(sorted(x for x in c if ord(x)>=32)))"

:: 2. 重新裁三个
python -m fontTools.subset _src\sans400.ttf --text-file=_src\chars.txt --output-file=assets\fonts\NotoSansSC-Regular.ttf --name-IDs=* --no-hinting
python -m fontTools.subset _src\sans500.ttf --text-file=_src\chars.txt --output-file=assets\fonts\NotoSansSC-Medium.ttf --name-IDs=* --no-hinting
python -m fontTools.subset _src\serif500.ttf --text-file=_src\chars.txt --output-file=assets\fonts\NotoSerifSC-Medium.ttf --name-IDs=* --no-hinting
```

**以后上 Android / iOS 就不用了。** 系统自带中文（苹方 / Noto Sans CJK），把 `pubspec.yaml` 里 fonts 那段删掉、`theme.dart` 里 `kSans` 改回 `null` 即可，包体还能小 0.5 MB。

---

## 五、这一步做了什么，没做什么

**做了**：12 屏全部可点通，视觉和原型 1:1，中文和衬线标题都在，能直接 `flutter run` 看到真东西。

**没做**（都不是漏了，是刻意留给后面）：

| 没做 | 留给第几步 |
| --- | --- |
| 接 DeepSeek，真生成回复 | 第 3 步 |
| 聊天记录本地存储、关掉 App 还在 | 第 4 步 |
| 截图 OCR（ML Kit） | 第 3 步之后 |
| 新建对象表单落库 | 第 4 步 |
| Key 存 secure storage | 第 4 步 |
| 页面切换动效、按钮按压反馈 | 最后 10% |
| Android / iOS 平台目录 | 装完 Android Studio 再加 |

---

## 六、已知问题（都是真的，不是猜的）

1. **`flutter analyze` 在这个中文路径下会崩**（analysis server exit 255）。用 `dart analyze`，它正常工作，且已经零问题。
2. **Web 版直连大模型会被浏览器的跨域策略拦掉。** 这是 CORS，不是代码写错。要跑真 API 用 `flutter run -d windows` 或 Android 真机。
3. **`flutter run -d windows` 需要 Visual Studio 的 C++ 桌面开发组件**，本机还没装。装的时候注意装到 E 盘（C 盘只剩 15G）。
4. **Web 首次打开约 2–3 MB**（CanvasKit 运行时），本地秒开，挂公网才有感。
5. **截图缩略图是画出来的**，不是真图片。真 OCR 接进来时换成 `Image.file`。

---

## 七、下一步

按路线图的四步走，现在站在第 1 步的终点、第 2 步的起点：

| 步 | 做什么 | 验收 |
| --- | --- | --- |
| **2** | 把 `data.dart` 的写死数据换成 model 读取 | 点首页对象卡进面板，数据来自同一份 model |
| **3** | 粘贴聊天 → 调 DeepSeek → 结果进面板 | 真粘贴一段，真生成一条回复 |
| **4** | 补本地存储 | 关掉 App 再开，对象和记录还在 |

**先做第 2 步再碰 API。** 现在 12 屏的数据全写死在 `data.dart`，先让它们从一处读，接 API 时才不用改 12 个文件。

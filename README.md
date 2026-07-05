# Kibble / Food Diary

A pixel cat sits in a small window on your desktop. Drag a food photo onto it, the cat chews, an old paper scroll slides out for you to scribble a note, and the photo is committed and pushed to your private "kibble" git repository. That's the whole app.

The bundled Codex skill is now **Food Diary**. It can log food directly from chat images, local image paths, receipts, menus, or text notes; the Kibble app inbox is still supported as a legacy input path.

![screenshot](docs/screenshot.png)

## Why

Logging meals shouldn't be a form. It should be a gesture. Kibble is one gesture; sending a meal photo directly to Codex is another. Food Diary handles the downstream work: classification, timestamps, meal type, nutrition lookup, Apple Health sync, and reports.

## Quick start

### 1. Prepare your data repository

```bash
# create a private repo on GitHub (or wherever), then:
git clone git@github.com:you/kibble-data.git ~/kibble-data
```

Make sure `git push` works in that directory before launching Kibble. Kibble does not handle authentication.

### 2. Configure Kibble

Run Kibble once. It will create a config template at `~/.config/kibble/config.toml` (or `%APPDATA%\kibble\config.toml` on Windows) and print the path to the console.

Edit `repo_path` to point at your clone:

```toml
repo_path = "/Users/you/kibble-data"
```

### 3. Build & run

```bash
cd app
pnpm install
pnpm tauri dev      # development
pnpm tauri build    # production binary
```

## How it works

1. Drop image(s) onto the cat.
2. Cat chews. A scroll slides out. Type a note (or don't). Press Enter.
3. Kibble copies the images into `<repo_path>/inbox/`, writes a sibling `.note.txt` if you wrote a note, and runs `git add -A && git commit && git push` in that directory.
4. Cat looks satisfied on success, confused on failure. Errors go to stderr.

The downstream Skill (see `skill/SKILL.md`) is responsible for everything else. By default it accepts direct chat images and local image paths; it only drains `<repo_path>/inbox/` when you explicitly ask it to process the legacy Kibble inbox.

## Food Diary skill

The bundled skill lives in `skill/` and is named `food-diary`.

Use it for:

- Direct food photos or screenshots sent to Codex.
- Local image paths.
- Receipts, menus, and delivery screenshots.
- Text-only meal notes.
- Legacy Kibble inbox processing.
- Apple Health sync and diet/health Word reports.

## File layout in the data repo

```
kibble-data/
└── inbox/
    ├── IMG_1234.jpg
    ├── IMG_1234.jpg.note.txt    # only if you wrote a note
    └── IMG_1235.jpg
```

Filenames are preserved. Collisions get a `_1`, `_2` suffix.

## License

MIT

---

# Kibble / Food Diary（中文）

桌面上一个小窗口里坐着一只像素猫。把吃的照片拖到它身上，猫嚼一嚼，一卷羊皮纸从底部滑出来让你写备注，然后照片就被 commit + push 到你的私有 "kibble" 数据仓库里。整个 App 就这么多。

这个 repo 里自带的 Codex skill 现在叫 **Food Diary**。它可以直接处理你发给 Codex 的食物图片、本地图片路径、收据、菜单或文字记录；Kibble app 的 inbox 仍然作为旧入口保留。

![screenshot](docs/screenshot.png)

## 为什么

记录饮食不应该是填表单，应该是一个动作。Kibble 是一种动作，直接把饭图发给 Codex 也是一种动作。Food Diary 负责后续处理：分类、时间、餐别、营养查询、Apple Health 同步和报告。

## 快速开始

### 1. 准备数据仓库

```bash
# 在 GitHub（或别的地方）建一个私有 repo，然后：
git clone git@github.com:you/kibble-data.git ~/kibble-data
```

确认在该目录下 `git push` 可以直接成功，再启动 Kibble。Kibble 不处理认证。

### 2. 配置 Kibble

首次运行 Kibble，会自动在 `~/.config/kibble/config.toml`（Windows 上是 `%APPDATA%\kibble\config.toml`）生成一个带注释的模板，并在控制台输出路径。

编辑 `repo_path`，指向你的本地仓库：

```toml
repo_path = "/Users/you/kibble-data"
```

### 3. 构建 & 运行

```bash
cd app
pnpm install
pnpm tauri dev      # 开发模式
pnpm tauri build    # 打包二进制
```

## 工作流程

1. 把图片拖到猫上。
2. 猫开始嚼，卷轴滑出。写备注（或者不写）。按 Enter。
3. Kibble 把图片复制到 `<repo_path>/inbox/`，如果有备注就写一个同名 `.note.txt`，然后在该目录里 `git add -A && git commit && git push`。
4. 成功猫露出满足表情，失败猫一脸困惑。详细错误打到 stderr。

下游的 Skill（见 `skill/SKILL.md`）负责其他所有事情。默认入口是直接发送给 Codex 的图片或本地图片路径；只有你明确要求处理旧的 Kibble inbox 时，它才会读取 `<repo_path>/inbox/`。

## Food Diary skill

自带 skill 位于 `skill/`，名字是 `food-diary`。

适合处理：

- 直接发给 Codex 的食物照片或截图。
- 本地图片路径。
- 收据、菜单、外卖截图。
- 纯文字饮食记录。
- 旧的 Kibble inbox。
- Apple Health 同步和饮食/健康 Word 报告。

## 数据仓库里的文件结构

```
kibble-data/
└── inbox/
    ├── IMG_1234.jpg
    ├── IMG_1234.jpg.note.txt    # 只有当你写了备注时才有
    └── IMG_1235.jpg
```

原文件名保留。重名追加 `_1`、`_2` 后缀。

## License

MIT

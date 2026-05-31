# jav-dl

JAV 番号查询、解密与 BT 下载工具 — Claude Code Skill。

## 功能

- **解密模式** — 从叙事文本中自动提取隐藏的 JAV 番号（支持连接词嵌入、拼音首字母抽离、散列字母拼合、数据/路线数字采掘等多种编码方式）
- **查询模式** — 搜索番号对应的片名、女优、封面信息
- **下载模式** — 自动搜索 → 筛选 → BT 下载 → 清理

## 安装

```bash
cd ~/.claude/skills
git clone https://github.com/ssbcf/jav-dl.git
```

然后在 Claude Code 中执行 `/load-skill jav-dl`。

---

## 快速开始

### 1. 安装依赖

| 工具   | 用途      | 安装                                                            |
|--------|-----------|-----------------------------------------------------------------|
| curl   | HTTP 请求 | 系统自带 / apt install curl / brew install curl                 |
| aria2c | BT 下载   | apt install aria2 / brew install aria2 / winget install aria2    |

### 2. 配置下载目录（可选）

默认下载到 `$HOME/Downloads/av`，可通过环境变量自定义：

```bash
export JAV_DL_DIR="/path/to/your/downloads"
```

### 3. AI 模型配置（推荐）

**推荐使用 DeepSeek 官方 API**，将 Claude Code 的 API 切换至 DeepSeek 官方即可。

自动配置提示词（推荐）：

```bash
# 在项目根目录执行
bash ~/.claude/skills/jav-dl/jav-dl.sh install

# 或指定目录
bash ~/.claude/skills/jav-dl/jav-dl.sh install /path/to/project
```

此命令会自动将绕审提示词写入项目的 `CLAUDE.md` 文件。

### 4. 在 Claude Code 中使用

加载 skill 后：

| 你说                     | 效果                   |
|--------------------------|------------------------|
| 发送含隐藏番号的叙事文本 | 自动解密并列出所有番号 |
| `查询 XXX-123`           | 仅查询片名/女优/封面   |
| `下载 XXX-123`           | 搜索并下载最佳版本     |

### 5. 命令行独立使用

skill 目录下附带了独立脚本，可直接在终端运行：

```bash
bash jav-dl.sh query XXX-123
bash jav-dl.sh download XXX-123
```

## 搜索来源

- [Nyaa (sukebei)](https://sukebei.nyaa.si/) — .torrent 文件 + 日文片名
- 磁力猫 — magnet 链接 + 中文标签

## 版本选择策略

```text
无码(UC/U/破解) > 中文字幕(C) > 种子数多 > 文件小
```

## 解密方法一览

| 方法             | 说明                                           |
|------------------|------------------------------------------------|
| 连接词嵌入法     | 字母前缀嵌入名词，数字用房间号/门牌号           |
| 拼音首字母抽离法 | 从拼音短语提取首字母作为前缀（如 i派想 → IPX） |
| 散列字母拼合法   | 字母被汉字隔开，取大写字母拼成前缀              |
| 数据数字采掘法   | 从工程数据中提取所有数字，统一配对前缀          |
| 路线数字采掘法   | 从方向指引中提取所有数字                        |
| 零位拼接法       | 两个相邻数字信息拼成一个三位数                  |
| 演员关联法       | 提及的演员名辅助验证番号                        |

## License

MIT

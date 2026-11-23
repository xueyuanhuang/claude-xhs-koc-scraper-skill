# XHS KOC Scraper

> 通过关键词批量搜索小红书，提取符合画像的 KOC 候选人（含 bio/粉丝数），输出到飞书表格

## Trigger

- `/xhs-koc-scraper [画像描述]`
- "帮我从小红书找一批 KOC"
- "小红书上搜一下符合 XX 画像的博主"

## 前置条件

- `opencli` 已全局安装（`npm install -g @jackwener/opencli`）
- Chrome 已安装 OpenCLI Browser Bridge 扩展
- `opencli doctor` 三项全绿
- `lark-cli` 已配置（用于同步飞书表格）

## Input

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| 画像描述 | 是 | - | 目标人群的特征描述 |
| 参考案例链接 | 否 | - | 小红书用户/笔记链接，用于分析共性 |
| 粉丝数阈值 | 否 | 1000 | 最低粉丝数筛选 |
| 目标人数 | 否 | 100 | 期望产出的候选人数量 |

## Pipeline

### Step 1: 分析画像 & 提炼关键词

**如果用户提供了参考案例链接：**

```bash
# 笔记详情
opencli xiaohongshu note "<note_url>" -f json

# 用户帖子列表
opencli xiaohongshu user "<user_url>" -f json
```

分析案例共性，提炼出 20-30 个搜索关键词。关键词设计原则：
- 核心词 + 场景词组合（如 "AI 交易"、"Claude 炒股"）
- 工具名 + 领域词（如 "DeepSeek 选股"、"Karpathy 量化"）
- 行为词 + 领域词（如 "蒸馏 投资 skill"、"vibe coding 交易"）
- 避免单字搜索，2-4 个词组合效果最好

**输出关键词列表给用户确认后再进入 Step 2。**

### Step 2: 批量搜索

用 Python 脚本串行执行搜索，**严格控制节奏防风控**。

```python
import subprocess, json, time, random, re, os

NODE_BIN = os.path.expanduser("~/.nvm/versions/node/v22.22.2/bin")
env = {**os.environ, "PATH": NODE_BIN + ":" + os.environ.get("PATH", "")}

def search_xhs(keyword):
    cmd = ["opencli", "xiaohongshu", "search", keyword, "--limit", "20", "-f", "json"]
    r = subprocess.run(cmd, capture_output=True, text=True, timeout=45, env=env)
    if r.returncode == 0 and r.stdout.strip():
        return json.loads(r.stdout.strip())
    return []
```

**关键规则：**
- **没有 `--page` 参数**，每个关键词只返回一页（最多 20 条）
- 每次搜索间隔 **5-8 秒**（`random.uniform(5, 8)`）
- 按 user_id 去重（从 `author_url` 中提取 `/profile/([a-f0-9]+)`）
- 过滤自家账号（如自家品牌官方账号）
- **每搜完一个关键词就增量写文件保存**，防止崩溃丢数据

### Step 3: 拉取用户主页（bio + 粉丝数）

用 `opencli browser open` + `eval` 提取用户主页数据。

```python
JS_EXTRACT = r'''(() => {
  const text = document.body.innerText;
  const nickname = document.title.replace(" - 小红书", "").trim();
  const bioMatch = text.match(/IP属地：[^\n]+\n([\s\S]*?)(\d+)\s*\n\s*关注/);
  const bio = bioMatch ? bioMatch[1].trim() : "";
  const followersMatch = text.match(/(\d+[\.\d]*[万]?)\s*\n\s*粉丝/);
  return JSON.stringify({
    nickname,
    bio: bio.substring(0, 200),
    followers: followersMatch ? followersMatch[1] : ""
  });
})()'''

# 打开页面
subprocess.run(["opencli", "browser", "open", author_url],
               capture_output=True, text=True, timeout=30, env=env)
time.sleep(3)  # 等页面加载

# 提取数据
r = subprocess.run(["opencli", "browser", "eval", JS_EXTRACT],
                   capture_output=True, text=True, timeout=15, env=env)
data = json.loads(r.stdout.strip())
```

**关键规则：**
- author_url 来自搜索结果（带 xsec_token），无需登录即可查看公开信息
- 每个用户实际耗时 **10-12 秒**（open 加载 + sleep(3) + eval + sleep(3-5)）
- 268 个用户约 **50 分钟**，110 个约 **20 分钟**
- **每拉完一个用户就增量写入文件**，不要全存内存
- 完成后执行 `opencli browser close`

### Step 4: 筛选 + 生成合作意向备注

```python
def parse_followers(s):
    s = str(s).strip()
    if '万' in s:
        return int(float(s.replace('万', '')) * 10000)
    return int(re.sub(r'[^\d]', '', s) or 0)

def gen_cooperation_note(title, bio):
    text = (title + " " + bio).lower()
    tags = []
    if any(w in text for w in ["实盘", "实战", "赚", "收益", "盈利", "跑赢"]):
        tags.append("产品体验")
    if any(w in text for w in ["教程", "教你", "如何", "指南", "分享", "心得"]):
        tags.append("内容共创")
    if any(w in text for w in ["大赛", "比赛", "龙虾", "openclaw"]):
        tags.append("大赛参与")
    if any(w in text for w in ["agent", "skill", "蒸馏", "框架", "系统"]):
        tags.append("技术合作")
    if any(w in text for w in ["开源", "github", "工具", "模板"]):
        tags.append("工具推广")
    return "/".join(tags) if tags else "内容共创"
```

筛选条件：`parse_followers(粉丝数) >= 阈值`

### Step 5: 输出到本地 CSV + 飞书表格

**CSV 列结构：**

| 列名 | 说明 |
|------|------|
| 搜索关键词 | 哪个词搜到的 |
| 用户名 | 昵称 |
| 主页URL | 小红书主页链接 |
| 个人简介 | bio |
| 粉丝数 | followers |
| 命中笔记标题 | 搜到的那篇笔记 |
| 笔记URL | 笔记链接 |
| 合作意向备注 | 自动生成，人工可改 |
| 是否已联系 | 空，人工填 |

**本地 CSV：**
```
~/Projects/rf/xhs-koc-candidates.csv
```

**飞书表格：**
```bash
lark-cli sheets +create --title "标题" --headers '[JSON数组]'
lark-cli sheets +append --spreadsheet-token "TOKEN" --values "$(cat values.json)"
```

## Rules

1. **防风控第一**：所有请求严格串行，间隔 5-8 秒随机，遇到异常暂停 30 秒
2. **增量保存**：每完成一个搜索/一个用户主页就写文件，绝不全存内存
3. **脚本开头 import 所有依赖**：`import subprocess, json, time, random, re, csv, os`
4. **PATH 显式设置**：后台 Python 脚本需要把 nvm 的 node bin 加到 PATH
5. **不要传不存在的参数**：opencli xiaohongshu search 没有 `--page`
6. **先跑 3 条验证**：大批量任务前先搜 3 个关键词确认能出数据
7. **关键词先确认**：Step 1 提炼的关键词列表要给用户确认
8. **飞书表格先确认**：写飞书前确认标题和列结构

## Error Handling

| Situation | Action |
|-----------|--------|
| 搜索返回空结果 | 跳过该关键词，继续下一个 |
| opencli browser open 超时 | 跳过该用户，记录失败列表 |
| JS eval 解析失败 | bio 留空，粉丝数留空，继续 |
| 脚本中途崩溃 | 读取已保存的增量文件，从断点继续 |
| opencli doctor 不通 | 提示用户检查 Chrome 扩展和 Daemon |
| 小红书风控（验证码/限流） | 暂停 60 秒后重试，连续 3 次失败则停止 |
| 飞书表格写入失败 | 本地 CSV 已有完整数据，手动重试飞书同步 |

## Examples

### 示例 1：找 AI×投资类 KOC
```
/xhs-koc-scraper AI工具类KOC，对AI有研究、用AI做投资交易、社媒活跃
参考：https://www.xiaohongshu.com/user/profile/xxx
粉丝阈值：1000
目标：100人
```

### 示例 2：找特定产品的用户
```
/xhs-koc-scraper 使用 Claude Code 做量化交易的博主
粉丝阈值：500
目标：50人
```

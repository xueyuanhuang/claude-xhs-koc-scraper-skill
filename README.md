# /xhs-koc-scraper — Claude Code KOC Discovery Skill

通过关键词批量搜索小红书，自动挖掘符合目标画像的 KOC（Key Opinion Consumer）候选人，提取 bio、粉丝数等信息，筛选后输出到飞书表格。

## 效果

告诉 Claude 你要找什么样的人（比如"用 AI 做投资交易的博主"），它会自动：
- 分析案例共性，生成 20-30 个搜索关键词
- 逐个搜索小红书，提取去重后的作者列表
- 打开每个作者的主页，抓取 bio 和粉丝数
- 按粉丝数筛选，自动生成合作意向标签
- 输出到本地 CSV + 飞书表格，方便团队协作标注

一次跑完可以拿到 100+ 符合条件的 KOC 候选人。

## 安装

1. 复制 `skill.md` 到 `~/.claude/skills/xhs-koc-scraper/`
2. 在 `~/.claude/CLAUDE.md` 的 Skills 部分添加：
   ```
   - `/xhs-koc-scraper [画像描述]` — 小红书KOC批量挖掘：分析画像→提炼关键词→OpenCLI批量搜索→拉主页补bio/粉丝数→筛选→飞书表格
   ```

### 前置依赖

- [OpenCLI](https://github.com/jackwener/OpenCLI) + Chrome Browser Bridge 扩展
- [lark-cli](https://github.com/nicepkg/lark-cli)（可选，用于同步飞书表格）

## 使用

```bash
# 找 AI×投资类 KOC
/xhs-koc-scraper 用AI做投资交易的博主，对AI有研究、社媒活跃

# 带参考案例
/xhs-koc-scraper 找和这个博主类似的人 https://www.xiaohongshu.com/user/profile/xxx

# 指定粉丝阈值
/xhs-koc-scraper 量化交易博主，粉丝>5000
```

## 流程

1. **分析画像** — 如有参考案例，先抓取分析共性特征
2. **提炼关键词** — 生成 20-30 个搜索词，确认后执行
3. **批量搜索** — OpenCLI 串行搜索，5-8 秒间隔防风控
4. **拉用户主页** — browser open + JS eval 提取 bio/粉丝数
5. **筛选输出** — 按粉丝数过滤 → 生成合作标签 → CSV + 飞书表格

## 注意事项

- 全量跑 100+ 人约需 40-50 分钟（防风控限速）
- 依赖 OpenCLI 的 Chrome 扩展，需要 Chrome 保持运行
- 搜索结果基于小红书的搜索排序，不保证覆盖所有相关账号

## License

MIT

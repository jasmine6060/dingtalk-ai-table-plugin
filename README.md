# dingtalk-ai-table — Claude Code Plugin

钉钉 AI 表格 / 多维表 (Notable) OpenAPI 集成手册，沉淀自周报收集机器人项目。

包含一个 skill：
- `skills/dingtalk-ai-table` — 涵盖正确的 URL 路径、auth header、必传 operatorId、应用权限、按列类型的值格式，以及一套 fields → records 的诊断流程。

## 安装

在 Claude Code 里执行：

```
/plugin marketplace add <你的-GitHub-账号>/dingtalk-ai-table-plugin
/plugin install dingtalk-ai-table@wuqiong-skills
```

## 本地开发 / 调试

```bash
claude --plugin-dir ./
```

进入会话后 skill 即可被自动触发（描述命中相关场景时）。

## 目录结构

```
.
├── .claude-plugin/
│   ├── plugin.json          # 插件清单
│   └── marketplace.json     # 市场清单（让别人能 add 此仓库）
├── skills/
│   └── dingtalk-ai-table/
│       └── SKILL.md         # 技能正文
└── README.md
```

## 更新

修改 `skills/dingtalk-ai-table/SKILL.md` 后，bump `plugin.json` 与 `marketplace.json` 中的 `version`，提交并推送：

```bash
git add .
git commit -m "feat: <changes>"
git push
```

订阅方执行 `/plugin update dingtalk-ai-table` 即可拉到新版本。

# Clash 模式切换 Web 页 — 设计规格

日期：2026-11-10
状态：已批准（2026-11-10 对话确认）

## 背景

用户用 mihomo（v1.19.28，用户级 systemd 服务）做代理，需要频繁在「规则连接」和「直连」间切换。已有 CLI 脚本 `~/.local/bin/clash-mode`，现需要浏览器页面直接控制，免敲命令。

## 目标

一个极简 Web 页面，打开即显示当前代理模式，点按钮一键切换 direct / rule。脚本保留不删。

## 架构与部署

- 单文件 `index.html`：原生 JS + CSS，无任何外部依赖
- 存放路径：`~/Documents/clash/ui/index.html`
- 复用 mihomo 自带 external-ui 机制：在**当前激活的 profile**（由 `mihomo.service` 的 ExecStart `-f` 指向，当前为 `~/Documents/clash/profiles/1793493955081.yml`）的 `external-controller: 127.0.0.1:9090` 后增加一行 `external-ui: ui`（相对路径，相对 `-d ~/Documents/clash` 解析，即 `~/Documents/clash/ui/`），重启 `mihomo.service` 生效
- 访问地址：`http://127.0.0.1:9090/ui/`（页面与 API 同源，浏览器直接 fetch，无 CORS、无额外进程、随服务自启）
- 未来重拉订阅的合并流程只替换 proxies 块，`external-ui` 字段自然保留，合并脚本无需改动

## 功能（最小集，已与用户确认）

1. 页面加载时 `GET /configs`，解析 `mode` 字段（`rule` / `direct`），以中文显示「规则连接」/「直连」
2. 两个大按钮：点击后 `PATCH /configs`，body `{"mode":"rule"}` 或 `{"mode":"direct"}`
3. PATCH 成功后重新 `GET /configs` 刷新显示，当前模式按钮高亮
4. 错误处理：
   - 初次加载失败 → 显示「无法连接 mihomo API（服务是否在运行？）」
   - 切换失败（PATCH 失败或切换后刷新失败）→ 显示「切换失败：<原因>」
   - PATCH 响应可能为 204 空 body（已知行为），判断只看 HTTP 状态码，不看 body

## 非目标（YAGNI）

- 不做节点列表 / 延迟显示 / 节点切换（用户明确选了最小集）
- 不做登录鉴权（页面与 API 仅监听 127.0.0.1，本机访问）
- 不做轮询自动刷新（加载时查询 + 切换后刷新足够）

## 验证

1. `systemctl --user restart mihomo` 后 `curl -s http://127.0.0.1:9090/ui/` 返回 200 且含 HTML
2. 用 curl 模拟页面调用路径：`GET /configs` 取 mode；`PATCH /configs` 切 direct/rule，确认返回 204 且 mode 生效
3. 浏览器打开 `http://127.0.0.1:9090/ui/` 人工点按验证
4. 确认 `~/.local/bin/clash-mode` 脚本功能不受影响

## 备注

- 工作区 `~/0.code` 非 git 仓库（CLAUDE.md 规定不 git init），本 spec 不做 git 提交
- 实施完成后将 `external-ui` 用法记入记忆 `mihomo-clash-setup`

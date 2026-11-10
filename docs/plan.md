# Clash 模式切换 Web 页 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 一个托管在 mihomo external-ui（`http://127.0.0.1:9090/ui/`）上的极简中文页面，显示当前代理模式并一键切换 direct / rule。

**Architecture:** 单文件 `index.html`（原生 JS + CSS，零依赖）放入 `~/Documents/clash/ui/`；在 mihomo 当前激活 profile 中加一行 `external-ui: ui` 使 mihomo 自行托管该页面。页面与 API 同源，直接用 `fetch` 调 `/configs`（GET 读模式、PATCH 切模式）。

**Tech Stack:** HTML5 + 原生 JavaScript + CSS；mihomo v1.19.28 Clash API（127.0.0.1:9090，无鉴权 secret）。

## Global Constraints

- 当前激活 profile：`/home/eason/Documents/clash/profiles/1793493955081.yml`（由 `mihomo.service` 的 ExecStart `-f` 指向；动手前可用 `grep ExecStart ~/.config/systemd/user/mihomo.service` 复核，若路径变了则改目标文件）
- mihomo API：`http://127.0.0.1:9090`，无 secret；`GET /configs` 返回 JSON 含 `mode`（值为 `rule`/`direct`/`global`）；`PATCH /configs` body `{"mode":"..."}`，成功返回 **204 空 body** —— 判断只许看 HTTP 状态码，不许看 body
- `~/Documents/clash/` 不是 git 仓库：不执行 git 提交；改 profile 前必须先备份
- 页面文案用中文；不引入任何外部 CDN/框架
- `external-ui` 相对路径相对于 mihomo 工作目录 `-d /home/eason/Documents/clash` 解析
- 所有 curl 测试命令里含 `127.0.0.1` 的请求不受 `http_proxy` 环境变量影响（no_proxy 已含 127.0.0.0/8），无需特殊处理

---

### Task 1: 启用 mihomo external-ui

**Files:**
- Modify: `/home/eason/Documents/clash/profiles/1793493955081.yml`（在 `external-controller: 127.0.0.1:9090` 行后插入一行）
- Create: `/home/eason/Documents/clash/ui/`（空目录）

**Interfaces:**
- Consumes: 无
- Produces: `http://127.0.0.1:9090/ui/` 路径由 mihomo 托管，后续任务往该目录放 `index.html` 即被服务

- [ ] **Step 1: 备份当前 profile**

```bash
cp /home/eason/Documents/clash/profiles/1793493955081.yml /home/eason/Documents/clash/profiles/1793493955081.yml.bak-external-ui
```

预期：命令无输出，备份文件存在。

- [ ] **Step 2: 在 profile 中插入 external-ui 配置**

先确认现有 external-controller 行及上下文：

```bash
grep -n -A2 'external-controller' /home/eason/Documents/clash/profiles/1793493955081.yml
```

预期输出包含 `external-controller: 127.0.0.1:9090`。然后执行插入（external-controller 行后加 `external-ui: ui`）：

```bash
sed -i '/^external-controller: 127.0.0.1:9090$/a external-ui: ui' /home/eason/Documents/clash/profiles/1793493955081.yml
```

- [ ] **Step 3: 验证配置已写入且只有一处**

```bash
grep -n 'external-ui' /home/eason/Documents/clash/profiles/1793493955081.yml
```

预期：恰好一行 `external-ui: ui`（行号紧跟在 external-controller 之后）。若出现多行，回滚重来：

```bash
cp /home/eason/Documents/clash/profiles/1793493955081.yml.bak-external-ui /home/eason/Documents/clash/profiles/1793493955081.yml
```

- [ ] **Step 4: 创建 ui 目录并重启 mihomo**

```bash
mkdir -p /home/eason/Documents/clash/ui
systemctl --user restart mihomo
sleep 2
systemctl --user is-active mihomo
```

预期：输出 `active`。

- [ ] **Step 5: 验证 mihomo 配置解析成功、/ui/ 路径已接管**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9090/ui/
```

预期：`404`（目录为空、还没有 index.html —— 404 说明 external-ui 已生效；若返回 `404 page not found` 之类的 mihomo 兜底响应也是 404 状态码）。同时确认服务日志无配置错误：

```bash
journalctl --user -u mihomo -n 5 --no-pager
```

预期：无 `level=error` / `level=fatal`。

---

### Task 2: 创建模式切换页面 index.html

**Files:**
- Create: `/home/eason/Documents/clash/ui/index.html`

**Interfaces:**
- Consumes: Task 1 的 external-ui 托管路径（`/ui/`，API 同源 `/configs`）
- Produces: 浏览器页面。页面行为契约（供 Task 3 验证）：
  - 加载时 `GET /configs` → 显示「当前模式：规则连接 / 直连 / 全局代理」，对应按钮高亮
  - 点按钮 → `PATCH /configs` `{"mode":"rule"|"direct"}` → 成功（2xx/204）后重新 GET 刷新显示
  - 任何 fetch 异常或非成功状态 → 页面显示「切换失败：<原因>」；初次加载失败 → 「无法连接 mihomo API（服务是否在运行？）」

- [ ] **Step 1: 写完整页面文件**

```bash
cat > /home/eason/Documents/clash/ui/index.html <<'EOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Clash 模式切换</title>
<style>
  body { font-family: system-ui, -apple-system, "Segoe UI", sans-serif; background:#1b1e24; color:#e6e8ee; display:flex; align-items:center; justify-content:center; min-height:100vh; margin:0; }
  .card { background:#262a33; border-radius:16px; padding:32px 40px; text-align:center; box-shadow:0 8px 30px rgba(0,0,0,.4); min-width:320px; }
  h1 { font-size:20px; margin:0 0 4px; }
  .status { font-size:14px; color:#9aa3b2; margin-bottom:24px; }
  .btn { display:block; width:100%; padding:14px; margin:12px 0; font-size:16px; border-radius:10px; border:2px solid #3a4050; background:#2f3542; color:#e6e8ee; cursor:pointer; transition:all .15s; }
  .btn:hover { border-color:#6a7690; }
  .btn.active { background:#2f6fed; border-color:#2f6fed; color:#fff; }
  .err { color:#ff7b72; font-size:14px; margin-top:12px; min-height:18px; }
</style>
</head>
<body>
<div class="card">
  <h1>Clash 代理模式</h1>
  <div class="status" id="status">读取中…</div>
  <button class="btn" id="btn-rule" onclick="setMode('rule')">规则连接</button>
  <button class="btn" id="btn-direct" onclick="setMode('direct')">直连</button>
  <div class="err" id="err"></div>
</div>
<script>
const NAMES = { rule: '规则连接', direct: '直连', global: '全局代理' };

async function getMode() {
  const r = await fetch('/configs');
  if (!r.ok) throw new Error('HTTP ' + r.status);
  const d = await r.json();
  return d.mode || 'rule';
}

function render(mode) {
  document.getElementById('status').textContent = '当前模式：' + (NAMES[mode] || mode);
  document.getElementById('btn-rule').classList.toggle('active', mode === 'rule');
  document.getElementById('btn-direct').classList.toggle('active', mode === 'direct');
}

async function setMode(mode) {
  const err = document.getElementById('err');
  err.textContent = '';
  try {
    const r = await fetch('/configs', { method: 'PATCH', body: JSON.stringify({ mode }) });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    render(await getMode());
  } catch (e) {
    err.textContent = '切换失败：' + e.message;
  }
}

(async () => {
  try {
    render(await getMode());
  } catch (e) {
    document.getElementById('err').textContent = '无法连接 mihomo API（服务是否在运行？）';
  }
})();
</script>
</body>
</html>
EOF
```

- [ ] **Step 2: 验证页面可被 mihomo 托管**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9090/ui/
curl -s http://127.0.0.1:9090/ui/ | grep -c 'btn-direct\|Clash 代理模式'
```

预期：第一行 `200`；第二行输出 `3`（标题行 + HTML 按钮行 + JS getElementById 行各含一处匹配）。

- [ ] **Step 3: 验证页面依赖的 API 契约（模拟页面 JS 的调用路径）**

```bash
curl -s http://127.0.0.1:9090/configs | grep -o '"mode":"[a-z]*"'
curl -s -o /dev/null -w '%{http_code}\n' -X PATCH http://127.0.0.1:9090/configs -d '{"mode":"direct"}'
curl -s http://127.0.0.1:9090/configs | grep -o '"mode":"[a-z]*"'
curl -s -o /dev/null -w '%{http_code}\n' -X PATCH http://127.0.0.1:9090/configs -d '{"mode":"rule"}'
curl -s http://127.0.0.1:9090/configs | grep -o '"mode":"[a-z]*"'
```

预期输出依次为：初始 mode、`204`、`"mode":"direct"`、`204`、`"mode":"rule"`。注意 PATCH 的 204 是空 body——页面 JS 已按 `r.ok`（状态码）判断，本步验证的正是这个契约。

---

### Task 3: 端到端验收与记忆更新

**Files:**
- Modify: `/home/eason/.claude/projects/-home-eason-0-code/memory/mihomo-clash-setup.md`（追加 external-ui 记录）
- Modify: `/home/eason/0.code/docs/superpowers/specs/2026-11-10-clash-mode-web-ui-design.md`（实现完成后无内容变化则不动；仅当实现与规格有出入时更新）

**Interfaces:**
- Consumes: Task 2 的页面与 API 契约
- Produces: 无（收尾任务）

- [ ] **Step 1: 确认与 clash-mode 脚本互不干扰**

```bash
clash-mode
clash-mode toggle && clash-mode toggle
```

预期：先显示当前模式，随后两次 `已切换: X -> Y`（脚本功能正常，与页面共用一个 API，无冲突）。

- [ ] **Step 2: 浏览器人工验收（需用户在浏览器操作）**

让用户打开 `http://127.0.0.1:9090/ui/`：
1. 页面显示「当前模式：规则连接」（或直连），对应按钮蓝色高亮
2. 点「直连」→ 高亮切到直连按钮
3. 点「规则连接」→ 高亮切回
4. （可选）`systemctl --user stop mihomo` 后刷新页面 → 显示「无法连接 mihomo API（服务是否在运行？）」；`systemctl --user start mihomo` 后刷新恢复

预期：全部符合。

- [ ] **Step 3: 更新记忆文件**

在 `/home/eason/.claude/projects/-home-eason-0-code/memory/mihomo-clash-setup.md` 的「模式切换脚本」条目后追加：

```markdown
- 模式切换 Web 页（2026-11-10 加）：profile 有 `external-ui: ui`，页面在 `~/Documents/clash/ui/index.html`，浏览器开 `http://127.0.0.1:9090/ui/` 即可一键切 direct/rule（同源直调 /configs API）。重拉订阅合并只换 proxies 块，external-ui 字段自然保留
```

- [ ] **Step 4: 清理备份文件**

```bash
rm /home/eason/Documents/clash/profiles/1793493955081.yml.bak-external-ui
```

预期：无输出。（保留与否二选一：如用户希望保留备份可跳过本步。）

---

## Self-Review 记录

- **Spec 覆盖**：架构（Task 1）、页面功能/错误处理（Task 2 代码含初次加载失败与切换失败两条错误路径）、非目标（无节点列表/轮询/鉴权——页面代码中确无）、验证 4 条（Task 1 Step 5 / Task 2 Step 2-3 / Task 3 Step 1-2）、记忆更新（Task 3 Step 3）——全部有对应任务。
- **占位符扫描**：无 TBD/TODO；所有代码步骤含完整代码；所有命令含预期输出。
- **类型/命名一致性**：`mode` 取值 `rule`/`direct` 在页面、PATCH body、验证命令中一致；页面函数 `getMode()`/`setMode(m)`/`render(mode)` 在 Task 2 定义、Task 3 验收描述中引用一致。

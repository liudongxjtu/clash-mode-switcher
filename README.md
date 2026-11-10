# clash-mode-switcher

在 mihomo（Clash Meta）代理的「规则连接 / 直连」模式间一键切换的两种方式：**Web 页面**和 **CLI 脚本**。两种方式共用一个 API（`PATCH /configs` 的 `mode` 字段），互不干扰。

- 设计文档：`docs/design.md`
- 实现计划：`docs/plan.md`

## 方式一：Web 页面

mihomo 自带 external-ui 机制，一个 `index.html` 即可，零依赖、零额外进程。

### 部署

```bash
# 1. 放页面（mihomo 工作目录假设为 ~/Documents/clash，下同）
mkdir -p ~/Documents/clash/ui
cp index.html ~/Documents/clash/ui/

# 2. 在激活的 profile 里 external-controller 行后加一行：
#    external-ui: ui

# 3. 重启
systemctl --user restart mihomo
```

浏览器打开 **http://127.0.0.1:9090/ui/** 即可查看当前模式并一键切换。

### 坑

- **MetaCubeXD 自动下载**：mihomo 在 `ui` 目录为空且配置重载时会自动下载 MetaCubeXD 面板并清空重建该目录；目录非空则跳过。**切勿清空 `ui` 目录后重启/重载 mihomo**。
- `PATCH /configs` 成功返回 **204 空 body**，判断结果只看 HTTP 状态码。
- 页面与 API 仅监听 127.0.0.1；mihomo API 默认无 secret 且带 `Access-Control-Allow-Origin: *`，如需收紧可给 `external-controller` 加 secret（会连带改动页面与脚本）。

## 方式二：CLI 脚本

```bash
cp clash-mode ~/.local/bin/ && chmod +x ~/.local/bin/clash-mode

clash-mode             # 查看当前模式
clash-mode direct      # 切直连
clash-mode rule        # 切规则代理
clash-mode toggle      # 一键互切
```

## 适用版本

mihomo v1.19.28（实测）；API 与 external-ui 为 Clash Meta 长期稳定功能，其他相近版本应同样适用。

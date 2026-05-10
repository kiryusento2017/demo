## 安装 s-ui 一键脚本

```bash
bash <(curl -Ls https://raw.githubusercontent.com/kiryusento2017/demo/main/s-ui-install.sh)
```
````markdown
# Reality 域名延迟测试工具

一个用于 Reality / VLESS 节点筛选目标域名（SNI）的在线小工具。

可随机生成域名测速脚本，用于快速测试：

- TLS 握手速度
- 域名连通性
- 延迟情况
- Reality 目标站点可用性

---

# 在线使用

## 网页入口

👉 https://kiryusento2017.github.io/demo/

---

# 功能介绍

- 随机生成 Reality 测试域名
- 一键复制 shell 测试命令
- 支持快速筛选低延迟 SNI
- 适用于：
  - VLESS Reality
  - Xray
  - sing-box
  - s-ui

---

# 使用方法

复制网页生成的命令，在 Linux VPS 中执行：

```bash
for d in xxx.com ; do
...
done
````

即可测试目标域名 TLS 延迟。

---

# 推荐用途

适用于：

* 自建节点
* Reality SNI 筛选
* 晚高峰稳定性测试
* 目标站点可用性检测

---

# 技术实现

本项目基于：

* HTML
* CSS
* JavaScript
* GitHub Pages

无需后端即可运行。

---

# 部署方式

本项目可直接托管于 GitHub Pages。

只需：

1. Fork 仓库
2. 开启 GitHub Pages
3. 访问生成的网址

即可使用。

---

# 注意事项

Reality 目标域名建议选择：

* TLS1.3 支持良好
* H2 支持正常
* 延迟低
* 稳定性高
* 大厂域名

部分域名可能：

* 地区不可达
* TLS 握手失败
* 被主动阻断

属于正常现象。

---

# License

MIT

```
```

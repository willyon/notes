# OpenClash：直接在 `config.yaml` 里定制策略组与规则（可立即生效版）

> 适用前提：你目前的 OpenClash 环境里，**直接编辑订阅生成的 `config.yaml` 已验证能生效**；而 Onekey Create / Overwrite 的方案在你的版本未跑通。  
> 本指南只讲“**编辑 config.yaml → 让它生效 → 验证是否命中**”。

---

## 1. 你需要知道的两个按钮：Commit vs Apply

在 OpenWrt / LuCI 里编辑文件通常会出现两个动作：

- **Commit / Commit Settings**：把你在网页里改的内容**写入配置/文件**（保存到磁盘）
- **Apply / Apply Settings**：让服务/进程**重新加载配置并生效**（重载/重启）

**最佳实践：**
1) 改完文件先点 **Commit**（保存）  
2) 然后点 **Apply**（让 OpenClash 重新加载并生效）  

> 只有 Commit 没 Apply：文件改了但运行未必立刻变  
> 只有 Apply 没 Commit：有可能你改的内容没真正保存

---

## 2. 修改 `config.yaml` 的正确位置（最常见的坑）

你要改的主要是两块：

1) `proxy-groups:`（策略组）
2) `rules:`（规则）

### 2.1 策略组写在哪里？

在 `config.yaml` 中找到：

```yaml
proxy-groups:
  ...
```

把你的 OpenAI 策略组加入其中（建议放在靠前位置，便于维护）。

#### 示例：创建一个 OpenAI 专用策略组（手动选择）

> 注意：`proxies:` 里写的是**节点名称**，必须与你配置里 `proxies:` 列表中的名字完全一致（大小写/空格/符号都必须一致）。

```yaml
proxy-groups:
  - { name: '🤖 OpenAI', type: select, proxies: [pro-日本01, pro-日本02, pro-新加坡01, pro-新加坡02, pro-美国01, pro-美国02] }
```

你也可以写成多行 YAML（更清晰）：

```yaml
proxy-groups:
  - name: '🤖 OpenAI'
    type: select
    proxies:
      - pro-日本01
      - pro-日本02
      - pro-新加坡01
      - pro-新加坡02
      - pro-美国01
      - pro-美国02
```

> 两种写法都行：**行内写法**更短，**多行写法**更不容易写错。

---

### 2.2 规则写在哪里？必须放在“更前面”才能抢先生效

找到：

```yaml
rules:
  ...
```

在 `rules:` **开头**（越靠前越优先）加入：

```yaml
rules:
  - 'DOMAIN-SUFFIX,chatgpt.com,🤖 OpenAI'
  - 'DOMAIN-SUFFIX,openai.com,🤖 OpenAI'
```

✅ 推荐覆盖范围说明：

- `DOMAIN-SUFFIX,chatgpt.com`：覆盖 `chatgpt.com` 及所有子域（例如 `ab.chatgpt.com`）
- `DOMAIN-SUFFIX,openai.com`：覆盖 `openai.com` 及所有子域（例如 `chat.openai.com`、`ios.chat.openai.com`）

> 因此通常不再需要单独写 `DOMAIN,chat.openai.com,...`  
> 但保留也不坏，只是“重复”。

---

## 3. 为什么你日志里会看到 `ws.chatgpt.com` 走了某个节点？

ChatGPT 会用到 WebSocket（`ws.chatgpt.com`）、静态资源、API 等不同域名。  
你已经用 `DOMAIN-SUFFIX,chatgpt.com` 覆盖了它的子域，所以按理会命中同一策略组。

如果你发现它走了你不想要的地区（例如香港），通常不是规则没覆盖，而是：

- 你的 `🤖 OpenAI` 组里包含了香港节点（或“自动选择”最终挑了香港）
- 或者你的节点命名/分组不一致导致回退到其它组

**解决思路：**
- 确保 `🤖 OpenAI` 组里**只放**你希望的节点（日本/新加坡/美国等）
- 不要把“自动选择”“香港”“Lite”混进 OpenAI 专组里（除非你明确想要）

---

## 4. 让修改生效：最稳的操作顺序

1) 在 OpenClash 的 **Config Manage**（或你当前编辑 config.yaml 的界面）改完后  
2) 点击 **Commit / Commit Settings**（保存文件）  
3) 点击 **Apply / Apply Settings**（生效）  
4) 回到 OpenClash 主界面，建议额外做一次：
   - **Restart OpenClash**（重启服务，确保完全加载新配置）

---

## 5. 如何验证“规则真的命中 🤖 OpenAI”？

### 5.1 直接看面板（推荐）

在 OpenClash Dashboard / 面板里：

- 打开 **Connections / 连接**
- 访问 ChatGPT
- 看对应连接的 **Rule / Proxy** 是否显示为 `🤖 OpenAI`

### 5.2 看日志关键词（你现在的方式也可以）

你可以在日志里搜索：

- `chatgpt.com`
- `openai.com`
- `ws.chatgpt.com`

然后观察其后续显示的出口节点/策略组名称。

---

## 6. 重要：订阅“自动更新”会不会覆盖你改的内容？

**会。**  
如果你的 `config.yaml` 是订阅自动拉取生成的，那么一旦自动更新订阅：

- 订阅端重新下发了完整 `config.yaml`
- 你的手动修改**可能被覆盖回原样**

### 6.1 你有 3 个现实可用的应对方式（按稳妥程度排序）

#### A) 关闭自动更新（最简单、最稳但要你手动更新）
- 在 OpenClash 的订阅设置里把自动更新关掉
- 你需要更新节点时手动点一次更新，然后再检查你的规则是否仍在

#### B) 每次更新后，把你的规则块“复制粘贴回去”（最符合现状的办法）
- 你已经验证“直接改 config.yaml 有效”
- 那么就把你这段内容保存为“固定片段”，每次更新后快速贴回 `rules:` 开头和 `proxy-groups:` 中

建议你把这两段保存到备忘录（或你的教程文档里），更新后 30 秒即可恢复。

#### C) 用 OpenClash 提供的“自定义规则/附加规则”入口（看版本是否支持）
> 你的版本界面里能看到 “Overwrite Settings → Rules Setting → Custom Clash Rules” 的话，理论上可以把规则放那里以避免改订阅文件。  
> 但你明确说目前未跑通其它方案，所以这里不作为本指南主方案。

---

## 7. 你可以直接复制使用的最小模板（推荐）

把下面两段分别放进 `proxy-groups:` 和 `rules:`（rules 放开头）：

### 7.1 `proxy-groups`（OpenAI 专组）

```yaml
  - name: '🤖 OpenAI'
    type: select
    proxies:
      - pro-日本01
      - pro-日本02
      - pro-新加坡01
      - pro-新加坡02
      - pro-美国01
      - pro-美国02
```

### 7.2 `rules`（放在最前面）

```yaml
  - 'DOMAIN-SUFFIX,chatgpt.com,🤖 OpenAI'
  - 'DOMAIN-SUFFIX,openai.com,🤖 OpenAI'
```

---

## 8. 常见问题排查（快速）

### Q1：为什么规则写了，但还是走别的组？
- 规则没放在 `rules:` 的靠前位置，被前面的规则抢先匹配了  
✅ 解决：把 OpenAI 两条规则放到 `rules:` 最前面

### Q2：为什么写了策略组但面板里找不到？
- `proxy-groups:` YAML 缩进错误 / 引号不匹配
- 或者策略组名与规则引用不一致（规则里写 `🤖 OpenAI`，组名却不完全相同）  
✅ 解决：组名和规则里的策略名必须一字不差

### Q3：节点名写了但无法选择/显示异常？
- `proxies:` 中的节点名字与你填的不一致（常见：多了空格、符号不同）  
✅ 解决：从 `proxies:` 列表里复制节点名粘贴到组里

---

## 9. 最后建议（你当前目标的最佳实践）

- 你现在的目标是：**ChatGPT / OpenAI 稳定走非香港节点**
- 那么最有效且最少折腾的方式就是：
  1) `🤖 OpenAI` 组里只放你想要的节点（JP/SG/US）
  2) `rules:` 开头强制命中 `chatgpt.com` / `openai.com`
  3) 改完 Commit + Apply +（必要时）Restart

这样你就能快速稳定地控制 OpenAI 的出口路线。

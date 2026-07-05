# 吴老师亚马逊父商品流量异动分析 Skill

## 一、这个 Skill 用来做什么

用于按吴老师亚马逊运营逻辑，分析父商品维度的流量上涨或流量下降原因。

它只处理流量异动，不处理转化率异动，不输出优化建议。

适用场景：

- 父 ASIN 或子 ASIN 出现 Sessions / Page Views 增加或减少。
- 周数据分析 Skill 已经判断存在流量异动，需要继续拆解流量原因。
- 需要验证库存、广告花费、Deals、站外推广、变体、Listing 状态、自然关键词、联盟客等因素。

启动示例：

```text
使用吴老师父商品流量异动 Skill，分析 B0XXXXXXXX 在美国站 2026 年 6 月的流量下降原因。
```

```text
用 $kuajing-wulaoshi-amazon-traffic-anomaly 分析这个 ASIN 的流量上涨，只输出命中的可能因素。
```

## 二、首次安装

1. 登录 GitHub 并提交用户名
2. 接受私有仓库邀请
3. 在 Codex 中发出安装指令
4. 重启 Codex 让 Skill 生效

### 1. 登录 GitHub 并提交用户名

首次安装前，需要先将自己的 GitHub 用户名提交给跨境吴老师。

操作步骤：

```text
登录 GitHub
-> 点击右上角头像
-> 选择 Your profile
-> 进入个人主页
-> 复制浏览器地址栏中的完整网址
-> 将网址发送给跨境吴老师
```

注意：

- GitHub 用户名不是邮箱。
- 请不要发送邮箱密码、GitHub 密码或其他授权信息。

### 2. 接受私有仓库邀请

跨境吴老师收到 GitHub 用户名后，会发送私有仓库访问邀请。

用户操作：

```text
登录 GitHub
-> 打开 GitHub 发送的邀请邮件
-> 点击 View invitation
-> 点击 Accept invitation
```

如果可以看到仓库页面和文件列表，说明已经获得访问权限。

### 3. 在 Codex 中发出安装指令

打开 Codex，新建一个对话，输入：

```text
请从以下 GitHub 私有仓库安装吴老师亚马逊父商品流量异动分析 Skill：
https://github.com/defway888-design/kuajing-wulaoshi-amazon-traffic-anomaly-skill
```

按照 Codex 提示完成 GitHub 授权。

### 4. 重启 Codex

安装完成后，关闭并重新打开 Codex，使 Skill 生效。

## 三、运行前 MCP 配置

本 Skill 不包含任何密钥。每个使用者需要配置自己的 MCP。

必需或建议配置：

- 领星 MCP：用于确认父商品、市场、子体、库存、广告花费。
- 卖家精灵 MCP：用于读取 Deals、变体数量、Listing 状态等数据。
- SIF MCP：用于读取自然关键词排名、自然关键词数量、主要关键词搜索趋势。
- 互联网搜索：用于验证站外推广和联盟客推广证据。

领星 MCP 示例配置中只填写自己的 Key：

```json
{
  "mcpServers": {
    "LingXing-MCP": {
      "type": "streamableHttp",
      "url": "https://openmcp.lingxing.com/mcp-servers/lingxing-mcp",
      "headers": {
        "X-Mcp-Key": "填写自己的领星 MCP Key"
      }
    }
  }
}
```

SIF MCP 示例配置中只填写自己的 Key：

```json
{
  "mcpServers": {
    "SIF-MCP": {
      "type": "streamableHttp",
      "url": "https://mcp.sif.com/mcp",
      "headers": {
        "secret-key": "填写自己的 SIF secret-key"
      }
    }
  }
}
```

## 四、执行后会输出什么

Skill 只输出命中的可能因素。

输出结构：

```text
流量异动方向：流量增加/流量减少

可能的因素：

1. 因素名称
   数据来源：领星 MCP/卖家精灵 MCP/SIF MCP/互联网搜索
   数据证据：具体数值、状态、时间或链接证据。
   可能原因：说明为什么它可能造成本次流量变化。
```

不会输出：

- 未命中的排查项。
- 可控因素/不可控因素分组。
- 转化率异常判断。
- Coupon 促销判断。
- 优化建议或运营动作。

## 五、仓库文件

- `SKILL.md`：Skill 主规则。
- `agents/openai.yaml`：Codex 展示与默认启动信息。
- `references/traffic-anomaly-playbook.md`：完整业务逻辑、时间口径、数据验证规则和输出模板。

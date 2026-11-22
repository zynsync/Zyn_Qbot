# Zyn_Qbot
注：源码去release拿！

一个跑在本地、接自己 AI 模型的 QQ 机器人。单文件 Python，没有框架，没有数据库，配置全在网页上点。
做成微agent了（

>  9069 行 Python + 1402 行 HTML ｜ 依赖只有 `requests` 和 `websocket-client`

---
# 使用教程
**一.安装依赖**
```bash
pip install requests websocket-client
```
**二.去release下载admin.html
与Zyn_Qbot.py
或直接下载zip解压**

**三.cd 到文件目录然后运行命令**
```bash
python3 qq_bot.py
```
就能跑起来了！

## 它是干什么的

QQ 官方的机器人接口只能收发消息，不会思考。这个项目接上你自己部署的模型（OpenAI 兼容接口都行），让它能：

- 在群里听懂人话，自己决定要不要回、回什么
- 需要查东西时自己去搜网页、搜图片、读文件
- 记得住之前聊过什么，不会每次从零开始

核心玩法是**让模型自己决定调用哪个工具**，而不是用正则表达式猜用户想干什么。猜错一次就够烦了，不如把判断权交给模型。

---

## 几个实际跑过的东西

**引用消息**

群友用"引用回复"时，模型看到的不是糊在一起的两段话，而是：

```
┌─ 引用 @张三|DCD5058E9DED7680F735390D8A26D48C 的消息
│ 这个方案我坚决反对，谁同意谁傻
└─ 本人回复：你到底同不同意
```

这样它不会把张三的立场当成发言人自己的立场。

**发言人标识**

每条消息前面都挂着 `[昵称|完整openid]`。群里两个人都叫"小明"、或者有人频繁改昵称，靠 openid 分得清。

**回复方式可选**

默认是 @ 对方。也支持改成引用回复——原话挂在上面，多人同时聊天时上下文清楚得多。两种模式网页上点一下切换。

**图片**

`search_image` 工具由模型主动调。会做多引擎降级（Bing → 搜狗 → 360），带相关度校验。Bing 对三个词以上的查询会退化成只搜第一个字，这个坑踩过，所以长查询会自动精简再搜。

**文件**

群友发的 txt/md/json 会自动下载到 `work/uploads/`，文本内容直接读进上下文。模型不会说"我读不了文件"——文件已经在它本地了。

---

## 快速开始

### 环境

```bash
Python 3.8+
pip install requests websocket-client
```

### 跑起来

```bash
python3 qq_bot.py
```

第一次启动没有配置，浏览器打开 `http://localhost:8888`，填四样东西：

| 字段 | 从哪拿 |
|---|---|
| AppID | QQ 开放平台 → 你的机器人 → 开发设置 |
| AppSecret | 同上 |
| 机器人名称 | 随便填，@ 识别会用 |
| AI 接口地址 | 你的模型服务，比如 `http://127.0.0.1:11434/v1` |

保存后自动连接。终端会打出：

```
╔══════════════════════════════════════════════╗
║   Zyn_Qbot  v1.0.0                           ║
╚══════════════════════════════════════════════╝
[WS] READY, session_id=...
```

然后在群里 @ 它试试。

### 要群管理功能的话

禁言、入群审批、踢人这些接口，**需要先把机器人设成群管理员**。不是管理员会返回 `11703`，这不是 bug。

群成员列表、黑名单、群基本信息还需要开放平台的内邀白名单，没申请到会返回 `11253`。代码里都写好了，申请下来打开开关就能用。

---

## 工具清单

模型能调的全部能力，一共 30 个：

**消息**
`send_image` `send_markdown` `reply_quote` `list_quotable` `recall_message` `at_everyone`

**查询**
`get_user_info` `get_group_info` `get_bot_group_state` `get_group_members` `get_group_member_detail` `get_group_mute_state` `get_join_requests`

**群管理**
`set_group_mute` `approve_join_request` `kick_group_member` `get_group_blacklist` `group_blacklist_op`

**联网**
`web_search` `http_get` `search_image` `get_weather` `get_time`

**文件**
`read_file` `write_file` `list_files` `search_files` `download_file` `delete_file` `exec_command`

---

## 配置文件

`qq_bot_config.json` 存所有设置，72 个字段，网页上都能点。常用的几个：

```json
{
  "app_id": "你的 AppID",
  "api_url": "http://127.0.0.1:8790/v1",
  "model_name": "你的模型名",
  "master_openid": "主人的 openid",
  "ai_history_limit": 6,
  "ai_group_reply_mode": "at"
}
```

`qq_memory.json` 是聊天记忆，按群/私聊分开存，超过阈值自动压缩成摘要。

---

## 网页控制台

34 个开关，分 14 个区块：

凭证、手动发送、API 配置、记忆策略、记忆管理、会话提示词、群管理、群成员/黑名单、消息能力、文件处理、事件订阅、AI 协议、调试、分段处理。

改完保存即时生效，改坏了点回去就行。敏感或试验性的功能都默认关着。

---

## 一些设计上的选择

**为什么不用框架**

一开始就想做成能直接 `scp` 到手机上跑的单个文件。现在 9000 行确实有点长，但好处是排查问题的时候不用在十几个模块之间跳。

**为什么用 JSON 决策而不是函数调用**

模型对 JSON 的理解普遍比 function calling 稳，尤其是一些小模型。文本协议和原生 function call 两套都实现了，网页上能切。

**为什么默认关掉正则兜底**

之前做过"用正则猜用户要不要图"的安全网。结果模型说不搜，安全网还是发了张图。这种嘴口不一比不干活更难受。现在改成模型自己判断，正则逻辑留着但默认关闭。

**记忆里为什么要存 openid**

最早只存昵称。后来发现群里两个人同名，模型完全分不清谁是谁。现在每条消息都带完整 openid。

---

## 已知限制

- 群成员列表、黑名单、群信息查询需要内邀白名单，普通开发者拿不到
- 图片和文件只在收到消息时那个链接有效，过时就下不了
- `msg_idx` 只存在内存里，重启后要等有人说话才能引用
---

## 目录

```
qq_bot.py              主程序
admin.html             网页控制台
qq_bot_config.json     配置
qq_memory.json         聊天记忆
qq_messages.json       消息记录
qq_stats.json          运行统计
work/uploads/          群友发来的文件
```

---

## 说明

项目按现状提供，在 Ubuntu 24.04 / Python 3.12 上跑过，其他环境没测。

QQ 开放平台的接口变得挺勤，协议部分跟进到 2026-11。要是哪天突然报错，先去翻官方变更记录，大概率不是代码的问题。

---

## 许可

GPL-3.0。完整条款见 [LICENSE](LICENSE)。

简单说：拿去用、改、商用都行，但如果分发修改后的版本，得同样开源，并且说明改了什么。

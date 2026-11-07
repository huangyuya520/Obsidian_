## hermes agent
```
hermes             #交互式CLI -启动对话
hermes model        #选择您的LLM提供商和模型
hermes tools         #配置启用哪些工具
/reasoning /reasoning <none|minimal|low|medium|high|xhigh|show|hide> 切换agent思考程度

hermes config set <xxx>       #设置单个配置值
hermes config           #查看当前配置
hermes config edit      #打开 config.yaml

hermes gateway        #启动消息网关(Telegram, Discord,etc)
hermes setup gateway      #添加通讯渠道
hermes gateway start / stop / restart   #启动、停止或重启网关服务
hermes gateway status         #查看网关连接状态

hermes setup          #运行完整的安装向导（一次配置所有内容）
hermes claw migrate          #从OpenClaw迁移数据（如果来自OpenClaw）

hermes update           #更新到最新版本
hermes doctor           #诊断任何问题

hermes setup model     #增加配模型服务商置


hermes sessions list         #列出历史会话
hermes --resume <会话名>        #返回上一次会话

hermes cache clean         #清理缓存和临时文件

```

```
hermes skills browse               # 浏览所有可用 skill
hermes skills search <关键词>       # 搜索
hermes skills install <skill-id>   # 安装
hermes skills inspect <skill-id>   # 预览内容（不安装）
hermes skills list                 # 查看已安装
hermes skills update               # 更新全部skill

```

## hermes-web-ui
- **常用命令**
**hermes-web-ui status    查看hermes-web-ui启动状态**
**hermes-web-ui stop       停止启动hermes-web-ui**
**hermes-web-ui start --port 8648     在指定端口启动hermes-web-ui**
**hermes-web-ui update    hermes-web-ui更新**














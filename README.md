# 薪事力 AI Agent Skills

为 AI Agent 提供薪事力能力的 Skill 集合，目前支持智能人事查询、发起流程等功能。

## 安装

```bash
npx skills add x-dhr/skills
```

## 配置

推荐在宿主中配置名为 `x-dhr` 的 MCP server。需要降级使用 curl 调用 MCP HTTP 接口时，再设置薪事力 API Token：

1. 前往 [API Key 管理页面](https://yourdomain/n/skills) 获取你的 API Token
2. 设置环境变量：
   - X_DHR_MCP_URL：MCP 端点地址
   - X_DHR_MCP_TOKEN：API Token

> 注意：API Token 绑定用户身份，请妥善保管。

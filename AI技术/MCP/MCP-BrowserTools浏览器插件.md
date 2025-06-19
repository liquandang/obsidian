https://browsertools.agentdesk.ai/installation

BrowserTools 使 AI 代码编辑器和智能体（agent）能够监控您的网页浏览器并与之交互，从而实现高效调试和更流畅的开发者体验——整个过程安全可靠。

借助这款 MCP 服务器工具，您可以授权 AI 代码编辑器和智能体访问以下内容：

- 控制台日志和错误
- XHR 网络请求/响应
- <mark style="background: #BBFABBA6;">屏幕截图功能</mark>（可选择自动粘贴到 Cursor 中）
- 当前选中的 DOM 元素
- 通过 Lighthouse 运行 SEO、性能和代码质量扫描
- 运行针对 NextJS 的特定 SEO 审计
- 进入“调试模式”，该模式会使用多种工具和提示来修复 bug
- 进入“审计模式”，以执行全面的 Web 应用审计

1、下载插件
https://github.com/AgentDeskAI/browser-tools-mcp/releases/download/v1.2.0/BrowserTools-1.2.0-extension.zip

2、浏览器安装
![[image-1 1.png]]

![[image-3 1.png]]


3、mcpclient设置server，可选择cusor、cline等
```
npx @agentdeskai/browser-tools-mcp@1.2.0

{
	"mcpServers": {
	
		"browser-tools": {
		
			"command": "npx",
			
			"args": [
			
				"-y",
				
				"@agentdeskai/browser-tools-mcp@1.2.0"
			]
		}
	}
}
```

4、本地连接mcpserver
```
npx @agentdeskai/browser-tools-server@1.2.0
```

![[image-2 1.png]]

5、cursor等mcpclient执行query

![[image-4 1.png]]

# Claude Code CLI 源码架构分析

> ⚠️ **声明：本文为社区学习研究文档，非 Anthropic 官方出品。** 内容基于对 Claude Code 源码的阅读分析，仅供技术学习参考，不代表 Anthropic 立场。Claude Code 是 Anthropic 的商业产品，相关商标和版权归 Anthropic 所有。

---

## 项目概述

Claude Code 是一个基于 **React + Ink** 构建的终端 AI 编程助手。它允许用户在终端中与 Claude AI 进行交互式对话，执行文件操作、运行命令、管理项目等任务。

**核心特性：**
- 基于 React/Ink 的终端 UI（支持鼠标、文本选择、搜索）
- 43+ 种工具（文件读写、编辑、Shell、Agent 等）
- 支持子 Agent 并行执行任务
- MCP（Model Context Protocol）扩展支持
- 持久化记忆系统（MEMORY.md）
- 完整的 Vim 模式支持
- 语音输入模式
- 远程控制桥接（CCR）

---

## 项目架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      User Interface Layer                       │
├─────────────────────────────────────────────────────────────────┤
│  REPL.tsx          →  主交互界面                                │
│  PromptInput       →  文本输入（Vim 模式、自动补全）             │
│  Messages          →  消息列表（虚拟滚动、搜索）                 │
│  Permission Dlg    →  工具权限请求对话框                         │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    Query Engine (Core)                           │
├─────────────────────────────────────────────────────────────────┤
│  query.ts        →  核心对话循环（流式 API + 工具执行）          │
│  QueryEngine.ts  →  SDK / 无头模式封装                          │
│  Autocompact     →  上下文自动压缩                               │
│  Token Budget    →  Token 预算管理                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                        Tool System                               │
├─────────────────────────────────────────────────────────────────┤
│  File Tools   : Read / Edit / Write / Glob / Grep               │
│  Shell Tools  : Bash / PowerShell                               │
│  Agent Tools  : TaskCreate / AgentTool / TeamCreate             │
│  Web Tools    : WebFetch / WebSearch                            │
│  System Tools : LSPTool / MCPTool / ConfigTool                  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                      Services Layer                              │
├─────────────────────────────────────────────────────────────────┤
│  services/api/         →  Anthropic API 客户端                   │
│  services/mcp/         →  MCP 连接管理                           │
│  services/lsp/         →  语言服务器协议集成                      │
│  services/analytics/   →  遥测和分析                              │
│  services/SessionMemory/ → 会话记忆提取                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                     State & Storage                              │
├─────────────────────────────────────────────────────────────────┤
│  AppState         →  全局状态管理（Zustand-like）                │
│  Memory System    →  MEMORY.md + 持久化记忆                      │
│  Settings         →  用户配置管理                                 │
│  Session History  →  会话历史记录                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 目录结构

```
Claude Code源码/
│
├── main.tsx                        # CLI 主入口（Commander.js, ~4700行）
├── Tool.ts                         # Tool 类型定义 + buildTool() 工厂
├── Task.ts                         # Task 类型定义
├── tools.ts                        # 工具注册表
├── tasks.ts                        # 任务注册表
├── commands.ts                     # 命令注册表（100+ 斜杠命令）
├── query.ts                        # 核心查询循环（~1700行）
├── QueryEngine.ts                  # SDK/无头查询引擎（~1300行）
├── context.ts                      # 上下文管理（git状态、CLAUDE.md）
├── setup.ts                        # 会话设置
├── history.ts                      # 命令历史（JSONL存储）
├── cost-tracker.ts                 # Token/费用跟踪
├── costHook.ts                     # 费用摘要 Hook
├── projectOnboardingState.ts       # 新项目引导状态
├── dialogLaunchers.tsx             # 对话框启动器
├── interactiveHelpers.tsx          # 交互式UI工具
├── replLauncher.tsx                # REPL启动器
├── ink.ts                          # Ink 公共API门面
│
├── entrypoints/                    # 入口点
│   ├── cli.tsx                     #   实际入口（--version等快速路径）
│   ├── mcp.ts                      #   MCP 服务器模式
│   ├── init.ts                     #   初始化逻辑
│   ├── agentSdkTypes.ts            #   SDK 类型导出
│   ├── sandboxTypes.ts             #   沙箱配置 Schema
│   └── sdk/                        #   SDK 核心
│       ├── coreTypes.ts            #     核心类型
│       ├── coreSchemas.ts          #     核心 Schema
│       └── controlSchemas.ts       #     控制 Schema
│
├── bootstrap/
│   └── state.ts                    # 全局可变状态单例（~1760行）
│
├── screens/                        # 主界面
│   ├── REPL.tsx                    #   主 REPL 界面（~5000行）
│   ├── Doctor.tsx                  #   诊断界面（/doctor）
│   └── ResumeConversation.tsx      #   会话恢复
│
├── components/                     # UI 组件（144+个）
│   ├── PromptInput/                #   主文本输入（~2340行主文件, Vim/自动补全）
│   ├── Messages.tsx                #   消息列表渲染
│   ├── VirtualMessageList.tsx      #   虚拟滚动容器
│   ├── Message.tsx / MessageRow.tsx
│   ├── Spinner.tsx / Spinner/      #   加载动画（12文件）
│   ├── Markdown.tsx                #   Markdown渲染
│   ├── ModelPicker.tsx             #   模型选择
│   ├── ThemePicker.tsx             #   主题选择
│   ├── Onboarding.tsx              #   首次运行引导
│   ├── ExitFlow.tsx                #   退出确认
│   ├── TaskListV2.tsx              #   任务列表
│   ├── LogSelector.tsx             #   会话选择器
│   ├── messages/                   #   ★ 各类型消息渲染（33文件: AssistantText/UserToolResult等）
│   ├── design-system/              #   设计原语（Dialog/Pane/FuzzyPicker/Tabs等）
│   ├── permissions/                #   权限请求对话框（Bash/File/Edit等）
│   ├── mcp/                        #   MCP 对话框
│   ├── sandbox/                    #   沙箱配置UI（5文件）
│   ├── Settings/                   #   设置页面（Config/Settings/Status/Usage）
│   ├── diff/                       #   Diff 显示
│   ├── grove/                      #   Grove 集成
│   ├── agents/                     #   Agent 创建向导
│   ├── skills/                     #   技能 UI
│   ├── tasks/                      #   任务 UI
│   ├── teams/                      #   团队 UI
│   ├── shell/                      #   Shell 组件
│   ├── hooks/                      #   共享 Hooks
│   ├── ui/                         #   基础 UI 组件
│   ├── memory/                     #   记忆管理 UI
│   ├── CustomSelect/               #   自定义选择器
│   ├── FeedbackSurvey/             #   反馈调查
│   ├── HighlightedCode/            #   代码高亮
│   ├── StructuredDiff/             #   结构化差异
│   ├── LogoV2/                     #   Logo/欢迎页/提示系统
│   ├── TrustDialog/                #   工作区信任对话框
│   ├── Passes/                     #   Pass 卡券 UI
│   ├── DesktopUpsell/              #   桌面版推广
│   ├── HelpV2/                     #   帮助界面
│   ├── LspRecommendation/          #   LSP 推荐
│   ├── ManagedSettingsSecurityDialog/
│   ├── ClaudeCodeHint/
│   └── wizard/                     #   向导组件
│
├── ink/                            # Ink 终端 UI 框架（自定义fork）
│   ├── ink.tsx                     #   核心渲染引擎（~1720行）
│   ├── reconciler.ts               #   React 协调器
│   ├── dom.ts                      #   自定义 DOM 实现
│   ├── screen.ts                   #   屏幕缓冲区（差分渲染）
│   ├── renderer.ts                 #   帧渲染器
│   ├── root.ts                     #   createRoot API
│   ├── selection.ts                #   文本选择状态机
│   ├── searchHighlight.ts          #   搜索高亮
│   ├── events/                     #   事件系统（键盘/鼠标/焦点/点击）
│   ├── layout/                     #   Yoga Flexbox 布局引擎
│   │   ├── engine.ts
│   │   ├── node.ts
│   │   ├── geometry.ts
│   │   └── yoga.ts
│   ├── components/                 #   基础组件（Box/Text/Button/Link/ScrollBox等）
│   ├── hooks/                      #   Ink Hooks（useInput/useApp/useStdin等）
│   └── termio/                     #   终端I/O（ANSI解析: sgr/osc/csi/dec/esc）
│
├── tools/                          # 43+ 个工具实现
│   ├── BashTool/                   #   Shell 命令执行（沙箱/安全/权限）
│   ├── PowerShellTool/             #   Windows PowerShell
│   ├── FileReadTool/               #   文件读取（图片/行偏移/限制）
│   ├── FileEditTool/               #   文件编辑（搜索/替换块）
│   ├── FileWriteTool/              #   文件写入/创建
│   ├── GlobTool/                   #   文件模式匹配
│   ├── GrepTool/                   #   内容搜索（正则）
│   ├── AgentTool/                  #   子Agent（含内置: explore/plan/verify等）
│   │   ├── built-in/               #     内置Agent定义
│   │   ├── runAgent.ts             #     Agent运行器
│   │   ├── forkSubagent.ts         #     子Agent分叉
│   │   └── agentMemory.ts          #     Agent记忆
│   ├── REPLTool/                   #   REPL 模式封装 + 原始工具集
│   ├── TaskCreateTool/             #   创建后台任务
│   ├── TaskGetTool/                #   获取任务结果
│   ├── TaskUpdateTool/             #   更新任务状态
│   ├── TaskListTool/               #   列出所有任务
│   ├── TaskStopTool/               #   停止任务
│   ├── TaskOutputTool/             #   任务输出
│   ├── TodoWriteTool/              #   待办事项管理
│   ├── WebFetchTool/               #   网页获取
│   ├── WebSearchTool/              #   网页搜索
│   ├── LSPTool/                    #   语言服务器操作
│   ├── MCPTool/                    #   MCP 工具调用
│   ├── McpAuthTool/                #   MCP 认证
│   ├── ListMcpResourcesTool/       #   列出MCP资源
│   ├── ReadMcpResourceTool/        #   读取MCP资源
│   ├── SkillTool/                  #   技能调用
│   ├── BriefTool/                  #   摘要/附件生成
│   ├── ConfigTool/                 #   配置管理
│   ├── SendMessageTool/            #   发送消息给Agent
│   ├── TeamCreateTool/             #   创建团队
│   ├── TeamDeleteTool/             #   删除团队
│   ├── EnterPlanModeTool/          #   进入计划模式
│   ├── ExitPlanModeTool/           #   退出计划模式
│   ├── EnterWorktreeTool/          #   进入Git Worktree
│   ├── ExitWorktreeTool/           #   退出Git Worktree
│   ├── NotebookEditTool/           #   Jupyter Notebook编辑
│   ├── AskUserQuestionTool/        #   交互式用户提问
│   ├── ToolSearchTool/             #   工具搜索/延迟加载
│   ├── ScheduleCronTool/           #   定时任务（创建/删除/列表）
│   ├── RemoteTriggerTool/          #   远程Agent触发
│   ├── SyntheticOutputTool/        #   内部输出合成
│   ├── SleepTool/                  #   Agent休眠
│   ├── shared/                     #   共享工具（多Agent/git跟踪）
│   └── testing/                    #   测试权限工具
│
├── tasks/                          # 任务系统实现
│   ├── types.ts                    #   TaskState 联合类型
│   ├── stopTask.ts                 #   共享停止逻辑
│   ├── pillLabel.ts                #   底部标签生成
│   ├── LocalMainSessionTask.ts     #   主会话后台化（Ctrl+B）
│   ├── LocalShellTask/             #   后台Shell任务
│   ├── LocalAgentTask/             #   本地子Agent任务
│   ├── RemoteAgentTask/            #   远程云Agent（ultraplan等）
│   ├── InProcessTeammateTask/      #   进程内队友（Swarm成员）
│   └── DreamTask/                  #   自动梦境（记忆整理）
│
├── services/                       # 服务层（36个条目：20目录 + 16独立文件）
│   ├── api/                        #   Anthropic API客户端（~3420行, 流式）
│   ├── mcp/                        #   MCP连接管理（OAuth/传输/VS Code SDK）
│   ├── lsp/                        #   语言服务器协议（诊断/被动反馈）
│   ├── analytics/                  #   遥测（Datadog/GrowthBook/1P事件）
│   ├── SessionMemory/              #   后台会话记忆提取（forked子Agent）
│   ├── autoDream/                  #   夜间记忆整理
│   ├── compact/                    #   上下文压缩
│   ├── MagicDocs/                  #   文档处理
│   ├── plugins/                    #   插件安装管理
│   ├── teamMemorySync/             #   团队记忆同步
│   ├── settingsSync/               #   设置同步
│   ├── tips/                       #   提示注册与调度
│   ├── policyLimits/               #   使用策略限制
│   ├── PromptSuggestion/           #   提示自动补全
│   ├── oauth/                      #   OAuth 流程
│   ├── tools/                      #   工具使用摘要
│   ├── extractMemories/            #   记忆提取服务
│   ├── remoteManagedSettings/      #   远程托管配置
│   ├── AgentSummary/               #   Agent摘要
│   ├── toolUseSummary/             #   工具使用统计
│   ├── voice.ts                    #   音频录制（cpal/SoX/arecord）
│   ├── voiceStreamSTT.ts           #   语音流式STT
│   ├── voiceKeyterms.ts            #   关键词提取
│   ├── notifier.ts                 #   OS通知服务
│   ├── preventSleep.ts             #   防止系统休眠
│   ├── vcr.ts                      #   录制/回放
│   ├── awaySummary.ts              #   离开摘要
│   ├── mcpServerApproval.tsx       #   MCP服务器审批UI
│   ├── claudeAiLimits.ts           #   Claude AI 使用限制
│   ├── claudeAiLimitsHook.ts       #   使用限制Hook
│   ├── tokenEstimation.ts          #   Token估算
│   ├── diagnosticTracking.ts       #   诊断追踪
│   ├── internalLogging.ts          #   内部日志
│   ├── rateLimitMessages.ts        #   速率限制消息
│   ├── rateLimitMocking.ts         #   速率限制模拟
│   └── mockRateLimits.ts           #   模拟速率限制
│
├── state/                          # 状态管理
│   ├── AppStateStore.ts            #   AppState 类型定义（~569行）
│   ├── AppState.tsx                #   React Provider + Hooks
│   ├── store.ts                    #   通用 createStore（Zustand-like）
│   ├── selectors.ts                #   派生状态选择器
│   ├── onChangeAppState.ts         #   状态变更副作用
│   └── teammateViewHelpers.ts      #   队友视图状态机
│
├── query/                          # 查询系统内部
│   ├── deps.ts                     #   依赖注入（callModel/compact/uuid）
│   ├── config.ts                   #   不可变查询配置快照
│   ├── stopHooks.ts                #   停止Hook执行（~473行）
│   └── tokenBudget.ts             #   Token预算跟踪
│
├── context/                        # React Context Providers
│   ├── mailbox.tsx                 #   组件间异步消息（pub/sub）
│   ├── notifications.tsx           #   优先级通知队列
│   ├── overlayContext.tsx           #   Overlay Escape键协调
│   ├── modalContext.tsx            #   模态对话框尺寸
│   ├── QueuedMessageContext.tsx     #   排队消息布局
│   ├── promptOverlayContext.tsx     #   提示浮层Portal
│   ├── fpsMetrics.tsx              #   FPS性能监控
│   ├── voice.tsx                   #   语音输入状态
│   └── stats.tsx                   #   内存统计（计数器/直方图/集合）
│
├── memdir/                         # 持久化记忆系统
│   ├── memdir.ts                   #   核心记忆管理（~507行）
│   ├── memoryTypes.ts              #   四种类型: user/feedback/project/reference
│   ├── paths.ts                    #   记忆目录路径
│   ├── teamMemPaths.ts             #   团队记忆路径
│   ├── teamMemPrompts.ts           #   团队记忆提示
│   ├── findRelevantMemories.ts     #   记忆搜索/相关性
│   ├── memoryScan.ts               #   记忆文件扫描
│   └── memoryAge.ts                #   记忆年龄追踪
│
├── bridge/                         # 远程控制桥（31文件）
│   ├── bridgeMain.ts               #   主轮询循环（~3000行）
│   ├── bridgeApi.ts                #   环境 API 客户端
│   ├── sessionRunner.ts            #   子进程会话生成
│   ├── replBridge.ts               #   REPL 集成
│   ├── initReplBridge.ts           #   REPL 桥初始化
│   ├── bridgeUI.ts                 #   实时状态显示
│   ├── bridgeConfig.ts             #   桥接配置
│   ├── pollConfig.ts               #   轮询间隔配置
│   ├── jwtUtils.ts                 #   JWT 令牌刷新
│   ├── workSecret.ts               #   工作密钥解码
│   ├── remoteBridgeCore.ts         #   核心远程桥逻辑
│   ├── createSession.ts            #   会话创建
│   ├── codeSessionApi.ts           #   CCR v2 API
│   ├── types.ts                    #   协议类型定义
│   └── ... (其他: 消息路由/权限/调试)
│
├── remote/                         # 远程会话客户端
│   ├── RemoteSessionManager.ts     #   WebSocket管理/权限/重连（~343行）
│   ├── SessionsWebSocket.ts        #   会话订阅/心跳/重连（~404行）
│   ├── sdkMessageAdapter.ts        #   SDK消息适配
│   └── remotePermissionBridge.ts   #   远程权限桥接
│
├── upstreamproxy/                  # CCR容器代理
│   ├── upstreamproxy.ts            #   MITM代理初始化（~285行）
│   └── relay.ts                    #   TCP隧道/WebSocket中继（~455行）
│
├── coordinator/
│   └── coordinatorMode.ts          # 协调者/工作者多Agent模式（~369行）
│
├── server/                         # 直连服务器
│   ├── directConnectManager.ts     #   WebSocket管理/消息路由
│   ├── createDirectConnectSession.ts
│   └── types.ts                    #   会话状态/配置类型
│
├── assistant/
│   └── sessionHistory.ts           # 会话事件历史分页
│
├── plugins/                        # 插件系统
│   ├── builtinPlugins.ts           # 内置插件注册表
│   └── bundled/index.ts            # 捆绑插件初始化
│
├── skills/                         # 技能系统
│   ├── bundledSkills.ts            # 捆绑技能注册表
│   ├── loadSkillsDir.ts            #   技能目录加载器（~1086行）
│   ├── mcpSkillBuilders.ts         #   MCP技能构建器（循环断路）
│   └── bundled/index.ts            #   12+个内置技能初始化
│
├── commands/                       # 100+ 斜杠命令（86目录 + 15独立文件）
│   │
│   │ # ── 会话管理 ──
│   ├── clear/                      #   清空对话历史
│   ├── compact/                    #   压缩对话（保留摘要）
│   ├── resume/                     #   恢复之前的对话
│   ├── rename/                     #   重命名对话
│   ├── rewind/                     #   回退到之前的检查点
│   ├── branch/                     #   创建对话分支
│   ├── session/                    #   显示远程会话URL/二维码
│   ├── exit/                       #   退出 REPL
│   │
│   │ # ── 模型配置 ──
│   ├── model/                      #   设置 AI 模型
│   ├── effort/                     #   设置推理努力级别
│   ├── fast/                       #   切换快速模式
│   ├── advisor.ts                  #   配置顾问模型
│   ├── brief.ts                    #   切换简洁模式
│   │
│   │ # ── Git/PR ──
│   ├── commit.ts                   #   自动生成 git commit
│   ├── commit-push-pr.ts           #   提交+推送+开PR一步完成
│   ├── diff/                       #   查看未提交变更
│   ├── review.ts                   #   审查 Pull Request
│   ├── review/                     #   审查（目录版）
│   ├── security-review.ts          #   安全审查代码变更
│   ├── pr_comments/                #   获取PR评论
│   │
│   │ # ── 规划 ──
│   ├── plan/                       #   计划模式
│   ├── ultraplan.tsx               #   远程多Agent规划
│   ├── init-verifiers.ts           #   创建验证技能
│   ├── btw/                        #   快速旁问
│   │
│   │ # ── 配置 ──
│   ├── config/                     #   配置面板
│   ├── permissions/                #   工具权限规则
│   ├── keybindings/                #   快捷键配置
│   ├── hooks/                      #   Hook 配置
│   ├── memory/                     #   编辑记忆文件
│   ├── mcp/                        #   MCP 服务器管理
│   ├── sandbox-toggle/             #   沙箱模式
│   ├── theme/                      #   终端主题
│   ├── color/                      #   提示栏颜色
│   ├── vim/                        #   Vim 模式切换
│   ├── voice/                      #   语音模式切换
│   ├── output-style/               #   输出样式（已弃用）
│   ├── statusline.tsx              #   状态栏 UI 设置
│   ├── terminalSetup/              #   安装终端键绑定
│   │
│   │ # ── 账户 ──
│   ├── login/                      #   登录 Anthropic
│   ├── logout/                     #   登出
│   ├── upgrade/                    #   升级到 Max 计划
│   ├── usage/                      #   显示用量限制
│   ├── cost/                       #   显示会话费用
│   ├── extra-usage/                #   超额计费配置
│   ├── privacy-settings/           #   隐私设置
│   ├── rate-limit-options/         #   速率限制选项（内部）
│   │
│   │ # ── 扩展 ──
│   ├── plugin/                     #   插件管理
│   ├── skills/                     #   列出技能
│   ├── reload-plugins/             #   重新加载插件
│   ├── agents/                     #   Agent 配置管理
│   │
│   │ # ── 项目初始化 ──
│   ├── init.ts                     #   初始化 CLAUDE.md
│   ├── install.tsx                 #   安装原生版本
│   ├── install-github-app/         #   设置 GitHub Actions
│   ├── install-slack-app/          #   安装 Slack 应用
│   ├── doctor/                     #   诊断安装问题
│   │
│   │ # ── 远程连接 ──
│   ├── bridge/                     #   远程控制模式
│   ├── bridge-kick.ts              #   桥接故障注入（内部）
│   ├── remote-env/                 #   远程环境配置
│   ├── remote-setup/               #   claude.ai 设置
│   ├── ide/                        #   IDE 集成管理
│   ├── desktop/                    #   在 Desktop 中继续
│   ├── chrome/                     #   Chrome 集成
│   ├── mobile/                     #   手机 App 二维码
│   │
│   │ # ── 信息/导出 ──
│   ├── help/                       #   帮助信息
│   ├── status/                     #   版本/模型/账户状态
│   ├── stats/                      #   使用统计
│   ├── insights.ts                 #   深度使用洞察
│   ├── context/                    #   上下文可视化
│   ├── files/                      #   列出上下文文件
│   ├── copy/                       #   复制回复到剪贴板
│   ├── export/                     #   导出对话
│   ├── feedback/                   #   提交反馈
│   ├── release-notes/              #   发行说明
│   ├── version.ts                  #   版本号
│   ├── passes/                     #   推荐码
│   ├── stickers/                   #   订购贴纸
│   ├── add-dir/                    #   添加工作目录
│   ├── tasks/                      #   后台任务列表
│   │
│   │ # ── 季节性/隐藏 ──
│   ├── thinkback/                  #   年度回顾
│   ├── thinkback-play/             #   回顾动画播放
│   ├── heapdump/                   #   JS堆转储（调试）
│   ├── tag/                        #   会话标签
│   │
│   │ # ── 禁用桩文件（内部功能占位） ──
│   ├── ant-trace/ autofix-pr/ backfill-sessions/ break-cache/
│   ├── bughunter/ ctx_viz/ debug-tool-call/ env/
│   ├── good-claude/ issue/ mock-limits/ oauth-refresh/
│   ├── onboarding/ perf-issue/ reset-limits/ share/
│   ├── summary/ teleport/
│   └── createMovedToPluginCommand.ts  # 已迁移命令的辅助工具
│
├── hooks/                          # React Hooks（85+条目，104+文件）
│   ├── useGlobalKeybindings.tsx     #   全局快捷键
│   ├── useCommandKeybindings.tsx    #   命令快捷键
│   ├── useVimInput.ts              #   Vim输入处理
│   ├── useTextInput.ts             #   文本输入基础Hook
│   ├── useVirtualScroll.ts         #   虚拟滚动
│   ├── useTerminalSize.ts          #   终端尺寸跟踪
│   ├── useSearchInput.ts           #   搜索输入
│   ├── useHistorySearch.ts         #   Ctrl+R历史搜索
│   ├── useTypeahead.tsx            #   自动补全
│   ├── useIDEIntegration.tsx       #   IDE连接
│   ├── useReplBridge.tsx           #   远程桥接
│   ├── useVoice.ts                 #   语音模式
│   ├── useMergedClients.ts         #   MCP客户端合并
│   ├── useMergedTools.ts           #   工具合并
│   ├── notifs/                     #   通知Hooks（模型/速率/插件/MCP等）
│   └── toolPermission/             #   工具权限处理（交互/协调/Swarm）
│
├── keybindings/                    # 键盘快捷键（14文件）
│   ├── defaultBindings.ts          #   默认映射（15+上下文）
│   ├── KeybindingContext.tsx       #   React上下文（和弦支持）
│   ├── KeybindingProviderSetup.tsx #   Provider组件
│   ├── resolver.ts                 #   键→动作解析器
│   ├── parser.ts                   #   快捷键字符串解析
│   ├── schema.ts                   #   JSON Schema验证
│   ├── validate.ts                 #   用户绑定验证
│   ├── loadUserBindings.ts         #   用户绑定加载
│   ├── match.ts                    #   键匹配逻辑
│   ├── reservedShortcuts.ts        #   保留快捷键
│   ├── shortcutFormat.ts           #   显示格式
│   ├── template.ts                 #   模板生成
│   ├── useKeybinding.ts            #   注册Hook
│   └── useShortcutDisplay.ts       #   显示文本Hook
│
├── vim/                            # Vim 模式（5文件）
│   ├── motions.ts                  #   移动（h/j/k/l/w/b/e/$/G）
│   ├── operators.ts                #   操作符（d/c/y/r/缩进）
│   ├── textObjects.ts              #   文本对象（iw/aw/i"/a(）
│   ├── transitions.ts              #   模式状态机转换
│   └── types.ts                    #   Vim状态类型
│
├── native-ts/                      # 纯TypeScript原生绑定
│   ├── color-diff/                 #   语法高亮Diff（~999行, highlight.js）
│   ├── file-index/                 #   模糊文件搜索（~370行, nucleo风格）
│   └── yoga-layout/                #   Flexbox布局引擎（~2700行, yoga移植）
│
├── buddy/                          # 宠物伴侣系统
│   ├── types.ts                    #   18种物种/稀有度/属性
│   ├── companion.ts                #   确定性生成（hash(userId)）
│   ├── sprites.ts                  #   ASCII精灵渲染
│   ├── CompanionSprite.tsx         #   React组件
│   ├── useBuddyNotification.tsx    #   通知Hook
│   └── prompt.ts                   #   性格系统提示
│
├── voice/
│   └── voiceModeEnabled.ts         #   语音模式功能门控
│
├── moreright/
│   └── useMoreRight.tsx            #   内部特性桩（外部构建占位）
│
├── outputStyles/
│   └── loadOutputStylesDir.ts      #   自定义输出样式加载
│
├── migrations/                     # 配置迁移（11文件）
│   ├── migrateAutoUpdatesToSettings.ts
│   ├── migrateBypassPermissionsAcceptedToSettings.ts
│   ├── migrateSonnet45ToSonnet46.ts
│   ├── migrateFennecToOpus.ts
│   └── ... (模型/配置迁移)
│
├── schemas/
│   └── hooks.ts                    #   Hook配置Zod Schema
│
├── types/                          # 类型定义
│   ├── command.ts                  #   命令类型（Prompt/Local/LocalJSX）
│   ├── permissions.ts              #   权限类型（模式/结果/规则）
│   ├── hooks.ts                    #   Hook类型
│   ├── plugin.ts                   #   插件类型
│   ├── ids.ts                      #   品牌ID类型（SessionId/AgentId）
│   ├── logs.ts                     #   会话日志类型
│   ├── textInputTypes.ts           #   文本输入类型
│   └── generated/                  #   Protobuf生成类型
│
├── constants/                      # 常量定义
│   ├── prompts.ts                  #   系统提示构建（~914行）
│   ├── tools.ts                    #   工具允许/拒绝集
│   ├── system.ts                   #   系统提示前缀
│   ├── systemPromptSections.ts     #   可组合提示段
│   ├── product.ts                  #   产品URL
│   ├── betas.ts                    #   Beta功能标志
│   ├── oauth.ts                    #   OAuth配置
│   ├── files.ts                    #   文件路径常量
│   ├── keys.ts                     #   快捷键常量
│   ├── messages.ts                 #   消息格式
│   ├── xml.ts                      #   XML标签
│   ├── apiLimits.ts                #   API速率限制
│   ├── toolLimits.ts               #   工具执行限制
│   ├── outputStyles.ts             #   输出格式样式
│   ├── common.ts                   #   共享工具
│   ├── errorIds.ts                 #   错误ID
│   ├── figures.ts                  #   Unicode图形
│   ├── spinnerVerbs.ts             #   加载动画动词
│   ├── turnCompletionVerbs.ts      #   完成动词
│   ├── github-app.ts               #   GitHub App配置
│   └── cyberRiskInstruction.ts     #   安全指令
│
├── utils/                          # 工具函数（300+模块, 560+文件）
│   ├── settings/                   #   设置管理（验证/缓存/变更/MDM）
│   ├── permissions/                #   权限系统
│   ├── git/                        #   Git操作（配置/文件系统/ignore）
│   ├── model/                      #   模型选择/成本计算
│   ├── messages/                   #   消息创建/规范化
│   ├── telemetry/                  #   遥测/追踪
│   ├── bash/                       #   Bash工具辅助
│   ├── sandbox/                    #   沙箱管理
│   ├── swarm/                      #   Swarm多Agent（后端: Tmux/iTerm/Pane/InProcess）
│   ├── hooks/                      #   Hook执行
│   ├── mcp/                        #   MCP辅助
│   ├── memory/                     #   记忆辅助
│   ├── skills/                     #   技能辅助
│   ├── plugins/                    #   插件辅助
│   ├── github/                     #   GitHub API
│   ├── dxt/                        #   DXT扩展
│   ├── computerUse/                #   计算机使用
│   ├── deepLink/                   #   claude:// 协议处理
│   ├── filePersistence/            #   文件持久化
│   ├── secureStorage/              #   安全存储
│   ├── processUserInput/           #   用户输入处理
│   ├── task/                       #   任务辅助
│   ├── todo/                       #   Todo辅助
│   ├── ultraplan/                  #   超级计划
│   ├── suggestions/                #   输入建议
│   ├── shell/                      #   Shell辅助
│   ├── powershell/                 #   PowerShell辅助
│   ├── nativeInstaller/            #   原生安装器
│   ├── claudeInChrome/             #   Chrome集成
│   └── background/                 #   后台任务辅助
│
├── cli/                            # CLI辅助
│   ├── handlers/
│   ├── transports/
│   ├── exit.ts / print.ts / update.ts
│   ├── remoteIO.ts / structuredIO.ts
│   └── ndjsonSafeStringify.ts
│
└── .sisyphus/plans/                # 工作计划目录
```

---

## 模块详解

### 1. 入口与启动流程

**主入口链：**
```
cli.tsx → main.tsx → run() → setup() → REPL.tsx
```

| 阶段 | 文件 | 职责 |
|------|------|------|
| 入口 | `entrypoints/cli.tsx` | 处理快速路径（`--version`, `--dump-system-prompt`） |
| 主逻辑 | `main.tsx` (~4700行) | Commander.js 定义，初始化，路由到 REPL/Doctor/Headless |
| 初始化 | `entrypoints/init.ts` | 环境变量、OpenTelemetry、代理配置 |
| 会话设置 | `setup.ts` | Worktree创建、Hooks加载、插件初始化、权限验证 |
| 全局状态 | `bootstrap/state.ts` (~1760行) | 会话ID、费用、模型设置、功能标志 |

### 2. 核心类型系统

#### Tool 系统
- 每个能力都是一个 `Tool<Input, Output, P>`，包含 40+ 个方法
- `call()` 执行工具、`checkPermissions()` 权限检查、`prompt()` 系统提示
- `buildTool()` 工厂函数提供安全默认值
- `tools.ts` 注册表：`getAllBaseTools()` → 过滤权限 → 合并 MCP 工具

#### Task 系统
- 实际6种后台任务实现：LocalShellTask, LocalAgentTask, RemoteAgentTask, InProcessTeammateTask, DreamTask, LocalMainSessionTask（另有 LocalWorkflowTask 和 MonitorMcpTask 类型定义但无独立目录）
- 支持 Ctrl+B 后台化主会话

#### Query 循环 (`query.ts`, ~1700行)
- 核心对话引擎：流式 API + 工具执行循环
- 自动压缩（autocompact）、Token 预算管理
- 错误恢复：提示过长→压缩重试、最大输出Token→续查、模型回退

#### Command 系统
- 三种类型：`PromptCommand`（模型提示）、`LocalCommand`（本地执行）、`LocalJSXCommand`（React UI）
- 100+ 斜杠命令（86目录 + 15独立文件），支持特性门控

**会话管理**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| clear | `/clear` (别名: `/reset`, `/new`) | 清空对话历史，释放上下文 |
| compact | `/compact [指令]` | 清空历史但保留摘要，可自定义摘要指令 |
| resume | `/resume` (别名: `/continue`) | 恢复之前的对话（按ID或搜索词） |
| rename | `/rename [名称]` | 重命名当前对话 |
| rewind | `/rewind` (别名: `/checkpoint`) | 将代码和/或对话恢复到之前的检查点 |
| branch | `/branch [名称]` (别名: `/fork`) | 从当前点创建对话分支 |
| session | `/session` (别名: `/remote`) | 显示远程会话URL和二维码（仅远程模式） |
| tag | `/tag <标签>` | 切换当前会话的可搜索标签 |
| exit | `/exit` (别名: `/quit`) | 退出 REPL |

**模型与推理配置**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| model | `/model [模型]` | 设置 AI 模型 |
| effort | `/effort [low\|medium\|high\|max\|auto]` | 设置模型推理努力级别 |
| fast | `/fast [on\|off]` | 切换快速模式（限制为快速模型） |
| advisor | `/advisor [模型\|off]` | 配置二级顾问模型审阅响应 |
| brief | `/brief` | 切换简洁模式（用BriefTool压缩输出） |

**Git 与代码审查**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| commit | `/commit` | 分析改动并自动生成 git commit |
| commit-push-pr | `/commit-push-pr` | 一步完成：创建分支→提交→推送→开PR |
| diff | `/diff` | 查看未提交的变更和每轮差异 |
| review | `/review [PR号]` | 审查 Pull Request（本地分析） |
| security-review | `/security-review` | 安全审查分支代码变更 |
| pr_comments | `/pr-comments` | 获取 GitHub PR 评论（已迁移到插件） |

**规划与高级工作流**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| plan | `/plan [open\|描述]` | 启用计划模式或查看当前会话计划 |
| ultraplan | `/ultraplan <提示>` | 启动远程多Agent规划（Opus, 10-30分钟） |
| init-verifiers | `/init-verifiers` | 创建自动验证代码变更的验证技能 |
| btw | `/btw <问题>` | 快速旁问，不打断主对话 |

**配置与设置**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| config | `/config` (别名: `/settings`) | 打开配置/设置面板 |
| permissions | `/permissions` (别名: `/allowed-tools`) | 管理工具权限规则（允许/拒绝） |
| keybindings | `/keybindings` | 打开快捷键配置文件 |
| hooks | `/hooks` | 查看工具事件的 Hook 配置 |
| memory | `/memory` | 编辑记忆文件（MEMORY.md等） |
| mcp | `/mcp [enable\|disable [服务器]]` | 管理 MCP 服务器 |
| sandbox-toggle | `/sandbox` | 切换和配置沙箱模式 |
| theme | `/theme` | 更改终端主题 |
| color | `/color <颜色\|default>` | 设置提示栏颜色 |
| vim | `/vim` | 切换 Vim / 普通编辑模式 |
| voice | `/voice` | 切换语音输入模式 |
| output-style | `/output-style` | 已弃用，请用 /config |
| statusline | `/statusline` | 设置状态栏 UI |
| terminalSetup | `/terminal-setup` | 安装 Shift+Enter 换行键绑定 |

**账户与认证**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| login | `/login` | 登录 Anthropic 账户 |
| logout | `/logout` | 登出 |
| upgrade | `/upgrade` | 升级到 Max 计划 |
| usage | `/usage` | 显示当前计划用量限制 |
| cost | `/cost` | 显示本次会话的总费用和时长 |
| extra-usage | `/extra-usage` | 配置超额计费，限额后继续工作 |
| privacy-settings | `/privacy-settings` | 查看和更新隐私设置 |
| rate-limit-options | `/rate-limit-options` | 达到速率限制时的选项（内部/隐藏） |

**插件、技能与扩展**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| plugin | `/plugin` (别名: `/plugins`, `/marketplace`) | 管理插件（安装/卸载/启用/禁用） |
| skills | `/skills` | 列出可用技能 |
| reload-plugins | `/reload-plugins` | 重新加载插件变更 |
| agents | `/agents` | 管理 Agent 配置 |

**项目初始化**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| init | `/init` | 初始化 CLAUDE.md 文件（自动生成项目文档） |
| install | `/install` | 安装 Claude Code 原生版本 |
| install-github-app | `/install-github-app` | 为仓库设置 GitHub Actions |
| install-slack-app | `/install-slack-app` | 安装 Claude Slack 应用 |
| doctor | `/doctor` | 诊断 Claude Code 安装和配置 |

**远程与连接**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| bridge | `/remote-control` (别名: `/rc`) | 连接终端用于远程控制会话（桥接模式） |
| bridge-kick | `/bridge-kick` | 注入桥接故障状态用于恢复测试（内部） |
| remote-env | `/remote-env` | 配置远程环境（teleport会话） |
| remote-setup | `/web-setup` | 在 claude.ai 上设置 Claude Code |
| ide | `/ide [open]` | 管理 IDE 集成，显示连接状态 |
| desktop | `/desktop` (别名: `/app`) | 在 Claude Desktop 中继续当前会话 |
| chrome | `/chrome` | Chrome 集成设置（Beta） |
| mobile | `/mobile` (别名: `/ios`, `/android`) | 显示二维码下载手机 App |

**信息与导出**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| help | `/help` | 显示帮助和可用命令 |
| status | `/status` | 显示版本、模型、账户、API连接状态 |
| stats | `/stats` | 显示使用统计和活动 |
| insights | `/insights` | 用 Opus 分析会话历史生成深度洞察 |
| context | `/context` | 可视化当前上下文使用量（彩色网格） |
| files | `/files` | 列出当前上下文中的所有文件 |
| copy | `/copy [N]` | 复制最近一条（或第N条）回复到剪贴板 |
| export | `/export [文件名]` | 导出对话到文件或剪贴板 |
| feedback | `/feedback` (别名: `/bug`) | 提交反馈或 Bug 报告 |
| release-notes | `/release-notes` | 查看发行说明 |
| version | `/version` | 打印当前版本号 |
| passes | `/passes` | 分享 Claude Code 免费周（推荐码） |
| stickers | `/stickers` | 订购 Claude Code 贴纸 |
| add-dir | `/add-dir <路径>` | 添加新的工作目录到会话 |
| tasks | `/tasks` (别名: `/bashes`) | 列出和管理后台任务 |

**季节性/功能门控**

| 命令 | 斜杠名 | 用途 |
|------|--------|------|
| thinkback | `/think-back` | 2025 Claude Code 年度回顾 |
| thinkback-play | `/thinkback-play` | 播放 thinkback 动画（隐藏，由技能调用） |

**内部/调试（桩文件，外部版本不可用）**

| 命令 | 说明 |
|------|------|
| ant-trace, autofix-pr, backfill-sessions, break-cache, bughunter, ctx_viz, debug-tool-call, env, good-claude, issue, mock-limits, oauth-refresh, onboarding, perf-issue, reset-limits, share, summary, teleport | 18个禁用桩文件，代表内部功能的外部占位 |

### 3. UI 系统 (Ink)

**Ink 框架** (`ink/`) — React for CLI 的自定义 fork
- 自定义 React 协调器 + Yoga 布局引擎
- 屏幕缓冲区差分渲染（StylePool/CharPool/HyperlinkPool）
- 完整 ANSI 转义序列解析（SGR/OSC/CSI/DEC/ESC）
- 鼠标跟踪、文本选择（拖拽/词语/行选择/复制）、搜索高亮

**主要 UI 组件：**
| 组件 | 功能 |
|------|------|
| `PromptInput` (~2340行) | 文本输入：Vim模式、自动补全、图片粘贴、历史导航 |
| `Messages` + `VirtualMessageList` | 消息列表：虚拟滚动、搜索、粘性头部 |
| `messages/` (33文件) | 各类型消息渲染：AssistantText/UserToolResult/Advisor等 |
| `PermissionRequest` 系列 | 工具权限对话框：Bash/File/Edit/Write/MCP等 |
| `design-system/` | 设计原语：Dialog/Pane/FuzzyPicker/Tabs/ProgressBar |
| `sandbox/` (5文件) | 沙箱配置UI：依赖项/配置选项卡 |
| `Settings/` (4文件) | 设置页面：Config/Settings/Status/Usage |

### 4. 服务层

| 服务 | 功能 |
|------|------|
| `services/api/` (~3420行) | Anthropic API 流式客户端：消息规范化、错误处理、使用量追踪 |
| `services/mcp/` | MCP连接管理：OAuth认证、多传输层、VS Code SDK集成 |
| `services/lsp/` | 语言服务器协议：诊断注册、被动反馈、符号操作 |
| `services/analytics/` | 遥测管道：Datadog指标、GrowthBook功能标志、1P事件日志 |
| `services/SessionMemory/` | 后台子Agent定期提取会话笔记到Markdown |
| `services/autoDream/` | 夜间记忆整理（dream/summarize） |
| `services/compact/` | 上下文压缩（trim对话以适应Token限制） |
| `services/notifier.ts` | 操作系统通知 |
| `services/tokenEstimation.ts` | Token估算 |
| `services/vcr.ts` | 会话录制/回放 |

### 5. 记忆系统

**memdir/** — 持久化文件记忆
- `MEMORY.md`: 索引文件（最多200行），始终加载到上下文
- 四种类型：`user`（偏好）、`feedback`（反馈）、`project`（项目上下文）、`reference`（外部引用）
- 支持团队记忆（`memory/team/` 共享目录）

**SessionMemory/** — 自动记忆提取
- 后台 fork 子Agent定期提取对话笔记

### 6. 远程控制

**bridge/** (31文件) — 远程控制守护进程
- `claude remote-control`: 注册到 claude.ai，接受工作请求
- 支持多会话模式（worktree隔离）、指数退避、心跳
- 轮询 → 生成会话 → 执行 → 返回结果

**remote/** — 远程会话客户端
- WebSocket订阅接收消息、HTTP POST发送消息
- 权限请求/响应流、中断支持、重连

**upstreamproxy/** — CCR容器代理
- MITM代理注入组织凭证到出站请求
- 安全：令牌读后删文件、prctl防ptrace

### 7. 扩展系统

**plugins/** — 插件系统
- 内置插件（用户可开关），提供命令、Hooks、MCP服务器

**skills/** — 技能系统
- **捆绑技能**: 编译进CLI（12+个：verify/debug/remember/dream/hunter等）
- **用户技能**: 从 `~/.claude/skills/` 和 `.claude/skills/` 加载
- **技能格式**: 目录 + `SKILL.md`（YAML frontmatter + 提示模板）

### 8. 原生 TS 模块

**native-ts/** — 纯 TypeScript 实现（无需编译原生二进制）

| 模块 | 功能 | 行数 |
|------|------|------|
| `color-diff/` | 语法高亮Diff渲染（highlight.js + word diff） | ~999 |
| `file-index/` | 模糊文件搜索引擎（位图预过滤，O(1)淘汰） | ~370 |
| `yoga-layout/` | Flexbox布局引擎（Facebook yoga-layout移植） | ~2700 |

### 9. 其他重要模块

| 模块 | 功能 |
|------|------|
| `vim/` (5文件) | 完整Vim模拟：移动(h/j/k/l/w/b)、操作符(d/c/y)、文本对象(iw/aw/i")、模式转换 |
| `keybindings/` (14文件) | 15+上下文的快捷键系统，支持和弦(ctrl+x ctrl+k)、用户自定义JSON |
| `buddy/` (6文件) | 宠物伴侣：18种物种、稀有度、属性、帽子、确定性生成 |
| `coordinator/` | 协调者/工作者多Agent模式（369行系统提示） |
| `voice/` | 语音输入：原生录音(cpal/SoX/arecord) → 流式STT |

---

## 数据流

```
用户输入
  │
  ▼
REPL.tsx（主界面）
  │
  ▼
processUserInput()（处理输入：文本/斜杠命令/Shell命令）
  │
  ▼
query() 循环
  │
  ├──▶ services/api/claude.ts → Anthropic API（流式响应）
  │         │
  │         ▼
  │     tool_use 块 → StreamingToolExecutor → Tool.call()
  │         │
  │         ▼
  │     Tool Result → 追加到消息 → 继续循环
  │
  ├──▶ Autocompact（上下文超限时自动压缩）
  │
  ├──▶ Token Budget（预算耗尽时决定是否继续）
  │
  └──▶ Stop Hooks（轮次结束后执行：记忆提取、提示建议等）
```

---

## 技术栈

| 技术 | 用途 |
|------|------|
| **TypeScript** | 主要语言 |
| **React** | UI组件模型 |
| **Ink** (自定义fork) | 终端UI渲染框架 |
| **Yoga** (纯TS移植) | Flexbox布局引擎 |
| **Zustand-like Store** | 状态管理 |
| **Zod** | Schema验证 |
| **Commander.js** | CLI参数解析 |
| **Anthropic Claude API** | AI模型调用 |
| **MCP** | 外部工具协议 |
| **LSP** | 语言服务器集成 |
| **Datadog / GrowthBook** | 遥测和功能标志 |
| **Node.js 18+** | 运行时 |

---

## 关键文件规模

| 文件 | 行数 | 说明 |
|------|------|------|
| `screens/REPL.tsx` | ~5000 | 主REPL界面 |
| `main.tsx` | ~4700 | CLI主入口，Commander.js程序 |
| `bridge/bridgeMain.ts` | ~3000 | 远程控制主循环 |
| `services/api/claude.ts` | ~3420 | Anthropic API流式客户端 |
| `components/PromptInput/PromptInput.tsx` | ~2340 | 文本输入组件 |
| `native-ts/yoga-layout/` | ~2700 | Flexbox布局引擎 |
| `ink/ink.tsx` | ~1720 | Ink核心渲染引擎 |
| `query.ts` | ~1700 | 核心查询循环 |
| `bootstrap/state.ts` | ~1760 | 全局状态单例 |
| `skills/loadSkillsDir.ts` | ~1086 | 技能加载器 |
| `native-ts/color-diff/` | ~999 | Diff渲染 |
| `constants/prompts.ts` | ~914 | 系统提示构建 |
| `memdir/memdir.ts` | ~507 | 记忆管理 |

---

## 统计

- **总文件数**: ~1900
- **工具数量**: 43+（42目录 + utils.ts）
- **服务数量**: 36个（20目录 + 16独立文件）
- **斜杠命令**: 100+（86目录 + 15独立文件）
- **工具函数**: 300+ 模块（560+ 文件，36 子目录）
- **React Hooks**: 85+（含子目录共104+文件）
- **配置迁移**: 11个
- **键盘快捷键上下文**: 15+

---

*此 README 由 AI 自动生成，基于对源代码的深度分析（7个并行代理扫描全部1900+文件，含完整性验证）。*

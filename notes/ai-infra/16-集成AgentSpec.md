# 集成 AgentSpec

修改 Nacos 中的 AgentSpec 的格式：

   `main_agent`

   ```json
   {
     "description": "主Agent配置",
     "model": {
       "name": "chat-standard",
       "thinking": false
     },
     "skills": [
       "pptx",
       "frontend-design"
     ],
     "mcps": [
       "langchain-docs",
       "microsoft-docs"
     ],
     "tools": [
       "ask_user_question",
       "get_current_time"
     ],
     "subagents": [
       "web_agent"
     ]
   }
   ```

   `web_agent`

   ```json
   {
     "description": "负责网络搜索、抓取网页正文、核实公开信息并总结分析。适合需要查询最新资料、查找官方文档或新闻、提炼指定 URL 内容、比较多个公开来源，或需要附带可追溯来源链接的任务。",
     "model": {
       "name": "chat-basic",
       "thinking": false
     },
     "skills": [],
     "mcps": [],
     "tools": [
       "get_current_time",
       "web_search",
       "fetch_url"
     ]
   }
   ```
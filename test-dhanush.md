you are a coding assistant. I want to build a desktop app that is an AGI-like application. It should maily help the user in his everyday task like email, calendar, reminder, automation, task planning, etc. I will define the capabilites in more detail below. I want this to be an agentic application. the GUI must be built using flet. the LLM should be running on Ollama running locally (model choice is key). It must have "tabs" or "sessions" and each session must maintain context in between prompts, there is no need to maintain context between multiple sessions/tabs. Each session must maintain a file for latest context. use an embedding framework like chromaDB to store context of each session in a vectorDB. It will make it easier to reference a particular previous session from a current session if the user explicitly says something like "from a previous session" or "from a previous tab". This agentic application must use MCP for any agent that has one.

The following are the agents that we want to implement:
1. Email agent (Gmail)
2. Calendar Agent (Coogle Calendar)
3. File handling (Only reading existing files, can write new files but not write to existing ones)
4. Web search agent (using linkup API). To be used only when there is not enough context. No user data/files must be sent to this API for privacy reasons.
5. translation between languages agent
6. A code assistant (use claude API?)
7. Summarizer agent that can summarize the outputs of other agents.


I want the following features / functionalities : 
1. Memory management within a session : compaction of context, contxtual understanding of phrases liek "it", "that", etc
2. Tab management : like what chatgpt does
3. Robust agentic loop with the think, act validate cycle

Use all Best coding practices and make it production ready and bug free. I want clean well commented code. 

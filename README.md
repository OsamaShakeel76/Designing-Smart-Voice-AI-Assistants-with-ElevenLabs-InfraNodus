# AI Voice Chat Agent with ElevenLabs + InfraNodus (Graph RAG) — n8n Workflow

A **voice-enabled AI chat agent** that routes user questions to multiple **InfraNodus Graph RAG “experts”** (knowledge graphs) and returns a **conversational response** back to the caller (e.g., ElevenLabs Conversational AI widget). Built as an **n8n orchestration workflow** with memory + tool-based expert selection.

---

## What This Does

- **Voice chat frontend (ElevenLabs Conversational AI)** sends user messages to an **n8n Webhook**
- **n8n AI Agent** decides which expert tool(s) to call (min 1, max 3)
- Each expert is an **InfraNodus graph** queried via HTTP (Graph RAG response + summary + statements)
- **Memory node** keeps session context (`sessionId`) for multi-turn conversations
- Final answer is returned via **Respond to Webhook** (and can be forwarded to Telegram or other channels)

---

## Architecture

**ElevenLabs Agent (Voice/UI) → n8n Webhook → AI Agent (Router) → InfraNodus Tools (Experts) → Response → Webhook Reply**

### Key Components (n8n Nodes)
- **Webhook**: Receives `{ prompt, sessionId }` from ElevenLabs
- **AI Agent**: Chooses the best expert tools based on the user prompt
- **Simple Memory**: Stores chat history per session (`body.sessionId`)
- **HTTP Request Tools (InfraNodus Experts)**:
  - `special_agents_manual`
  - `waves_into_patterns`
  - `the_flow_and_notion`
  - `polysingularity_letters`
- **LLM**: Google Gemini Flash (fast) or OpenAI (more precise tool calling)
- **Respond to Webhook**: Returns the final assistant message back to the caller

---

## Requirements

### Accounts / Keys
- **n8n** (self-hosted or n8n cloud)
- **ElevenLabs** (Conversational AI feature enabled)
- **InfraNodus** API access (Bearer token)
- **LLM credentials**:
  - Google Gemini (PaLM API) **or**
  - OpenAI

### Credentials Used in Workflow
- `httpBearerAuth` → InfraNodus API token
- `googlePalmApi` → Gemini API key
- *(Optional)* `openAiApi` → OpenAI key (node present but disabled)

---

## Setup Guide (Step-by-Step)

### 1) Import Workflow into n8n
- Create a new workflow in n8n
- Import the JSON (this workflow export)

### 2) Configure InfraNodus Expert Tools
Each expert uses:
- **POST** `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&addStats=true&optimize=develop&includeGraph=false&includeGraphSummary=true&includeStatements=true`
- Body parameters:
  - `name` → your InfraNodus graph name (e.g., `waves_into_patterns`)
  - `prompt` → user prompt (adjusted by AI Agent)
  - `requestMode` → `response`
  - `aiTopics` → `true`

✅ To add a new expert:
1. Duplicate an existing **httpRequestTool**
2. Change `body.name` to your graph name
3. Update the **toolDescription** (important for the agent’s routing accuracy)

---

### 3) Configure the LLM
This workflow includes:
- **Google Gemini Chat Model** (`models/gemini-2.5-flash-preview-04-17`) for speed
- **OpenAI Model** node exists but is disabled

You can:
- Keep Gemini for fast routing
- Switch to OpenAI if you want stronger tool selection / reasoning

---

### 4) Set Up ElevenLabs Conversational AI → n8n Webhook
In ElevenLabs:
1. Create a **Conversational AI Agent**
2. Add a **Tool** with name: `knowledge_base`
3. Tool method: **POST**
4. Tool URL: your n8n webhook URL  
   Example: `https://<your-n8n-domain>/webhook/171bf9a6-1390-4195-bd6b-ff3df2e27d1c`

**Body Parameters to send**
- `prompt` (type: LLM prompt) → user message
- `sessionId` (dynamic variable) → `system__conversation_id`

#### Suggested ElevenLabs System Prompt
```txt
You are a voice AI assistant that answers using the knowledge_base tool.
1) Briefly acknowledge the user (e.g., “Let me check that…”).
2) Send the user's message to the knowledge_base tool without changing it.
3) Use the returned answer to respond, making it concise and conversational while keeping specifics.
IMPORTANT: Always use knowledge_base for answers.


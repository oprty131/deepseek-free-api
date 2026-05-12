
# DeepSeek Free API Proxy
 [![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)
 [![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
 [![FastAPI](https://img.shields.io/badge/FastAPI-teal)](https://fastapi.tiangolo.com/)
 Reverse the **DeepSeek web free chat** (chat.deepseek.com) into an **OpenAI-compatible API**. Supports dynamic model discovery, automatic PoW solving, automatic token refresh, and provides a pure chat version (no-tools branch, no tool-call prompt injection).
 All code modified in this project is AI-generated, with **zero human-written code** — please note!
 
💡 No tool calls needed? For pure conversation use cases (writing, translation, coding, Q&A), we recommend the  no-tools  branch — no tool prompt injection, cleaner context, higher output quality.
 
Reference project: NIyueeE/ds-free-api (Rust version). This Python edition is a full rewrite.
The original Rust version uses browser automation (Playwright/Chrome), while this Python version uses pure HTTP forwarding (curl_cffi simulating Chrome TLS fingerprint) with much lower resource usage.
 
Table of Contents
 
- Features
- Architecture
- Quick Start- One-command deployment (recommended)
- Manual installation
- Credential Configuration- Method 1: Phone / Email login (recommended)
- Method 2: cURL import
- Method 3: Cookie import
- API Usage- List models
- Non-streaming chat
- Streaming chat
- Model refresh
- Anthropic Messages API
- Responses API
- Model System- Dynamic model discovery
- Currently available models
- Tool Call Details
- No-Tools Branch (no-tools)
- PoW Solving Mechanism
- Automatic Token Refresh
- Management Commands
- Project Structure
- Configuration Reference
- Dependencies
- Limitations & Known Issues
- FAQ
- License & Credits
 
Features
 
- Full OpenAI compatibility —  /v1/chat/completions  (stream/non-stream),  /v1/models ,  /v1/models/refresh ,  /v1/responses  endpoints
- OpenAI Responses API — New  /v1/responses  create/retrieve/delete/input_items/cancel/compact, full SSE lifecycle events, Structured Output support
- Pure chat proxy — No tool-call prompt injection, cleaner output, model focuses on user queries
- Dynamic model discovery — Real-time model list from DeepSeek official API on startup, auto-refresh every hour (including context size and full details)
- Automatic PoW solving — Node.js WASM main solver + Python pure-algorithm fallback; auto-fetch challenge and solve before requests
- Automatic token refresh — Auto-relogin with saved password on 401, no manual intervention
- Deep reasoning — Supports DeepSeek  <thought>  tags, separated into  reasoning_content  in streaming output
- Vision image understanding — Supports image upload, parsing, and conversation
- Text file upload — Direct upload of .txt/.md/.py and other text files via ref_file_ids (same as web)
- Web search — Supports  search_enabled  parameter for search model variants
- Management panel — Embedded single-file Web UI, supports phone/email login, cURL import
- Pure HTTP solution — No browser/Playwright/Chrome dependency; uses curl_cffi to mimic Chrome TLS fingerprint
- No-tools branch — Dedicated branch removing tool-call logic for pure chat scenarios with better output quality
 
Architecture
 
plaintext
  
┌──────────────────────────────────────────────────────────┐
│                   OpenAI Compatible Clients                    │
│          (ChatBox / LobeChat / curl / Cline)               │
└───────────────┬──────────────────────────────────────────┘
                │  /v1/chat/completions
                ▼
┌──────────────────────────────────────────────────────────┐
│               DeepSeek Free API Proxy (FastAPI)             │
│  ┌─────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ Router  │  │  tool_call   │  │    tool_sieve    │  │   tool_dsml    │  │   curl_cffi client        │ │
│  │ /v1/*   │──│ (DSML prompt) │──│ (stream filter)  │──│ (DSML parsing) │──│ (simulate Chrome fingerprint)│ │
│  └─────────┘  └──────────────┘  └──────────────────────┘ │
│  ┌─────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ Model   │  │   PoW Solver │  │   Auto Token Refresh    │ │
│  │ Discovery│  │ (Node+Python) │  │ (save password & relogin)│ │
│  └─────────┘  └──────────────┘  └──────────────────────┘ │
│  ┌─────────┐  ┌──────────────────────────┐              │
│  │ Vision  │  │ File Upload / Parsing           │              │
│  │ Support │  │ (image: upload→fork→wait)        │              │
│  └─────────┘  │ (text: upload→wait)             │              │
│               └──────────────────────────┘              │
└───────────────┬──────────────────────────────────────────┘
                │  HTTPS (curl_cffi, Chrome fingerprint)
                ▼
┌──────────────────────────────────────────────────────────┐
│        DeepSeek API (chat.deepseek.com)                   │
│  /api/v0/chat/completion (SSE)                            │
│  /api/v0/users/login                                     │
│  /api/v0/chat_session/create                             │
│  /api/v0/chat/create_pow_challenge                       │
│  /api/v0/client/settings?scope=model                     │
│  /api/v0/file/upload_file + fork_file_task               │
└──────────────────────────────────────────────────────────┘
 
 
Quick Start
 
One-command deployment (recommended)
 
bash
  
# Install Node.js first (required for PoW solver)
# Termux:
pkg install nodejs
# Linux:
# sudo apt install nodejs

# Clone (recommended)
git clone https://github.com/Fly143/deepseek-free-api.git
cd deepseek-free-api
chmod +x deploy.sh

# Run in foreground (stop with Ctrl+C)
./deploy.sh

# Or run in background
./deploy.sh --bg

# Check status
./deploy.sh --status

# Stop
./deploy.sh --stop
 
 
After deployment, visit: http://localhost:8000/admin
 
💡 No tool calls needed? Clone the  no-tools  branch for a cleaner pure-chat version (no prompt injection, higher quality output).
 
Manual installation
 
bash
  
# 1. Ensure Python 3.10+ and Node.js are installed
python3 --version
node --version

# 2. Install Python dependencies
pip install fastapi uvicorn curl-cffi python-dotenv

# 3. Start the proxy
python3 proxy.py
 
 
Credential Configuration
 
Open the admin panel at http://localhost:8000/admin to configure.
 
Method 1: Phone / Email login (recommended)
 
The easiest way, same as web login:
 
1. Select Phone or Email tab
2. Enter phone number (default area code +86) or email
3. Enter password
4. Click Login
 
The system will automatically: login to get Token → create chat session → save config to  token.json  (including password for auto-refresh).
 
Method 2: cURL import
 
1. Log in to chat.deepseek.com
2. Open DevTools → Network panel
3. Send a message, find the  completion  request
4. Right-click → Copy as cURL
5. In the admin panel, expand Advanced: Paste cURL manually, paste it
6. Click Save cURL
 
Method 3: Cookie import
 
1. Log in to chat.deepseek.com
2. Open DevTools → Application → Cookies
3. Find cookies for  chat.deepseek.com 
4. Export the cookie string containing  userToken 
5. Paste into admin panel → Save
 
API Usage
 
1. List models
 
bash
  
curl http://localhost:8000/v1/models
 
 
Returns all dynamically detected available models, including  max_input_tokens ,  max_output_tokens , and other details.
 
2. Non-streaming chat
 
bash
  
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-default",
    "messages": [
      {"role": "user", "content": "Write a quicksort in Python"}
    ]
  }'
 
 
3. Streaming chat
 
bash
  
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-reasoner",
    "messages": [
      {"role": "user", "content": "Explain quantum entanglement"}
    ],
    "stream": true
  }'
 
 
In streaming responses, reasoning content appears in  delta.reasoning_content , official output in  delta.content .
 
4. File upload (text & image)
 
Text file upload (supported by all models, no fork, uses  ref_file_ids ):
 
bash
  
# Encode file to base64
FILE_B64=$(base64 -w0 three-body-intro.txt)
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-default",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "What is this file about?"},
        {"type": "file", "file": {"filename": "three-body-intro.txt", "file_data": "'"$FILE_B64"'"}}
      ]
    }]
  }'
 
 
Vision image upload (requires Vision model; fork to vision type after upload):
 
bash
  
# Encode image to base64
IMG_B64=$(base64 -w0 photo.png)
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-vision",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "Describe this image"},
        {"type": "image_url", "image_url": {"url": "data:image/png;base64,'"$IMG_B64"'"}}
      ]
    }]
  }'
 
 
Note: Text files are not forked; wait for DeepSeek parsing then reference raw  file_id . Images must be forked to  "vision"  to be read by Vision models.
 
5. Responses API (OpenAI compatible)
 
Supports OpenAI’s latest  /v1/responses  endpoint.
 
Non-streaming:
 
bash
  
curl http://localhost:8000/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek",
    "input": "Write a quicksort in Python",
    "stream": false
  }'
 
 
Streaming (with full SSE lifecycle events):
 
bash
  
curl http://localhost:8000/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek",
    "input": "Explain quantum entanglement",
    "stream": true
  }'
 
 
Events:  response.created  →  response.in_progress  →  response.output_item.added  →  response.content_part.added  →  response.output_text.delta  (chunk by chunk) →  response.output_text.done  →  response.content_part.done  →  response.output_item.done  →  response.completed 
 
Other endpoints (supports streaming replay):
 
bash
  
# Retrieve
curl http://localhost:8000/v1/responses/{response_id}

# Input items
curl http://localhost:8000/v1/responses/{response_id}/input_items

# Cancel
curl -X POST http://localhost:8000/v1/responses/{response_id}/cancel

# Delete
curl -X DELETE http://localhost:8000/v1/responses/{response_id}

# Compact multi-turn conversation
curl -X POST http://localhost:8000/v1/responses/{response_id}/compact \
  -H "Content-Type: application/json" \
  -d '{"instructions": "Please answer all following questions in Chinese"}'
 
 
Structured Output (json_schema):
 
bash
  
curl http://localhost:8000/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek",
    "input": "Beijing weather today 25°C, return structured data",
    "text": {
      "format": {
        "type": "json_schema",
        "schema": {
          "type": "object",
          "properties": {
            "city": {"type": "string"},
            "temperature": {"type": "integer"},
            "unit": {"type": "string"}
          },
          "required": ["city", "temperature", "unit"]
        }
      }
    }
  }'
 
 
The Responses API complements the existing  /v1/chat/completions ; both can be used together.
 
6. Anthropic Messages API
 
This proxy is fully compatible with Anthropic Messages API format, supporting seamless integration with clients like RikkaHub.
 
Auth: Use  x-api-key  header or  Authorization: Bearer :
 
bash
  
# x-api-key (recommended)
curl http://localhost:8000/v1/messages \
  -H "x-api-key: sk-dsapi" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-default",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Write a quicksort in Python"}
    ]
  }'
 
 
Streaming (reasoning + text):
 
bash
  
curl http://localhost:8000/v1/messages \
  -H "x-api-key: sk-dsapi" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-reasoner",
    "max_tokens": 1024,
    "stream": true,
    "messages": [
      {"role": "user", "content": "Explain quantum entanglement"}
    ]
  }'
 
 
Reasoning streams as  thinking  block; text as  text  block.
 
Available endpoints:
 
Method Endpoint Description 
POST  /v1/messages  Send message (text/reasoning/tool call) 
POST  /v1/messages/count_tokens  Count tokens 
GET  /v1/messages/{id}  Query sent message 
POST  /v1/messages/batches  Create batch request 
GET  /v1/messages/batches  List batch requests 
GET  /v1/messages/batches/{id}  Get batch details 
POST  .../cancel  Cancel batch 
GET  .../results  Download batch results 
DELETE  /v1/messages/batches/{id}  Delete batch 
 
Note: The  /v1/messages  endpoint in the no-tools branch does not support the  tools  parameter; cleaner for pure chat.
 
Anthropic model name mapping
 
Tools like Claude Code CLI expect Anthropic-style model names (e.g.,  claude-sonnet-4-6 ) and cannot use raw  deepseek-*  names. This proxy maps them automatically for the Anthropic endpoint:
 
Claude Model Name → DeepSeek Internal Reasoning Web Search 
 claude-opus-4-6   deepseek-expert-reasoner  ✓ ✗ 
 claude-opus-4-6-search   deepseek-expert-reasoner-search  ✓ ✓ 
 claude-sonnet-4-6   deepseek-reasoner  ✓ ✗ 
 claude-sonnet-4-6-search   deepseek-reasoner-search  ✓ ✓ 
 claude-haiku-4-5   deepseek-default  ✗ ✗ 
 claude-sonnet-4-6-nothinking   deepseek-default  ✗ ✗ 
 claude-3-7-sonnet   deepseek-reasoner  ✓ ✗ 
 claude-3-5-sonnet   deepseek-default  ✗ ✗ 
 claude-3-opus   deepseek-expert-reasoner  ✓ ✗ 
 
Legacy Claude 4.x names ( claude-sonnet-4-5 ,  claude-opus-4-1 , etc.) and  -nothinking  variants are also supported. DeepSeek native names ( deepseek-* ) continue to work directly;  /v1/models  returns native names unchanged.
 
bash
  
# Works with Claude model names too
curl http://localhost:8000/v1/messages \
  -H "x-api-key: sk-dsapi" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":100,"messages":[{"role":"user","content":"hi"}]}'
 
 
7. Model refresh
 
bash
  
# Force refresh model list (skip 1-hour cache)
curl -X POST http://localhost:8000/v1/models/refresh
 
 
Model System
 
Dynamic model discovery
 
On startup, automatically call DeepSeek official API  GET /api/v0/client/settings?scope=model  to get current available model configs.
 
Core discovery logic ( proxy.py:418 ):
 
python
  
def _discover_models():
    resp = cffi_requests.get(
        "https://chat.deepseek.com/api/v0/client/settings?scope=model",
        headers={"Authorization": f"Bearer {token}", ...}
    )
    # Parse model_configs, generate base/reasoning/search/reasoning+search variants by model_type
 
 
- Auto-detect: No manual model list updates
- 1-hour cache: Avoid frequent requests
- Manual refresh:  POST /v1/models/refresh 
- Fault-tolerant: Detection failure does not break cached list
 
Each model returns:
 
-  max_input_tokens  — max input tokens
-  max_output_tokens  — max output tokens (including reasoning)
-  thinking_enabled  — supports deep reasoning
-  search_enabled  — supports web search
 
Currently available models
 
Model list changes dynamically with DeepSeek official updates. Currently detected: 3 base models × 4 variants = 12 models:
 
Model ID Display Name Description Reasoning Search 
 deepseek-default  DeepSeek V4 Flash Base Fast base model ✗ ✗ 
 deepseek-reasoner  DeepSeek V4 Flash Reasoning + deep reasoning ✓ ✗ 
 deepseek-search  DeepSeek V4 Flash Search + web search ✗ ✓ 
 deepseek-reasoner-search  DeepSeek V4 Flash Reason+Search + reasoning + search ✓ ✓ 
 deepseek-expert  DeepSeek V4 Pro Base Pro expert model ✗ ✗ 
 deepseek-expert-reasoner  DeepSeek V4 Pro Reasoning + deep reasoning ✓ ✗ 
 deepseek-expert-search  DeepSeek V4 Pro Search + web search ✗ ✓ 
 deepseek-expert-reasoner-search  DeepSeek V4 Pro Reason+Search + reasoning + search ✓ ✓ 
 deepseek-vision  DeepSeek Vision Base Image understanding ✗ ✗ 
 deepseek-vision-reasoner  DeepSeek Vision Reasoning + deep reasoning ✓ ✗ 
 
Notes:
 
- New DeepSeek models are auto-discovered; no code changes needed
- All models explicitly set  model_type  ( default / expert / vision ) for correct routing
- Model names are English IDs; Chinese display names in table above
 
Branch Info
 
Two branches available:
 
Branch Features 
 main  (current) Full-featured — supports DSML tool calls, streaming filtering, session management. Use when tool calls are needed. 
 no-tools  Pure chat proxy — no tool-call prompt injection, cleaner output. Ideal for writing, translation, coding. 
 
You are currently on  main . To switch to pure chat:
 
bash
  
git checkout no-tools
 
 
Tool Call Details
 
DeepSeek web does not support OpenAI function-calling format. This proxy implements tool calls via DSML prompt injection + multi-strategy extraction:
 
bash
  
curl http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer sk-dsapi" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "What’s the weather in Beijing?"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather info",
        "parameters": {
          "type": "object",
          "properties": {"city": {"type": "string"}},
          "required": ["city"]
        }
      }
    }]
  }'
 
 
DSML prompt injection
 
Convert OpenAI tools definition to DSML format and inject into system message:
 
xml
  
<|DSML|tool_calls>
  <|DSML|invoke name="search_file">
    <|DSML|parameter name="query"><![CDATA[config.yaml]]></|DSML|parameter>
  </|DSML|invoke>
</|DSML|tool_calls>
 
 
Extraction strategies (priority order)
 
Priority Format Description 
DSML  <|DSML|tool_calls><|DSML|invoke name="X">...</|DSML|invoke></|DSML|tool_calls>  Primary format, 7 noise-tolerant variants 
TOOL_CALL  TOOL_CALL: name(key=value)  Legacy fallback 
JSON  {"name":"x","arguments":{...}}  JSON block parsing 
XML  <tool_call><function=NAME>...</function></tool_call>  Native XML 
Hybrid  <function_call>{...}</function_call>  XML + JSON 
 
Fault tolerance
 
- Noise resistance: Supports missing pipes, duplicate  < , full-width  ｜ , hyphenated  dsml- , 7 variants
- Code block skip: Auto-skip DSML examples inside markdown code blocks
- JSON repair: Auto-fix unquoted keys, missing array brackets
- CDATA protection: Preserve raw strings for content/command/prompt
- Missing opening tags: Auto-repair when closing tags exist without opening
 
PoW Solving Mechanism
 
DeepSeek requires Proof of Work (PoW) verification for the  /api/v0/chat/completion  endpoint.
 
Flow
 
1. Before each request: call  POST /api/v0/chat/create_pow_challenge  to get challenge
2. Solve challenge → get  x-ds-pow-response  header
3. Attach solution to chat request headers
 
Dual solver
 
Solver Method Speed Compatibility 
Node.js WASM  node pow_solver.js  subprocess Fast (seconds) Matches official algorithm 
Python fallback Pure Python  hashlib.sha3_256  Slower Fallback when Node.js unavailable 
 
Requires Node.js +  sha3_wasm_bg.wasm  (included).
 
Algorithm
 
DeepSeek uses custom  DeepSeekHashV1 , essentially SHA3-256 hash collision. WASM version (called via Node.js) matches official exactly.
 
Automatic Token Refresh
 
Token valid for ~24 hours. On 401 response:
 
1. Detect 401 → trigger  relogin() 
2. Re-login with saved password via  POST /api/v0/users/login 
3. Get new Token → create new Session → save to  token.json 
4. Retry current request with new Token (transparent to user)
 
Prerequisite: First config must use account-password login. Pure cURL/Cookie imports lack password and cannot auto-refresh.
 
Management Commands
 
bash
  
# Run in foreground
python3 proxy.py

# Start in background
./deploy.sh --bg

# Check status
./deploy.sh --status

# Stop background process
./deploy.sh --stop

# View realtime logs (background mode)
tail -f ~/dsapi.log

# Set custom port
PROXY_PORT=9000 python3 proxy.py

# Force refresh model list
curl -X POST http://localhost:8000/v1/models/refresh

# Health check
curl http://localhost:8000/health
 
 
After startup:
 
Address Description 
 http://localhost:8000/admin  Web admin (login/config) 
 http://localhost:8000/v1  OpenAI-compatible API root 
 http://localhost:8000/health  Health check endpoint 
 
Project Structure
 
plaintext
  
ds-free-api/
├── proxy.py              # Main: FastAPI app, SSE parsing, OpenAI endpoints, admin UI
├── response_store.py     # Responses API local persistence (JSON files)
├── pow_native.py         # PoW solver: Node.js WASM + Python fallback
├── pow_solver.js         # Node.js PoW script (calls WASM)
├── sha3_wasm_bg.wasm     # SHA3 WASM binary
├── deploy.sh             # One-click deploy script
├── requirements.txt      # Python dependencies
├── token.example.json    # Config template
└── token.json            # Actual config (.gitignore, includes credentials)
 
 
Core files
 
File Responsibility 
 proxy.py  Entry, routing, SSE parsing, DeepSeek API interaction, token refresh, admin panel 
 response_store.py  Responses API persistence (thread-safe JSON) 
 pow_native.py  PoW solving logic 
 deploy.sh  Deployment & process management 
 
Configuration Reference
 
Full  token.json  schema:
 
json
  
{
  "token": "eyJ...",
  "session_id": "abc-def-123...",
  "headers": {
    "content-type": "application/json",
    "origin": "https://chat.deepseek.com",
    "referer": "https://chat.deepseek.com/",
    "user-agent": "Mozilla/5.0 ...",
    "x-client-version": "2.0.2",
    "x-client-platform": "web",
    "authorization": "Bearer YOUR_TOKEN"
  },
  "account": "+86 138xxxx",
  "login_type": "phone",
  "_password": "your_password",
  "_email": "",
  "_mobile": "138xxxx",
  "_area_code": "+86"
}
 
 
Field Description Auto-generated 
 token  Bearer Token (~24h valid) ✓ 
 session_id  Chat session UUID ✓ 
 headers  Request headers (UA, auth, etc.) ✓ 
 account  Account label (display) ✓ 
 login_type   phone  /  email  Set on first login 
 _password  Login password (for refresh) Set on first login 
 _mobile  Phone (for refresh) Set on first login 
 _email  Email (for refresh) Set on first login 
 _area_code  Area code (default +86) Set on first login 
 
Security note:  _password  stored in plaintext locally. Secure  token.json  with  chmod 600  and exclude from packaging.
 
Environment variable:  PROXY_PORT  — listen port (default  8000 )
 
Dependencies
 
Python (pip)
 
bash
  
pip install fastapi uvicorn curl-cffi python-dotenv
 
 
Package Purpose 
 fastapi  Web framework 
 uvicorn  ASGI server 
 curl-cffi  HTTP client (simulate Chrome TLS fingerprint) 
 python-dotenv  Env loader 
 
System
 
- Node.js — PoW solver (required)
- Python 3.10+ — Runtime
 
Limitations & Known Issues
 
Limitation Description 
Token expiry ~24 hours; password login required for auto-refresh 
Concurrency limit ~2 concurrent requests per free account 
API coverage Chat Completions + Responses only; no Embeddings/Fine-tuning 
PoW overhead Each request requires challenge solve (Node.js: ~1–3s) 
Non-stream uses SSE DeepSeek only provides SSE; non-stream buffers full response 
Vision non-stream Vision models have no output in streaming mode; fetched non-stream then wrapped to SSE 
 
FAQ
 
Q: /admin shows blank after startup?
A: Admin panel is embedded HTML in  proxy.py . Check JS console (F12) for errors. Access directly at  http://localhost:8000/admin .
 
Q: "Update to latest version to use Expert/Vision"?
A:  x-client-version  must match DeepSeek web (currently  2.0.2 ). Proxy sets it automatically.
 
Q: PoW solve failed?
A: Verify Node.js is installed ( node --version ). If Node.js fails, proxy falls back to Python solver (slower but no external dependencies).
 
Q: Login says wrong password?
A: Confirm password is correct (min 8 chars, letters + digits). Complete captcha on web first if needed.
 
Q: What if Token expires?
A: If logged in with account/password, proxy auto-refreshes on 401. If imported via cURL/Cookie, re-import manually.
 
Q: expert model shows as default (fast mode) in history?
A: Usually due to expired Token/Session. DeepSeek falls back to default on auth failure. Fix: re-login via  /admin .
 
Q: Can I deploy to public server?
A: Yes; recommend Nginx reverse proxy + HTTPS + IP whitelist. API key not validated (any value works); control access externally.
 
License & Credits
 
MIT License
 
References:
 
- NIyueeE/ds-free-api — Rust original, API reverse-engineering & PoW reference
- CJackHwang/ds2api — DSML tool format, streaming filter, session logic reference
- GoblinHonest/mimo2api_mimoapi — Session management reference
- Acidmoon — PR #2: OpenAI Responses API compatibility
- xstjmark21-cmyk — Vision feature testing & token support

# CLAUDE.md - AI Assistant Guide for GCLI2API

## Project Overview

**GCLI2API** (GeminiCLI to API) is a Python-based proxy server that converts Google's GeminiCLI into both OpenAI-compatible and Gemini-native API interfaces. This enables developers to use Gemini models through familiar OpenAI API formats while also supporting native Gemini API calls.

### Core Purpose
- Convert Gemini OAuth credentials to REST API endpoints
- Provide OpenAI-compatible chat completion endpoints (`/v1/chat/completions`)
- Support Gemini native endpoints (`/v1/models/{model}:generateContent`)
- Manage multiple Google OAuth credentials with automatic rotation
- Offer a web-based control panel for credential and system management
- Support distributed deployment via Redis, Postgres, or MongoDB

### License
**Cooperative Non-Commercial License (CNC-1.0)** - Strictly non-commercial use only

---

## Technology Stack

### Core Framework
- **FastAPI** - Modern async web framework
- **Hypercorn** - ASGI server for production deployment
- **Pydantic** - Data validation and settings management
- **HTTPX** - Async HTTP client with SOCKS proxy support

### Storage Backends (Priority Order)
1. **Redis** - Highest priority, best performance (via `redis-py`)
2. **Postgres** - Relational database option (via `asyncpg`)
3. **MongoDB** - Document database option (via `motor`)
4. **File System** - Default fallback using TOML files

### Authentication & OAuth
- **PyJWT** - JWT token generation and validation
- **oauthlib** - OAuth 2.0 implementation
- **google-auth** - Google authentication library

### Utilities
- **python-dotenv** - Environment variable management
- **aiofiles** - Async file operations
- **python-multipart** - File upload handling
- **toml** - Configuration file parsing
- **pypinyin** - Chinese text processing

---

## Architecture Overview

### Application Structure

```
┌─────────────────────────────────────────────────────────────┐
│                         web.py                              │
│                   (Main Entry Point)                        │
└──────────────┬──────────────────────────────────────────────┘
               │
       ┌───────┴────────┐
       │   FastAPI App  │
       └───────┬────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼────┐ ┌──▼───┐ ┌────▼────┐
│OpenAI  │ │Gemini│ │  Web    │
│Router  │ │Router│ │ Routes  │
└───┬────┘ └──┬───┘ └────┬────┘
    │         │          │
    └─────────┼──────────┘
              │
    ┌─────────▼─────────┐
    │  Credential       │
    │  Manager          │
    └─────────┬─────────┘
              │
    ┌─────────▼─────────┐
    │  Storage          │
    │  Adapter          │
    └─────────┬─────────┘
              │
    ┌─────────┴─────────┐
    │ Redis/Postgres/   │
    │ MongoDB/File      │
    └───────────────────┘
```

### Request Flow

#### OpenAI Format Request
```
Client → OpenAI Router → Format Detector → OpenAI Transfer
  → Credential Manager → Google Chat API → Gemini API
    → Response Converter → Client
```

#### Gemini Format Request
```
Client → Gemini Router → Credential Manager
  → Google Chat API → Gemini API → Client
```

---

## Directory Structure

```
cccc-gcli2api/
├── web.py                      # Main application entry point
├── config.py                   # Configuration management & getters
├── log.py                      # Logging setup
├── requirements.txt            # Python dependencies
├── Dockerfile                  # Docker build configuration
├── docker-compose.yml          # Docker Compose setup
├── .env.example                # Environment variables template
├── .gitignore                  # Git ignore patterns
│
├── src/                        # Core source code
│   ├── openai_router.py       # OpenAI-compatible API routes
│   ├── gemini_router.py       # Gemini native API routes
│   ├── web_routes.py          # Control panel & auth routes
│   ├── auth.py                # OAuth flow & authentication
│   ├── credential_manager.py  # Multi-credential management
│   ├── openai_transfer.py     # OpenAI ↔ Gemini conversion
│   ├── format_detector.py     # Auto-detect request format
│   ├── google_chat_api.py     # Google Gemini API client
│   ├── google_oauth_api.py    # Google OAuth API client
│   ├── httpx_client.py        # Shared HTTP client manager
│   ├── state_manager.py       # State persistence layer
│   ├── storage_adapter.py     # Storage backend abstraction
│   ├── usage_stats.py         # Usage tracking & quotas
│   ├── task_manager.py        # Async task lifecycle
│   ├── anti_truncation.py     # Stream anti-truncation logic
│   ├── utils.py               # Utility functions
│   ├── models.py              # Pydantic data models
│   │
│   └── storage/               # Storage implementations
│       ├── redis_manager.py   # Redis backend
│       ├── postgres_manager.py # Postgres backend
│       ├── mongodb_manager.py # MongoDB backend
│       ├── file_storage_manager.py # File system backend
│       └── cache_manager.py   # Caching layer
│
├── front/                     # Frontend HTML files
│   ├── control_panel.html     # Desktop control panel
│   ├── control_panel_mobile.html # Mobile control panel
│   └── multi_user_auth_web.html  # Multi-user auth UI
│
├── docs/                      # Documentation
│   ├── README_EN.md           # English README
│   └── DONATE.md              # Donation information
│
├── .github/workflows/         # CI/CD pipelines
│   └── docker-build.yml       # Docker image build
│
├── test_*.py                  # Test files
│   ├── test_tool_calling.py   # Tool calling tests
│   ├── test_multi_turn_tool_calls.py
│   └── test_edge_cases.py
│
└── install scripts            # Platform-specific installers
    ├── install.sh             # Linux installer
    ├── install.ps1            # Windows installer
    ├── darwin-install.sh      # macOS installer
    ├── termux-install.sh      # Termux installer
    ├── start.sh               # Linux/macOS starter
    ├── start.bat              # Windows starter
    └── termux-start.sh        # Termux starter
```

---

## Core Modules Reference

### 1. `web.py` - Application Entry Point
**Purpose**: Main application initialization and lifecycle management

**Key Components**:
- `lifespan()` - Async context manager for app startup/shutdown
- `global_credential_manager` - Singleton credential manager
- Route registration for all routers
- Static file serving for `/docs`
- Health check endpoint (`/keepalive`)

**Startup Sequence**:
1. Initialize credential manager
2. Auto-load environment variable credentials (if enabled)
3. Start FastAPI with Hypercorn
4. Mount routers: OpenAI, Gemini, Web

**Shutdown Sequence**:
1. Close all async tasks (via `task_manager`)
2. Close credential manager
3. Clean up resources

---

### 2. `config.py` - Configuration Management
**Purpose**: Centralized configuration with priority: ENV > Storage > Default

**Key Features**:
- Async configuration getters
- Model name parsing and feature detection
- Safety settings for Google API
- Proxy and network configuration

**Important Functions**:
- `get_config_value(key, default, env_var)` - Universal config getter
- `get_api_password()` / `get_panel_password()` - Separate password support
- `get_available_models(router_type)` - Generate model list with features
- `is_fake_streaming_model()` / `is_anti_truncation_model()` - Feature detection
- `get_base_model_name()` - Extract base model from variant

**Configuration Priority**:
```
Environment Variable → Storage Backend → Default Value
```

**Model Naming Convention**:
- **Prefix features**: `假流式/`, `流式抗截断/`
- **Suffix features**: `-maxthinking`, `-nothinking`, `-search`
- **Examples**:
  - `gemini-2.5-pro-maxthinking`
  - `假流式/gemini-2.5-flash-search`
  - `流式抗截断/gemini-2.5-pro-nothinking`

---

### 3. `src/credential_manager.py` - Credential Rotation
**Purpose**: Manage multiple OAuth credentials with intelligent rotation

**Key Features**:
- Round-robin credential rotation
- Automatic credential disabling on errors (401, 403, 429)
- Quota-based reset timing (UTC+7 timezone)
- Concurrent request safety with asyncio.Lock
- Email fetching for credentials

**Important Methods**:
- `initialize()` - Load all credentials from storage
- `get_next_credential()` - Get next available credential
- `rotate_credential()` - Force rotation to next credential
- `disable_credential(filename, error_code)` - Disable on error
- `enable_credential(filename)` - Re-enable credential
- `refresh_all_credentials()` - Refresh all access tokens

**Credential State Tracking**:
- `disabled: bool` - Manual or auto-disabled
- `error_codes: List[int]` - Recent error codes
- `last_success: str` - Last successful use timestamp
- `call_count: int` - Number of API calls made
- `quota_reset_time: str` - Next quota reset time

---

### 4. `src/openai_router.py` - OpenAI Compatible API
**Purpose**: Handle OpenAI-format requests and convert to Gemini

**Endpoints**:
- `POST /v1/chat/completions` - Chat completion (stream/non-stream)
- `GET /v1/models` - List available models

**Request Flow**:
1. Authenticate via Bearer token
2. Detect format (OpenAI or Gemini native)
3. Convert to Gemini format if needed
4. Get credential from manager
5. Call Google Chat API
6. Convert response back to OpenAI format
7. Handle streaming or non-streaming response

**Features**:
- Automatic format detection
- Multi-modal support (text + images)
- Tool calling support
- Reasoning content separation
- Anti-truncation for long responses

---

### 5. `src/gemini_router.py` - Gemini Native API
**Purpose**: Handle Gemini-format requests directly

**Endpoints**:
- `POST /v1/models/{model}:generateContent` - Non-streaming generation
- `POST /v1/models/{model}:streamGenerateContent` - Streaming generation
- `GET /v1/models` - List models
- `GET /v1/models/{model}` - Get model info

**Authentication Methods**:
1. `Authorization: Bearer <token>`
2. `x-goog-api-key: <token>`
3. URL parameter: `?key=<token>`

**Features**:
- Native Gemini request/response format
- Direct passthrough to Google API
- Minimal conversion overhead
- Full Gemini feature support

---

### 6. `src/web_routes.py` - Control Panel Routes
**Purpose**: Web interface for credential and system management

**Route Groups**:

#### Authentication (`/auth/*`)
- `/auth/login` - Panel login
- `/auth/start` - Start OAuth flow
- `/auth/callback` - OAuth callback handler
- `/auth/status/{project_id}` - Check auth status

#### Credential Management (`/creds/*`, `/auth/*`)
- `/creds/status` - List all credentials
- `/creds/action` - Enable/disable/delete credential
- `/creds/batch-action` - Bulk operations
- `/auth/upload` - Upload credential files (ZIP support)
- `/creds/download/{filename}` - Download credential
- `/creds/download-all` - Download all as ZIP
- `/creds/fetch-email/{filename}` - Fetch Google account email
- `/creds/refresh-all-emails` - Bulk email refresh

#### Configuration (`/config/*`)
- `/config/get` - Get current config
- `/config/save` - Save config to storage

#### Logging (`/auth/logs/*`)
- `/auth/logs/clear` - Clear logs
- `/auth/logs/download` - Download log file
- `/auth/logs/stream` (WebSocket) - Real-time log streaming

#### Usage Statistics (`/usage/*`)
- `/usage/stats` - Get usage statistics
- `/usage/aggregated` - Get aggregated stats
- `/usage/update-limits` - Update rate limits
- `/usage/reset` - Reset usage counters

---

### 7. `src/storage_adapter.py` - Storage Abstraction
**Purpose**: Unified interface for all storage backends

**Storage Priority**:
```
Redis (fastest) > Postgres > MongoDB > File System (default)
```

**Auto-Detection Logic**:
1. Check for `REDIS_URI` → Use Redis
2. Check for `POSTGRES_DSN` → Use Postgres
3. Check for `MONGODB_URI` → Use MongoDB
4. Fallback → Use file system

**Common Interface**:
```python
class StorageAdapter:
    async def get_credential(filename: str) → dict
    async def save_credential(filename: str, data: dict)
    async def delete_credential(filename: str)
    async def list_credentials() → List[str]
    async def get_credential_status(filename: str) → dict
    async def update_credential_status(filename: str, status: dict)
    async def get_config(key: str) → Any
    async def save_config(key: str, value: Any)
```

**Key Features**:
- Transparent backend switching
- Automatic migration support
- Consistent API across all backends
- Built-in caching for frequently accessed data

---

### 8. `src/openai_transfer.py` - Format Conversion
**Purpose**: Bidirectional conversion between OpenAI and Gemini formats

**Key Functions**:

#### Request Conversion
- `openai_to_gemini(request: OpenAIChatCompletionRequest)` → `GeminiRequest`
  - Converts messages to contents/systemInstruction
  - Maps temperature, top_p, max_tokens, etc.
  - Handles multi-modal content (text + images)
  - Processes tool definitions and tool choice

#### Response Conversion
- `gemini_to_openai(response: GeminiResponse)` → `OpenAIChatCompletionResponse`
  - Converts candidates to choices
  - Separates thinking from final answer
  - Maps finish reasons
  - Calculates token usage

#### Streaming Conversion
- `gemini_stream_to_openai_stream()` - Async generator for SSE

**Special Handling**:
- **Thinking Content**: Automatically separated into `reasoning_content` field
- **Tool Calls**: Converted between formats with unique IDs
- **Images**: Base64 inline data and file references
- **System Instructions**: Compatibility mode converts to user messages

---

### 9. `src/google_chat_api.py` - Google API Client
**Purpose**: Direct interface to Google Gemini API

**Key Functions**:
- `call_google_api(credential, model, payload, stream=False)`
- `handle_429_with_retry()` - Automatic retry on rate limits
- `extract_quota_reset_time()` - Parse reset time from headers

**Error Handling**:
- **429 (Rate Limit)**: Auto-retry with exponential backoff
- **401/403 (Auth Error)**: Disable credential, rotate to next
- **500+ (Server Error)**: Log and retry
- **Network Errors**: Retry with timeout

**Features**:
- Automatic credential rotation on error
- Precise quota reset time tracking (millisecond accuracy)
- Stream and non-stream support
- Proxy configuration support

---

### 10. `src/auth.py` - OAuth Flow Manager
**Purpose**: Handle Google OAuth 2.0 authentication flow

**OAuth Flow**:
1. `start_oauth_flow()` - Generate auth URL
2. User authorizes in browser
3. `handle_oauth_callback()` - Exchange code for tokens
4. `save_credentials()` - Persist to storage
5. `refresh_access_token()` - Refresh when expired

**Key Functions**:
- `auto_load_env_credentials_on_startup()` - Load from `GCLI_CREDS_*` env vars
- `upload_credential_file()` - Import JSON credentials
- `validate_credential()` - Test credential validity
- `get_user_email()` - Fetch associated email

**Environment Variable Credentials**:
```bash
export GCLI_CREDS_1='{"client_id":"...","refresh_token":"..."}'
export GCLI_CREDS_myproject='{"client_id":"...","project_id":"myproject"}'
export AUTO_LOAD_ENV_CREDS=true
```

---

### 11. `src/anti_truncation.py` - Stream Continuation
**Purpose**: Detect and continue truncated streaming responses

**Truncation Detection**:
- Response ends without proper finish reason
- Content stops mid-sentence
- No `STOP` or `MAX_TOKENS` finish reason

**Continuation Strategy**:
1. Detect truncation in stream
2. Append "continue" message to conversation
3. Request continuation from API
4. Merge continued content seamlessly
5. Repeat up to `ANTI_TRUNCATION_MAX_ATTEMPTS` times

**Usage**:
```python
model = "流式抗截断/gemini-2.5-pro"
```

---

### 12. `src/usage_stats.py` - Usage Tracking
**Purpose**: Track API usage per credential with daily quotas

**Tracked Metrics**:
- Total API calls per credential
- Gemini 2.5 Pro specific calls
- Quota reset timestamps (UTC+7)
- Success/failure rates

**Key Functions**:
- `increment_usage(credential_filename, model)`
- `get_usage_stats()` - Retrieve all stats
- `reset_daily_usage()` - Reset counters (auto at UTC+7 midnight)
- `check_quota_exceeded(credential_filename)` - Quota enforcement

**Daily Limits**:
- Default: 1000 requests per credential per day
- Configurable via `update_limits()`

---

### 13. `src/task_manager.py` - Async Task Lifecycle
**Purpose**: Global async task registry and cleanup

**Key Functions**:
- `register_task(task: asyncio.Task, name: str)` - Register task
- `unregister_task(task_id: str)` - Remove from registry
- `shutdown_all_tasks(timeout: float)` - Graceful shutdown

**Use Cases**:
- Long-running background tasks
- Streaming response handlers
- Periodic cleanup jobs
- OAuth callback servers

---

## Storage System Details

### Redis Backend (`src/storage/redis_manager.py`)
**Advantages**: Fastest, best for distributed deployment

**Configuration**:
```bash
REDIS_URI=redis://localhost:6379
REDIS_DATABASE=0
```

**Data Structure**:
- Credentials: `cred:{filename}` (JSON hash)
- Status: `status:{filename}` (JSON hash)
- Config: `config:{key}` (string)

**Features**:
- Connection pooling
- Automatic reconnection
- TTL support for temporary data
- Pub/Sub for real-time updates

---

### Postgres Backend (`src/storage/postgres_manager.py`)
**Advantages**: ACID compliance, relational queries

**Configuration**:
```bash
POSTGRES_DSN=postgresql://user:pass@localhost:5432/gcli2api
```

**Schema**:
```sql
CREATE TABLE credentials (
    filename TEXT PRIMARY KEY,
    data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE credential_status (
    filename TEXT PRIMARY KEY,
    status JSONB NOT NULL,
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE config (
    key TEXT PRIMARY KEY,
    value JSONB NOT NULL,
    updated_at TIMESTAMP DEFAULT NOW()
);
```

---

### MongoDB Backend (`src/storage/mongodb_manager.py`)
**Advantages**: Flexible schema, easy scaling

**Configuration**:
```bash
MONGODB_URI=mongodb://localhost:27017
MONGODB_DATABASE=gcli2api
```

**Collections**:
- `credentials` - Credential JSON documents
- `credential_status` - Status tracking
- `config` - Configuration key-value pairs

**Indexes**:
- `filename` (unique) on all collections
- `updated_at` for sorting

---

### File System Backend (`src/storage/file_storage_manager.py`)
**Advantages**: Simple, no external dependencies

**Structure**:
```
./creds/
├── credential1.json
├── credential2.json
├── config.toml
└── state.toml
```

**Locking**:
- File-based locks for concurrent access
- TOML format for human readability
- Atomic writes with temp files

---

## Development Workflow

### Setting Up Development Environment

```bash
# Clone repository
git clone <repository-url>
cd cccc-gcli2api

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# OR
venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt

# Copy environment template
cp .env.example .env
# Edit .env with your configuration

# Run development server
python web.py
```

### Running Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest test_tool_calling.py

# Run with coverage
pytest --cov=src --cov-report=html
```

**Test Files**:
- `test_tool_calling.py` - Tool/function calling tests
- `test_multi_turn_tool_calls.py` - Multi-turn conversation tests
- `test_edge_cases.py` - Edge case and error handling tests

---

### Git Workflow

**Branch Strategy**:
- `main` / `master` - Production-ready code
- `claude/*` - Feature branches created by Claude AI
- Feature branches should be descriptive

**Commit Message Style** (from git log):
```
feat: add ARM64 support to Docker build workflow
refactor(zeabur): 简化密码配置，使用通用PASSWORD
fix(并发): 彻底解决凭证刷新时的死锁与兼容性问题
```

**Format**: `<type>(<scope>): <description>`
- **Types**: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`
- **Scope**: Module or feature area (optional)
- **Description**: Brief summary in Chinese or English

**Important Git Commands**:
```bash
# Push to feature branch
git push -u origin claude/<branch-name>

# Retry on network failure (exponential backoff)
# Retry up to 4 times: 2s, 4s, 8s, 16s delays

# Fetch specific branch
git fetch origin <branch-name>

# Pull with rebase
git pull --rebase origin <branch-name>
```

---

### Docker Development

**Build Image**:
```bash
docker build -t gcli2api:dev .
```

**Run Container**:
```bash
docker run -d \
  --name gcli2api-dev \
  -p 7861:7861 \
  -e PASSWORD=devpwd \
  -e LOG_LEVEL=debug \
  -v $(pwd)/creds:/app/creds \
  gcli2api:dev
```

**Docker Compose**:
```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

---

## Configuration Reference

### Environment Variables

**Priority**: `ENV > Storage > Default`

#### Server Configuration
- `HOST` - Server listen address (default: `0.0.0.0`)
- `PORT` - Server port (default: `7861`)

#### Password Configuration
- `PASSWORD` - Universal password (highest priority)
- `API_PASSWORD` - Chat API password (default: inherits `PASSWORD` or `pwd`)
- `PANEL_PASSWORD` - Control panel password (default: inherits `PASSWORD` or `pwd`)

#### Storage Configuration (Priority Order)
1. **Redis** (highest priority)
   - `REDIS_URI` - Redis connection string
   - `REDIS_DATABASE` - Database index (0-15, default: `0`)

2. **Postgres**
   - `POSTGRES_DSN` - Postgres connection string

3. **MongoDB**
   - `MONGODB_URI` - MongoDB connection string
   - `MONGODB_DATABASE` - Database name (default: `gcli2api`)

4. **File System** (default if none set)
   - `CREDENTIALS_DIR` - Credential directory (default: `./creds`)

#### Credential Management
- `AUTO_LOAD_ENV_CREDS` - Auto-load from `GCLI_CREDS_*` env vars (default: `false`)
- `CALLS_PER_ROTATION` - Calls before rotating credential (default: `100`)

#### Error Handling
- `AUTO_BAN` - Auto-disable credentials on errors (default: `false`)
- `AUTO_BAN_ERROR_CODES` - Error codes to ban on (default: `400,403`)
- `RETRY_429_ENABLED` - Retry on 429 errors (default: `true`)
- `RETRY_429_MAX_RETRIES` - Max retries for 429 (default: `5`)
- `RETRY_429_INTERVAL` - Retry interval in seconds (default: `1`)
- `MAX_429_ERRORS_PER_CREDENTIAL` - Max 429 errors before daily ban (default: `5`)

#### Advanced Features
- `ANTI_TRUNCATION_MAX_ATTEMPTS` - Max continuation attempts (default: `3`)
- `COMPATIBILITY_MODE` - Convert system messages to user (default: `true`)

#### Proxy Configuration
- `PROXY` - HTTP/HTTPS/SOCKS5 proxy URL
- `OAUTH_PROXY_URL` - OAuth endpoint proxy (default: `https://oauth2.googleapis.com`)
- `GOOGLEAPIS_PROXY_URL` - Google APIs proxy (default: `https://www.googleapis.com`)
- `RESOURCE_MANAGER_API_URL` - Resource Manager API URL
- `SERVICE_USAGE_API_URL` - Service Usage API URL
- `CODE_ASSIST_ENDPOINT` - Code Assist endpoint

#### Logging
- `LOG_LEVEL` - Log level: `debug`, `info`, `warning`, `error`, `critical` (default: `info`)
- `LOG_FILE` - Log file path (default: `log.txt`)

---

### TOML Configuration

**File Location**: `{CREDENTIALS_DIR}/config.toml`

**Example**:
```toml
# Passwords
api_password = "your_api_password"
panel_password = "your_panel_password"
password = "your_common_password"  # Overrides above

# Rotation
calls_per_rotation = 100

# Retry Configuration
[retry]
retry_429_enabled = true
retry_429_max_retries = 5
retry_429_interval = 1

# Logging
[logging]
log_level = "info"
log_file = "log.txt"

# Auto Ban
[auto_ban]
auto_ban_enabled = false
auto_ban_error_codes = [400, 403]

# Anti-truncation
[anti_truncation]
max_attempts = 3
```

---

## Key Coding Conventions

### Async/Await Patterns

**Always use async for I/O operations**:
```python
# Good
async def get_credential(filename: str):
    async with aiofiles.open(filename, 'r') as f:
        return await f.read()

# Bad
def get_credential(filename: str):
    with open(filename, 'r') as f:
        return f.read()
```

**Use asyncio.Lock for concurrent access**:
```python
class CredentialManager:
    def __init__(self):
        self._lock = asyncio.Lock()

    async def rotate_credential(self):
        async with self._lock:
            # Critical section
            self.current_index = (self.current_index + 1) % len(self.credentials)
```

---

### Error Handling Patterns

**Specific exception handling**:
```python
try:
    response = await call_google_api(credential, model, payload)
except httpx.HTTPStatusError as e:
    if e.response.status_code == 429:
        # Handle rate limit
        await self.disable_credential(filename, 429)
        return await self.retry_with_next_credential()
    elif e.response.status_code in [401, 403]:
        # Handle auth error
        await self.disable_credential(filename, e.response.status_code)
        raise
    else:
        # Generic HTTP error
        log.error(f"HTTP error {e.response.status_code}: {e}")
        raise
except asyncio.TimeoutError:
    log.error("Request timeout")
    raise
except Exception as e:
    log.error(f"Unexpected error: {e}")
    raise
```

**Graceful degradation**:
```python
async def get_user_email(credential):
    try:
        email = await fetch_email_from_api(credential)
        return email
    except Exception as e:
        log.warning(f"Failed to fetch email: {e}")
        return "unknown@example.com"  # Fallback value
```

---

### Logging Conventions

**Use structured logging**:
```python
from log import log

# Include context in log messages
log.info(f"Rotating to credential: {filename} (index: {index})")
log.warning(f"Credential {filename} disabled due to error {error_code}")
log.error(f"API call failed: {error_message}", exc_info=True)
```

**Beijing Time (UTC+8) for timestamps**:
All logs are automatically timestamped in Beijing Time for consistency.

---

### Configuration Access Patterns

**Always use async getters**:
```python
# Good
from config import get_api_password, get_retry_429_enabled

async def authenticate(token: str):
    api_password = await get_api_password()
    return token == api_password

# Bad
from config import get_api_password

def authenticate(token: str):
    api_password = get_api_password()  # Will fail - not awaited
    return token == api_password
```

**Respect environment variable priority**:
```python
# Environment variables always take precedence
# Never override env vars in code
async def get_proxy():
    # This function checks ENV first, then storage, then default
    return await get_config_value("proxy", env_var="PROXY")
```

---

### Model Name Handling

**Always extract base model**:
```python
from config import get_base_model_from_feature_model, get_base_model_name

# Remove feature prefixes
model = "流式抗截断/gemini-2.5-pro-maxthinking"
base_with_suffix = get_base_model_from_feature_model(model)
# → "gemini-2.5-pro-maxthinking"

# Remove feature suffixes
final_base = get_base_model_name(base_with_suffix)
# → "gemini-2.5-pro"
```

**Check features**:
```python
from config import (
    is_fake_streaming_model,
    is_anti_truncation_model,
    is_search_model,
    is_maxthinking_model,
    get_thinking_budget
)

if is_anti_truncation_model(model):
    # Enable anti-truncation logic
    max_attempts = await get_anti_truncation_max_attempts()

if is_search_model(model):
    # Add search grounding
    payload["tools"] = [{"googleSearch": {}}]

thinking_budget = get_thinking_budget(model)
if thinking_budget:
    payload["generationConfig"]["thinkingConfig"] = {
        "maxThinkingTime": thinking_budget
    }
```

---

### Response Streaming Patterns

**Use async generators for SSE**:
```python
async def stream_response(response: httpx.Response):
    async for line in response.aiter_lines():
        if line.startswith("data: "):
            data = line[6:]
            if data == "[DONE]":
                break
            yield f"data: {data}\n\n"
```

**Wrap in StreamingResponse**:
```python
from fastapi.responses import StreamingResponse

@router.post("/v1/chat/completions")
async def chat_completions(request: ChatCompletionRequest):
    if request.stream:
        return StreamingResponse(
            stream_generator(),
            media_type="text/event-stream"
        )
    else:
        return await non_stream_handler()
```

---

## Common Development Tasks

### Adding a New API Endpoint

1. **Define Pydantic model** in `src/models.py`:
```python
class NewFeatureRequest(BaseModel):
    param1: str
    param2: int
```

2. **Add route** in appropriate router:
```python
# In src/openai_router.py or src/gemini_router.py
@router.post("/v1/new-feature")
async def new_feature(request: NewFeatureRequest):
    # Implementation
    return {"result": "success"}
```

3. **Update documentation** in this file and README.md

---

### Adding a New Configuration Option

1. **Add to `.env.example`**:
```bash
# New feature configuration
NEW_FEATURE_ENABLED=true
NEW_FEATURE_TIMEOUT=30
```

2. **Add async getter to `config.py`**:
```python
async def get_new_feature_enabled() -> bool:
    env_value = os.getenv("NEW_FEATURE_ENABLED")
    if env_value:
        return env_value.lower() in ("true", "1", "yes", "on")
    return bool(await get_config_value("new_feature_enabled", True))

async def get_new_feature_timeout() -> int:
    env_value = os.getenv("NEW_FEATURE_TIMEOUT")
    if env_value:
        try:
            return int(env_value)
        except ValueError:
            pass
    return int(await get_config_value("new_feature_timeout", 30))
```

3. **Use in code**:
```python
from config import get_new_feature_enabled, get_new_feature_timeout

if await get_new_feature_enabled():
    timeout = await get_new_feature_timeout()
    # Use feature
```

---

### Adding a New Storage Backend

1. **Create manager in `src/storage/`**:
```python
# src/storage/new_backend_manager.py
class NewBackendManager:
    async def initialize(self):
        # Setup connection
        pass

    async def get_credential(self, filename: str):
        # Implementation
        pass

    # Implement all StorageAdapter methods
```

2. **Update `src/storage_adapter.py`**:
```python
async def get_storage_adapter():
    # Add priority check
    if os.getenv("NEW_BACKEND_URI"):
        from src.storage.new_backend_manager import NewBackendManager
        manager = NewBackendManager()
        await manager.initialize()
        return manager

    # Continue with existing priority chain
```

3. **Add configuration**:
- Update `.env.example`
- Add async getter to `config.py`
- Document in README.md

---

### Handling New Google API Features

1. **Update `src/google_chat_api.py`** for new API parameters
2. **Update `src/models.py`** with new Pydantic fields
3. **Update `src/openai_transfer.py`** for format conversion
4. **Add configuration** if feature requires settings
5. **Write tests** in `test_*.py` files

---

### Debugging Common Issues

#### Credential Rotation Not Working
```python
# Check credential manager state
manager = get_credential_manager()
print(f"Current index: {manager.current_index}")
print(f"Total credentials: {len(manager.credentials)}")
print(f"Available credentials: {[c for c in manager.credentials if not c.get('disabled')]}")
```

#### Storage Backend Not Detected
```python
# Check detection logic
from src.storage_adapter import get_storage_adapter
storage = await get_storage_adapter()
print(f"Using storage backend: {type(storage).__name__}")
```

#### Streaming Response Cut Off
- Check `ANTI_TRUNCATION_MAX_ATTEMPTS` setting
- Verify client supports SSE properly
- Check network timeout settings
- Review `src/anti_truncation.py` logic

#### 429 Rate Limit Issues
- Check `RETRY_429_ENABLED` and related settings
- Verify `quota_reset_time` in credential status
- Review `MAX_429_ERRORS_PER_CREDENTIAL` limit
- Check credential rotation is working

---

## API Endpoint Reference

### OpenAI Compatible Endpoints

#### POST `/v1/chat/completions`
**Authentication**: `Authorization: Bearer <api_password>`

**Request Body**:
```json
{
  "model": "gemini-2.5-pro",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "Hello"}
  ],
  "temperature": 0.7,
  "max_tokens": 1000,
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string"}
          },
          "required": ["location"]
        }
      }
    }
  ]
}
```

**Response** (non-stream):
```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "gemini-2.5-pro",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you?",
        "reasoning_content": "Thinking process..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 20,
    "total_tokens": 30
  }
}
```

---

#### GET `/v1/models`
**Authentication**: `Authorization: Bearer <api_password>`

**Response**:
```json
{
  "object": "list",
  "data": [
    {
      "id": "gemini-2.5-pro",
      "object": "model",
      "owned_by": "google"
    },
    {
      "id": "流式抗截断/gemini-2.5-pro-maxthinking",
      "object": "model",
      "owned_by": "google"
    }
  ]
}
```

---

### Gemini Native Endpoints

#### POST `/v1/models/{model}:generateContent`
**Authentication**: `x-goog-api-key: <api_password>` or `Authorization: Bearer <api_password>` or `?key=<api_password>`

**Request Body**:
```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{"text": "Hello"}]
    }
  ],
  "systemInstruction": {
    "parts": [{"text": "You are a helpful assistant"}]
  },
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 1000
  }
}
```

**Response**:
```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [{"text": "Hello! How can I help you?"}]
      },
      "finishReason": "STOP",
      "index": 0
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 10,
    "candidatesTokenCount": 20,
    "totalTokenCount": 30
  }
}
```

---

#### POST `/v1/models/{model}:streamGenerateContent`
Same as above but returns Server-Sent Events (SSE) stream.

---

### Web Control Panel Endpoints

All control panel endpoints require authentication via JWT token obtained from `/auth/login`.

#### POST `/auth/login`
**Request**:
```json
{
  "password": "your_panel_password"
}
```

**Response**:
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

#### GET `/creds/status`
**Headers**: `Authorization: Bearer <jwt_token>`

**Response**:
```json
{
  "credentials": [
    {
      "filename": "credential1.json",
      "project_id": "my-project-123",
      "status": {
        "disabled": false,
        "error_codes": [],
        "last_success": "2025-11-16T10:30:00+08:00",
        "call_count": 45
      },
      "email": "user@gmail.com"
    }
  ]
}
```

---

## Testing Guidelines

### Test Structure

Each test file should follow this pattern:
```python
import pytest
from httpx import AsyncClient
from web import app

@pytest.mark.asyncio
async def test_feature_name():
    """Description of what is being tested"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/v1/chat/completions",
            json={"model": "gemini-2.5-pro", "messages": [...]},
            headers={"Authorization": "Bearer pwd"}
        )
        assert response.status_code == 200
        data = response.json()
        assert "choices" in data
```

### Test Categories

1. **Unit Tests**: Test individual functions
2. **Integration Tests**: Test API endpoints
3. **Edge Case Tests**: Test error handling and boundaries

### Running Tests

```bash
# All tests
pytest

# Specific file
pytest test_tool_calling.py

# Specific test
pytest test_tool_calling.py::test_function_name

# With verbose output
pytest -v

# With coverage
pytest --cov=src --cov-report=term-missing
```

---

## Deployment Guidelines

### Local Deployment

```bash
# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env

# Run server
python web.py
```

Access at: `http://127.0.0.1:7861`

---

### Docker Deployment

**Single Container**:
```bash
docker run -d \
  --name gcli2api \
  -p 7861:7861 \
  -e PASSWORD=your_password \
  -v ./creds:/app/creds \
  ghcr.io/cyskysky/cccc-gcli2api:latest
```

**With Redis**:
```bash
docker run -d \
  --name gcli2api \
  --network my_network \
  -p 7861:7861 \
  -e PASSWORD=your_password \
  -e REDIS_URI=redis://redis:6379 \
  ghcr.io/cyskysky/cccc-gcli2api:latest
```

**Docker Compose** (see `docker-compose.yml`)

---

### Zeabur Deployment

Configuration in `zeabur.yaml`. Use the deploy button in README.md.

**Required Environment Variables**:
- `PASSWORD` or `API_PASSWORD` + `PANEL_PASSWORD`
- Optional: `REDIS_URI`, `MONGODB_URI`, etc.

---

### Platform-Specific Notes

**Termux (Android)**:
```bash
curl -o termux-install.sh "https://raw.githubusercontent.com/.../termux-install.sh"
chmod +x termux-install.sh
./termux-install.sh
```

**Windows**:
```powershell
iex (iwr "https://raw.githubusercontent.com/.../install.ps1" -UseBasicParsing).Content
```

**macOS/Linux**:
```bash
curl -o install.sh "https://raw.githubusercontent.com/.../install.sh"
chmod +x install.sh
./install.sh
```

---

## Important Notes for AI Assistants

### When Making Changes

1. **Always read relevant files first** before making modifications
2. **Maintain async/await patterns** throughout the codebase
3. **Test configuration changes** with different priority levels (ENV > Storage > Default)
4. **Update documentation** when adding features or changing behavior
5. **Follow existing code style** and conventions
6. **Add logging** for important operations
7. **Handle errors gracefully** with specific exception handling
8. **Consider backward compatibility** when modifying APIs

### Code Review Checklist

- [ ] All I/O operations use async/await
- [ ] Configuration uses async getters from `config.py`
- [ ] Error handling covers specific exception types
- [ ] Logging includes sufficient context
- [ ] New endpoints have authentication
- [ ] Changes are documented in CLAUDE.md or README.md
- [ ] Environment variables added to `.env.example`
- [ ] Storage operations work with all backends
- [ ] Model name handling uses helper functions
- [ ] No hardcoded credentials or secrets

### Common Pitfalls to Avoid

1. **Blocking I/O in async functions**
   ```python
   # Bad
   async def load_data():
       with open("file.json") as f:  # Blocking!
           return json.load(f)

   # Good
   async def load_data():
       async with aiofiles.open("file.json") as f:
           return json.loads(await f.read())
   ```

2. **Not respecting environment variable priority**
   ```python
   # Bad
   async def get_password():
       return "hardcoded"  # Ignores ENV and storage!

   # Good
   async def get_password():
       return await get_config_value("password", "default", "PASSWORD")
   ```

3. **Missing error handling for credentials**
   ```python
   # Bad
   credential = manager.get_next_credential()  # May fail!
   response = await call_api(credential)

   # Good
   try:
       credential = await manager.get_next_credential()
       response = await call_api(credential)
   except NoAvailableCredentialsError:
       return {"error": "All credentials exhausted"}
   ```

4. **Not cleaning up async tasks**
   ```python
   # Bad
   asyncio.create_task(background_task())  # No tracking!

   # Good
   from src.task_manager import register_task
   task = asyncio.create_task(background_task())
   register_task(task, "background_task")
   ```

### Performance Considerations

- **Use Redis** for production deployments (fastest storage)
- **Configure credential rotation** appropriately (`CALLS_PER_ROTATION`)
- **Enable 429 retry** for better reliability
- **Use anti-truncation** sparingly (adds latency)
- **Monitor log file size** and rotate periodically
- **Limit concurrent requests** to avoid overwhelming credentials

### Security Best Practices

- **Never commit** `.json` credential files
- **Never commit** `.env` files
- **Always use** environment variables for secrets
- **Validate** all user inputs
- **Authenticate** all sensitive endpoints
- **Use HTTPS** in production
- **Rotate** API passwords regularly
- **Monitor** for unusual credential usage patterns

---

## Quick Reference

### Start Application
```bash
python web.py
```

### Access Points
- **Control Panel**: `http://127.0.0.1:7861`
- **OpenAI API**: `http://127.0.0.1:7861/v1`
- **Gemini API**: `http://127.0.0.1:7861`

### Key Files
- `web.py` - Entry point
- `config.py` - Configuration
- `src/credential_manager.py` - Credential logic
- `src/openai_router.py` - OpenAI endpoints
- `src/gemini_router.py` - Gemini endpoints
- `src/web_routes.py` - Control panel

### Environment Variables (Most Important)
```bash
PORT=7861
PASSWORD=your_password
REDIS_URI=redis://localhost:6379  # Optional
LOG_LEVEL=info
```

### Model Examples
- `gemini-2.5-pro`
- `假流式/gemini-2.5-flash`
- `流式抗截断/gemini-2.5-pro-maxthinking`
- `gemini-2.5-pro-search`

---

## Additional Resources

### Documentation Files
- `README.md` - User-facing documentation (Chinese)
- `docs/README_EN.md` - English documentation
- `TOOL_CALLING_ANALYSIS.md` - Tool calling implementation details
- `CODE_REVIEW.md` - Code review findings
- `PRODUCTION_FIXES.md` - Production issue fixes

### External Links
- FastAPI Documentation: https://fastapi.tiangolo.com
- Pydantic Documentation: https://docs.pydantic.dev
- HTTPX Documentation: https://www.python-httpx.org
- Google Gemini API: https://ai.google.dev/docs

---

## Changelog

### Recent Updates (from README.md)

**2025-11-11**:
- Refactored credential rotation logic for fairness
- Implemented API-driven precise disabling based on `quotaResetTimeStamp`
- Fixed concurrency race conditions with `asyncio.Lock`
- Enhanced logging with Beijing Time (UTC+8) and credential indexing
- Fixed "zombie data" issue in credential deletion

**Previous**:
- Added ARM64 Docker support
- Simplified password configuration with universal `PASSWORD`
- Fixed concurrent credential refresh deadlocks
- Fixed `PORT` environment variable handling

---

**Document Version**: 1.0
**Last Updated**: 2025-11-16
**Maintained By**: AI Assistants working on GCLI2API

---

*This document is meant to be a living reference. Update it as the codebase evolves.*

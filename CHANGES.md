# Changes: Streamable HTTP Transport + Bearer Token Auth

## Summary

Added support for the MCP **Streamable HTTP transport** alongside the existing stdio transport, and added **Bearer token authentication** that bypasses the full OAuth flow. This makes the server usable in HTTP-based deployments where a pre-authenticated Google access token is available (e.g. from a proxy or gateway that handles OAuth).

## What Changed

**Single file modified:** `src/index.ts`

### 1. Streamable HTTP Transport
- New `--transport http` flag (or `GOOGLE_DRIVE_MCP_TRANSPORT=http` env var)
- Configurable port via `--port <n>` or `GOOGLE_DRIVE_MCP_PORT` (default: 3001)
- Endpoint: `POST /mcp` with stateful session management (`mcp-session-id` header)
- Each session gets its own `Server` + `StreamableHTTPServerTransport` instance

### 2. Bearer Token Authentication
- `Authorization: Bearer <token>` header on HTTP requests — per-request auth, no OAuth needed
- `GOOGLE_ACCESS_TOKEN` env var — checked once at first authentication, works with both stdio and HTTP
- Both bypass the OAuth credential file + token refresh flow entirely

### 3. Refactored Handler Registration
- Extracted `registerHandlers(server)` function so MCP request handlers can be registered on both the global stdio server and per-session HTTP servers
- No changes to auth module, tool modules, types, or any other files

## Test Evidence

### HTTP Transport + `GOOGLE_ACCESS_TOKEN` env var
```
$ GOOGLE_ACCESS_TOKEN=ya29.a0ATko... node dist/index.js --transport http --port 3464

# Initialize session
$ curl -s -D - -X POST http://localhost:3464/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize",...}'

HTTP/1.1 200 OK
mcp-session-id: 92330889-af5f-407c-adc2-4268ef2491d9
content-type: text/event-stream

event: message
data: {"result":{"protocolVersion":"2025-03-26","capabilities":{"resources":{},"tools":{}},"serverInfo":{"name":"google-drive-mcp","version":"1.7.4"}},"jsonrpc":"2.0","id":1}

# Search Drive (real API call, returned actual files)
$ curl -s -X POST http://localhost:3464/mcp \
  -H "mcp-session-id: 92330889-af5f-407c-adc2-4268ef2491d9" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search","arguments":{"query":"test","pageSize":2}}}'

event: message
data: {"result":{"content":[{"type":"text","text":"Found 2 files:\nagent tests (application/vnd.google-apps.spreadsheet) [id: 1oyO...]\ntest-gsheet-fivetran (application/vnd.google-apps.folder) [id: 1Axz...]\n\nMore results available..."}],"isError":false},"jsonrpc":"2.0","id":2}
```

### HTTP Transport + `Authorization: Bearer` header (separate session)
```
$ curl -s -X POST http://localhost:3464/mcp \
  -H "Authorization: Bearer ya29.a0ATko..." \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize",...}'
# -> 200 OK, new session created

$ curl -s -X POST http://localhost:3464/mcp \
  -H "mcp-session-id: <new-session>" \
  -H "Authorization: Bearer ya29.a0ATko..." \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"search","arguments":{"query":"test","pageSize":2}}}'

event: message
data: {"result":{"content":[{"type":"text","text":"Found 2 files:\nagent tests...\ntest-gsheet-fivetran..."}],"isError":false},"jsonrpc":"2.0","id":3}
```

### Stdio Transport (backward compatibility)
```
$ echo '{"jsonrpc":"2.0","id":1,"method":"initialize",...}' | GOOGLE_ACCESS_TOKEN=test node dist/index.js

{"result":{"protocolVersion":"2025-03-26","capabilities":{"resources":{},"tools":{}},"serverInfo":{"name":"google-drive-mcp","version":"1.7.4"}},"jsonrpc":"2.0","id":1}
```

### Typecheck + Build
```
$ npm run build
> tsc --noEmit    # clean
Build complete!
```

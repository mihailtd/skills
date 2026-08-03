---
name: python-fastmcp-security-discovery
description: Guides teams to implement security controls (API key authentication, sliding-window rate limiting, request/response audit logging) and dynamic service discovery / capability registries for MCP servers.
---

# Python FastMCP: Security & Discovery

This skill helps AI implement security controls and service discovery interfaces for Model Context Protocol systems. As MCP enables AI systems to execute actions and access sensitive enterprise data, servers require robust authentication, authorization, rate limiting, and audit logging — alongside registry capabilities to dynamically advertise and discover server capabilities across large ecosystems.

---

## When to use this skill

Use this skill when you need to:

- protect an MCP server with authentication middleware (e.g. API keys or secret tokens),
- enforce sliding-window rate limits to prevent resource exhaustion or rogue agent loops,
- build structured request/response audit logging for security compliance and monitoring,
- implement principle of least privilege and defense-in-depth across MCP tools and resources,
- build or integrate an MCP server discovery registry to dynamically advertise server capabilities and tags.

---

## Outcome

Produce an MCP server security wrapper or discovery registry that:

- authenticates incoming requests before invoking tool handlers or reading resources,
- tracks request rates per client ID and rejects requests exceeding sliding window limits,
- records audit log entries for every request and response, including timestamps, method names, and payload metrics,
- maintains a registry of available MCP servers and allows capability/tag matching for dynamic service discovery.

## Instructions for the AI

1. **Implement Multi-Layered Security Architecture**
   - Wrap the MCP server's request dispatcher to intercept every request before execution.
   - Enforce core security layers:
     - **Explicit User Consent & Approval Guardrails:** Any tool execution (arbitrary code execution path) or resource data transmission MUST enforce explicit user consent. Tool calls with real-world side effects (financial, destructive, or external data sharing) require explicit approval before invocation.
     - **OAuth 2.1 & RFC 8707 Resource Indicators:** For Streamable HTTP transports, use OAuth 2.1 bearer tokens. Validate that the token's `aud` (audience) claim matches the server's canonical URI (`resource` parameter per RFC 8707) to reject tokens replayed from other servers.
     - **Capability & Attribute-Based Authorization (ABAC):** Evaluate fine-grained user identity, environmental conditions, and action attributes rather than relying solely on static RBAC.
     - **Intent-Based Authorization & Policy Engine:** Evaluate whether an incoming tool request aligns with declared agent intent and workflow stage using real-time policy-as-code evaluation before executing side effects.
     - **Data Lineage & Reasoning Step Audit Logging:** Audit logs must record not just raw RPC inputs and outputs, but correlation IDs, agent reasoning steps, intent metadata, and data provenance to track how sensitive data is combined and transformed across servers.
     - **Data Minimization & Automatic Expiration:** Resource handlers must return the minimum data necessary for the intent and enforce automatic data expiration.
     - **Sliding-Window Rate Limiting:** Enforce maximum requests per minute per client using a sliding window to block rogue agent loops.

2. **Implement Security Middleware Pattern**
   - Example implementation pattern for an MCP server security wrapper:
     ```python
     from datetime import datetime
     import time
     from typing import Any, Dict, List, Optional
     from mcp.server import Server

     class SecureMCPServer:
         """An MCP server wrapper with authentication, rate limiting, and audit logging."""

         def __init__(self, name: str, api_key: str, max_rpm: int = 100):
             self.server = Server(name)
             self.api_key = api_key
             self.max_rpm = max_rpm
             self.rate_limits: Dict[str, List[float]] = {}
             self.audit_log: List[Dict[str, Any]] = []
             self._setup_security()

         def _setup_security(self) -> None:
             """Wrap the internal handle request method with security middleware."""
             original_handle_request = self.server._handle_request

             async def secure_handle_request(request: Dict[str, Any]) -> Dict[str, Any]:
                 # 1. Authenticate
                 if not await self._authenticate_request(request):
                     raise ValueError("Authentication failed: invalid or missing API key")

                 # 2. Check rate limit
                 if not await self._check_rate_limit(request):
                     raise ValueError("Rate limit exceeded")

                 # 3. Log request
                 await self._log_request(request)

                 # 4. Process request
                 response = await original_handle_request(request)

                 # 5. Log response
                 await self._log_response(request, response)
                 return response

             self.server._handle_request = secure_handle_request

         async def _authenticate_request(self, request: Dict[str, Any]) -> bool:
             auth_header = request.get("auth")
             return auth_header == self.api_key

         async def _check_rate_limit(self, request: Dict[str, Any]) -> bool:
             client_id = request.get("client_id", "unknown")
             now = time.time()
             if client_id not in self.rate_limits:
                 self.rate_limits[client_id] = []

             # Retain timestamps from the last 60 seconds
             self.rate_limits[client_id] = [
                 t for t in self.rate_limits[client_id] if now - t < 60
             ]
             if len(self.rate_limits[client_id]) >= self.max_rpm:
                 return False

             self.rate_limits[client_id].append(now)
             return True

         async def _log_request(self, request: Dict[str, Any]) -> None:
             self.audit_log.append({
                 "timestamp": datetime.now().isoformat(),
                 "type": "request",
                 "method": request.get("method"),
                 "client_id": request.get("client_id", "unknown"),
                 "request_id": request.get("id"),
             })

         async def _log_response(self, request: Dict[str, Any], response: Dict[str, Any]) -> None:
             self.audit_log.append({
                 "timestamp": datetime.now().isoformat(),
                 "type": "response",
                 "request_id": request.get("id"),
                 "success": "error" not in response,
                 "response_size": len(str(response)),
             })

         def get_audit_log(self) -> List[Dict[str, Any]]:
             return self.audit_log.copy()
     ```

3. **Implement Discovery & Capability Registry Services**
   - In larger architectures, servers register their endpoint, capabilities, and tags with a central registry service ("yellow pages").
   - Expose registration and discovery tools: `register_server`, `discover_servers`, and `get_server_info`.
   - Filter servers by matching all required `capabilities` and at least one required `tag`.
   - Example pattern:
     ```python
     from datetime import datetime
     import json
     from typing import Any, Dict
     from mcp.server.fastmcp import FastMCP

     mcp = FastMCP("mcp-discovery-service")
     REGISTERED_SERVERS: Dict[str, Dict[str, Any]] = {}

     @mcp.tool()
     def register_server(
         name: str,
         description: str,
         endpoint: str,
         capabilities: list[str],
         tags: list[str] = [],
     ) -> str:
         """Register an MCP server with capability metadata."""
         REGISTERED_SERVERS[name] = {
             "name": name,
             "description": description,
             "endpoint": endpoint,
             "capabilities": capabilities,
             "tags": tags,
             "registered_at": datetime.now().isoformat(),
         }
         return f"Server '{name}' registered successfully."

     @mcp.tool()
     def discover_servers(
         capabilities: list[str] = [],
         tags: list[str] = [],
     ) -> str:
         """Discover servers matching capabilities or tags."""
         matching = []
         for server in REGISTERED_SERVERS.values():
             if capabilities and not all(c in server["capabilities"] for c in capabilities):
                 continue
             if tags and not any(t in server["tags"] for t in tags):
                 continue
             matching.append(server)
         return json.dumps({"matches": len(matching), "servers": matching}, indent=2)
     ```

---

## Decision points and guidance

- **Authentication Transport?** Pass authentication credentials via transport connection metadata or request authorization headers, never in prompt text.
- **Rate Limit Window?** Choose sliding window duration and max request counts based on server capability and downstream API quota restrictions.
- **Security Auditing?** Audit logs must be copyable (`audit_log.copy()`) or sent directly to persistent security logging pipelines (e.g. cloud logging / Redis streams) to prevent tampering.

---

## Quality criteria

- **Zero Unauthenticated Access:** Requests without valid credentials are rejected immediately before handler invocation.
- **Resilient Rate Control:** Rate limiting correctly evicts expired timestamps and blocks requests exceeding limits.
- **Complete Audit Trail:** Both request metadata and response status/size are recorded for every invocation.
- **Precise Discovery:** Discovery queries accurately evaluate required capabilities and tags.

---

## Example prompts

- "Add API key authentication and rate limiting middleware to our FastMCP server."
- "Implement an audit logging system for all incoming MCP tool calls and responses."
- "Create a discovery service that registers MCP servers and lets clients search by capability tags."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-client-integration
- python-fastmcp-resource-providers

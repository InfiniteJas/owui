# Rate-Limiting Proxy for OpenWebUI with Keycloak Group Integration

This document outlines the design for an intermediate rate-limiting layer (proxy) to protect OpenWebUI's Large Language Model (LLM) API endpoints. The rate limits will be based on user group memberships extracted from Keycloak JWT tokens.

## 1. Architectural Overview

The core idea is to place an API Gateway or a Reverse Proxy in front of the OpenWebUI application. This proxy will intercept all incoming traffic, specifically targeting requests to OpenWebUI's LLM API endpoints.

**Flow:**

1.  User makes a request to an OpenWebUI LLM endpoint (e.g., sending a message in a chat).
2.  The request hits the Rate-Limiting Proxy first.
3.  The Proxy extracts the JWT Bearer token from the `Authorization` header.
4.  It validates the JWT token against Keycloak (signature, issuer, audience, expiry).
5.  If the token is valid, the proxy extracts the user's group memberships (e.g., `epir_test`, `epir_prod`, `unlimited_access`) from the `groups` claim in the token.
6.  Based on the user's group(s), the proxy checks if the request exceeds the predefined rate limits for that group within a specific time window.
7.  **If rate-limited:** The proxy returns a `429 Too Many Requests` HTTP error.
8.  **If not rate-limited:** The proxy forwards the request to the OpenWebUI backend.
9.  OpenWebUI processes the request and sends the response back through the proxy to the user.

**Target API Paths:**
The specific OpenWebUI API paths that interact with LLMs need to be identified and configured in the proxy. These might include paths like:
*   `/api/chat`
*   `/api/generate`
*   `/v1/chat/completions` (if OpenWebUI exposes an OpenAI-compatible endpoint that should be rate-limited)
For this design, we'll refer to a generic path as `/api/llm_request_path`.

## 2. Technology Suggestion

While several technologies can implement this, a common and powerful solution is **Nginx with the `lua-nginx-module`**. This is often packaged as OpenResty, which bundles Nginx with Lua and many other useful modules.

**Why Nginx + Lua?**
*   **Performance:** Nginx is renowned for its high performance and low resource consumption.
*   **Flexibility:** Lua scripting within Nginx allows for complex custom logic directly at the request processing phase.
*   **Ecosystem:** Libraries like `lua-resty-openidc` for OIDC token validation and `lua-resty-redis` for Redis communication are readily available.

**Alternatives:**
*   **Kong:** A popular API gateway built on top of Nginx and Lua, offering many features out-of-the-box, including OIDC validation and rate limiting. Might be an overkill if only simple rate-limiting is needed but good for future expansion.
*   **Tyk:** Another open-source API gateway with similar capabilities.
*   **Custom Microservice:** A dedicated microservice written in Python (Flask/FastAPI), Node.js (Express), or Go. This service would handle token validation and rate limiting, then proxy valid requests. This offers maximum flexibility but requires more development and operational effort.
*   **Cloud-Native API Gateways:** Solutions like AWS API Gateway, Google Cloud API Gateway, or Azure API Management often have built-in JWT validation and rate-limiting features.

For illustration, the following sections will focus on an Nginx + Lua example.

## 3. Core Logic for the Proxy (Nginx + Lua Example)

This section describes the core logic components if using Nginx with Lua.

### 3.1. Token Validation

1.  **Extract Token:** Retrieve the JWT from the `Authorization: Bearer <token>` header.
2.  **Fetch JWKS:** Get Keycloak's public keys from its JWKS (JSON Web Key Set) URI: `http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>/protocol/openid-connect/certs`.
    *   These keys should be cached by the proxy for a configurable duration (e.g., few hours) to avoid excessive requests to Keycloak.
3.  **Validate Signature:** Verify the token's signature using the appropriate public key from the JWKS.
4.  **Validate Claims:**
    *   **Issuer (`iss`):** Must match `http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>`.
    *   **Audience (`aud`):** Should include or be exactly `openwebui-client` (or your configured client ID).
    *   **Expiry (`exp`):** Ensure the token is not expired.
    *   The `lua-resty-openidc` library can handle most of this automatically.

### 3.2. Group Extraction

*   If the token is successfully validated, parse the JWT payload.
*   Extract the `groups` claim. This claim will contain an array of group names the user belongs to (e.g., `["epir_test", "another_group"]`). This was configured via the "Group Membership" mapper in Keycloak.

### 3.3. Rate Limiting

1.  **Redis Connection:** Establish a connection to a Redis instance.
2.  **Define Limits:** Configure rate limits per group. For example:
    *   `epir_test`: 20 requests / 1 hour
    *   `epir_prod`: 100 requests / 1 hour
    *   `unlimited_access`: No limit (or a very high limit)
3.  **Request Processing:**
    *   For each incoming request to a protected LLM endpoint:
        *   Identify the relevant group(s) for the user (e.g., if a user is in `epir_test` and `another_group`, you might apply the strictest limit or sum them, or have a priority system. For simplicity, let's assume we check the first relevant group found with a defined limit).
        *   If the user belongs to the `unlimited_access` group, bypass rate limiting.
        *   For a user in a monitored group (e.g., `epir_test`):
            *   **Determine Current Window:** Calculate the start timestamp of the current window (e.g., for an hourly window, this is the start of the current hour).
            *   **Construct Redis Key:** Create a unique key for the group and the current window. Example: `rate_limit:epir_test:<YYYYMMDDHH>`.
                *   *Note on Granularity:* For per-user-in-group limits, the key could be `rate_limit:<group_name>:<user_id>:<YYYYMMDDHH>`. For this example, we assume a simple group-level shared quota.
            *   **Increment and Check:**
                *   Use `INCR` to increment the counter for the key.
                *   If it's a new key (first request in the window for this group), Redis `INCR` will create it and set it to 1. Use `EXPIRE` to set its TTL (Time To Live) to the window duration (e.g., 3600 seconds for 1 hour). This should be done atomically or in a transaction if possible, or ensure `EXPIRE` is set on the first increment.
                *   Compare the incremented value against the group's defined limit (e.g., 20 for `epir_test`).
            *   **Action:**
                *   If counter > limit: Return `429 Too Many Requests`.
                *   If counter <= limit: Proceed to proxy the request.

### 3.4. Proxying

*   If the request passes the rate limit check (or is exempt), Nginx proxies the request to the OpenWebUI backend server (e.g., `http://openwebui-backend-host:8080`).

### 3.5. Conceptual Configuration Snippets (Nginx + Lua)

These are illustrative and not complete, production-ready configurations.

**`nginx.conf` (Conceptual Snippet):**

```nginx
# Ensure lua-nginx-module is loaded, often done in main nginx.conf or via OpenResty
# E.g., load_module /path/to/ngx_http_lua_module.so;

# Define where Keycloak's JWKS can be found and other OIDC settings
# lua-resty-openidc can use these if configured globally or per location
# lua_shared_dict oidc_config 1m; # For lua-resty-openidc discovery cache
# lua_shared_dict oidc_jwks 1m;   # For lua-resty-openidc JWKS cache

# Redis upstream (if using lua-resty-redis)
# upstream redis_backend {
#     server <redis_ip>:<redis_port>;
#     keepalive 100;
# }

http {
    # ... other http configurations ...

    # Lua script path
    # lua_package_path "/path/to/lua/scripts/?.lua;;";

    server {
        listen 80; # Or 443 for HTTPS
        server_name <your_proxy_domain>;

        # Location for OpenWebUI's LLM API endpoints
        location /api/llm_request_path { # Or more specific paths like /api/chat
            access_by_lua_block {
                -- This block will contain Lua code to:
                -- 1. Extract Authorization Bearer token
                -- 2. Validate token using lua-resty-openidc (configured with Keycloak details)
                --    - Keycloak URL: http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>
                --    - Client ID: openwebui-client
                -- 3. Extract 'groups' claim from the validated token
                -- 4. Connect to Redis (e.g., using lua-resty-redis)
                -- 5. Implement rate limiting logic based on groups and defined limits/windows
                --    - Read limits from a config file or hardcode for simplicity in example
                -- 6. If rate limited, return ngx.exit(ngx.HTTP_TOO_MANY_REQUESTS)
                -- 7. If not, allow request to proceed to proxy_pass

                require("rate_limiter").handle_request()
            }

            # If access_by_lua_block doesn't exit, request is proxied
            proxy_pass http://<openwebui_internal_host>:<openwebui_internal_port>; # e.g., http://localhost:8080
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            # Potentially pass validated user info to OpenWebUI if needed
            # proxy_set_header X-Authenticated-User $jwt_subject; # Subject from JWT
            # proxy_set_header X-Authenticated-Groups $jwt_groups; # Groups from JWT
        }

        # Other locations for OpenWebUI static assets, UI, etc.
        location / {
            proxy_pass http://<openwebui_internal_host>:<openwebui_internal_port>;
            proxy_set_header Host $host;
            # ... other proxy headers ...
        }
    }
}
```

**Conceptual Lua Script (`rate_limiter.lua`):**

```lua
-- rate_limiter.lua
local redis = require "resty.redis"
local cjson = require "cjson"
-- For OIDC validation, you'd typically use a library like lua-resty-openidc
-- local oidc = require "resty.openidc"

local _M = {}

-- Configuration for rate limits (could be loaded from a file or env vars)
local group_limits = {
    epir_test = { limit = 20, window = 3600 }, -- 20 requests per hour
    epir_prod = { limit = 100, window = 3600 }, -- 100 requests per hour
    -- 'unlimited_access' group members would bypass this logic
}

function _M.get_jwt_claims()
    -- Placeholder for JWT extraction and validation logic
    -- In a real scenario, use lua-resty-openidc:
    -- local res, err = oidc.authenticate({
    --     redirect_uri_path = ngx.var.request_uri, -- Or specific callback
    --     discovery = "http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>/.well-known/openid-configuration",
    --     client_id = "openwebui-client",
    --     -- Other oidc opts: scope, use_pkce, ssl_verify etc.
    -- })
    -- if not res then
    --     ngx.log(ngx.ERR, "Failed to authenticate: ", err)
    --     return nil, "Auth Error"
    -- end
    -- return res.id_token, nil -- Return the claims from the ID token

    -- Simplified mock for demonstration:
    local auth_header = ngx.req.get_headers()["Authorization"]
    if not auth_header or not string.match(auth_header, "^Bearer ") then
        ngx.log(ngx.WARN, "No Bearer token found")
        return nil, "Missing Token"
    end
    local token_str = string.sub(auth_header, 8) -- Crude extraction

    -- !! In a real implementation, ALWAYS validate the token signature and claims !!
    -- This is a MOCK decoding for structure illustration ONLY
    -- local jwt_decode = require "resty.jwt" -- Example library
    -- local claims, err = jwt_decode:verify(token_str, secret_or_jwks)
    -- if err then return nil, "Invalid Token" end

    -- Mocked claims for this example
    if token_str == "valid_token_epir_test" then
        return {
            sub = "user123",
            groups = {"epir_test"},
            iss = "http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>",
            aud = "openwebui-client"
            -- ... other claims
        }, nil
    elseif token_str == "valid_token_unlimited" then
         return { sub = "user456", groups = {"unlimited_access"} }, nil
    else
        return nil, "Invalid Token Mock"
    end
end

function _M.handle_request()
    local claims, err = _M.get_jwt_claims()

    if not claims then
        ngx.log(ngx.ERR, "JWT validation failed: ", err)
        ngx.exit(ngx.HTTP_UNAUTHORIZED)
        return
    end

    local user_groups = claims.groups
    if not user_groups or type(user_groups) ~= "table" then
        ngx.log(ngx.WARN, "No groups claim in token for user: ", claims.sub)
        ngx.exit(ngx.HTTP_FORBIDDEN) -- Or allow with a default very strict limit
        return
    end

    local applicable_limit_config = nil
    local user_is_unlimited = false

    for _, group_name in ipairs(user_groups) do
        if group_name == "unlimited_access" then
            user_is_unlimited = true
            break -- Unlimited access overrides other group limits
        end
        if group_limits[group_name] then
            -- Simple: take the first configured limit found.
            -- More complex logic could pick the most restrictive or a specific one.
            if not applicable_limit_config then
                 applicable_limit_config = group_limits[group_name]
                 -- Potentially store the group_name for the Redis key
            end
        end
    end

    if user_is_unlimited then
        ngx.log(ngx.INFO, "User ", claims.sub, " has unlimited access.")
        return -- Allow request
    end

    if not applicable_limit_config then
        ngx.log(ngx.INFO, "User ", claims.sub, " in groups ", cjson.encode(user_groups), " has no specific rate limit defined. Allowing by default.")
        -- Or, alternatively, deny:
        -- ngx.log(ngx.WARN, "User ", claims.sub, " has no applicable rate limit defined. Denying.")
        -- ngx.exit(ngx.HTTP_FORBIDDEN)
        return -- Allow request if no specific limit is found and not unlimited
    end

    -- Connect to Redis
    local red, err_redis = redis:new()
    if not red then
        ngx.log(ngx.ERR, "Failed to instantiate redis: ", err_redis)
        ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
        return
    end

    -- TODO: Set actual Redis host and port
    local ok, err_connect = red:connect("<redis_ip>", <redis_port>)
    if not ok then
        ngx.log(ngx.ERR, "Failed to connect to redis: ", err_connect)
        ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
        return
    end
    -- Use red:set_keepalive() for connection pooling in production

    local current_time = ngx.time()
    -- For hourly window, floor to the hour
    local window_start_timestamp = math.floor(current_time / applicable_limit_config.window) * applicable_limit_config.window
    -- The Redis key should represent the group being limited and the specific time window
    -- For simplicity, using the first applicable group name found.
    local first_limited_group_name = ""
    for _, group_name in ipairs(user_groups) do
        if group_limits[group_name] then
            first_limited_group_name = group_name
            break
        end
    end

    local redis_key = "rate_limit:" .. first_limited_group_name .. ":" .. window_start_timestamp

    local count, err_incr = red:incr(redis_key)
    if err_incr then
        ngx.log(ngx.ERR, "Redis INCR failed: ", err_incr)
        red:close()
        ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
        return
    end

    if count == 1 then -- First request in this window for this group
        local _, err_expire = red:expire(redis_key, applicable_limit_config.window)
        if err_expire then
            ngx.log(ngx.ERR, "Redis EXPIRE failed for key ", redis_key, ": ", err_expire)
            -- Request still proceeds, but key won't auto-expire if this fails
        end
    end

    red:close() -- Close connection if not using keepalive

    if count > applicable_limit_config.limit then
        ngx.log(ngx.WARN, "Rate limit exceeded for group '", first_limited_group_name, "' (user ", claims.sub, "). Count: ", count)
        -- Optional: Add Retry-After header
        -- ngx.header["Retry-After"] = applicable_limit_config.window - (current_time - window_start_timestamp)
        ngx.exit(ngx.HTTP_TOO_MANY_REQUESTS)
        return
    end

    ngx.log(ngx.INFO, "Rate limit check passed for group '", first_limited_group_name, "' (user ", claims.sub, "). Count: ", count)
    -- Request proceeds
end

return _M
```

## 4. Considerations

*   **Identifying Target API Paths:** Accurately identify all OpenWebUI backend API endpoints that involve LLM processing and should be subject to rate limits. These paths must be correctly configured in the proxy's location blocks.
*   **Redis Setup:** A stable and accessible Redis instance is required for the rate-limiting counters. Ensure appropriate network configuration and Redis persistence if needed (though rate limit counters are often acceptable to be transient).
*   **Security:**
    *   The proxy itself becomes a critical security component. It must be secured and kept up-to-date.
    *   Communication between the proxy and the OpenWebUI backend should occur over a trusted network (e.g., Docker internal network, private subnet). If not, HTTPS should be used for this internal leg.
    *   The JWT validation (especially signature verification against Keycloak's JWKS) is paramount.
*   **Flexibility & Complexity:**
    *   The Nginx/Lua approach is highly flexible and can be extended for more sophisticated rules (e.g., different limits for different API paths, more complex group logic, dynamic limit configuration).
    *   Using a library like `lua-resty-openidc` is highly recommended for robust OIDC token validation, as handling all edge cases of JWTs and OIDC manually is complex and error-prone.
*   **Error Handling:** Implement comprehensive error handling in the Lua script (e.g., Redis connection failures, unexpected token structures).
*   **Configuration Management:** Rate limit values (requests, window) and group definitions should ideally be managed via configuration files or environment variables rather than being hardcoded in scripts.
*   **Performance:** While Nginx and Lua are fast, poorly written Lua scripts or too many synchronous operations can impact performance. Optimize Redis interactions (e.g., use connection pooling with `set_keepalive`).
*   **Deployment:** This proxy would be deployed as a separate container or service in your infrastructure, routing traffic to the OpenWebUI container(s).

This design provides a solid foundation for implementing group-based rate limiting for OpenWebUI using Keycloak for authentication.

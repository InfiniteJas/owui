# OpenWebUI with Keycloak Authentication and Group-Based Rate Limiting: A Comprehensive Integration Guide

## Introduction

This guide provides comprehensive instructions for setting up OpenWebUI with secure authentication via Keycloak (OpenID Connect) and implementing group-based rate limiting for its API endpoints using an intermediate proxy. This solution enhances security, user management, and resource control for your OpenWebUI deployment.

---

## Part 1: Deploying OpenWebUI

This section covers the initial deployment of OpenWebUI using Docker.

### 1.1 Prerequisites

Ensure that Docker is installed and running on your system. You can find installation instructions for your operating system on the [official Docker website](https://docs.docker.com/get-docker/).

### 1.2 Using `docker run`

This command will download the latest OpenWebUI image and run it in a detached container.

```bash
docker run -d -p <host_port>:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

**Explanation of the command:**

*   `-d`: Runs the container in detached mode (in the background).
*   `-p <host_port>:8080`: Maps port `<host_port>` on your host machine to port 8080 in the container.
    *   You can change `<host_port>` to any available port on your system (e.g., `80`, `8080`, `3000`).
    *   **Important:** If you change `<host_port>` from the default `3000`, you will also need to update the `Valid Redirect URIs` in your Keycloak client configuration and the OpenWebUI public URL (e.g., `OIDC_ISSUER_URL` or similar environment variables if you are configuring OpenWebUI directly) to reflect this new port.
*   `--add-host=host.docker.internal:host-gateway`: Adds a host entry for `host.docker.internal`, which allows the OpenWebUI container to connect to services running on your host machine (like Ollama). This is crucial if Ollama is running directly on your Docker host and not in a container.
*   `-v open-webui:/app/backend/data`: Creates and mounts a Docker volume named `open-webui` to `/app/backend/data` inside the container. This volume is used to persist OpenWebUI's data, such as user information, chat history, and settings, even if the container is removed or updated.
*   `--name open-webui`: Assigns a name to the container for easy reference (e.g., when stopping or removing it).
*   `--restart always`: Configures the container to automatically restart if it stops or if the Docker daemon restarts. This helps ensure OpenWebUI remains available.
*   `ghcr.io/open-webui/open-webui:main`: Specifies the Docker image to use. This pulls the `main` tag (often the latest stable version) from the GitHub Container Registry.

### 1.3 Using `docker-compose.yml`

Docker Compose allows you to define and manage multi-container Docker applications. Create a file named `docker-compose.yml` with the following content:

```yaml
version: '3.8'

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "<host_port>:8080" # Change <host_port> if needed (e.g., "3000:8080", "80:8080")
    volumes:
      - open-webui:/app/backend/data
    # This allows OpenWebUI to connect to Ollama or other services running on the host machine.
    # It maps host.docker.internal to the host's IP address within the container.
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always

volumes:
  open-webui:
    # This defines the named volume 'open-webui' for persistent data storage.
    # Docker will manage this volume.
```

**To use Docker Compose:**

1.  Save the content above into a file named `docker-compose.yml` in a directory of your choice.
2.  Open a terminal, navigate to that directory.
3.  Run the command: `docker-compose up -d`

**Explanation of the `docker-compose.yml` file:**

*   `version: '3.8'`: Specifies the version of the Docker Compose file format.
*   `services:`: Defines the different services (containers) that make up your application.
    *   `open-webui:`: Defines the OpenWebUI service.
        *   `image: ghcr.io/open-webui/open-webui:main`: Specifies the Docker image to use.
        *   `container_name: open-webui`: Sets the name of the container.
        *   `ports:`: Maps ports between the host and the container.
            *   `"<host_port>:8080"`: Maps `<host_port>` on the host to port 8080 in the container. Remember to replace `<host_port>` (e.g., `3000`).
            *   **Important:** If you change `<host_port>` from the default `3000`, you will also need to update the `Valid Redirect URIs` in your Keycloak client configuration and the OpenWebUI public URL.
        *   `volumes:`: Mounts volumes for persistent data.
            *   `open-webui:/app/backend/data`: Mounts the named volume `open-webui` to `/app/backend/data` in the container.
        *   `extra_hosts:`: Adds custom host mappings.
            *   `"host.docker.internal:host-gateway"`: Allows connection to services on the host machine.
        *   `restart: always`: Ensures the container restarts automatically.
*   `volumes:`: Defines named volumes.
    *   `open-webui:`: Declares the `open-webui` volume, which Docker will manage for data persistence.

By default, OpenWebUI will be accessible at `http://localhost:<host_port>` after running either of these commands. Remember to replace `<host_port>` with the actual port number you've chosen (e.g., `http://localhost:3000`).

---

## Part 2: Configuring Keycloak for OpenWebUI

This guide details the steps to configure Keycloak as an OpenID Connect (OIDC) provider for OpenWebUI. This setup enables centralized user authentication and management for your OpenWebUI instance.

**Assumptions:**
*   You have a running Keycloak instance accessible at `http://<keycloak_ip>:<keycloak_port>` (e.g., `http://localhost:8080` if running Keycloak locally).
*   You have administrative access to your Keycloak instance.
*   OpenWebUI will be accessible at `http://<your_openwebui_ip_or_domain>:<port>`.

### 2.1 Step 1: Create a New Realm

A realm in Keycloak is a way to isolate tenants. It manages a set of users, credentials, roles, and groups.

1.  Open your Keycloak administration console (e.g., `http://<keycloak_ip>:<keycloak_port>/auth/admin/`).
2.  If you are in the `master` realm, hover over `master` in the top-left corner and click **Add realm**.
3.  **Realm Name:** Enter a name for your realm. For example, `OpenWebUI_Realm`. You can choose any name, but be consistent.
4.  Ensure **Enabled** is ON.
5.  Click **Create**.
6.  You will be automatically switched to the newly created realm. You can always switch between realms using the dropdown in the top-left corner.

### 2.2 Step 2: Create a New Client for OpenWebUI

A client in Keycloak is an entity that can request authentication of a user. In this case, OpenWebUI is the client.

1.  In your `OpenWebUI_Realm` (or your chosen realm name), navigate to **Clients** from the left-hand menu.
2.  Click **Create** on the top right.
3.  **Client ID:** Enter a unique ID for your client. For example, `openwebui-client`. This ID will be used in OpenWebUI's configuration.
4.  **Client Protocol:** Select `openid-connect`. This is the standard protocol OpenWebUI uses.
5.  **Root URL:** You can leave this blank or set it to your OpenWebUI base URL (e.g., `http://<your_openwebui_ip_or_domain>:<port>`).
6.  Click **Save**. This will take you to the client's detailed configuration page.

7.  **Configure Client Settings:**
    *   **Access Type:** Set this to `confidential`. This means the client (OpenWebUI) needs to provide a secret to exchange the authorization code for a token. This is more secure for server-side applications like OpenWebUI.
    *   **Standard Flow Enabled:** Ensure this is ON (usually default).
    *   **Direct Access Grants Enabled:** Ensure this is ON (usually default).
    *   **Valid Redirect URIs:** This is a critical setting. It's a list of URLs where Keycloak is allowed to redirect the user after successful authentication.
        *   The `Valid Redirect URIs` (e.g., `http://<your_openwebui_host_or_ip>:<port>/auth/callback` or `http://<your_openwebui_host_or_ip>:<port>/oauth/oidc/callback`) must exactly match what OpenWebUI uses.
        *   When using `OLLAMA_AUTH_PROVIDER=keycloak` (as described in Part 3), this URI might be specific (e.g., often ending in `/auth/callback` or similar, but this can vary).
        *   **If unsure of the exact URI, you can often determine the correct one by attempting an initial login from OpenWebUI after configuring it for Keycloak. When OpenWebUI redirects you to the Keycloak login page, inspect the `redirect_uri` query parameter in your browser's address bar. This URL is what Keycloak expects. Add this discovered URI to the list of Valid Redirect URIs in Keycloak.**
        *   You might also need to add your OpenWebUI base URL (e.g., `http://<your_openwebui_ip_or_domain>:<port>/*`) for features like logout.
        *   **Important:** Replace `<your_openwebui_ip_or_domain>` with the actual IP address or domain name where OpenWebUI is accessible, and `<port>` with the port OpenWebUI is running on (e.g., `3000` or `80`).
    *   **Web Origins:** You might need to add `+` to allow all origins or specify `http://<your_openwebui_ip_or_domain>:<port>`. This helps with CORS.

8.  Click **Save** at the bottom of the page.

9.  **Retrieve Client Secret:**
    *   Navigate to the **Credentials** tab for the `openwebui-client`.
    *   You will see a field labeled **Secret**. Copy this value.
    *   **This secret is crucial and will be used in OpenWebUI's environment variable configuration (e.g., as `OLLAMA_AUTH_KEYCLOAK_CLIENT_SECRET` as per Part 3).** Keep it secure.

### 2.3 Step 3: Configure Mappers for Group Information

To allow OpenWebUI (and potentially a rate-limiting proxy) to make decisions based on user groups, you need to ensure group membership information is included in the OIDC tokens.

1.  Still within the `openwebui-client` configuration, navigate to the **Mappers** tab.
2.  Click **Create**.
3.  **Name:** Give the mapper a descriptive name, e.g., `groups-mapper` or simply `groups`.
4.  **Mapper Type:** Select `Group Membership` from the dropdown.
5.  **Token Claim Name:** Enter `groups`. This is the name of the claim that will appear in the JWT token containing the user's group memberships. The rate-limiting proxy will use this claim. OpenWebUI might also use it if configured (e.g., via `OLLAMA_AUTH_KEYCLOAK_GROUP_CLAIM`).
6.  **Full group path:** Set this to **OFF**. Typically, only the group name is needed, not its entire path in the group hierarchy.
7.  **Add to ID token:** Set to **ON**.
8.  **Add to access token:** Set to **ON**. (While OpenWebUI primarily uses the ID token for this, including it in the access token is good practice if other services consume it).
9.  **Add to userinfo:** Set to **ON**.
10. Click **Save**.

This mapper ensures that when a user logs in, their group memberships (e.g., `epir_test`, `unlimited_access`) are embedded within the JWT token. OpenWebUI can then parse this token to identify the user's groups.

### 2.4 Step 4: Create User Groups

Define the groups that users can be assigned to. These groups can be used by OpenWebUI for role-based access control or by the rate-limiting proxy to apply different policies.

1.  In your `OpenWebUI_Realm`, navigate to **Groups** from the left-hand menu.
2.  Click **New** (or **Create group** depending on Keycloak version).
3.  **Name:** Enter the desired group name (e.g., `epir_test`). Click **Save** (or **Create**).
4.  Repeat for other groups, for example:
    *   `epir_prod`
    *   `unlimited_access`
    *   Any other groups relevant to your access control policies.

### 2.5 Step 5: Assign Users to Groups

Once groups are created, you can assign users to them.

1.  In your `OpenWebUI_Realm`, navigate to **Users** from the left-hand menu.
2.  Select the user you want to assign to a group.
3.  Go to the **Groups** tab for that user.
4.  In the **Available Groups** section, select the group(s) you want to assign the user to (e.g., `epir_test`).
5.  Click **Join**. The group will move to the **Assigned Groups** (or similar section).

Users can be members of multiple groups.

### 2.6 Summary of Key Information for OpenWebUI Configuration

You will need the following pieces of information from this Keycloak setup to configure OpenWebUI:

*   **OIDC Issuer URL:** `http://<keycloak_ip>:<keycloak_port>/auth/realms/OpenWebUI_Realm` (or your realm name)
*   **OIDC Client ID:** `openwebui-client` (or your client ID)
*   **OIDC Client Secret:** The secret you copied from the client's 'Credentials' tab.
*   **OIDC Group Claim Name (if not default):** `groups` (if you changed the `Token Claim Name` in the mapper).

This completes the Keycloak configuration for OpenWebUI. Remember to replace placeholders like `<keycloak_ip>`, `<your_openwebui_ip_or_domain>`, and `<port>` with your actual values.

---

## Part 3: Configuring OpenWebUI for Keycloak Integration

To integrate OpenWebUI with your Keycloak setup, you need to configure several environment variables in your OpenWebUI Docker deployment. These variables tell OpenWebUI how to communicate with Keycloak for authentication, assuming OpenWebUI uses a Keycloak-specific set of environment variables when `OLLAMA_AUTH_PROVIDER=keycloak` is set.

Refer to your `docker run` command or `docker-compose.yml` file to set these variables.

**Important Note on Redirect URI:**
When using `OLLAMA_AUTH_PROVIDER=keycloak`, the exact redirect URI that OpenWebUI will use needs to be configured in your Keycloak client settings under 'Valid Redirect URIs'. This is often `http://<your_openwebui_host_or_ip>:<port>/auth/callback` or similar (the exact path `/auth/callback` may vary depending on OpenWebUI's implementation for this provider). Please verify this by checking the initial redirection URL from OpenWebUI to Keycloak during the first login attempt if the documentation for this auth provider isn't explicit about the callback path. Update the "Valid Redirect URIs" in Keycloak (Part 2, Step 2.2) accordingly.

### 3.1 Required Keycloak Environment Variables for OpenWebUI

Here's a list of essential environment variables based on the `OLLAMA_AUTH_PROVIDER=keycloak` pattern:

*   **`OLLAMA_AUTH_PROVIDER=keycloak`**
    *   **Explanation:** Specifies Keycloak as the authentication provider for OpenWebUI.

*   **`OLLAMA_AUTH_KEYCLOAK_URL=http://<keycloak_host_or_ip>:<keycloak_port>/realms/<your_realm_name>`**
    *   **Explanation:** The full URL to your Keycloak realm. This is often referred to as the "Issuer URL".
    *   **Example:** `http://localhost:8080/realms/OpenWebUI_Realm` (if Keycloak is on localhost, port 8080, and realm is `OpenWebUI_Realm`).
    *   **Placeholders:** Replace `<keycloak_host_or_ip>`, `<keycloak_port>`, and `<your_realm_name>` with your actual values.

*   **`OLLAMA_AUTH_KEYCLOAK_CLIENT_ID=<openwebui-client_id_from_keycloak>`**
    *   **Explanation:** The Client ID of the client you created in Keycloak for OpenWebUI.
    *   **Example:** `openwebui-client` (if you used this name during Keycloak client setup).
    *   **Placeholder:** Replace `<openwebui-client_id_from_keycloak>` with your actual Client ID.

*   **`OLLAMA_AUTH_KEYCLOAK_CLIENT_SECRET=<your_client_secret_from_keycloak>`**
    *   **Explanation:** The client secret obtained from the 'Credentials' tab of your `openwebui-client` in Keycloak.
    *   **Placeholder:** Replace `<your_client_secret_from_keycloak>` with your actual client secret.

*   **(Optional) `OLLAMA_AUTH_KEYCLOAK_GROUP_CLAIM=groups`**
    *   **Explanation:** If OpenWebUI has specific features to utilize Keycloak group claims directly (beyond what the external proxy handles for rate limiting), specify the name of the group claim in the JWT token (e.g., `groups`). This must match the 'Token Claim Name' of your group mapper in Keycloak (configured in Part 2, Step 2.3). For the primary goal of rate limiting via an external proxy, this claim will be read by the proxy.
    *   **Default:** If not set, OpenWebUI might not use group claims directly, or it might have a default value.

*   **User Signup and Login Form Behavior Note:**
    *   The variables `ENABLE_OAUTH_SIGNUP=True` and `ENABLE_LOGIN_FORM=False` were relevant for the generic OIDC configuration. When `OLLAMA_AUTH_PROVIDER=keycloak` is used, OpenWebUI's behavior regarding user registration and the visibility of its native login form might change.
    *   **Verification Needed:** It's crucial to test whether new users from Keycloak can be automatically provisioned in OpenWebUI and whether the local login form is appropriately hidden. If OpenWebUI's documentation for the `keycloak` provider specifies different variables for these controls (e.g., a general `ENABLE_SIGNUP`), use those. If users are expected to be pre-provisioned in OpenWebUI, then `ENABLE_SIGNUP` (or its equivalent) should be `False`. The goal is typically to have Keycloak as the sole point of entry.

*   **`WEBUI_SECRET_KEY=<your_strong_random_secret_key>`**
    *   **Explanation:** An essential security key used by OpenWebUI for signing session cookies and other security-related functions. It should be a long, random, and unpredictable string. This is independent of Keycloak but vital for OpenWebUI's overall security.
    *   **Recommendation:** Generate a strong random string (e.g., using a password manager or `openssl rand -hex 32`).
    *   **Default (Docker):** Randomly generated on first start. **Important:** For consistent behavior across restarts or in multi-container setups, set this explicitly.

### 3.2 Verifying the Keycloak Connection

After configuring these environment variables and restarting your OpenWebUI container:

1.  **Restart OpenWebUI:**
    *   If using `docker run`: `docker stop open-webui && docker rm open-webui && <your_full_docker_run_command_with_new_env_vars>`
    *   If using `docker-compose`: `docker-compose down && docker-compose up -d` (ensure your `.env` file or `docker-compose.yml` reflects the new variables).

2.  **Access OpenWebUI:** Open your OpenWebUI URL (e.g., `http://<openwebui_ip_or_domain>:<openwebui_port>`) in a web browser.

3.  **Redirection to Keycloak:** You should be automatically redirected to the Keycloak login page. Verify if OpenWebUI's native login form is hidden as expected (behavior might depend on how `OLLAMA_AUTH_PROVIDER=keycloak` interacts with form visibility).

4.  **Keycloak Login:** Log in with a user account that exists in your Keycloak realm (and is assigned to relevant groups if testing group functionality).

5.  **Redirection back to OpenWebUI:** After successful authentication with Keycloak, you should be redirected back to the OpenWebUI interface, and you should be logged in as the Keycloak user.

6.  **Troubleshooting (Optional):**
    *   If the login fails or you encounter issues, check the OpenWebUI container logs for any auth-related error messages: `docker logs open-webui`.
    *   Double-check that the `OLLAMA_AUTH_KEYCLOAK_URL` is correct and accessible from the OpenWebUI container.
    *   Verify that the Client ID and Secret are correctly copied.
    *   Ensure the "Valid Redirect URIs" in Keycloak correctly matches the URI OpenWebUI is attempting to redirect to after Keycloak authentication (see note at the beginning of Part 3).

By correctly setting these environment variables, you can effectively delegate OpenWebUI's authentication to Keycloak.

---

## Part 4: Implementing Request Limiting with an Intermediate Proxy

This section outlines the design for an intermediate rate-limiting layer (proxy) to protect OpenWebUI's Large Language Model (LLM) API endpoints. The rate limits will be based on user group memberships extracted from Keycloak JWT tokens.

### 4.1. Architectural Overview

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

### 4.2. Technology Suggestion

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

### 4.3. Core Logic for the Proxy (Nginx + Lua Example)

This section describes the core logic components if using Nginx with Lua.

#### 4.3.1. Token Validation

1.  **Extract Token:** Retrieve the JWT from the `Authorization: Bearer <token>` header.
2.  **Fetch JWKS:** Get Keycloak's public keys from its JWKS (JSON Web Key Set) URI: `http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>/protocol/openid-connect/certs`.
    *   These keys should be cached by the proxy for a configurable duration (e.g., few hours) to avoid excessive requests to Keycloak.
3.  **Validate Signature:** Verify the token's signature using the appropriate public key from the JWKS.
4.  **Validate Claims:**
    *   **Issuer (`iss`):** Must match `http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>`.
    *   **Audience (`aud`):** Should include or be exactly `openwebui-client` (or your configured client ID).
    *   **Expiry (`exp`):** Ensure the token is not expired.
    *   The `lua-resty-openidc` library can handle most of this automatically.

#### 4.3.2. Group Extraction

*   If the token is successfully validated, parse the JWT payload.
*   Extract the `groups` claim. This claim will contain an array of group names the user belongs to (e.g., `["epir_test", "another_group"]`). This was configured via the "Group Membership" mapper in Keycloak.

#### 4.3.3. Rate Limiting

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

#### 4.3.4. Proxying

*   If the request passes the rate limit check (or is exempt), Nginx proxies the request to the OpenWebUI backend server (e.g., `http://openwebui-backend-host:8080`).

#### 4.3.5. Conceptual Configuration Snippets (Nginx + Lua)

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

### 4.4. Considerations

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

---

## Part 5: Testing the Setup

This section outlines the testing plan for the integrated OpenWebUI, Keycloak, and rate-limiting proxy solution. The goal is to ensure secure authentication, correct authorization, and effective rate limiting based on user groups.

### 5.1. Prerequisites for Testing

Before commencing testing, ensure the following components are deployed, configured, and operational:

1.  **OpenWebUI:**
    *   Deployed as a Docker container (or target deployment method).
    *   Environment variables for Keycloak OIDC integration are correctly set (as per Part 3).
2.  **Keycloak:**
    *   Running and accessible.
    *   A realm (e.g., `OpenWebUI_Realm`) is created and enabled.
    *   A client (e.g., `openwebui-client`) is configured with `openid-connect` protocol, `confidential` access type, and correct `Valid Redirect URIs`.
    *   The `groups` mapper is configured for the client to include group memberships in tokens.
    *   User groups are created: `epir_test`, `epir_prod`, `unlimited_access`.
3.  **Rate-Limiting Proxy:**
    *   Deployed in front of OpenWebUI (e.g., Nginx + Lua).
    *   Configured to validate JWTs from Keycloak (using the correct realm and client details).
    *   Connected to a running Redis instance.
    *   Rate limit rules for `epir_test`, `epir_prod` groups are defined (e.g., `epir_test`: 20 reqs/hour, `epir_prod`: 100 reqs/hour).
    *   Logic for handling the `unlimited_access` group (bypassing limits) is in place.
    *   The specific LLM API paths of OpenWebUI (e.g., `/api/chat`, `/api/generate`, or a generic `/api/llm_request_path`) are correctly targeted by the proxy for rate limiting.
4.  **Test Users (in Keycloak):**
    *   `user_test`: Member of the `epir_test` group.
    *   `user_prod`: Member of the `epir_prod` group.
    *   `user_unlimited`: Member of the `unlimited_access` group.
    *   `user_nogroup`: Not a member of `epir_test`, `epir_prod`, or `unlimited_access`. (This user might fall into a default policy, e.g., no access if not in a recognized group, or a very restrictive default limit, or unlimited if not specified otherwise).
    *   Ensure these users have passwords set.

### 5.2. Authentication Tests

These tests verify the OIDC integration between OpenWebUI and Keycloak.

*   **Test Case 5.2.1: Successful Login (`user_test`)**
    *   **Steps:**
        1.  Open a new incognito browser window.
        2.  Navigate to the OpenWebUI URL.
        3.  Attempt to log in using the credentials for `user_test`.
    *   **Expected Outcome:**
        *   User is redirected to the Keycloak login page.
        *   After entering valid credentials for `user_test`, login is successful.
        *   User is redirected back to OpenWebUI.
        *   User is successfully logged into OpenWebUI and the interface is accessible.
        *   The displayed username/email in OpenWebUI corresponds to `user_test`.

*   **Test Case 5.2.2: Successful Login (Other Groups)**
    *   **Steps:** Repeat Test Case 5.2.1 for `user_prod` and `user_unlimited`.
    *   **Expected Outcome:** Successful login and access to OpenWebUI for both users.

*   **Test Case 5.2.3: Failed Login (Invalid Credentials)**
    *   **Steps:**
        1.  Open a new incognito browser window.
        2.  Navigate to the OpenWebUI URL.
        3.  On the Keycloak login page, enter the username for `user_test` but an incorrect password.
    *   **Expected Outcome:**
        *   Keycloak displays an "Invalid username or password" (or similar) error message.
        *   User is not redirected back to OpenWebUI.
        *   User cannot access OpenWebUI.

*   **Test Case 5.2.4: Access Protected Page Without Login**
    *   **Steps:**
        1.  Open a new incognito browser window.
        2.  Attempt to navigate directly to a known protected OpenWebUI URL (e.g., the main chat interface or settings page if known, otherwise just the base URL).
    *   **Expected Outcome:** User is redirected to the Keycloak login page.

### 5.3. Authorization Tests

These tests verify basic access control based on user roles/groups, if OpenWebUI is configured to interpret roles beyond simple group membership for access.

*   **Test Case 5.3.1: Admin User Access (If Applicable)**
    *   **Prerequisite:** An admin role (e.g., `admin`) is defined in Keycloak, mapped from Keycloak to OpenWebUI (e.g., via `OAUTH_ADMIN_ROLES` and `ENABLE_OAUTH_ROLE_MANAGEMENT` in OpenWebUI), and a test user (e.g., `user_admin`) is assigned this role in Keycloak.
    *   **Steps:** Log in to OpenWebUI as `user_admin`.
    *   **Expected Outcome:** `user_admin` has access to administrative functionalities in OpenWebUI (e.g., user management, system settings if available).

*   **Test Case 5.3.2: Regular User Access**
    *   **Steps:** Log in to OpenWebUI as `user_test` (who is not an admin).
    *   **Expected Outcome:** `user_test` has standard user access, without administrative privileges. Admin sections should be hidden or inaccessible.

*(Note: If OpenWebUI's role management is primarily based on allowing access if a user is authenticated via Keycloak and is part of *any* recognized group, then these tests primarily confirm that authenticated users can access the application, while unauthenticated users cannot.)*

### 5.4. Rate Limiting Tests

These tests are crucial for verifying the functionality of the rate-limiting proxy.

*   **Test Case 5.4.1: Rate Limit for `epir_test` Group**
    *   **Setup:** Configured limit for `epir_test` (e.g., 20 requests / 1 hour).
    *   **Steps:**
        1.  Log in to OpenWebUI as `user_test`.
        2.  Identify the specific API endpoint used for LLM interactions (e.g., `/api/chat` or `/api/llm_request_path`) by using browser developer tools (Network tab) while sending a message, or refer to the proxy configuration.
        3.  Sequentially send requests to the LLM via the OpenWebUI interface (e.g., by sending chat messages) or by using a tool like `curl` or Postman to hit the proxied API endpoint directly with `user_test`'s JWT.
        4.  Send 20 requests.
        5.  Send the 21st request.
    *   **Expected Outcome:**
        *   The first 20 requests are successful (HTTP 200 OK or other success status from OpenWebUI).
        *   The 21st request (and subsequent ones within the hour) receives an `HTTP 429 Too Many Requests` error from the proxy.
        *   Check Redis: A counter key (e.g., `rate_limit:epir_test:<window_timestamp>`) should exist and its value should be >= 20.

*   **Test Case 5.4.2: Rate Limit for `epir_prod` Group**
    *   **Setup:** Configured limit for `epir_prod` (e.g., 100 requests / 1 hour).
    *   **Steps:** Repeat Test Case 5.4.1, logging in as `user_prod` and sending up to 101 requests.
    *   **Expected Outcome:**
        *   The first 100 requests are successful.
        *   The 101st request receives an `HTTP 429 Too Many Requests` error.
        *   Check Redis for the corresponding counter.

*   **Test Case 5.4.3: No Rate Limit for `unlimited_access` Group**
    *   **Setup:** `unlimited_access` group should bypass defined numerical limits.
    *   **Steps:**
        1.  Log in to OpenWebUI as `user_unlimited`.
        2.  Send more requests (e.g., 30 or 150) than the highest limit defined for other groups.
    *   **Expected Outcome:** All requests are successful. No `HTTP 429` errors are received. Redis should not have a counter for this group/user that leads to blocking.

*   **Test Case 5.4.4: Rate Limit for `user_nogroup` (Default Behavior)**
    *   **Steps:**
        1.  Log in to OpenWebUI as `user_nogroup`.
        2.  Send requests to the LLM endpoint.
    *   **Expected Outcome:**
        *   The behavior depends on the proxy's default policy:
            *   **Option A (Deny by Default):** If users must be in a specific group to get access, `user_nogroup` might be denied by the proxy (e.g., HTTP 403 Forbidden) even before hitting rate limits.
            *   **Option B (Default Limited Pool):** `user_nogroup` might be subject to a very restrictive default rate limit.
            *   **Option C (Default Unlimited/Less Restricted):** `user_nogroup` might fall into the same category as `unlimited_access` if not explicitly restricted.
        *   This test clarifies and verifies the intended default behavior. For this plan, assume the default behavior is documented by the proxy implementer (e.g., if not in `epir_test` or `epir_prod`, the user is treated as `unlimited_access` or subject to a different default limit). Test against this documented default.

*   **Test Case 5.4.5: Rate Limit Window Reset**
    *   **Steps:**
        1.  Using `user_test`, execute Test Case 5.4.1 to hit the rate limit (receive `HTTP 429`).
        2.  Note the time.
        3.  Wait for the configured time window to expire (e.g., 1 hour from the time the first request in the limited window was made).
        4.  After the window expires, attempt to send a new request to the LLM endpoint as `user_test`.
    *   **Expected Outcome:**
        *   The new request is successful (HTTP 200 OK).
        *   The counter in Redis for the previous window has expired, or a new counter for the new window has started at 1.

### 5.5. Logging and Monitoring

Throughout all testing phases, active monitoring of system logs is essential.

*   **Components to Monitor:**
    *   **OpenWebUI Container:** `docker logs <openwebui_container_name_or_id>`
    *   **Keycloak Server:** Access Keycloak's server logs (location depends on Keycloak deployment). Look for login successes, failures, token issuance.
    *   **Rate-Limiting Proxy (Nginx/Lua):** Check Nginx's `access.log` and `error.log`. Lua scripts should log relevant information (token validation results, group extraction, Redis interactions, rate-limiting decisions).
    *   **Redis:**
        *   Use `redis-cli MONITOR` (with caution in shared/dev environments, as it streams all commands) to observe SET, INCR, EXPIRE commands.
        *   Periodically check keys using `redis-cli KEYS "rate_limit:*"`, and `redis-cli GET <key>` or `redis-cli TTL <key>` to inspect values and expiry times.

*   **Expected Logging Behavior:**
    *   Successful operations should be logged appropriately (e.g., successful token validation, request proxying).
    *   Errors (e.g., failed token validation, Redis connection issues, rate limit exceeded) should be clearly logged with sufficient detail to diagnose the issue.
    *   Proxy logs should indicate when a request is blocked due to rate limits.

This testing plan provides a structured approach to verifying the core functionalities of the integrated system. Each test case should be documented with its actual outcome, and any deviations from the expected outcome should be investigated.

---

## Part 6: Additional Considerations

This section addresses further questions related to debugging, security, error handling, and scalability of the integrated OpenWebUI, Keycloak, and rate-limiting proxy solution.

### 6.1 Какие логи нужно проверять для отладки? (Which logs need to be checked for debugging?)

Effective debugging requires checking logs from all involved components:

*   **OpenWebUI:**
    *   **Container Logs:** Access via `docker logs <openwebui_container_name_or_id>`. These logs are crucial for diagnosing OIDC integration problems (e.g., misconfigured OIDC variables), application startup issues, and general backend errors within OpenWebUI.
*   **Keycloak:**
    *   **Server Logs:** The location depends on your Keycloak deployment method (e.g., standalone, Docker, Kubernetes). Typically accessible through the Keycloak Admin Console (Server Info -> Logs) or directly on the server's filesystem. These logs will show details about user authentication attempts, authorization decisions, token issuance, and any errors within Keycloak itself.
*   **Rate-Limiting Proxy (e.g., Nginx + Lua):**
    *   **Nginx Access Logs (`access.log`):** Shows all incoming requests, status codes, and basic request information. Useful for seeing if requests are reaching the proxy and how they are being handled (e.g., proxied or rejected).
    *   **Nginx Error Logs (`error.log`):** Contains detailed error messages from Nginx and potentially from Lua scripts. Essential for debugging issues with proxy configuration, Lua script execution errors (e.g., problems connecting to Redis or Keycloak's JWKS endpoint), and SSL/TLS problems.
    *   **Custom Lua Logging:** Lua scripts should include their own logging (e.g., using `ngx.log(ngx.ERR, "message")`) to trace token validation steps, group extraction, and rate-limiting decisions.
*   **Redis:**
    *   **Redis Server Logs:** Useful if Redis itself is experiencing issues (e.g., persistence problems, memory limits, connection errors). Location depends on Redis setup.
    *   **`MONITOR` command (via `redis-cli`):** Streams all commands processed by the Redis server. This can be very helpful for seeing exactly what operations the proxy is performing on Redis keys (e.g., `INCR`, `EXPIRE`). **Use with caution in production environments as it impacts performance.**

### 6.2 Какие best practices по безопасности следует учитывать при интеграции Keycloak? (Which security best practices should be considered for Keycloak integration?)

Integrating Keycloak and securing the overall solution involves multiple layers:

*   **HTTPS Everywhere:** Enforce HTTPS (TLS) for all communications:
    *   User to Proxy.
    *   Proxy to OpenWebUI.
    *   Proxy to Keycloak.
    *   Proxy to Redis (if on an untrusted network and Redis supports it).
*   **Strong Secrets & Keys:**
    *   Protect Keycloak's client secrets diligently.
    *   Use a strong, unique `WEBUI_SECRET_KEY` for OpenWebUI and ensure it's managed securely.
    *   Secure any API keys or credentials used by the proxy or Keycloak.
*   **Realm Security (Keycloak):**
    *   Implement strong password policies.
    *   Enable brute force detection and account lockout mechanisms.
    *   Configure appropriate session timeouts and token lifespans (access token, refresh token).
    *   Restrict direct access to the Keycloak admin console.
*   **Meticulous Token Validation:**
    *   The rate-limiting proxy (or any service consuming JWTs) must rigorously validate tokens: signature (against JWKS), issuer (`iss`), audience (`aud`), and expiry (`exp`). Use well-tested OIDC/JWT libraries.
*   **Principle of Least Privilege:**
    *   In Keycloak, assign users only the minimum necessary roles and group memberships.
    *   The Keycloak client for OpenWebUI should have only necessary protocol mappers and scopes enabled.
*   **Regular Software Updates:** Keep all components updated to their latest stable versions to patch known vulnerabilities:
    *   OpenWebUI
    *   Keycloak
    *   Proxy software (Nginx, Lua modules)
    *   Redis
    *   Operating System and other underlying infrastructure.
*   **Secure Proxy Configuration:**
    *   Harden the reverse proxy configuration (e.g., disable unused HTTP methods, set security headers like HSTS, CSP, X-Content-Type-Options).
    *   Prevent information leakage (e.g., server version tokens).
*   **Firewall Rules & Network Segmentation:**
    *   Restrict network access to Keycloak, Redis, and the internal OpenWebUI port. Only the proxy should be publicly accessible (or accessible to the intended users).
    *   Place components in appropriate network segments.
*   **CSRF and XSS Protection:** While Keycloak and OpenWebUI should have their own protections, ensure proxy configurations do not inadvertently create new vulnerabilities.

### 6.3 Как обрабатывать ошибки аутентификации/авторизации? (How to handle authentication/authorization errors?)

Error handling should be clear to the user where appropriate and provide enough information for debugging:

*   **Keycloak:**
    *   Handles its own login error displays directly on its login pages (e.g., "Invalid username or password").
*   **OpenWebUI:**
    *   If an unauthenticated user attempts to access a protected resource, it should gracefully redirect to Keycloak for login.
    *   Errors during the OIDC callback process (e.g., state mismatch, token validation failure by OpenWebUI itself) might result in a generic error message displayed in the OpenWebUI interface. Detailed errors would be found in OpenWebUI container logs.
*   **Rate-Limiting Proxy:** This is where specific HTTP error codes are crucial:
    *   **Invalid/Expired/Malformed Token:** Return `HTTP 401 Unauthorized`. The client should then attempt to re-authenticate (e.g., redirect to Keycloak).
    *   **Token Valid but Insufficient Permissions (e.g., user not in an allowed group, if proxy enforces this):** Return `HTTP 403 Forbidden`.
    *   **Rate Limit Exceeded:** Return `HTTP 429 Too Many Requests`. Optionally include a `Retry-After` header indicating when the client can try again.
    *   **Internal Proxy Errors (e.g., cannot connect to Redis/Keycloak JWKS):** Return `HTTP 500 Internal Server Error` or `HTTP 503 Service Unavailable`.
*   **User Experience:**
    *   For user-facing errors (like those from Keycloak or a redirect from OpenWebUI), the experience should be as user-friendly as possible.
    *   For API errors returned by the proxy, they are typically consumed by the client application (OpenWebUI's frontend). The frontend should ideally handle these gracefully (e.g., display a notification "Too many requests, please try again later").

### 6.4 Как масштабировать решение? (How to scale the solution?)

Scalability needs to address each component of the architecture:

*   **OpenWebUI:**
    *   **Horizontal Scaling:** Run multiple instances of the OpenWebUI Docker container. These would typically sit behind the rate-limiting proxy, which also acts as a load balancer.
    *   **Session Management:** If OpenWebUI instances maintain state, a shared session store might be needed. OpenWebUI documentation mentions `REDIS_URL` for app-state and `WEBSOCKET_REDIS_URL` for websocket communication, which are essential for multi-node/worker clusters. Ensure `WEBUI_SECRET_KEY` is identical across all instances for session cookie consistency.
*   **Keycloak:**
    *   Keycloak is designed for scalability and supports various deployment modes:
        *   **Standalone Mode:** For smaller deployments.
        *   **Clustered Mode:** Provides high availability and load distribution by running multiple Keycloak nodes in a cluster. Requires a shared database and often an external load balancer.
        *   **Cross-Site Replication:** For geographically distributed deployments.
*   **Rate-Limiting Proxy (e.g., Nginx):**
    *   **Vertical Scaling:** Increase resources (CPU/memory) on the proxy server. Nginx is very efficient.
    *   **Horizontal Scaling:** Run multiple instances of the proxy, with a load balancer (e.g., HAProxy, AWS ELB, Nginx itself) distributing traffic among them. This requires careful consideration of how shared state (like OIDC discovery caches) is handled if not using a common mechanism like Redis for that too.
*   **Redis:**
    *   **Vertical Scaling:** Increase resources on the Redis server.
    *   **Redis Sentinel:** Provides high availability for Redis. A master-replica setup with Sentinels managing failover. Rate-limiting data would still be on a single master at a time.
    *   **Redis Cluster:** For very high throughput and data sharding. Distributes data across multiple Redis nodes. This provides scalability for both reads and writes. The Lua script connecting to Redis would need to be cluster-aware (some client libraries handle this).

By addressing these aspects, the entire solution can be scaled to handle increased load and provide high availability.

---

## Conclusion

This guide has provided a step-by-step approach to integrating OpenWebUI with Keycloak for robust authentication and implementing a group-based rate-limiting proxy. By following these instructions, administrators can enhance the security, manageability, and operational control of their OpenWebUI deployment. Thorough testing, as outlined, is crucial to ensure all components function correctly together. Remember to adapt configurations to your specific environment and security requirements.

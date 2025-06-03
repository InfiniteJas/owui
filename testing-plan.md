# Testing Plan: OpenWebUI, Keycloak, and Rate-Limiting Proxy

This document outlines the testing plan for the integrated OpenWebUI, Keycloak, and rate-limiting proxy solution. The goal is to ensure secure authentication, correct authorization, and effective rate limiting based on user groups.

## 1. Prerequisites for Testing

Before commencing testing, ensure the following components are deployed, configured, and operational:

1.  **OpenWebUI:**
    *   Deployed as a Docker container (or target deployment method).
    *   Environment variables for Keycloak OIDC integration are correctly set (as per `openwebui-oidc-env-variables.md`).
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

## 2. Authentication Tests

These tests verify the OIDC integration between OpenWebUI and Keycloak.

*   **Test Case 1.1: Successful Login (`user_test`)**
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

*   **Test Case 1.2: Successful Login (Other Groups)**
    *   **Steps:** Repeat Test Case 1.1 for `user_prod` and `user_unlimited`.
    *   **Expected Outcome:** Successful login and access to OpenWebUI for both users.

*   **Test Case 1.3: Failed Login (Invalid Credentials)**
    *   **Steps:**
        1.  Open a new incognito browser window.
        2.  Navigate to the OpenWebUI URL.
        3.  On the Keycloak login page, enter the username for `user_test` but an incorrect password.
    *   **Expected Outcome:**
        *   Keycloak displays an "Invalid username or password" (or similar) error message.
        *   User is not redirected back to OpenWebUI.
        *   User cannot access OpenWebUI.

*   **Test Case 1.4: Access Protected Page Without Login**
    *   **Steps:**
        1.  Open a new incognito browser window.
        2.  Attempt to navigate directly to a known protected OpenWebUI URL (e.g., the main chat interface or settings page if known, otherwise just the base URL).
    *   **Expected Outcome:** User is redirected to the Keycloak login page.

## 3. Authorization Tests

These tests verify basic access control based on user roles/groups, if OpenWebUI is configured to interpret roles beyond simple group membership for access.

*   **Test Case 2.1: Admin User Access (If Applicable)**
    *   **Prerequisite:** An admin role (e.g., `admin`) is defined in Keycloak, mapped from Keycloak to OpenWebUI (e.g., via `OAUTH_ADMIN_ROLES` and `ENABLE_OAUTH_ROLE_MANAGEMENT` in OpenWebUI), and a test user (e.g., `user_admin`) is assigned this role in Keycloak.
    *   **Steps:** Log in to OpenWebUI as `user_admin`.
    *   **Expected Outcome:** `user_admin` has access to administrative functionalities in OpenWebUI (e.g., user management, system settings if available).

*   **Test Case 2.2: Regular User Access**
    *   **Steps:** Log in to OpenWebUI as `user_test` (who is not an admin).
    *   **Expected Outcome:** `user_test` has standard user access, without administrative privileges. Admin sections should be hidden or inaccessible.

*(Note: If OpenWebUI's role management is primarily based on allowing access if a user is authenticated via Keycloak and is part of *any* recognized group, then these tests primarily confirm that authenticated users can access the application, while unauthenticated users cannot.)*

## 4. Rate Limiting Tests

These tests are crucial for verifying the functionality of the rate-limiting proxy.

*   **Test Case 3.1: Rate Limit for `epir_test` Group**
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

*   **Test Case 3.2: Rate Limit for `epir_prod` Group**
    *   **Setup:** Configured limit for `epir_prod` (e.g., 100 requests / 1 hour).
    *   **Steps:** Repeat Test Case 3.1, logging in as `user_prod` and sending up to 101 requests.
    *   **Expected Outcome:**
        *   The first 100 requests are successful.
        *   The 101st request receives an `HTTP 429 Too Many Requests` error.
        *   Check Redis for the corresponding counter.

*   **Test Case 3.3: No Rate Limit for `unlimited_access` Group**
    *   **Setup:** `unlimited_access` group should bypass defined numerical limits.
    *   **Steps:**
        1.  Log in to OpenWebUI as `user_unlimited`.
        2.  Send more requests (e.g., 30 or 150) than the highest limit defined for other groups.
    *   **Expected Outcome:** All requests are successful. No `HTTP 429` errors are received. Redis should not have a counter for this group/user that leads to blocking.

*   **Test Case 3.4: Rate Limit for `user_nogroup` (Default Behavior)**
    *   **Steps:**
        1.  Log in to OpenWebUI as `user_nogroup`.
        2.  Send requests to the LLM endpoint.
    *   **Expected Outcome:**
        *   The behavior depends on the proxy's default policy:
            *   **Option A (Deny by Default):** If users must be in a specific group to get access, `user_nogroup` might be denied by the proxy (e.g., HTTP 403 Forbidden) even before hitting rate limits.
            *   **Option B (Default Limited Pool):** `user_nogroup` might be subject to a very restrictive default rate limit.
            *   **Option C (Default Unlimited/Less Restricted):** `user_nogroup` might fall into the same category as `unlimited_access` if not explicitly restricted.
        *   This test clarifies and verifies the intended default behavior. For this plan, assume the default behavior is documented by the proxy implementer (e.g., if not in `epir_test` or `epir_prod`, the user is treated as `unlimited_access` or subject to a different default limit). Test against this documented default.

*   **Test Case 3.5: Rate Limit Window Reset**
    *   **Steps:**
        1.  Using `user_test`, execute Test Case 3.1 to hit the rate limit (receive `HTTP 429`).
        2.  Note the time.
        3.  Wait for the configured time window to expire (e.g., 1 hour from the time the first request in the limited window was made).
        4.  After the window expires, attempt to send a new request to the LLM endpoint as `user_test`.
    *   **Expected Outcome:**
        *   The new request is successful (HTTP 200 OK).
        *   The counter in Redis for the previous window has expired, or a new counter for the new window has started at 1.

## 5. Logging and Monitoring

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

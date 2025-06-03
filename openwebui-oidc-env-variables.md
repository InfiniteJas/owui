# OpenWebUI Environment Variables for Keycloak OIDC Integration

To integrate OpenWebUI with your Keycloak OIDC setup, you need to configure several environment variables in your OpenWebUI Docker deployment. These variables tell OpenWebUI how to communicate with Keycloak for authentication.

Refer to your `docker run` command or `docker-compose.yml` file to set these variables.

## Required OIDC Environment Variables

Here's a list of essential environment variables and their explanations:

*   **`ENABLE_OAUTH_SIGNUP=True`**
    *   **Explanation:** Enables user account creation through the OIDC provider (Keycloak). If a new user logs in via Keycloak and doesn't have an account in OpenWebUI, one will be created for them.
    *   **Default:** `False`

*   **`ENABLE_LOGIN_FORM=False`**
    *   **Explanation:** Disables OpenWebUI's built-in email/password login form. This is crucial when OIDC is intended to be the sole authentication method.
    *   **Warning:** As per OpenWebUI documentation, `ENABLE_LOGIN_FORM` must be set to `False` when `ENABLE_OAUTH_SIGNUP` is `True`. Failure to do so will result in the inability to log in.
    *   **Default:** `True`

*   **`OAUTH_CLIENT_ID=<your_openwebui_client_id_from_keycloak>`**
    *   **Explanation:** The Client ID of the client you created in Keycloak for OpenWebUI.
    *   **Example:** `openwebui-client` (if you used the example name from the Keycloak setup guide).
    *   **Placeholder:** Replace `<your_openwebui_client_id_from_keycloak>` with your actual Client ID.

*   **`OAUTH_CLIENT_SECRET=<your_client_secret_from_keycloak>`**
    *   **Explanation:** The client secret obtained from the 'Credentials' tab of your `openwebui-client` in Keycloak.
    *   **Placeholder:** Replace `<your_client_secret_from_keycloak>` with your actual client secret.

*   **`OPENID_PROVIDER_URL=http://<keycloak_ip>:<keycloak_port>/realms/<your_realm_name>`**
    *   **Explanation:** The full URL to your Keycloak realm's OpenID configuration. This is often referred to as the "Issuer URL". OpenWebUI uses this to discover OIDC endpoints.
    *   **Example:** `http://localhost:8180/realms/OpenWebUI_Realm` (if Keycloak is on localhost, port 8180, and realm is `OpenWebUI_Realm`).
    *   **Placeholders:**
        *   `<keycloak_ip>`: IP address or domain of your Keycloak server.
        *   `<keycloak_port>`: Port your Keycloak server is running on.
        *   `<your_realm_name>`: The name of the realm you created in Keycloak (e.g., `OpenWebUI_Realm`).

*   **`OPENID_REDIRECT_URI=http://<openwebui_ip_or_domain>:<openwebui_port>/oauth/oidc/callback`**
    *   **Explanation:** The exact URL where Keycloak should redirect the user after successful authentication. This **must** match one of the "Valid Redirect URIs" configured in your Keycloak client.
    *   **Example:** `http://localhost:3000/oauth/oidc/callback` (if OpenWebUI is accessible at `localhost:3000`).
    *   **Placeholders:**
        *   `<openwebui_ip_or_domain>`: The IP address or domain name where your OpenWebUI instance is accessible.
        *   `<openwebui_port>`: The port on which your OpenWebUI instance is running.
    *   **Default (from docs):** `<backend>/oauth/oidc/callback` (OpenWebUI replaces `<backend>` internally, ensure your external URL matches this structure).

*   **`OAUTH_SCOPES="openid email profile groups"`**
    *   **Explanation:** Specifies the scopes OpenWebUI should request from Keycloak.
        *   `openid`: Standard scope for OIDC.
        *   `email`: To get the user's email address.
        *   `profile`: To get user profile information like name.
        *   `groups`: Essential for fetching group membership information, which will be used by the rate-limiting proxy and potentially OpenWebUI itself. This must align with the "Token Claim Name" of your 'groups' mapper in Keycloak.
    *   **Default (from docs):** `openid email profile` (you **must** add `groups`).

*   **`OAUTH_USERNAME_CLAIM=preferred_username`**
    *   **Explanation:** The OIDC token claim that Keycloak uses for the username. `preferred_username` is a standard claim often containing the login name.
    *   **Default (from docs):** `name`. For Keycloak, `preferred_username` is typically more appropriate for the unique login username. `name` might be the full display name. Adjust if your Keycloak token provides username differently.

*   **`OAUTH_EMAIL_CLAIM=email`**
    *   **Explanation:** The OIDC token claim for the user's email address. `email` is standard.
    *   **Default:** `email`

*   **`OAUTH_GROUP_CLAIM=groups`**
    *   **Explanation:** The name of the claim in the OIDC token that contains the user's group memberships. This **must** match the "Token Claim Name" you configured for the `Group Membership` mapper in Keycloak (e.g., `groups`).
    *   **Default:** `groups`

*   **`ENABLE_OAUTH_GROUP_MANAGEMENT=True`**
    *   **Explanation:** This variable suggests that OpenWebUI can utilize group information from the OIDC token. While our primary group-based logic (rate limiting) will be handled by an external proxy, enabling this might offer additional group-related features within OpenWebUI itself or simplify user role mapping if OpenWebUI supports it directly based on OIDC groups.
    *   **Default (from docs):** `False`

*   **`WEBUI_SECRET_KEY=<your_strong_random_secret_key>`**
    *   **Explanation:** An essential security key used by OpenWebUI for signing session cookies and other security-related functions. It should be a long, random, and unpredictable string.
    *   **Recommendation:** Generate a strong random string (e.g., using a password manager or `openssl rand -hex 32`).
    *   **Default (Docker):** Randomly generated on first start. **Important:** For consistent behavior across restarts or in multi-container setups, set this explicitly.

## Optional OIDC Environment Variables

*   **`OAUTH_PROVIDER_NAME="Login with Keycloak"`**
    *   **Explanation:** Customizes the text displayed on the OIDC login button on the OpenWebUI login page.
    *   **Default:** `SSO`
    *   **Example:** You could set it to "Login with Company SSO" or "Keycloak".

*   **`OAUTH_PICTURE_CLAIM=picture`**
    *   **Explanation:** The OIDC token claim for the user's profile picture URL. `picture` is a standard claim.
    *   **Default:** `picture`

*   **`OAUTH_UPDATE_PICTURE_ON_LOGIN=True`**
    *   **Explanation:** If enabled, OpenWebUI will update the user's profile picture with the one provided by Keycloak upon each login.
    *   **Default:** `False`

## Verifying the Keycloak Connection

After configuring these environment variables and restarting your OpenWebUI container:

1.  **Restart OpenWebUI:**
    *   If using `docker run`: `docker stop open-webui && docker rm open-webui && <your_full_docker_run_command_with_new_env_vars>`
    *   If using `docker-compose`: `docker-compose down && docker-compose up -d` (ensure your `.env` file or `docker-compose.yml` reflects the new variables).

2.  **Access OpenWebUI:** Open your OpenWebUI URL (e.g., `http://<openwebui_ip_or_domain>:<openwebui_port>`) in a web browser.

3.  **Redirection to Keycloak:** You should be automatically redirected to the Keycloak login page. The `ENABLE_LOGIN_FORM=False` setting ensures the standard OpenWebUI login form is hidden.

4.  **Keycloak Login:** Log in with a user account that exists in your Keycloak realm (and is assigned to relevant groups if testing group functionality).

5.  **Redirection back to OpenWebUI:** After successful authentication with Keycloak, you should be redirected back to the OpenWebUI interface, and you should be logged in as the Keycloak user.

6.  **Troubleshooting (Optional):**
    *   If the login fails or you encounter issues, check the OpenWebUI container logs for any OIDC-related error messages: `docker logs open-webui`
    *   Double-check that all URLs (`OPENID_PROVIDER_URL`, `OPENID_REDIRECT_URI`) are correct and accessible from the OpenWebUI container and your browser, respectively.
    *   Verify that the Client ID and Secret are correctly copied.
    *   Ensure the "Valid Redirect URIs" in Keycloak exactly matches the `OPENID_REDIRECT_URI` value.

By correctly setting these environment variables, you can effectively delegate OpenWebUI's authentication to Keycloak.

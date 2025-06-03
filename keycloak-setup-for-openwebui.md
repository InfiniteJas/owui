# Configuring Keycloak for OpenWebUI Authentication

This guide details the steps to configure Keycloak as an OpenID Connect (OIDC) provider for OpenWebUI. This setup enables centralized user authentication and management for your OpenWebUI instance.

Assumptions:
*   You have a running Keycloak instance accessible at `http://<keycloak_ip>:<keycloak_port>` (e.g., `http://localhost:8080` if running Keycloak locally).
*   You have administrative access to your Keycloak instance.
*   OpenWebUI will be accessible at `http://<your_openwebui_ip_or_domain>:<port>`.

## Step 1: Create a New Realm

A realm in Keycloak is a way to isolate tenants. It manages a set of users, credentials, roles, and groups.

1.  Open your Keycloak administration console (e.g., `http://<keycloak_ip>:<keycloak_port>/auth/admin/`).
2.  If you are in the `master` realm, hover over `master` in the top-left corner and click **Add realm**.
3.  **Realm Name:** Enter a name for your realm. For example, `OpenWebUI_Realm`. You can choose any name, but be consistent.
4.  Ensure **Enabled** is ON.
5.  Click **Create**.
6.  You will be automatically switched to the newly created realm. You can always switch between realms using the dropdown in the top-left corner.

## Step 2: Create a New Client for OpenWebUI

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
        *   Add `http://<your_openwebui_ip_or_domain>:<port>/oauth/oidc/callback`
        *   You might also need to add `http://<your_openwebui_ip_or_domain>:<port>/*` or `http://<your_openwebui_ip_or_domain>:<port>` for logout or if you encounter redirect issues. Start with the callback URI.
        *   **Important:** Replace `<your_openwebui_ip_or_domain>` with the actual IP address or domain name where OpenWebUI is accessible, and `<port>` with the port OpenWebUI is running on (e.g., `3000` or `80`).
    *   **Web Origins:** You might need to add `+` to allow all origins or specify `http://<your_openwebui_ip_or_domain>:<port>`. This helps with CORS.

8.  Click **Save** at the bottom of the page.

9.  **Retrieve Client Secret:**
    *   Navigate to the **Credentials** tab for the `openwebui-client`.
    *   You will see a field labeled **Secret**. Copy this value.
    *   **This secret is crucial and will be used as `OIDC_CLIENT_SECRET` in OpenWebUI's environment variable configuration.** Keep it secure.

## Step 3: Configure Mappers for Group Information

To allow OpenWebUI (and potentially a rate-limiting proxy) to make decisions based on user groups, you need to ensure group membership information is included in the OIDC tokens.

1.  Still within the `openwebui-client` configuration, navigate to the **Mappers** tab.
2.  Click **Create**.
3.  **Name:** Give the mapper a descriptive name, e.g., `groups-mapper` or simply `groups`.
4.  **Mapper Type:** Select `Group Membership` from the dropdown.
5.  **Token Claim Name:** Enter `groups`. This is the name of the claim that will appear in the JWT token containing the user's group memberships. OpenWebUI will look for this claim (configurable via `OAUTH_GROUP_CLAIM` in OpenWebUI, defaults to `groups`).
6.  **Full group path:** Set this to **OFF**. Typically, only the group name is needed, not its entire path in the group hierarchy.
7.  **Add to ID token:** Set to **ON**.
8.  **Add to access token:** Set to **ON**. (While OpenWebUI primarily uses the ID token for this, including it in the access token is good practice if other services consume it).
9.  **Add to userinfo:** Set to **ON**.
10. Click **Save**.

This mapper ensures that when a user logs in, their group memberships (e.g., `epir_test`, `unlimited_access`) are embedded within the JWT token. OpenWebUI can then parse this token to identify the user's groups.

## Step 4: Create User Groups

Define the groups that users can be assigned to. These groups can be used by OpenWebUI for role-based access control or by the rate-limiting proxy to apply different policies.

1.  In your `OpenWebUI_Realm`, navigate to **Groups** from the left-hand menu.
2.  Click **New** (or **Create group** depending on Keycloak version).
3.  **Name:** Enter the desired group name (e.g., `epir_test`). Click **Save** (or **Create**).
4.  Repeat for other groups, for example:
    *   `epir_prod`
    *   `unlimited_access`
    *   Any other groups relevant to your access control policies.

## Step 5: Assign Users to Groups

Once groups are created, you can assign users to them.

1.  In your `OpenWebUI_Realm`, navigate to **Users** from the left-hand menu.
2.  Select the user you want to assign to a group.
3.  Go to the **Groups** tab for that user.
4.  In the **Available Groups** section, select the group(s) you want to assign the user to (e.g., `epir_test`).
5.  Click **Join**. The group will move to the **Assigned Groups** (or similar section).

Users can be members of multiple groups.

## Summary of Key Information for OpenWebUI Configuration

You will need the following pieces of information from this Keycloak setup to configure OpenWebUI:

*   **OIDC Issuer URL:** `http://<keycloak_ip>:<keycloak_port>/auth/realms/OpenWebUI_Realm` (or your realm name)
*   **OIDC Client ID:** `openwebui-client` (or your client ID)
*   **OIDC Client Secret:** The secret you copied from the client's 'Credentials' tab.
*   **OIDC Group Claim Name (if not default):** `groups` (if you changed the `Token Claim Name` in the mapper).

This completes the Keycloak configuration for OpenWebUI. Remember to replace placeholders like `<keycloak_ip>`, `<your_openwebui_ip_or_domain>`, and `<port>` with your actual values.

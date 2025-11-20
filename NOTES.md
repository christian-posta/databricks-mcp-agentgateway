
Step-by-step guide for setting up OAuth token exchange with Databricks:

## Step-by-Step Guide: Setting Up OAuth Token Exchange with Databricks

### Part 1: Configure Federation Policy in Databricks

**Step 1.1: Access Databricks Account Console**
- Log in as an account administrator
- Navigate to Account Settings
- Find the Federation Policies section

**Step 1.2: Create a Federation Policy**
- Create a new federation policy that defines:
  - Trusted IdP (issuer)
  - Allowed audiences
  - User mapping rules (how IdP users map to Databricks users/service principals)
  - Token validation rules

**Step 1.3: Configure Policy Details**
- Specify:
  - IdP issuer URL (e.g., `https://login.microsoftonline.com/<tenant-id>/v2.0` for Azure AD)
  - Allowed audiences
  - Token signing algorithm (RS256 or ES256)
  - User mapping (email, subject, etc.)

### Part 2: Configure Your Identity Provider (IdP)

**Step 2.1: Ensure IdP Issues JWTs**
- Configure your IdP (Azure AD, Okta, etc.) to issue JWTs
- Tokens must be signed with RS256 or ES256
- Tokens should include:
  - `iss` (issuer)
  - `sub` (subject/user identifier)
  - `aud` (audience)
  - `exp` (expiration)

**Step 2.2: Configure Token Claims**
- Ensure tokens include claims that match your federation policy
- Common claims: email, name, groups

### Part 3: Obtain IdP Token (User Authentication)

**Step 3.1: User Authenticates with IdP**
- User logs in via your IdP (e.g., Azure AD, Okta)
- IdP issues a JWT token
- Store this token securely (environment variable, secure storage, etc.)

**Example: Getting Token from Azure AD**
```bash
# Using Azure CLI (example)
az account get-access-token --resource https://<databricks-workspace-host>
```

### Part 4: Exchange IdP Token for Databricks OAuth Token

**Step 4.1: Make Token Exchange Request**

Use the Databricks token exchange endpoint:

```bash
curl --request POST https://<databricks-workspace-host>/oidc/v1/token \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data "subject_token=${FEDERATED_JWT_TOKEN}" \
  --data "subject_token_type=urn:ietf:params:oauth:token-type:jwt" \
  --data "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  --data "scope=all-apis"
```

Replace:
- `<databricks-workspace-host>` with your workspace URL (e.g., `adb-1234567890123456.7.azuredatabricks.net`)
- `${FEDERATED_JWT_TOKEN}` with the actual JWT from your IdP

**Step 4.2: Parse Response**

The response will contain:
```json
{
  "access_token": "databricks-oauth-token-here",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "all-apis"
}
```

### Part 5: Use Databricks OAuth Token

**Step 5.1: Use Token in API Calls**

Include the token in the Authorization header:

```bash
curl --request GET https://<databricks-workspace-host>/api/2.0/clusters/list \
  --header "Authorization: Bearer ${DATABRICKS_OAUTH_TOKEN}"
```

**Step 5.2: Use with Databricks SDK**

If using the Python SDK, you can configure it to use the token:

```python
from databricks.sdk import WorkspaceClient

# Option 1: Environment variable
import os
os.environ['DATABRICKS_TOKEN'] = databricks_oauth_token
client = WorkspaceClient()

# Option 2: Direct configuration
client = WorkspaceClient(
    host="https://<databricks-workspace-host>",
    token=databricks_oauth_token
)
```

### Part 6: Implement Token Exchange in Your MCP Server

**Step 6.1: Extract User Token from Request**

In your MCP server, extract the IdP token from the Authorization header:

```python
from fastapi import Request, Header
from typing import Optional

async def get_user_token(request: Request) -> Optional[str]:
    auth_header = request.headers.get("Authorization")
    if auth_header and auth_header.startswith("Bearer "):
        return auth_header[7:]  # Remove "Bearer " prefix
    return None
```

**Step 6.2: Exchange Token**

Create a function to exchange the IdP token:

```python
import httpx
import os

async def exchange_idp_token_for_databricks_token(idp_token: str) -> str:
    """Exchange IdP JWT for Databricks OAuth token"""
    databricks_host = os.getenv("DATABRICKS_HOST")
    
    response = httpx.post(
        f"https://{databricks_host}/oidc/v1/token",
        data={
            "subject_token": idp_token,
            "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
            "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
            "scope": "all-apis"
        }
    )
    response.raise_for_status()
    return response.json()["access_token"]
```

**Step 6.3: Create User-Scoped WorkspaceClient**

Use the exchanged token to create a user-specific client:

```python
from databricks.sdk import WorkspaceClient

async def get_user_workspace_client(idp_token: str) -> WorkspaceClient:
    """Get a WorkspaceClient scoped to the user's identity"""
    databricks_token = await exchange_idp_token_for_databricks_token(idp_token)
    
    return WorkspaceClient(
        host=os.getenv("DATABRICKS_HOST"),
        token=databricks_token
    )
```

**Step 6.4: Use in Tool Execution**

Modify your tools to use the user-scoped client:

```python
async def execute_tool_with_user_context(request: Request, tool_name: str, **kwargs):
    idp_token = await get_user_token(request)
    user_client = await get_user_workspace_client(idp_token)
    
    # Now use user_client instead of the service principal client
    # This will use the user's permissions, not the service principal's
    result = await call_databricks_api(user_client, tool_name, **kwargs)
    return result
```

### Part 7: Environment Configuration

**Step 7.1: Set Environment Variables**

```bash
export DATABRICKS_HOST="adb-1234567890123456.7.azuredatabricks.net"
# Optionally set default token (for service principal fallback)
export DATABRICKS_TOKEN="<service-principal-token>"
```

### Part 8: Testing

**Step 8.1: Test Token Exchange**

```bash
# Get IdP token (example with Azure AD)
IDP_TOKEN=$(az account get-access-token --resource https://<databricks-host> --query accessToken -o tsv)

# Exchange for Databricks token
DATABRICKS_TOKEN=$(curl -s -X POST "https://<databricks-host>/oidc/v1/token" \
  -d "subject_token=${IDP_TOKEN}" \
  -d "subject_token_type=urn:ietf:params:oauth:token-type:jwt" \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  -d "scope=all-apis" | jq -r '.access_token')

# Test API call
curl -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  "https://<databricks-host>/api/2.0/clusters/list"
```

**Step 8.2: Test MCP Server**

- Send a request with an IdP token in the Authorization header
- Verify the server exchanges it for a Databricks token
- Verify API calls use the user's permissions

### Important Notes

1. Federation policy must be configured before token exchange works
2. Token lifetime: Databricks tokens typically inherit lifetime from the IdP token (often ~1 hour)
3. Token refresh: Implement refresh logic if needed
4. Error handling: Handle exchange failures and invalid tokens
5. Security: Store tokens securely and never log them

### References

- Databricks OAuth Token Federation: `https://docs.databricks.com/<cloud>/en/dev-tools/auth/oauth-federation-exchange`
- Federation Policy Configuration: `https://learn.microsoft.com/en-us/azure/databricks/dev-tools/auth/oauth-federation-policy`

This enables per-user authentication where each user's requests use their own Databricks permissions, not the service principal's.

Can you point me to relevant documentation if so?
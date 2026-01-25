# Rancher OIDC Authentication Setup

This guide explains how to configure Rancher to use Authentik as the OIDC authentication provider.

## Prerequisites

- Rancher must be installed and accessible
- Authentik OAuth provider for Rancher must be deployed (via Fleet: `fleet/authentik-blueprints`)
- Bootstrap password for Rancher admin account (generated during Ansible installation)
- OAuth credentials (Client ID and Client Secret) stored in Ansible vault

## Step 1: Initial Rancher Setup

### Get the Bootstrap Password

The bootstrap password is randomly generated during Rancher installation. To retrieve it:

```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{ .data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

### Login to Rancher

1. Access Rancher UI at https://rancher.lab.vac.dev
2. Login with the bootstrap password retrieved above
3. Set the Rancher Server URL to: `https://rancher.lab.vac.dev`
4. Reset the admin password when prompted (use a secure password of at least 12 characters)

## Step 2: Configure OIDC Authentication

1. In the Rancher UI, click the hamburger menu (☰) in the top left
2. Navigate to **Users & Authentication**
3. Click on the **Auth Provider** tab
4. Find and select **Keycloak (OIDC)**

### Configuration Values

**Note**: The Client ID and Client Secret are stored in the Ansible vault and configured during deployment. Retrieve them from your vault or use the commands in the "Retrieving Credentials" section below.

Enter the following configuration:

| Field | Value |
|-------|-------|
| **Client ID** | Retrieved from Ansible vault or database (see below) |
| **Client Secret** | Retrieved from Ansible vault or Kubernetes secret (see below) |
| **Endpoints**
| **Issuer** | `https://authentik.lab.vac.dev/application/o/rancher/` |
| **Auth Endpoint** | `https://authentik.lab.vac.dev/application/o/authorize/` |
| **Token Endpoint** | `https://authentik.lab.vac.dev/application/o/token/` |
| **User Info Endpoint** | `https://authentik.lab.vac.dev/application/o/userinfo/` |
| **Scopes** | `openid profile email groups` |
| **Group Claim Name** | `groups` |

### Important Notes

- **Do NOT change the Rancher Server URL** after initial setup - it cannot be updated
- Make sure to enable the provider after configuration
- Test the authentication by logging out and logging back in with an Authentik account
- Keep local authentication enabled as a fallback

## Step 3: Verify Configuration

1. Click **Enable** to activate the OIDC authentication
2. Click **Test and Enable Authentication** to verify the connection
3. You should be redirected to Authentik for authentication
4. After successful authentication, you'll be redirected back to Rancher

## Troubleshooting

### Authentication Fails
- Verify the Authentik provider is deployed and accessible
- Check that the client ID and secret match the Authentik provider configuration
- Ensure the issuer URL ends with a trailing slash
- Verify network connectivity between Rancher and Authentik

### Groups Not Syncing
- Ensure the scope `groups` is included in the configuration
- Verify the Group Claim Name is set to `groups`
- Check that the Authentik scope mapping is configured correctly

### Cannot Access Rancher After Enabling OIDC
- Use local authentication as fallback: add `?local=true` to the Rancher URL
  - Example: `https://rancher.lab.vac.dev/dashboard/auth/login?local=true`
- Login with the admin username and password set during initial setup

## Retrieving Credentials

The OAuth credentials are managed via Ansible vault and deployed as Kubernetes secrets. To retrieve them:

### Method 1: From Ansible Vault (Recommended)
Access your Ansible vault where the credentials are stored:
- `RANCHER_OIDC_CLIENT_ID`
- `RANCHER_OIDC_CLIENT_SECRET`

### Method 2: From Kubernetes Secrets
```bash
# Get Client ID from Kubernetes secret
kubectl -n authentik get secret authentik-env -o jsonpath='{.data.RANCHER_OIDC_CLIENT_ID}' | base64 -d && echo

# Get Client Secret from Kubernetes secret
kubectl -n authentik get secret authentik-env -o jsonpath='{.data.RANCHER_OIDC_CLIENT_SECRET}' | base64 -d && echo
```

### Method 3: From Authentik Database
```bash
# Get Client ID
kubectl -n authentik exec authentik-postgresql-0 -- \
  env PGPASSWORD=vaclab psql -U authentik -d authentik \
  -c "SELECT client_id FROM authentik_providers_oauth2_oauth2provider WHERE name='rancher';" -t

# Note: Client Secret is hashed in the database, use Method 1 or 2 instead
```

## Authentik Provider Configuration

The Authentik OAuth provider for Rancher is managed via Fleet in `fleet/authentik-blueprints/blueprints.yaml`.

Key configuration:
- **Client Type**: Confidential
- **Redirect URIs**: `https://rancher.lab.vac.dev/verify-auth`
- **Scopes**: openid, profile, email, Rancher Groups Mapping
- **Authorization flow**: default-provider-authorization-implicit-consent

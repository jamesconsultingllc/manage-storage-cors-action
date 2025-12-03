# Manage Azure Storage CORS Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Manage%20Azure%20Storage%20CORS-blue?logo=github)](https://github.com/marketplace/actions/manage-azure-storage-cors)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A GitHub Action to manage CORS (Cross-Origin Resource Sharing) rules for Azure Blob Storage. Perfect for automatically adding CORS rules for preview environments and cleaning them up when environments are deleted.

## Features

- ➕ **Add CORS Rules** - Add new origins to your storage account
- ➖ **Remove CORS Rules** - Clean up CORS rules when environments are deleted
- 🔄 **Auto-Consolidation** - Automatically consolidates origins when Azure's 5-rule limit is reached
- 🛡️ **Safe Removal** - Preserves other CORS rules when removing a single origin
- 📊 **Detailed Logging** - Shows current and final CORS configuration

## Prerequisites

- Azure CLI must be installed on the runner
- Storage account key must be provided (typically from secrets)

## Quick Start

### Add CORS Rule

```yaml
- name: Add CORS for Preview Environment
  uses: jamesconsultingllc/manage-storage-cors-action@v1
  with:
    action: add
    storage-account: mystorageaccount
    storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
    origin: https://preview-123.azurestaticapps.net
```

### Remove CORS Rule

```yaml
- name: Remove CORS for Preview Environment
  uses: jamesconsultingllc/manage-storage-cors-action@v1
  with:
    action: remove
    storage-account: mystorageaccount
    storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
    origin: https://preview-123.azurestaticapps.net
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `action` | Action to perform: `add` or `remove` | Yes | - |
| `storage-account` | Azure Storage account name | Yes | - |
| `storage-account-key` | Azure Storage account key | Yes | - |
| `origin` | Origin URL to add/remove | Yes | - |
| `methods` | Allowed HTTP methods (space-separated) | No | `DELETE GET HEAD MERGE OPTIONS POST PUT` |
| `allowed-headers` | Allowed headers pattern | No | `*` |
| `exposed-headers` | Exposed headers pattern | No | `*` |
| `max-age` | Max age in seconds for preflight cache | No | `3600` |

### Valid HTTP Methods

The following HTTP methods are valid for Azure Storage CORS:

`DELETE` `GET` `HEAD` `MERGE` `OPTIONS` `POST` `PUT` `PATCH` `CONNECT` `TRACE`

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `cors-updated` | Whether CORS rules were modified | `true` |

## Examples

### Add CORS for Multiple Preview Environments

```yaml
- name: Add CORS Rule
  uses: jamesconsultingllc/manage-storage-cors-action@v1
  with:
    action: add
    storage-account: ${{ vars.STORAGE_ACCOUNT_NAME }}
    storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
    origin: https://${{ steps.deploy.outputs.static_web_app_url }}
```

### Custom Methods and Headers

```yaml
- name: Add Restricted CORS Rule
  uses: jamesconsultingllc/manage-storage-cors-action@v1
  with:
    action: add
    storage-account: mystorageaccount
    storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
    origin: https://myapp.azurestaticapps.net
    methods: 'GET HEAD OPTIONS'
    allowed-headers: 'Content-Type Authorization'
    exposed-headers: 'Content-Length'
    max-age: '7200'
```

### Cleanup on PR Close

```yaml
name: Cleanup Preview Environment

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Determine Preview URL
        id: preview
        run: |
          # Construct preview environment URL
          BRANCH="${{ github.head_ref }}"
          SANITIZED=$(echo "$BRANCH" | sed 's/[^a-zA-Z0-9]//g' | tr '[:upper:]' '[:lower:]')
          echo "url=https://${SANITIZED:0:20}-my-swa.azurestaticapps.net" >> "$GITHUB_OUTPUT"

      - name: Remove CORS Rule
        uses: jamesconsultingllc/manage-storage-cors-action@v1
        with:
          action: remove
          storage-account: ${{ vars.STORAGE_ACCOUNT_NAME }}
          storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
          origin: ${{ steps.preview.outputs.url }}
```

### Complete GitFlow Workflow

```yaml
name: Azure SWA CI/CD

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    types: [closed]
    branches: [develop]

jobs:
  deploy:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Resolve Environment
        id: resolve-env
        uses: jamesconsultingllc/resolve-swa-environment-action@v1

      - name: Build and Deploy
        id: deploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN }}
          deployment_environment: ${{ steps.resolve-env.outputs.target-slot }}
          app_location: "/"
          output_location: "dist"

      - name: Add CORS for Preview Environment
        if: steps.resolve-env.outputs.is-preview == 'true'
        uses: jamesconsultingllc/manage-storage-cors-action@v1
        with:
          action: add
          storage-account: ${{ vars.STORAGE_ACCOUNT_NAME }}
          storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
          origin: https://${{ steps.resolve-env.outputs.target-environment }}-my-swa.azurestaticapps.net

  cleanup:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Resolve Environment
        id: resolve-env
        uses: jamesconsultingllc/resolve-swa-environment-action@v1
        with:
          branch: ${{ github.event.pull_request.head.ref }}

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Find Actual SWA Slot
        id: find-slot
        run: |
          SLOTS=$(az staticwebapp environment list \
            --name my-swa \
            --resource-group my-rg \
            -o json)
          
          PATTERN="${{ steps.resolve-env.outputs.target-environment }}"
          SLOT=$(echo "$SLOTS" | jq -r --arg p "${PATTERN:0:10}" \
            '.[] | select(.hostname | startswith($p)) | .hostname | split("-")[0]' | head -1)
          
          echo "slot=$SLOT" >> "$GITHUB_OUTPUT"

      - name: Remove CORS Rule
        if: steps.find-slot.outputs.slot != ''
        uses: jamesconsultingllc/manage-storage-cors-action@v1
        with:
          action: remove
          storage-account: ${{ vars.STORAGE_ACCOUNT_NAME }}
          storage-account-key: ${{ secrets.STORAGE_ACCOUNT_KEY }}
          origin: https://${{ steps.find-slot.outputs.slot }}-my-swa.azurestaticapps.net

      - name: Delete Preview Environment
        if: steps.find-slot.outputs.slot != ''
        run: |
          az staticwebapp environment delete \
            --name my-swa \
            --resource-group my-rg \
            --environment-name "${{ steps.find-slot.outputs.slot }}" \
            --yes
```

## Behavior Details

### Adding Rules

When you add a CORS rule:

1. **Check Existence** - If the origin already exists in CORS rules, skip (idempotent)
2. **Count Rules** - Azure allows maximum 5 CORS rules per storage service
3. **Add or Consolidate**:
   - If < 5 rules: Add as a new rule
   - If = 5 rules: Consolidate all origins into a single rule

### Removing Rules

When you remove a CORS rule:

1. **Check Existence** - If origin doesn't exist, skip (idempotent)
2. **Clear All Rules** - Azure doesn't support partial deletion
3. **Re-add Remaining** - Re-add all rules except the one being removed

### Azure CORS Rule Limit

Azure Blob Storage allows a maximum of **5 CORS rules** per service. This action automatically handles this limit:

- When adding a 6th origin, all origins are consolidated into a single rule
- Each rule can have multiple comma-separated origins

### Graceful Degradation

If `storage-account-key` is not provided:
- Action logs a warning and exits successfully
- Returns `cors-updated=false`
- Useful for environments where storage isn't configured

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Storage key missing | Warning logged, action succeeds |
| Origin already exists (add) | Skip, return `updated=false` |
| Origin doesn't exist (remove) | Skip, return `updated=false` |
| Invalid action | Error, exit code 1 |
| Azure CLI error | Error with Azure message, exit code 1 |

## Requirements

- GitHub Actions runner with `bash` and `jq` installed (included in ubuntu-latest)
- Azure Storage account access key
- Azure CLI installed (included in ubuntu-latest)

## Related Actions

- [resolve-swa-environment-action](https://github.com/jamesconsultingllc/resolve-swa-environment-action) - Resolve environments from branch names
- [sync-swa-settings-action](https://github.com/jamesconsultingllc/sync-swa-settings-action) - Sync app settings between environments

## License

MIT License - see [LICENSE](LICENSE) for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Support

- 🐛 [Report Issues](https://github.com/jamesconsultingllc/manage-storage-cors-action/issues)
- 💡 [Request Features](https://github.com/jamesconsultingllc/manage-storage-cors-action/issues/new)
- 📖 [Documentation](https://github.com/jamesconsultingllc/manage-storage-cors-action#readme)

## Why This Action?

Azure Static Web Apps with blob storage backends need CORS configured for each preview environment. Without this:

- Preview environments can't load images or files from storage
- Manual CORS management is tedious and error-prone
- Orphaned CORS rules accumulate when environments are deleted

This action automates CORS lifecycle management, keeping your storage account clean and your preview environments functional.

# Azure Virtual Machine Auto-Shutdown

[![](https://github.com/gabrielecanepa/azure-vm-actions/actions/workflows/start-vm.yml/badge.svg)](https://github.com/gabrielecanepa/azure-vm-actions/actions/workflows/start-vm.yml)
[![](https://github.com/gabrielecanepa/azure-vm-actions/actions/workflows/stop-vm.yml/badge.svg)](https://github.com/gabrielecanepa/azure-vm-actions/actions/workflows/stop-vm.yml)

Useful GitHub Actions workflows to start and automatically stop an Azure virtual machine on a schedule, with optional email notification before shutdown.

## Why use this?

Azure provides a [built-in auto-shutdown feature](https://learn.microsoft.com/en-us/azure/virtual-machines/auto-shutdown-vm) for virtual machines, but it has limitations:

- **No pre-shutdown notifications** — the built-in feature only supports a basic webhook; it doesn't send emails
- **No start workflow** — there's no built-in scheduled way to start a VM back up
- **Not version-controlled** — shutdown settings live in the Azure portal, outside the codebase
- **Not easily togglable** — enabling or disabling requires navigating the portal, here it's a single repository variable (`RUN_STOP`)

## Workflows

| Name | Description | Default schedule | Trigger |
| --- | --- | --- | --- |
| [Start VM](.github/workflows/start-vm.yml) | Starts the virtual machine | — | `workflow_dispatch` |
| [Stop VM](.github/workflows/stop-vm.yml) | Sends an optional email notification, waits 30 minutes, then stops and deallocates the virtual machine | Every day at midnight | `schedule`, `workflow_dispatch` |

The Stop VM workflow only runs when the `RUN_STOP` variable is set to `true`, which makes it easy to temporarily disable automatic shutdown without editing the workflow file.

### API dispatch

A workflow can also be triggered programmatically via a repository dispatch event, useful for automation:

```sh
curl --request POST \
  --url 'https://api.github.com/repos/<repo>/dispatches' \
  --header 'authorization: Bearer <GITHUB_ACCESS_TOKEN>' \
  --data '{ "event_type": "start" }' # or "stop" to shutdown
```

## Configuration

### 1. Azure service principal

Create a service principal and grant it contributor access to your resource group. Open a local terminal or [Azure Cloud Shell](https://shell.azure.com) and run:

```sh
SERVICE_PRINCIPAL_NAME="auto-shutdown"
SUBSCRIPTION_ID="<SUBSCRIPTION_ID>"
RESOURCE_GROUP_NAME="<RESOURCE_GROUP_NAME>"

az ad sp create-for-rbac \
  --name $SERVICE_PRINCIPAL_NAME \
  --scopes /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP_NAME \
  --role contributor \
  --sdk-auth
```

The output is a JSON object — copy it in full, you'll need it in the next step:

```json
{
  "clientId": "<CLIENT_ID>",
  "clientSecret": "<CLIENT_SECRET>",
  "subscriptionId": "<SUBSCRIPTION_ID>",
  "tenantId": "<TENANT_ID>",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

### 2. Repository secrets

Go to **Settings → Secrets and variables → Actions → Secrets** and add:

| Secret | Description |
| --- | --- |
| `AZURE_CREDENTIALS` | The full JSON object from the previous step |
| `MAIL_USERNAME` | Gmail address used to send notifications (e.g. `bot@gmail.com`) |
| `MAIL_PASSWORD` | Gmail [App Password](https://myaccount.google.com/apppasswords) for the sender account |
| `MAIL_FROM` | Display name and address for the sender (e.g. `Auto Shutdown <bot@gmail.com>`) |
| `MAIL_TO` | Recipient address(es), comma-separated |

> [!NOTE]
> The `MAIL_*` secrets are only required if you enable email notifications (see `NOTIFY_STOP` below).

### 3. Repository variables

Go to **Settings → Secrets and variables → Actions → Variables** and add:

| Variable | Description | Required |
| --- | --- | --- |
| `AZURE_RESOURCE_GROUP` | Name of the Azure resource group containing the VM | Yes |
| `AZURE_VM` | Name of the virtual machine | Yes |
| `RUN_STOP` | Set to `true` to enable the scheduled stop workflow | Yes |
| `NOTIFY_STOP` | Set to `true` to send an email notification 30 minutes before stopping | No |
| `MAIL_SUBJECT` | Subject line for the notification email | No |
| `MAIL_BODY` | Body of the notification email (Markdown supported) | No |

## Email notifications

When `NOTIFY_STOP` is `true`, the Stop VM workflow sends an email via Gmail SMTP before the machine is shut down. The notification is sent **30 minutes before the actual stop**, giving time to save the work or postpone the shutdown by triggering the workflow manually.

The email body (`MAIL_BODY`) supports Markdown and is rendered as HTML in the message.


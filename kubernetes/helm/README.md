# WSO2 API Platform Helm charts

Use the Helm charts in this folder to install the WSO2 API Platform product offerings on Kubernetes.

- This page introduces each offering and shows which charts to install for it.
- Each chart's own README has the full installation and configuration steps.

## Product offerings

The WSO2 API Platform has the following offerings.

| Offering | What you get | Charts to install |
| --- | --- | --- |
| Standalone AI Gateway | A gateway for AI traffic, such as requests to LLM providers | `gateway` |
| AI Gateway and AI Workspace | An AI Gateway that you manage from the AI Workspace console | `ai-workspace`, `gateway` |
| Standalone API Gateway | A gateway for API traffic | `gateway` |
| API Gateway and API Portal | An API Gateway, and a developer portal where consumers find and use your APIs | `api-portal`, `gateway` |
| Standalone API Portal | A developer portal and MCP Hub | `api-portal` |
| AI or API Gateway, AI Workspace, and API Portal | An AI or API Gateway, the AI Workspace, and the API Portal, sharing one Platform API | `ai-workspace`, `api-portal`, `gateway` |

## Components

Each offering is built from the following components.

- **Gateway.** Receives AI and API traffic and applies policies. The AI Gateway and the API Gateway use the same [`gateway`](gateway-helm-chart/README.md) chart and the same settings.
- **Platform API.** The shared control plane. The AI Workspace, the API Portal, and connected gateways use it.
- **AI Workspace.** Web console for managing AI resources such as LLM providers, MCP proxies, and AI gateways.
- **API Portal.** Developer portal and MCP Hub where consumers find and use APIs.

The [`ai-workspace`](ai-workspace-helm-chart/README.md) and [`api-portal`](api-portal-helm-chart/README.md) charts are product package charts.

- Each one installs the Platform API together with its console, in one step.
- They use the `platform-api`, `ai-workspace-ui`, and `api-portal-ui` charts in this folder, so you don't install those yourself.

The components work together in the following way.

- AI and API clients send their requests to the gateway.
- The gateway connects to the Platform API with a registration token and receives the APIs and proxies deployed to it.
- The AI Workspace connects to the Platform API to manage AI resources.
- The Platform API publishes APIs to the API Portal. The API Portal sends API key and subscription events back to the Platform API.

## Before you start

Make sure you have the following.

- A Kubernetes cluster, version 1.24 or later
- Helm 3.12 or later
- `kubectl`
- `openssl`, to generate encryption keys
- `htpasswd` or Docker, used by the AI Workspace and API Portal secret scripts
- cert-manager installed in your cluster, for TLS certificates. See [Installing cert-manager](gateway-helm-chart/README.md#installing-cert-manager).

## Install an offering

Each section lists the steps for one offering. Install the charts in the order shown.

### Standalone AI Gateway or API Gateway

1. Install the `gateway` chart. Follow the steps in the [`gateway` README](gateway-helm-chart/README.md).

### AI Gateway and AI Workspace

1. Install the `ai-workspace` chart. Follow the steps in the [`ai-workspace` README](ai-workspace-helm-chart/README.md).
2. In the AI Workspace, go to **AI Gateways**, add a gateway, and copy the **Gateway Registration Token**.
3. Install the `gateway` chart with the following values. Follow the steps in the [`gateway` README](gateway-helm-chart/README.md).
   - Set `gateway.controller.controlPlane.host` to `ai-workspace-platform-api.ai-workspace.svc:9243`. This is the Platform API address when you use the default release and namespace names.
   - Set `gateway.controller.controlPlane.token.value` to the registration token.
4. In the AI Workspace, check that the gateway status is **Active**.

> **Tip:** The **Kubernetes** tab on the gateway page shows a `helm install` command you can copy.

> **Note:** You can use a registration token only once. To connect the gateway again, click **Reconfigure** on the gateway page to get a new token.

### API Gateway and API Portal

1. Install the `api-portal` chart. Follow the steps in the [`api-portal` README](api-portal-helm-chart/README.md).
2. Register the gateway with the Platform API and copy its registration token. <!-- TODO: add where the user gets the registration token for an API Gateway. -->
3. Install the `gateway` chart with the following values. Follow the steps in the [`gateway` README](gateway-helm-chart/README.md).
   - Set `gateway.controller.controlPlane.host` to `api-portal-platform-api.api-portal.svc:9243`. This is the Platform API address when you use the default release and namespace names.
   - Set `gateway.controller.controlPlane.token.value` to the registration token.

### Standalone API Portal

1. Install the `api-portal` chart. Follow the steps in the [`api-portal` README](api-portal-helm-chart/README.md).

### AI or API Gateway, AI Workspace, and API Portal

In this offering, all components share the Platform API that the `ai-workspace` chart installs. Run the commands from the `kubernetes/helm` folder.

1. Install the `ai-workspace` chart. Follow the steps in the [`ai-workspace` README](ai-workspace-helm-chart/README.md). The steps below use the default release and namespace name, `ai-workspace`.
2. Create the API Portal secret. Running the script with the `ai-workspace` namespace and release name gives the API Portal the public key of the shared Platform API.

   ```bash
   ./api-portal-helm-chart/generate-secrets.sh ai-workspace ai-workspace
   ```

3. Install the `api-portal` chart in the `ai-workspace` namespace with the following values. Follow the steps in the [`api-portal` README](api-portal-helm-chart/README.md), but skip its secrets step, because you created the secret in step 2.
   - Set `platform-api.enabled` to `false`.
   - Set `api-portal-ui.config.platformApi.baseUrl` to `https://ai-workspace-platform-api.ai-workspace.svc:9243`.
   - If the Platform API uses a self-signed certificate, also set `api-portal-ui.config.platformApi.insecure` to `true`.
4. In the AI Workspace, go to **AI Gateways**, add a gateway, and copy the **Gateway Registration Token**.
5. Install the `gateway` chart with the following values. Follow the steps in the [`gateway` README](gateway-helm-chart/README.md).
   - Set `gateway.controller.controlPlane.host` to `ai-workspace-platform-api.ai-workspace.svc:9243`.
   - Set `gateway.controller.controlPlane.token.value` to the registration token.
   - If the Platform API uses a self-signed certificate, also set `gateway.config.controller.controlplane.insecure_skip_verify` to `true`.

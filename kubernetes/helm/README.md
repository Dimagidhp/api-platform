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
| AI or API Gateway, AI Workspace, and API Portal | An AI or API Gateway, the AI Workspace, and the API Portal, sharing one Platform API | `ai-workspace` or `api-portal` with the other console added, and `gateway` |

## Components

Each offering is built from the following components.

- **Gateway.** Receives AI and API traffic and applies policies. The AI Gateway and the API Gateway use the same [`gateway`](gateway-helm-chart/README.md) chart and the same settings.
- **Platform API.** The shared control plane. The AI Workspace, the API Portal, and connected gateways use it.
- **AI Workspace.** Web console for managing AI resources such as LLM providers, MCP proxies, and AI gateways.
- **API Portal.** Developer portal and MCP Hub where consumers find and use APIs.

The [`ai-workspace`](ai-workspace-helm-chart/README.md) and [`api-portal`](api-portal-helm-chart/README.md) charts are umbrella charts.

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

To install these components as one product, use an umbrella chart and add the components you need to it as subcharts.

- You can use either umbrella chart, `ai-workspace` or `api-portal`. Start from one, and add the other console to it as a subchart.
- All components then share one Platform API.
- The steps below use the `ai-workspace` chart as an example. Run the commands from the `kubernetes/helm` folder.

1. Add the API Portal as a dependency in `ai-workspace-helm-chart/Chart.yaml`, after the existing dependencies.

   ```yaml
     - name: api-portal-ui
       version: "1.0.0"
       repository: "oci://ghcr.io/wso2/api-platform/helm-charts"
       condition: api-portal-ui.enabled
   ```

2. Create a values file for your own settings, for example `my_values.yaml`, and turn on the API Portal in it.

   ```yaml
   api-portal-ui:
     enabled: true
   ```

   - If the Platform API uses a self-signed certificate, also set `api-portal-ui.config.platformApi.insecure` to `true`.
3. Create the secrets, including the API Portal secret.

   ```bash
   API_PORTAL=true ./ai-workspace-helm-chart/generate-secrets.sh ai-workspace
   ```

4. Download the subcharts and install the chart with your values file. For details, see the [`ai-workspace` README](ai-workspace-helm-chart/README.md).

   ```bash
   helm dependency update ./ai-workspace-helm-chart
   helm upgrade --install ai-workspace ./ai-workspace-helm-chart -n ai-workspace \
     -f values-secrets.yaml -f my_values.yaml
   ```

5. In the AI Workspace, go to **AI Gateways**, add a gateway, and copy the **Gateway Registration Token**.
6. Install the `gateway` chart with the following values. Follow the steps in the [`gateway` README](gateway-helm-chart/README.md).
   - Set `gateway.controller.controlPlane.host` to `ai-workspace-platform-api.ai-workspace.svc:9243`.
   - Set `gateway.controller.controlPlane.token.value` to the registration token.
   - If the Platform API uses a self-signed certificate, also set `gateway.config.controller.controlplane.insecure_skip_verify` to `true`.

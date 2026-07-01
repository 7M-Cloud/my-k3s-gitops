K3s Platform AI Lab
Purpose: This repository contains the GitOps-managed home lab platform used to build an AI-enabled Kubernetes/OpenShift-style platform assistant. The platform combines K3s, Argo CD App-of-Apps, Prometheus, Grafana, Backstage, and a LangGraph/FastAPI AI Agent that answers natural-language platform questions using Prometheus metrics.

1. Current Working State
The lab currently has a working end-to-end flow:

text Backstage Platform AI UI ↓ /api/proxy/ai/ask ↓ AI Agent - FastAPI + LangGraph ↓ Prometheus API ↓ Natural-language answer + raw Prometheus response

Validated example:

bash curl http://backstage.localhost/api/proxy/ai/

Expected response:

json { "status": "AI Agent Running ✅", "prometheus_url": "http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090" }

Validated CPU query:

bash curl http://backstage.localhost/api/proxy/ai/cpu

Expected response pattern:

json { "status": "success", "data": { "resultType": "vector", "result": [ { "metric": {}, "value": [ 1782946796.729, "21.232870370370417" ] } ] } }

Validated natural-language UI example:

text Question: show current cluster cpu usage Answer: Current cluster CPU usage percentage is 21.23%.

2. Platform Components
Component	Namespace	Purpose	Status
K3s	N/A	Lightweight Kubernetes runtime	Working
Argo CD	argocd	GitOps controller	Working
Argo CD ApplicationSet	argocd	App-of-Apps orchestration	Working
Prometheus	monitoring	Metrics backend	Working
Grafana	monitoring	Dashboards and visualization	Working
Backstage	backstage	Developer portal and AI UI	Working
AI Agent	ai-agent	FastAPI/LangGraph AI backend	Working
3. Repository Structure
Current GitOps repository layout:

text my-k3s-gitops/ ├── bootstrap/ │ └── root-app.yaml ├── platform/ │ ├── appsets/ │ │ ├── platform-appset.yaml │ │ └── tenant-appset.yaml │ └── projects/ │ ├── platform-project.yaml │ └── tenants-project.yaml ├── tenants/ │ ├── monitoring/ │ │ ├── grafana/ │ │ │ └── values.yaml │ │ ├── prometheus/ │ │ │ └── values.yaml │ │ └── namespace.yaml │ └── platform-team/ │ ├── backstage/ │ │ └── values.yaml │ ├── ai-agent/ │ │ ├── namespace.yaml │ │ ├── deployment.yaml │ │ └── service.yaml │ └── namespaces.yaml └── README.md

4. GitOps Model
The platform uses a strict GitOps model.

text bootstrap/root-app.yaml ↓ platform/ ↓ projects + appsets ↓ Generated Argo CD Applications ↓ Platform workloads

Key Principle
Do not deploy platform components manually with kubectl apply unless explicitly troubleshooting.

Preferred workflow:

text Edit YAML → Git commit → Git push → Argo CD sync → Kubernetes reconciliation

5. Root Application
bootstrap/root-app.yaml is the top-level Argo CD application.

```yaml apiVersion: argoproj.io/v1alpha1 kind: Application metadata: name: root-app namespace: argocd spec: project: default

source: repoURL: https://github.com/7M-Cloud/my-k3s-gitops.git targetRevision: main path: platform directory: recurse: true

destination: server: https://kubernetes.default.svc namespace: argocd

ignoreDifferences: - group: apiextensions.k8s.io kind: CustomResourceDefinition name: applicationsets.argoproj.io

syncPolicy: automated: prune: true selfHeal: true syncOptions: - CreateNamespace=true - ServerSideApply=true ```

6. ApplicationSets
6.1 Helm-Based Platform AppSet
File:

text platform/appsets/platform-appset.yaml

Purpose:

text Deploy Helm-based platform applications: - Prometheus - Grafana - Backstage

Do not place raw Kubernetes YAML apps like ai-agent in this AppSet.

6.2 Raw Manifest Tenant AppSet
File:

text platform/appsets/tenant-appset.yaml

Purpose:

text Deploy raw Kubernetes manifests from Git.

Current application:

text ai-agent

Recommended tenant-appset.yaml:

```yaml apiVersion: argoproj.io/v1alpha1 kind: ApplicationSet metadata: name: tenant-appset namespace: argocd spec: generators: - list: elements: - name: ai-agent project: tenants namespace: ai-agent repo: https://github.com/7M-Cloud/my-k3s-gitops.git revision: main path: tenants/platform-team/ai-agent

template: metadata: name: "{{name}}" namespace: argocd finalizers: - resources-finalizer.argocd.argoproj.io labels: app.kubernetes.io/part-of: tenants app.kubernetes.io/managed-by: argocd spec: project: "{{project}}"

  destination:
    server: https://kubernetes.default.svc
    namespace: "{{namespace}}"

  source:
    repoURL: "{{repo}}"
    targetRevision: "{{revision}}"
    path: "{{path}}"

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

7. Argo CD Projects
7.1 Platform Project
File:

text platform/projects/platform-project.yaml

Purpose:

text Allows deployment of shared platform services into monitoring and kube-system namespaces.

7.2 Tenants Project
File:

text platform/projects/tenants-project.yaml

Current required destinations:

```yaml destinations: - namespace: backstage server: https://kubernetes.default.svc

namespace: ai-agent server: https://kubernetes.default.svc ```
If Argo CD shows this error:

text application destination server 'https://kubernetes.default.svc' and namespace 'ai-agent' do not match any of the allowed destinations in project 'tenants'

Then update tenants-project.yaml to allow the ai-agent namespace.

8. AI Agent
8.1 Runtime
The AI Agent is a Python application using:

text FastAPI LangGraph Requests Prometheus HTTP API

Runtime namespace:

text ai-agent

Service DNS:

text http://ai-agent.ai-agent.svc.cluster.local

8.2 AI Agent Manifests
Folder:

text tenants/platform-team/ai-agent

Expected files:

text namespace.yaml deployment.yaml service.yaml

8.3 AI Agent Deployment
yaml apiVersion: apps/v1 kind: Deployment metadata: name: ai-agent namespace: ai-agent spec: replicas: 1 selector: matchLabels: app: ai-agent template: metadata: labels: app: ai-agent spec: containers: - name: ai-agent image: docker.io/library/ai-agent:1.0 imagePullPolicy: Never ports: - containerPort: 8000

8.4 AI Agent Service
yaml apiVersion: v1 kind: Service metadata: name: ai-agent namespace: ai-agent spec: selector: app: ai-agent ports: - name: http port: 80 targetPort: 8000

8.5 Important Image Note
The AI image is local to K3s containerd.

Build locally:

bash cd ~/ai-agent sudo docker build -t ai-agent:1.0 .

Import into K3s:

bash sudo docker save ai-agent:1.0 | sudo k3s ctr images import -

Verify:

bash sudo k3s ctr images ls | grep ai-agent

Expected:

text docker.io/library/ai-agent:1.0

9. AI Agent Prometheus Configuration
The correct Prometheus service URL is:

text http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090

Earlier incorrect value:

text http://prometheus-prometheus-kube-prometheus-prometheus.monitoring.svc:9090

This caused:

text NameResolutionError Failed to resolve prometheus-prometheus-kube-prometheus-prometheus.monitoring.svc

Current PROM_URL
python PROM_URL = "http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090"

Useful PromQL Queries
Cluster CPU percentage:

promql 100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))

Cluster memory percentage:

promql 100 * (1 - (sum(node_memory_MemAvailable_bytes) / sum(node_memory_MemTotal_bytes)))

Namespace CPU:

promql topk(10, sum by(namespace) (rate(container_cpu_usage_seconds_total{container!="",pod!=""}[5m])))

Namespace memory:

promql topk(10, sum by(namespace) (container_memory_working_set_bytes{container!="",pod!=""}))

Unhealthy pods:

promql sum by(namespace, pod, phase) (kube_pod_status_phase{phase=~"Pending|Failed|Unknown"})

10. Backstage
10.1 Runtime
Backstage runs in namespace:

text backstage

URL:

text http://backstage.localhost

10.2 Custom Image
Image:

text backstage-ai:1.0

Current deployment image:

bash kubectl get deploy backstage -n backstage \ -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

Expected:

text backstage-ai:1.0

10.3 Build Backstage Image
From the Backstage source repo:

```bash cd ~/Desktop/backstage-ai

yarn install yarn tsc yarn build:backend

sudo docker build -f packages/backend/Dockerfile -t backstage-ai:1.0 . sudo docker save backstage-ai:1.0 | sudo k3s ctr images import - ```

If any legacy alias is still needed:

bash sudo k3s ctr images tag \ docker.io/library/backstage-ai:1.0 \ ghcr.io/docker.io/library/backstage-ai:1.0

Restart Backstage:

bash kubectl rollout restart deployment backstage -n backstage kubectl rollout status deployment backstage -n backstage

11. Backstage AI Proxy
Backstage proxy is configured to forward AI requests to the AI Agent service.

Expected log line:

text [HPM] Proxy created: /ai -> http://ai-agent.ai-agent.svc.cluster.local

Expected rewrite rule:

text [HPM] Proxy rewrite rule created: "^/api/proxy/ai/?" -> "/"

Backstage config:

yaml proxy: endpoints: '/ai': target: http://ai-agent.ai-agent.svc.cluster.local changeOrigin: true

Validate proxy config:

bash kubectl logs -n backstage deploy/backstage --tail=100 | grep -i proxy

Expected:

text Proxy created: /ai -> http://ai-agent.ai-agent.svc.cluster.local

Validate through Backstage:

bash curl http://backstage.localhost/api/proxy/ai/

Expected:

json { "status": "AI Agent Running ✅" }

12. Backstage Platform AI UI
12.1 Plugin Location
text plugins/platform-ai

Package name:

json "name": "@internal/backstage-plugin-platform-ai"

12.2 App Dependency
File:

text packages/app/package.json

Expected:

json "@internal/backstage-plugin-platform-ai": "workspace:*"

12.3 Workspace Configuration
File:

text package.json

Expected:

json "workspaces": [ "packages/*", "plugins/*" ]

12.4 Plugin Export
File:

text plugins/platform-ai/src/index.ts

Expected:

ts export { default } from './plugin';

12.5 Plugin Registration
File:

text packages/app/src/App.tsx

Expected:

```tsx import { createApp } from '@backstage/frontend-defaults'; import catalogPlugin from '@backstage/plugin-catalog/alpha'; import platformAiPlugin from '@internal/backstage-plugin-platform-ai'; import { navModule } from './modules/nav';

export default createApp({ features: [ catalogPlugin, platformAiPlugin, navModule, ], }); ```

Without this registration, /platform-ai returns:

text ERROR 404: PAGE NOT FOUND

12.6 Platform AI Page URL
text http://backstage.localhost/platform-ai

13. Validation Commands
13.1 Cluster Health
bash kubectl get nodes kubectl get pods -A

13.2 Argo CD Apps
bash argocd app list argocd app get root-app argocd app get backstage argocd app get ai-agent

13.3 AI Agent Health
bash kubectl get pods -n ai-agent kubectl get svc -n ai-agent kubectl get endpoints -n ai-agent ai-agent

Expected endpoint:

text 10.x.x.x:8000

13.4 Test AI Inside Cluster
bash kubectl run curl-test \ --image=curlimages/curl \ -it --rm --restart=Never -- sh

Inside the pod:

bash curl http://ai-agent.ai-agent.svc.cluster.local/ curl http://ai-agent.ai-agent.svc.cluster.local/cpu

13.5 Test Backstage Proxy
bash curl http://backstage.localhost/api/proxy/ai/ curl http://backstage.localhost/api/proxy/ai/cpu

13.6 Test AI Natural Language
bash curl -X POST http://backstage.localhost/api/proxy/ai/ask \ -H "Content-Type: application/json" \ -d '{"question":"show current cluster cpu usage"}'

Expected response:

json { "question": "show current cluster cpu usage", "intent": "cluster_cpu", "answer": "Current cluster CPU usage percentage is 21.23%." }

14. Troubleshooting Guide
Issue: AI Service Exists but No Pods
Symptoms:

```bash kubectl get svc -n ai-agent

service exists
kubectl get pods -n ai-agent

No resources found
kubectl get endpoints -n ai-agent ai-agent

```

Cause:

text Deployment missing, invalid, or not synced by Argo CD.

Fix:

bash argocd app resources ai-agent argocd app sync ai-agent kubectl get deploy -n ai-agent

Verify deployment.yaml exists:

bash find tenants/platform-team/ai-agent -maxdepth 1 -type f -print

Issue: Service Has No Endpoint
Symptoms:

bash kubectl get endpoints -n ai-agent ai-agent

Output:

text ENDPOINTS <none>

Cause:

text Service selector does not match pod labels, or pod is not running.

Fix service selector:

yaml selector: app: ai-agent

Fix pod label:

yaml labels: app: ai-agent

Issue: /cpu Returns Internal Server Error
Symptom:

bash curl http://ai-agent.ai-agent.svc.cluster.local/cpu Internal Server Error

Check logs:

bash kubectl logs -n ai-agent deploy/ai-agent --tail=100

Common cause:

text Wrong Prometheus service DNS.

Correct DNS:

text http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090

Issue: Cannot Resolve ai-agent.ai-agent.svc.cluster.local From Laptop
This is expected.

This DNS name only works inside Kubernetes.

Wrong from laptop:

bash curl http://ai-agent.ai-agent.svc.cluster.local/cpu

Correct from laptop:

bash curl http://backstage.localhost/api/proxy/ai/cpu

Correct from inside cluster:

bash curl http://ai-agent.ai-agent.svc.cluster.local/cpu

Issue: Backstage Proxy Not Working
Check logs:

bash kubectl logs -n backstage deploy/backstage --tail=100 | grep -i proxy

Expected:

text Proxy created: /ai -> http://ai-agent.ai-agent.svc.cluster.local

If missing, verify Backstage config contains:

yaml proxy: endpoints: '/ai': target: http://ai-agent.ai-agent.svc.cluster.local changeOrigin: true

Issue: /platform-ai Returns 404
Check App.tsx:

bash cat packages/app/src/App.tsx

Expected:

```tsx import platformAiPlugin from '@internal/backstage-plugin-platform-ai';

export default createApp({ features: [ catalogPlugin, platformAiPlugin, navModule, ], }); ```

If platformAiPlugin is missing, the plugin is built but not registered.

Issue: Yarn Workspace Not Found
Error:

text Workspace not found @internal/plugin-platform-ai

Check actual plugin name:

bash cat plugins/platform-ai/package.json | grep name

Expected:

json "name": "@internal/backstage-plugin-platform-ai"

Check app dependency:

bash cat packages/app/package.json | grep platform-ai

Expected:

json "@internal/backstage-plugin-platform-ai": "workspace:*"

Issue: Missing SmartToy Icon
Error:

text Cannot find module '@material-ui/icons/SmartToy'

Fix:

Remove the icon import from:

text plugins/platform-ai/src/plugin.tsx

Use simple plugin registration without icon.

15. Current Capability Matrix
Capability	Status
K3s cluster	Complete
Argo CD root app	Complete
Platform ApplicationSet	Complete
Tenant ApplicationSet	Complete
Prometheus metrics	Complete
Grafana dashboard	Basic complete
Backstage custom image	Complete
AI Agent FastAPI	Complete
AI Agent LangGraph workflow	Working
Backstage AI proxy	Complete
Platform AI UI	Working/in progress depending route validation
CPU query	Complete
Memory query	Complete
Namespace usage	Added/in progress
Unhealthy pods	Added/in progress
Right-sizing starter logic	Added/in progress
Excel/report generation	Not started
Enterprise RBAC	Not started
Multi-cluster ACM integration	Not started
16. Recommended Next Enhancements
16.1 Improve UI Rendering
Current UI shows:

text answer raw JSON

Next UI should show:

text Intent PromQL Summary Table result Raw result collapsible

16.2 Add Capacity Summary
Question:

text show cluster capacity summary

Response:

text CPU: 21.23% Memory: 48.10% Unhealthy pods: 0 Risk: Green

16.3 Add Right-Sizing Engine
Add P95-based analysis:

text current request p95 usage recommended request waste percentage risk level

16.4 Add Report Generation
Future endpoints:

text /report/capacity /report/showback /report/rightsizing

Export formats:

text JSON CSV Excel PDF

16.5 Add Enterprise Governance
Future controls:

text RBAC-aware answers approved PromQL only audit logs rate limits query timeout namespace ownership mapping CMDB integration

16.6 Add Multi-Cluster Support
Future architecture:

text Backstage ↓ AI Agent ↓ ACM Hub / Thanos / Prometheus ↓ Managed clusters

17. Enterprise Target Architecture
text Backstage Platform AI │ ▼ LangGraph AI Agent │ ├── Prometheus Tool ├── OpenShift/Kubernetes API Tool ├── Grafana Tool ├── Alertmanager Tool ├── Right-Sizing Tool └── Report Generator │ ▼ Capacity / Showback / Right-Sizing Insights

18. Safe Operating Principles
Use Git as the source of truth.
Do not deploy workloads manually unless troubleshooting.
Do not let AI generate unrestricted PromQL in production.
Use approved PromQL templates.
Keep AI read-only initially.
Require human approval before any right-sizing action.
Use RBAC before exposing to multiple users.
Audit all AI questions and executed queries.
19. Quick Recovery Checklist
If something breaks, run:

bash kubectl get pods -A argocd app list argocd app get root-app argocd app get backstage argocd app get ai-agent kubectl logs -n ai-agent deploy/ai-agent --tail=100 kubectl logs -n backstage deploy/backstage --tail=100 curl http://backstage.localhost/api/proxy/ai/ curl http://backstage.localhost/api/proxy/ai/cpu

Expected minimum healthy state:

text backstage pod Running ai-agent pod Running ai-agent endpoint exists Backstage proxy /ai works CPU query returns Prometheus data

20. Current MVP Completion
Approximate status:

text Platform foundation: 100% AI backend: 85-90% Backstage AI UI: 80-90% Enterprise features: 20% Overall MVP: ~90%

21. Next Recommended Sprint
Sprint Goal
text Make Platform AI demo-ready for capacity and right-sizing use cases.

Tasks
text 1. Confirm /platform-ai route works. 2. Improve UI formatting. 3. Add capacity summary intent. 4. Add namespace CPU/memory tables. 5. Add unhealthy pods table. 6. Add basic right-sizing output. 7. Add download/export roadmap.

22. Useful Commands
Restart Backstage
bash kubectl rollout restart deployment backstage -n backstage kubectl rollout status deployment backstage -n backstage

Restart AI Agent
bash kubectl rollout restart deployment ai-agent -n ai-agent kubectl rollout status deployment ai-agent -n ai-agent

Check Backstage Proxy
bash kubectl logs -n backstage deploy/backstage --tail=100 | grep -i proxy

Check AI Agent Logs
bash kubectl logs -n ai-agent deploy/ai-agent --tail=100

Test AI Through Backstage
bash curl http://backstage.localhost/api/proxy/ai/ curl http://backstage.localhost/api/proxy/ai/cpu

Test Ask Endpoint
bash curl -X POST http://backstage.localhost/api/proxy/ai/ask \ -H "Content-Type: application/json" \ -d '{"question":"show current cluster cpu usage"}'

23. Notes
This is a local lab implementation. For enterprise production use, replace the following:

text local K3s image imports → image registry pullPolicy: Never → IfNotPresent/Always in-memory/local databases → PostgreSQL guest auth → Entra ID/OIDC single cluster Prometheus → Thanos/ACM/Mimir manual image rebuilds → CI/CD pipeline

24. Version History
Date	Change
2026-07-01	K3s, Argo CD, Prometheus, Grafana, Backstage baseline working
2026-07-01	AI Agent deployed through GitOps
2026-07-01	Backstage proxy to AI Agent working
2026-07-01	/api/proxy/ai/cpu validated
2026-07-02	Platform AI UI created in Backstage
2026-07-02	Natural-language CPU query working from Backstage UI
25. Ownership
Lab owner:

text Platform Engineering / AI Platform Lab

Primary objective:

text Build an enterprise-grade AI-assisted platform operations assistant for Kubernetes/OpenShift capacity, showback, and right-sizing use cases.
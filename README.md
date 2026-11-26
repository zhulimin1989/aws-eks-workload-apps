# aws-eks-workload-apps

App of Apps **两层全集中** Helm chart,**多 env 平行子目录**(`prod/` + `nprod/`):

- 每个 env 一份独立 chart(`<env>/Chart.yaml + values.yaml + templates/`)
- 第二层 chart 在 `<env>/apps/`,`values/<product>.yaml` 一份,声明该产品所有 child Application
- 业务 manifest 还在产品 k8s 仓(`aws-eks-<product>-k8s/<env>/<service>/`)

## 结构

```
.
├── prod/                              ← 生产 env
│   ├── Chart.yaml + values.yaml       ← 第一层(applications: map + projects: map)
│   ├── templates/                     ← 第一层 range
│   │   ├── applications.yaml          ← (sync-wave 1)
│   │   ├── projects.yaml              ← (sync-wave -1 先装)
│   │   └── demo-podinfo.yaml          ← Day 1 验证 demo
│   └── apps/                          ← 第二层 chart
│       ├── Chart.yaml
│       ├── templates/applications.yaml ← 第二层 range
│       └── values/
│           ├── raytone.yaml           ← 5 services
│           ├── relitoken.yaml         ← 4 services
│           └── agentmatrix.yaml       ← 占位(空)
│
└── nprod/                             ← 非生产 env(sit / uat 共用,加 entry 区分)
    ├── (同 prod/ 结构)
    └── apps/values/                   ← child Application path 指向产品仓 sit/<service>
```

## 第一层 + 第二层 range template 支持的字段

`values.yaml` 每条 applications / projects entry 可选字段(全可选):

```yaml
applications:
  prod-relitoken-appofapps:
    enabled: true
    namespace: relitoken-prod
    source: { repoURL, targetRevision, path, helm: { valueFiles, parameters } }

    # ─── 增强字段(2026-05-14 加)──────────────────
    additionalAnnotations:
      notifications.argoproj.io/subscribe.on-deployed.slack: "release-alerts"
    additionalLabels:
      team: relitoken
    syncPolicy:                   # 覆盖默认 prune/selfHeal/retry
      automated: { ... }
    ignoreDifferences:            # 业务侧 mutating webhook / 自动注入的字段
      - group: ""
        kind: Service
        jsonPointers: ["/spec/clusterIP"]
    info:                         # Argo UI 显示的 link / 元信息
      - { name: Owner, value: "relitoken-team" }
```

`projects` entry 同样支持 `additionalAnnotations / additionalLabels / permitOnlyProjectScopedClusters / orphanedResources / syncWindows / signatureKeys`。

## Cluster 上下文自动注入

第一层 root Application 自动给每个 Application 加 helm parameters:

```yaml
helm:
  parameters:
    - { name: cluster.name,   value: "raytonelab-prod-sg" }
    - { name: cluster.domain, value: "raytone.ai" }
    - { name: cluster.env,    value: "prod" }
```

第二层 chart 透传这 3 个值给业务叶 chart(如 ingress host 拼 `app.{{ .Values.cluster.domain }}`)。

注入路径:`root-app` helm install `--set cluster.name=... --set cluster.domain=...` → 第一层 values.yaml `cluster:` 块 → 第一层 template parameters → 第二层 chart values → 业务叶 chart。

## 当前 applications

> **2026-05-14 阶段聚焦**:Day 1 只推 **relitoken** 上线;**raytone / agentmatrix 暂缓**(deferred,等业务 image / 域名 ready 后再 `enabled: true` 启用)。
> 占位 entry 留着是为了:① 加新产品不用改 chart 结构,只 enable;② 文档化"我们规划接入哪些产品"。

### prod
| 名字 | path | valueFiles | 产品 | 状态 |
|---|---|---|---|---|
| `prod-raytone-appofapps` | `./prod/apps` | `values/raytone.yaml` | raytone | ⏸️ deferred(Day 1 暂不推)|
| `prod-relitoken-appofapps` | `./prod/apps` | `values/relitoken.yaml` | relitoken | 🎯 **当前焦点**,业务 ready 后 `enabled: true` |
| `prod-agentmatrix-appofapps` | `./prod/apps` | `values/agentmatrix.yaml` | agentmatrix | ⏸️ deferred(等业务代码仓 / 域名)|

### nprod
| 名字 | path | valueFiles | 状态 |
|---|---|---|---|
| `nprod-raytone-appofapps` | `./nprod/apps` | `values/raytone.yaml` | ⏸️ deferred,占位 |
| `nprod-relitoken-appofapps` | `./nprod/apps` | `values/relitoken.yaml` | 占位,业务 ready 后启 |
| `nprod-agentmatrix-appofapps` | `./nprod/apps` | `values/agentmatrix.yaml` | ⏸️ deferred,占位 |

未来加 uat env:`nprod/values.yaml` 加 `uat-<product>-appofapps` entry + 复用同份 `nprod/apps/values/<product>.yaml`(或加 env-specific values 文件)。

## 启用产品线

只改对应 env 的 `values.yaml`:

```yaml
# prod/values.yaml
applications:
  prod-relitoken-appofapps:
    enabled: true            # 改 true + git push 即可
```

ArgoCD root-app sync → 第一层 range → 渲染 root Application → 第二层 range → 多个 child Application → 拉产品仓叶 manifest。

## 加新产品线

3 步,**零新建 template 文件**:

1. 各 env values.yaml 的 `applications:` 加一条 entry
2. 各 env values.yaml 的 `projects:` 加一条 AppProject(走 `*` 宽松)
3. 各 env `apps/values/<new-product>.yaml` 写 service map

## AppProject(集中 + 宽松)

`values.yaml` `projects:` map 集中声明,`templates/projects.yaml` range 渲染。

**宽松策略**:`sourceRepos: ['*']`、`destinations: [{namespace: '*', server: '*'}]`、`clusterResource/namespaceResourceWhitelist: [{group: '*', kind: '*'}]` — 隔离主要走 Argo RBAC roles + IRSA + NetworkPolicy。

**sync 顺序**:`projects.yaml` 标 `sync-wave: -1`,`applications.yaml` 标 `sync-wave: 1`。

## root-app 指向

`aws-eks-system-apps/argocd/root-app/` 渲染 root Application,scaffold 参数:
- `app_of_apps_repo_path = prod`(prod env)/ `nprod`(非 prod env)
- `app_of_apps_repo_name = aws-eks-workload-apps`
- 通过 `helm --set cluster.name=raytonelab-prod-sg --set cluster.domain=raytone.ai` 注入

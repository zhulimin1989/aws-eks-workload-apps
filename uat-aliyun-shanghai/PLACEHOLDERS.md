# uat-aliyun-shanghai app-of-apps 当前状态

本目录用于阿里云上海 UAT ACK：`raytone-uat-aliyun-sh-ack`。

当前采取安全默认值：

1. 第一层 `uat-aliyun-shanghai-raytone-appofapps.enabled` 为 `false`。
2. 第二层 `apps/values/raytone.yaml` 中所有 child Application 的 `enabled` 均为 `false`。
3. 只有 AppProject 会随 Helm 渲染出来，业务应用不会在参数回填前同步。
4. `repoURL` 沿用张家口 GitLab mirror：`http://39.103.223.155/agentsphere/...`。
5. `cluster.region` 固定为 `cn-shanghai`，`cluster.name` 固定为 `raytone-uat-aliyun-sh-ack`。

## 启用步骤

1. 在 `aws-eks-raytone-k8s/uat-aliyun-shanghai` 回填 ACR、RDS、Redis、Kafka、KMS、NAS、OSS、NLB/域名、Auth 和内部 token。
2. 先将 `apps/values/raytone.yaml` 中需要部署的 child Application 改为 `enabled: true`。
3. 最后将 `values.yaml` 中 `uat-aliyun-shanghai-raytone-appofapps.enabled` 改为 `true`。
4. 第一次启用建议按 `platform -> middleware -> backend -> frontend/ingress` 分批同步。

## 关联设计文档

`/Users/atom/work/isoftstone/AgentSphere/deploy/raytone-uat-aliyun-shanghai`

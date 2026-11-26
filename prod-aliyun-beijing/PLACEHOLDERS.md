# prod-aliyun-beijing 当前说明

1. 当前目录专用于阿里云北京 ACK，第一层只保留 `raytone` app-of-apps。
2. `agentmatrix` / `demo` / `relitoken` 的北京 values 已移除，不再参与该环境渲染。
3. `repoURL` 已切到张家口 GitLab mirror，供北京 ACK / Argo CD 拉取。
4. `cluster.region` 已固定为 `cn-beijing`；各业务叶 chart 仍需按服务逐个核对镜像与依赖参数。

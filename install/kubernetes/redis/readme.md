本部署yaml 通过helm渲染得到



## 使用Helm替代部署
# 添加仓库
$ helm repo add bitnami https://charts.bitnami.com/bitnami

# 安装 Redis
$ helm install redis-62 ./redis/redis-16.13.2.tgz \
  -n infra \
  --set-string image.registry=docker2.gf.com.cn \
  --set-string image.repository=aims2/bitnami/redis \
  --set-string image.tag=6.2.12 \
  --set replica.replicaCount=1 \
  --set auth.enabled=true \
  --set-string master.persistence.storageClass=local-path \
  --set-string replica.persistence.storageClass=local-path \
  --set auth.password=admin

# 卸载 Redis
# $ helm uninstall redis-62 -n common

## 参数说明
- `./redis/redis-16.13.2.tgz`为从bitnamit同步的Helm Chart
- 因项目对 Redis 版本有要求，指定 Redis Chart 的版本为 16.13.2 是因为其是最后一个使用 6.2.x 版本的 Helm Chart
- `--set-string master.persistence.storageClass=local-path`和`--set-string replica.persistence.storageClass=local-path`用于指定StorageClass, 若不指定将会使用Default Storage Class
- `-n infra`将redis部署至`infra namespace`，`--set auth.password=admin`为设置redis密码，此处与`redis.yaml`保持一致
- `--set-string image.registry=docker2.gf.com.cn`, `--set-string image.repository=aims2/bitnami/redis`和`--set-string image.tag=6.2.12`用于指定内网镜像
## 注意事项
若创建后redis相关PVC为Pending，则需要自行创建PV，使用磁盘创建PV的案例如下
---
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
  labels:
    type: local
spec:
  storageClassName: local-path # 与PVC匹配
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /data/redis-pv  # 此处为本地路径，需根据实际情况修改
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01  # 此处为指定节点名称，需根据实际情况修改
EOF
---

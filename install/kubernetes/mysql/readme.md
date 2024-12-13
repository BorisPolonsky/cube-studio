# k8s部署mysql 

## 为机器打label 
kubectl label node node1xx mysql=true --overwrite

## 创建pv，pvc，根据自己的实际情况创建(内置的账号密码为root/admin)
kubectl apply -f pv-pvc-hostpath.yaml   
kubectl apply -f service.yaml     
kubectl apply -f configmap-mysql.yaml   
kubectl apply -f deploy.yaml  

## 校验mysql的pv和pvc是否匹配完成

# 本地调试可以使用docker启动mysql
docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=admin -d mysql:8.0.32  


## 使用Helm部署Mysql
helm install mysql-80 ./mysql/mysql-9.7.1.tgz \
  -n infra \
  --set-string image.registry=docker2.gf.com.cn \
  --set-string image.repository=aims2/bitnami/mysql \
  --set-string image.tag=8.0.32-debian-11-r21 \
  --set-string primary.persistence.storageClass=local-path \
  --set-string architecture=standalone \
  --set auth.rootPassword=admin

## 参数说明
- `./mysql/mysql-9.7.1.tgz`为从bitnamit同步的Helm Chart
- 因项目对 MySQL 版本有要求，指定 MySQL Chart 的版本为 16.13.2 是因为其是最后一个使用 6.2.x 版本的 Helm Chart
---set-string primary.persistence.storageClass=local-path
- `-n infra`将mysql部署至`infra namespace`，`--set auth.password=admin`为设置redis密码，此处与`redis.yaml`保持一致
- `--set-string image.registry=docker2.gf.com.cn`, `--set-string image.repository=aims2/bitnami/redis`和`--set-string image.tag=6.2.12`用于指定内网镜像
- `--set-string architecture=standalone`为无主备模式
## 注意事项
若创建后mysql相关PVC为Pending，则需要自行创建PV，使用磁盘创建PV的案例如下
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-10-80-85-11
  labels:
    type: local
spec:
  storageClassName: local-path # 与PVC匹配
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /data/mysql-pv  # 此处为本地路径，需根据实际情况修改
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - 10-80-85-11  # 此处为指定节点名称，需根据实际情况修改
EOF

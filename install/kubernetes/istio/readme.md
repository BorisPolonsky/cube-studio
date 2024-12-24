version=1.14.1
```bash
wget -c https://github.com/istio/istio/releases/download/$version/istioctl-$version-linux-amd64.tar.gz
tar zxfv istioctl-$version-linux-amd64.tar.gz -C /usr/local/bin/

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.14.1 TARGET_ARCH=x86_64 sh -

istioctl manifest generate --set Values.global.jwtPolicy=first-party-jwt > install.yaml

kubectl apply -f install.yaml
```

其中主要是base、istiod、gateway三部分

istiod 将先前由 Pilot，Galley，Citadel 和 sidecar 注入器执行的功能统一为一个二进制文件。


# 由1.3.1版本升级到1.4.1+

需要先删除validatingwebhookconfigurations mutatingwebhookconfigurations  deployment和svc ds等

查看所有资源

```bash
namespace=istio-system
kubectl api-resources -o name --verbs=list --namespaced | xargs -n 1 kubectl get --show-kind --ignore-not-found -n $namespace

```



### 使用Helm部署istio
部署`istio`控制面之前，需先部署想要的CRD

利用`istio-base`模板部署`CRD`

```
helm install istio-base istio/base-1.15.7.tgz -n istio-system --set defaultRevision=default --create-namespace
```
参数说明
`istio/base-1.15.7.tgz`可通过添加官方`helm repo`后通过`helm fetch istio/base --version=1.15.7`同步到离线环境的的模板，其中`1.15.7`是`istio 1.15.*`的最后一个版本


检查安装情况
```
helm ls -n istio-system
NAME      	NAMESPACE   	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
istio-base	istio-system	1       	2024-12-16 11:24:58.572877125 +0800 CST	deployed	base-1.15.7	1.15.7
```


```
helm install istiod istiod-1.15.7.tgz -n istio-system --wait \
  --set-string global.hub=docker2.gf.com.cn/aims2/istio
```
参数说明
- `istiod-1.15.7.tgz`可通过添加官方`helm repo`后通过`helm fetch istio/istiod --version=1.15.7`同步到离线环境的的模板，其中`1.15.7`是`istio 1.15.*`的最后一个版本
- `--set defaultRevision=default` 根据官方文档
> When performing a revisioned installation, the base chart requires the --set defaultRevision=<revision> value to be set for resource validation to function. Below we install the default revision, so --set defaultRevision=default is configured.
- `--set-string global.hub=docker2.gf.com.cn/aims2/istio`全家配置需要拉去istio镜像的本地仓库


注意用helm部署完`istio`后，仍需手工部署`istio-gateway`
```
kubectl create namespace istio-ingress
helm install istio-ingress istio/gateway -n istio-ingress --wait
```
注意helm部署后ns和ingress的名称都为`istio-ingress`,与手工部署的配置(`ns`为`istio-system`，`ingress`名称为`istio-ingressgateway`

该模板默认参数会将流量导入到ingress容器内的80和443端口，在istio容器没有配置特权的情形下无法使用80,443等序号过低的容器内端口，可新建形如下列values.yaml并修改相应配置，例如以下配置适用于gateway使用`15021`,`8080`和`8443`端口，ingress以`NodePort`形式暴露的使用场景
```
cat <<EOF | helm install istio-ingress --namespace istio-ingress -f - isito/gateway
service:
  # Type of service. Set to "None" to disable the service entirely
  type: NodePort
  ports:
  - name: status-port
    nodePort: 30000
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http2
    nodePort: 30010
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 30020
    port: 443
    protocol: TCP
    targetPort: 8443
EOF
`

注意，对于内核<4版本的系统，需要修改securityContext，参考[ref1](https://github.com/istio/istio/issues/49549)，否则istio-ingress会启动失败
可部署后修改ingress的配置
kubectl edit istio-ingress -n istio-ingress
去掉下列内容
```
        securityContext:
          sysctls:
          - name: net.ipv4.ip_unprivileged_port_start
            value: "0"
```
或者直接将官方helm chart的相关内容去掉，制作新的helm chart部署

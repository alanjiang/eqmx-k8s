
推荐Helm安装

1  Helm 环境搭建

需要golang和 Helm ， 下载二进制包解压：

导出环境变量：
export PATH=$PATH:~/softwares/go/bin
export PATH=$PATH:~/softwares/linux-amd64

2 安装

2.1 基于源码安装

下载源码到本地执行安装。 本方式方便更改源文件。

$ git clone https://github.com/emqx/emqx-chart
 
$ cd emqx-chart $ helm install --name myemqx --set deployment.image="emqx/emqx-edge:latest" .
 
 
 2.2 repo安装
 
 2.2.1  add repo 
 
 helm repo add emqx https://repos.emqx.io/charts

 2.2.2  helm repo update 
 
 helm repo update
 
 2.2.3 执行Helm  (命名空间为：developer)
 
 helm install my-emqx emqx/emqx -n  developer
 
 2.3 my-emqx svc 使用负载均衡
 
 spec.type: LoadBalancer 
 
 $kubectl edit svc my-emqx -n developer 
 
 注： LoadBalancer 可以自己创建（在阿里云上，设备SVC spec.type: Balancer 会自动创建一个新的 LB）
 
 使用已有的LoadBalancer 需要新增注解
 
 annotations:
  service.beta.kubernetes.io/alicloud-loadbalancer-id: XXX
  service.beta.kubernetes.io/alicloud-loadbalancer-force-override-listeners: 'true'
 spec:
   type: LoadBalancer
 
 如果在线编辑不生效，运行 
 
 helm upgrade  my-emqx emqx/emqx  -n developer

完整的SVC配置如下：其中：49.45.116.73 为 SLB 的公网 

apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: my-emqx
    meta.helm.sh/release-namespace: developer
    service.beta.kubernetes.io/alicloud-loadbalancer-force-override-listeners: "true"
    service.beta.kubernetes.io/alicloud-loadbalancer-id: lb-bp1wrtjeumtikrw6yhx32
  creationTimestamp: "2022-09-20T10:59:01Z"
  finalizers:
  - service.k8s.alibaba/resources
  labels:
    app.kubernetes.io/instance: my-emqx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: emqx
    helm.sh/chart: emqx-5.0.3
    service.beta.kubernetes.io/hash: d15b4b0356143d36c3677816857526bcf95d003819b46f304aac42a4
  name: my-emqx
  namespace: developer
  resourceVersion: "569779201"
 
 
 uid: 2d6935d4-f42d-49ef-98af-536f7219e978
spec:
  clusterIP: 192.168.106.82
  clusterIPs:
  - 192.168.106.82
  externalTrafficPolicy: Cluster
  ports:
  - name: mqtt
    nodePort: 30494
    port: 1883
    protocol: TCP
    targetPort: mqtt
  - name: mqttssl
    nodePort: 32360
    port: 8883
    protocol: TCP
 
 
  targetPort: mqttssl
  - name: ws
    nodePort: 31870
    port: 8083
    protocol: TCP
    targetPort: ws
  - name: wss
    nodePort: 30377
    port: 8084
    protocol: TCP
    targetPort: wss
  - name: dashboard
    nodePort: 30511
    port: 18083
    protocol: TCP
    targetPort: dashboard
nodePort: 30511
    port: 18083
    protocol: TCP
    targetPort: dashboard
  selector:
    app.kubernetes.io/instance: my-emqx
    app.kubernetes.io/name: emqx
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 49.45.116.73
    
 
 
 
2.4 部署ingress 

2.4.1 创建证书

准备key和pom格式的证书文件。 创建名为 my-emqx-ssl的k8s密钥。

kubectl create secret tls my-emqx-ssl --key=/home/jiangpeng25/cert/my-emqx.key  --cert=/home/jiangpeng25/cert/my-emqx.pem  -n  developer 

2.4.2 定义 Ingress 

域名： my-emqx.XX.com , 

ingress.yaml 文件如下：spec.tls.hosts.secretName 使用my-emqx-ssl。

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  name: my-emqx-ingress
spec:
  tls:
  - hosts:
    - my-emqx.XX.com
    secretName: my-emqx-ssl
  rules:
  - host: my-emqx.XX.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
             service:
               name: my-emqx
               port:
                 number: 18083
        - path: /mqtt
          pathType: Prefix
          backend:
              service:
                name: my-emqx
                port:
                  number: 8083
        - path: /
          pathType: Prefix
          backend:
              service:
                name: my-emqx
                port:
                  number: 1883
 

$ kubectl apply -f ingress.ymal -n  developer 



注意： Helm 的安装包在：~/.cache/helm/repository 文件夹中可以找到。

---
title: '[k8s实践系列]基于动态jenkins-slave的CICD'
date: 2020-07-29 15:25:43
tags:
- k8s
- Jenkins
categories:
- 实践K8s
- Jenkins
description: 在k8s环境中部署jenkins并实现动态slave功能
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1596017743701&di=489a3aed2ef7d89f6f665bdf4e6b9362&imgtype=0&src=http%3A%2F%2Fimg4.mukewang.com%2F5b1e0cfc0001ef7b06000338.jpg
---



**在k8s中部署jenkins服务，并实现根据流水线自动创建slave工作节点。**



------



# 准备工作



## 新建namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-ops
```



执行下面的命令完成创建：

```bash
$ kubectl create -f namespace.yaml
```



## 创建storageclass对象

这里使用nfs作为后端存储，可以参考：{% post_link k8s实践系列-使用nfs存储  %}，需要创建一个名为`xjl-storageclass`的storageclass。



## 创建pvc

```yaml
# pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: opspvc
  namespace: kube-ops
spec:
  storageClassName: xjl-storageclass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```



执行下面的命令完成创建：

```bash
$ kubectl create -f pvc.yaml
```



![](pvc.png)

<br>



# 服务部署

## 创建serviceaccount并授权

```yaml
# jenkins-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins2
  namespace: kube-ops

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins2
  namespace: kube-ops
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins2
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins2
subjects:
  - kind: ServiceAccount
    name: jenkins2
    namespace: kube-ops
```



执行下面的命令完成创建：

```bash
$ kubectl create -f jenkins-serviceaccount.yaml
```



## 新建master deployment

```yaml
# jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins2
  namespace: kube-ops
  labels:
    app: jenkins2
spec:
  selector:
    matchLabels:
      app: jenkins2
  template:
    metadata:
      labels:
        app: jenkins2
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccount: jenkins2
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 2000m
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          subPath: jenkins2
          mountPath: /var/jenkins_home
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: opspvc
```



执行下面的命令完成创建：

```bash
$ kubectl create -f jenkins-deployment.yaml
```



## 新建service

```yaml
# jenkins-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins2
  namespace: kube-ops
  labels:
    app: jenkins2
spec:
  selector:
    app: jenkins2
  type: ClusterIP
  ports:
  - name: web
    port: 8080
    targetPort: 8080
    protocol: TCP
  - name: agent
    port: 50000
    targetPort: 50000
    protocol: TCP
```



执行下面的命令完成创建：

```bash
$ kubectl create -f jenkins-service.yaml
```



<br>



# 暴露服务

## 新建Ingress

设定jenkins的域名为：`jenkins-k8s.example.com`



```yaml
# jenkins-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins2
  namespace: kube-ops
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: jenkins-k8s.example.com
    http:
      paths:
      - backend:
          serviceName: jenkins2
          servicePort: 8080
        path: /
```



执行下面的域名完成创建：

```bash
$ kubectl create -f jenkins-ingress.yaml
```



<br>



## 查看服务运行情况

```bash
$ kubectl get pod,svc,ingress,pvc -n kube-ops
```

![](status.png)



<br>



## nginx设置反向代理

这里使用ingress-nginx作为服务入口，外部通过nginx最为ingress的代理，设置jenkins对应的nginx配置如下：

```nginx
# jenkins-k8s.conf
upstream jenkins-k8s {
    server 10.8.138.10:80 max_fails=3 fail_timeout=10s weight=2;
    server 10.8.138.9:80 max_fails=3 fail_timeout=10s weight=2;
}

server {
    listen 80;
    server_name jenkins-k8s.example.com;
    access_log /var/log/nginx/jenkins-k8s.example.com_access.log elk_json;
    error_log /var/log/nginx/jenkins-k8s.example.com_error.log;
    expires -1;

    location / {
        proxy_pass http://jenkins-k8s;
        proxy_set_header Host jenkins-k8s.example.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_buffering          off;
        proxy_redirect off;
        proxy_intercept_errors on;
        proxy_http_version 1.1;
        proxy_connect_timeout    30;
        proxy_read_timeout       30;
        proxy_send_timeout       30;
        proxy_buffer_size 64k;
        proxy_buffers 8 64k;
    }

    location = /favicon.ico {
        log_not_found off;
        access_log off;
		}

    location ~* /\.(svn|git)/ {
        return 404;
    }
}
```



- upstream中的两台机器是我的k8s中的ingress节点；
- 域名`jenkins-k8s.example.com`需要和nginx节点的IP做hosdt绑定；



执行下面的命令检查nginx并重载配置：

```bash
$ nginx -t
$ nginx -s reload 
```



<br>



# 访问测试

通过浏览器，使用域名访问jenkins，可以看到如下的页面：

![](jenkins-init.png)



使用如下的命令获取到token并解锁jenkins：

![](jenkins-password.png)



进入后根据实际情况安装插件，然后设置一个用户名和密码，就完成jenkins的部署了。

![](jenkins-home.png)



<br>



# jenkins汉化

选择 `Manage Jenkins` --> `Plugin Manager` --> `Available`，搜索插件`locale plugin`并安装。

> 一般安装完成就自动切换为中文了。



<br>



# 动态创建slave

## 什么是动态生成slave

Jenkins Master 和 Jenkins Slave 以 Pod 形式运行在 Kubernetes 集群的 Node 上，Master 运行在其中一个节点，并且将其配置数据存储到一个 Volume 上去，Slave 运行在各个节点上，并且它不是一直处于运行状态，它会按照需求动态的创建并自动删除。



这种方式的工作流程大致为：当 Jenkins Master 接受到 Build 请求时，会根据配置的 Label 动态创建一个运行在 Pod 中的 Jenkins Slave 并注册到 Master 上，当运行完 Job 后，这个 Slave 会被注销并且这个 Pod 也会自动删除，恢复到最初状态。

![](jenkins-slave)



## 优势

- 服务高可用，当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的 Jenkins Master 容器，并且将 Volume 分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。
- 动态伸缩，合理使用资源，每次运行 Job 时，会自动创建一个 Jenkins Slave，Job 完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes 会根据每个资源的使用情况，动态分配 Slave 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。
- 扩展性好，当 Kubernetes 集群的资源严重不足而导致 Job 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。



## 插件安装和设置

搜索插件`kubernetes`并安装。

![](kubernetes.png)



选择`Manage Jenkins` —> `Configure System` —> (拖到最下方)`Add a new cloud`，选择`kubernetes`，填写kubernetes和jenkins信息：

![](add-cloud.png)

- `namespace`这里填写的是`kube-ops`；
- `jenkins地址`按照k8s集群内域名的格式填写；
- 点击测试，如果出现`success`表名连接集群成功；



然后点击`pod template`，添加一个pod模板，就是`jenkins-slave`运行的模板：

![](pod-template.png)

> 这里的template名称和标签列表似乎只能是这个值，否则slave启动会失败。





需要挂载两个主机目录：

- ` /var/run/docker.sock`：该文件是用于 Pod 中的容器能够共享宿主机的 Docker（docker in docker 的方式）；
- ` /root/.kube` ：将这个目录挂载到容器的 /home/jenkins/.kube 目录下面这是为了让我们能够在 Pod 的容器中能够使用 kubectl 工具来访问 Kubernetes 集群，方便后面在 Slave Pod 部署 Kubernetes 应用；

![](volume.png)



这里设置一下超时时间和serviceaccount，防止出现权限问题。

![](timeout.png)

设置完成后保存。



**因为需要kubeconfig文件，所以从master上将/root/.kube拷贝到node节点上**



## 测试

新建一个任务：

![](demo.png)



这里的标签表达式填入上面设置的标签`haimaxy-jnlp`：

![](label.png)



然后在`构建`这个步骤中，选择`执行shell`，使用下面的脚本测试：

```bash
echo "测试 Kubernetes 动态生成 jenkins slave"echo "==============docker in docker==========="
docker info

echo "=============kubectl============="
kubectl get pods
```



点击保存后，执行构建，因为整个构建很简单，所以速度很快，查看构建日志可以看见构建成功了：

![](result.png)



或者可以在master节点上执行下面的命令动态查看构建过程：

```bash
watch kubectl get pod -n kube-ops
```

> 可以看到会创建一个slave的pod执行成功后自动删除。



<br>



# pipline结合动态slave

## 什么是pipeline

在 Jenkins 中的构建工作可以有多种方式，常用的是 Pipeline 这种方式。Pipeline就是一套运行在 Jenkins 上的工作流框架，将原来独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的工作。



## Pipeline 核心概念

- `Node`：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境，比如动态运行的Slave；
- `Stage`：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念，可以跨多个 Node
- `Step`：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令：sh 'make'，就相当于我们平时 shell 终端中执行 make 命令一样。



## 如何创建Pipline

- Pipeline 脚本是由 Groovy 语言实现的，但是我们没必要单独去学习 Groovy，当然你会的话最好
- Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法
- Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一个 Jenkinsfile 脚本文件放入项目源码库中
- 一般我们都推荐在 Jenkins 中直接从源代码控制(SCMD)中直接载入 Jenkinsfile Pipeline 这种方法

<br>



## 创建一个简单的pipline

新建任务，选择`流水线`：

![](pipline-demo.png)



在流水线脚本中，输入下面的脚本，然后点击保存：

![](pipline-script.png)

```groovy
node('haimaxy-jnlp') {    stage('Clone') {
      echo "1.Clone Stage"
    }
    stage('Test') {
      echo "2.Test Stage"
    }
    stage('Build') {
      echo "3.Build Stage"
    }
    stage('Deploy') {
      echo "4. Deploy Stage"
    }
}
```

> 这个构建脚本，首先给node添加了上面设置的动态slave标签，然后设置了4个简单的构建流程



直接点击构建，会看到构建成功：

![](pipline-build.png)



在k8s集群上执行下面的命令，也看到了动态创建的pod执行该构建，成功后被清除：

<img src="onk8s.png" style="zoom:67%;" />



<br>



# 通过pipeline部署服务到k8s

## 流程

要部署 Kubernetes大概的流程如下：

- `CI阶段`：编写代码 -- 测试 -- 编写 Dockerfile --构建打包 Docker 镜像 -- 推送 Docker 镜像到仓库；
- `CD阶段`：编写 Kubernetes YAML 文件 -- 利用 kubectl 工具部署应用



## jenkins添加凭证

由于我们是拉取私有仓库（harbor中）的镜像，所以需要密码认证。为了避免明文密码泄露，所以使用jenkins的凭证管理密码：

![](jenkins-certs.png)



>  注意这里的ID，后面要用到。



## 创建代码

在自己的gitlab中先创建一个空的项目（没有gitlab自行搭建一个就好），这里我的是`pipline-demo`：

![](gitinit.png)



将项目克隆到服务器上：

```bash
$ git clone http://10.8.138.11/root/pipeline-demo.git
```



添加代码：

```bash
$ cat > app.py << EOF
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0')
EOF
```



## 添加dockerfile

```dockerfile
FROM centos:7

WORKDIR /root

RUN yum install -y epel-release
RUN yum install -y python-pip && /usr/bin/pip install flask

ADD app.py /root

CMD python /root/app.py

EXPOSE 5000
```



## 添加部署yaml

创建一个deployment的yaml文件，用于部署服务：

```bash
$ cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-demo
spec:
  selector:
    matchLabels:
      app: jenkins-demo
  template:
    metadata:
      labels:
        app: jenkins-demo
    spec:
      containers:
      - image: 10.8.138.11:8181/python-demo/python-demo:<BUILD_TAG>
        imagePullPolicy: IfNotPresent
        name: jenkins-demo
EOF
```

**这里的镜像tag会在Jenkinsfile中导入**



##  添加Jenkinsfile

```bash
$ cat > Jenkinsfile << EOF
node('haimaxy-jnlp') {
    stage('Clone') {
      echo "1.Clone Stage"
      checkout scm
      script {
        build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
      }
    }

    stage('Test') {
      echo "2.Test Stage"
    }

    stage('Build') {
      echo "3.Build Docker Image Stage"
      sh "docker build -t 10.8.138.11:8181/python-demo/python-demo:${build_tag} ."
    }

    stage('Push') {
      echo "4.Push Docker Image Stage"
      withCredentials([usernamePassword(credentialsId: 'harbor', passwordVariable: 'harborPassword', usernameVariable: 'harborUser')]) {
        sh "docker login -u ${harborUser} -p ${harborPassword} 10.8.138.11:8181"
        sh "docker push 10.8.138.11:8181/python-demo/python-demo:${build_tag}"
      }
    }

    stage('Deploy') {
      echo "5. Deploy Stage"
      def userInput = input(
        id: 'userInput',
        message: 'Choose a deploy environment',
        parameters: [
          [
            $class: 'ChoiceParameterDefinition',
            choices: "Dev\nQA\nProd",
            name: 'Env'
          ]
        ]
      )
      echo "This is a deploy step to ${userInput}"
      sh "sed -i 's/<BUILD_TAG>/${build_tag}/' deployment.yaml"

      if (userInput == "Dev") {
        echo "Deploying to DEV ."
      } else if (userInput == "QA"){
        echo "Deploying to QA ."
      } else {
        echo "Deploying to Prod ."
      }

      sh "kubectl apply -f deployment.yaml"
    }
}
EOF
```

- 这个Jenkinsfile将流水线分为：获取代码、测试、构建、推送、部署这几步，其中测试暂时忽略；
- 克隆这一步中，使用script将上传代码的commit与镜像tag进行关联，方便后续问题定位；
- 推送镜像这一步，使用的是之前在jenkins中创建的凭证，`credentialsId`为凭证的ID，`passwordVariable`和`usernameVariable`的前缀为凭证ID；
- 一般情况下，部署的时候都会选择现部署到测试或者开发环境，再部署到生产，所以这里通过获取用户输入来选择部署到哪个环境（当然这里只是一个示例，并没有那么多环境）；
- 部署这一步中，通过`sed`命令将镜像tag进行替换；



## 推动代码到代码库

```bash
$ git add .
$ git commit -m 'init project'
$ git push origin master
```

![](gitlab-push.png)



## 修改流水线配置

编辑之前创建的`pipeline-demo`流水线，修改流水线定义为如下：

![](jenkinsfile-pipeline.png)



> 根据实际情况填写代码库地址、分支以及凭据；



## 运行流水线

直接点击构建，会经过5个构建步骤，当运行到部署这一步的时候，需要选择一个部署环境：

![](deploy-jenkins.png)

> 当然这里的环境选择是个假的，选任何环境都可以，实际是需要在jenkinsfile中具体设置的



最后运行成功了：

![](success.png)



在集群中，容器也已经启动了（默认是和jenkins在一个namespace下）

<img src="run-demo.png" style="zoom:67%;" />




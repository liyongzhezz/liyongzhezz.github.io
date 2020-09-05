---
title: '[k8s实践系列]Python调用kubernetes API管理Job资源'
date: 2020-09-05 16:40:38
tags:
- kubernetes api
categories:
- 实践K8s
- kubernetes api
description: 通过python调用kubernetes api管理Job资源。
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1599305444011&di=96b260e2ed93814697457e34e01387d5&imgtype=0&src=http%3A%2F%2Fimg2.ctoutiao.com%2Fuploads%2F2016%2F10%2F24%2F1477271707687844.jpg
---



**环境基于python3**

---





# 安装kubernetes sdk

直接使用pip安装kubernetes sdk即可：

```bash
$ pip install kubernetes
```



<br>



# 初始化

```python
from kubernetes.client import BatchV1Api
from kubernetes.config import load_kube_config

load_kube_config()
batch = BatchV1Api()
```



- `load_kube_config`是从`~/.kube/config`加载配置。若使用其他位置的文件，则通过第一个参数传入其路径。
- `BatchV1Api()`可以当做Job的客户端来用。命名上，Batch和Job是类似的概念，前者强调批量。



<br>





# 通过对象创建job

实例代码：

```python
def create_job_object():
    container = client.V1Container(
        name="pi",
        image="perl",
        command=["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"])
    
    template = client.V1PodTemplateSpec(
        metadata=client.V1ObjectMeta(labels={"app": "pi"}),
        spec=client.V1PodSpec(restart_policy="Never", containers=[container]))
    
    spec = client.V1JobSpec(
        template=template,
        backoff_limit=4)
    
    job = client.V1Job(
        api_version="batch/v1",
        kind="Job",
        metadata=client.V1ObjectMeta(name=JOB_NAME),
        spec=spec)

    return job


def create_job(api_instance, job):
    api_response = api_instance.create_namespaced_job(
        body=job,
        namespace="default")
    print("Job created. status='%s'" % str(api_response.status))
```

<br>



# 通过yaml创建job

示例yaml：

```yaml
# job.yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - name: echo
          image: alpine:3.11
          args:
            - 'echo'
            - 'Hello world!'
```



python读取上边的yaml文件进行创建：

```python
from kubernetes.client import V1Job
import yaml

with open('job.yaml') as file:
    cfg = yaml.safe_load(file)
job = batch.create_namespaced_job(namespace='default', body=cfg)
assert isinstance(job, V1Job)
```

- 这里返回的`V1Job`只是创建时的状态，但是会包含更多集群中的信息。



`create_namespaced_job`同样接受字典作为body输入，例如：

```python
cfg = {
    'apiVersion': 'batch/v1',
    'kind': 'Job',
    'metadata': {
        'name': 'hello'
    },
    'spec': {
        'template': {
            'spec': {
                'restartPolicy':
                'Never',
                'containers': [{
                    'name': 'upload',
                    'image': 'alpine:3.11',
                    'args': ['echo', 'Hello world!']
                }]
            }
        }
    }
}
batch.create_namespaced_job(namespace='default', body=cfg)
```



`dict`结构与YAML相同，而又没有类的束缚，所以也很灵活方便。

此外，从YAML读出为`dict`后，也可以通过修改部分字段，达到动态变化的效果。这种结合YAML和dict的使用方式，是对官方用法的最佳替代。

<br>



# 监控job运行

在创建Job后，通常需要监控Job的运行，做一些外围处理。Kubernetes提供了一个`Watch`机制，通过接收Event，实现对状态变化的掌控。Event只有在状态变化时才会有，所以是非常理想的回调。

```python
from kubernetes.client import V1Job
from kubernetes.watch import Watch

job_name = 'hello'
watcher = Watch()
for event in watcher.stream(
        batch.list_namespaced_job,
        namespace='default',
        label_selector=f'job-name={job_name}',
):  
    assert isinstance(event, dict)
    job = event['object']
    assert isinstance(job, V1Job)
```



`Watch().stream`就是前面说的理想回调，它第一个参数是列出类的函数，这里选择`list_namespaced_job`。后面的参数，都是`list_namespaced_job`的参数。除了必备的namespace以外，`label_selector也`是一个常用参数，可以避免关注无关的Job。



每个Job在创建后，都会自动带一个`f'job-name={job_name}'`的Label，可以借此筛选。`job_name`就是`metadata`里设置的`name`，如这里`job-name=hello`。



event是一个dict，只有三个值。其中`event['raw_object']`只是`event['object']`的`dict`形式，没有太大意义。`event['type']`常见三个值，对应增删改。

- `ADDED`，创建时的信息，和`create_namespaced_job`的返回值通常没有区别。
- `MODIFIED`，Job状态变化时的信息。
- `DELETED`，Job删除时的信息。

以上三个状态值，对其它类型的资源也是通用的，比如Pod、Deployment等。



<br>



# 列出Job

列出所有Job的`list_job_for_all_namespaces`不常用，一般只列出指定Namespace的Job。

```python
from kubernetes.client import V1JobList, V1Job

job_list = batch.list_namespaced_job(namespace='default')
assert isinstance(job_list, V1JobList)

assert isinstance(job_list.items, list)
for job in job_list.items:
    assert isinstance(job, V1Job)
```

与监控的示例相比，这里去掉了`label_selector`，可以获取Namespace中所有的Job。如果有需要，可以通过自定义Label把所有Job分类，并使用l`abel_selector`获取指定类型的Job。

<br>





# 读取Job

如果知道Job的`name`，可以直接通过`read_*`系列接口，获得指定的`V1Job`。

```python
from kubernetes.client import V1Job

job = batch.read_namespaced_job(name='hello', namespace='default')
assert isinstance(job, V1Job)
```

如果更看重状态，可以改用`read_namespaced_job_status`。虽然访问的API不同，但在Python的`V1Job`这个结果层面，没有本质差异。

<br>





# 列出一个Job的Pod

Pod是Kubernetes调度的最小单元，也是最常用的一种资源。

```python
from typing import List

from kubernetes.client import CoreV1Api, V1Pod


def get_pods_by(job_name: str) -> List[V1Pod]:
    core = CoreV1Api()
    pods = core.list_namespaced_pod(
        namespace='default',
        label_selector=f'job-name={job_name}',
        limit=1,
    )
    return pods.items
```

这里的`get_pods_by`，可以用`job_name`获取对应的Pod。`limit=1`是在已知Pod只有一个的情况下做出的优化，可按需调整或去掉。

<br>



# 删除Job

删除一个Job：

```python
from kubernetes.client import V1Status

status = batch.delete_namespaced_job(
    namespace='default',
    name=job_name,
    propagation_policy='Background',
)
assert isinstance(status, V1Status)
```

其中，`propagation_policy='Background'`是不可省略的关键，否则默认是`Orphan`，其Pod不会被删除。这属于API设计的一个失误，与`kubectl`的默认行为不符合。而且，应该没有人在删除了Job之后，还要保留Pod的吧。这里也可以选择`'Foreground'`，阻塞等待相关资源的删除完毕。



删除多个、或所有Job：

```python
status = batch.delete_collection_namespaced_job(
    namespace='default',
    propagation_policy='Background',
    label_selector='some-label=your-value',
)
assert isinstance(status, V1Status)
```

如果没有`label_selector`，那就是删除一个`Namespace`中的所有Job。

<br>



# 更新Job

这个比较少用，因为一般都是建新的。用法其实和`create_namespaced_job`差不多：

```python
def update_job(api_instance, job):
    job.spec.template.spec.containers[0].image = "perl"
    api_response = api_instance.patch_namespaced_job(
        name=JOB_NAME,
        namespace="default",
        body=job)
    print("Job updated. status='%s'" % str(api_response.status))
```


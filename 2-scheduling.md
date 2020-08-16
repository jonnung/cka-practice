## Scheduling
기본적으로 POD는 NodeName 필드를 갖고 있지만 세팅된 상태는 아니다.  
보통 POD 매니페이스트에 명시하지 않고, 쿠버네티스에서 자동으로 추가한다.  
쿠버네티스는 POD의 NodeName 속성을 직접 변경하는 것을 허용하지 않으며, Binding 오브젝트를 API 호출을 통해 생성해서 스케줄링을 모방할 수 있다.  

참고: [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)


### kube-scheduler 가 실행되고 있지 않을 때 POD를 직접 스케줄링 하기
- kube-scheduler가 실행되고 있지 않으면, POD의 `nodeSelector`를 쓸 수 없다. `nodeName` 속성으로 직접 노드 이름을 지정한다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

## labels & annotations
labels과 selector는 오브젝트를 선택하고 그룹화 하기 위해 사용된다.  
annotations는 상세 정보를 표현하는 목적으로 사용된다.  

▶︎ `env`라벨이 `prod`인 모든 오브젝트 찾기
```shell
$ kubectl get all -lenv=prod
```

▶︎ 여러 라벨로 POD 찾기
```shell
$ kubectl get po -lenv=prod,bu=finance,tier=frontend
```

## Taint & Toleration
Taints와 Toleration은 POD가 스케줄링 되는 것을 제한하는 목적으로 사용한다.  
Taints는 노드에 설정하고, Toleration은 POD에 설정한다.  

```shell
kubectl taint nodes node-name key=value:taint-effect

# Example
kubectl taint nodes node1 app=blue:NoSchedule
```

- Taint Effect : 이 Taint를 Tolerate하지 않는 POD에는 어떤 일이 벌어지는가?
	- `NoSchedule` : POD는 이 노드에 스케줄링 되지 않을 것이다. 
	- `PreferNoSchedule` :  시스템은 이 노드에 POD를 스케줄링을 하지 않으려고 노력하지만 반드시 보장하는 것은 아니다. 
	- `NoExecute` : 이 노드에 POD를 스케줄링하지 않으며, 이미 실행중인 POD가 있는 경우 Evict 된다. 

Taints와 Toleration은 POD가 어떤 노드로 스케줄링 될 지 결정하기 위해 사용하는 게 아니다.  
노드가 자신에게 부여된 Taints를 Toleration하는 POD만을 허용하는 것이다.  

참고: [테인트(Taints)와 톨러레이션(Tolerations) | Kubernetes](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/taint-and-toleration/)  

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

## Node Affinity
Node Affinity 기능의 목적은 POD를 특정 노드에 배치를 제한 하기 위함이다.  

- [노드에 파드 할당하기 | Kubernetes](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/assign-pod-node/#%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0-affinity-%EC%99%80-%EC%95%88%ED%8B%B0-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0-anti-affinity)
- [Advanced Scheduling and Node Affinity - Scheduling | Cluster Administration | OpenShift Container Platform 3.6](https://docs.openshift.com/container-platform/3.6/admin_guide/scheduling/node_affinity.html)


## Taint & Tolerations vs Node Affinity
Taint와 Toleration은 POD가 어떤 노드만 선호한다고 보장하지는 않는다.  

노드에 label을 지정하고, POD에 nodeSelector를 이용해서 해당 노드에 스케줄링 되도록 할 수 있다.
하지만 이 노드에는 다른 POD도 스케줄링 될 수 있다.

Taint와Toleration 그리고 NodeAffinity를 함께 사용하면 특정 POD만 스케줄링되는 전용 노드를 만들 수 있다.  

참고: [테인트(Taints)와 톨러레이션(Tolerations) | Kubernetes](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/taint-and-toleration/)

동일한 노드에 여러 Taint를 지정하거나 동일한 POD에 여러 Toleration을 지정할 수 있다.  
쿠버네티스가 여러 Taint 및 Toleration을 처리하는 방식은 필터와 같다.  
모든 노드의 Taint로 시작한 다음, POD에 일치하는 Toleration이 있는 것만 무시한다. 무시되지 않은 나머지 Taint는 POD에 표시된 effect를 적용한다. 


## Resource Limit & Requirements
모든 POD는 사용할 수 있는 리소스를 설정할 수 있다.  
쿠버네티스 스케줄러는 POD를 어떤 노드에 배치할 지 결정한다.  
스케줄러는 이 판단을 위해 POD가 필요로 하는 리소스의 양과 사용 가능한 노드를 파악한다.  
만약 충분한 리소스가 없다면 스케줄러는 그 노드에 POD를 배치하지 않으며, 충분한 리소스를 가진 노드에 배포한다.  
충분한 리소스를 확보한 노드가 없다면 스케줄러는 POD 배포를 중지하고 해당 POD는 PENDING 상태가 된다.  
그 POD의 상태를 확인하면 PENDING의 이유로 `insufficient cpu`라고 표시되는 것을 확인할 수 있다.   

CPU를 0.1로 표기하는 것은 Milli Core 단위를 의미하며 100m와 같은 의미를 가진다.  
최소 1m까지 지정할 수 있고, 1이라고 명시한 경우 1 vCPU를 의미한다.  
메모리도 유사하여 256 메가바이트는 Mi 라는 단위를 사용한다.  
기가 바이트는 G 또는 Gi로 표시할 수 있는데 1G의 경우 1000 메가 바이트를 의미하고, 1Gi는 1024 메가 바이트를 의미한다.  

POD가 사용할 수 있는 최대 리소스를 제한할 수 있다.  
쿠버네티스는 컨테이너가 제한된 CPU 리소스 이상을 넘어 사용될 수 없게 한다.
하지만 메모리의 경우 컨테이너가 제한된 리소스보다 더 사용하게 될 수 있다.  
이 경우 쿠버네티스는 해당 컨테이너를 종료해버린다. 


## DaemonSet
Daemonset은 Replicasets과 유사하게 여러 POD를 배포할 수 있게 해준다.  
하지만 POD를 클러스터의 각 노드마다 1개씩 유지될 수 있도록 한다는 점이 다르다.  
새로운 노드가 추가된다면 자동으로 Daemonsets의 리플리카가 생성된다.  
그리고 노드가 삭제되면 해당 노드의 Daemonsets POD도 삭제한다.  

쿠버네티스 아키텍처 상에 모든 워커 노드에 배포되어 있는 kube-proxy가 DaemonSet의 좋은 사례라고 할 수 있다.  

쿠버네티스 v1.12까지는 DaemonSet의 POD에는 `nodeName` 속성에 각 노드가 명시 되었지만 그 이후 버전부터는 NodeAffinity를 사용하고 있다.   


## Multiple Scheduler 
쿠버네티스는 확장성이 뛰어나다. 사용자가 직접 스케줄러 프로그램을 만들어서 기본 스케줄러와 함께 실행되도록 할 수 있다.  
kubeadm으로 클러스터를 구성했다면 `/etc/kubenetes/manifests/` 폴더에 Static POD로 실행될 kube-scheduler 매니페이스가 존재한다.  

가장 쉽게 커스텀 스케줄러를 만들기 위한 방법은 기존 kube-scheduler Static POD 매니페스트를 복사해서 POD의 name과 kube-scheduler 커맨드 옵션을 추가하는 방법이다.  

여러 마스터 노드에서 각 실행되는 스케줄러에 리더 선정을 위해 `--leader-elect` 옵션을 사용할 수 있다.  
이 옵션을 사용해서 여러 마스터 노드에 동일한 스케줄러가 실행 중일 때 반드시 1개만 리더로 활성화되어야 한다.  
그리고 `--lock-object-name` 옵션은 리더 선택 프로세스 중에 새로운 커스텀 스케줄러를 기본값과 구별하기 위해 사용된다.  

커스텀 스케줄러는 POD나 Deployment로 정의할 수 있다.  
그리고 만약 어떤 POD에서 특정 스케줄러만을 지정하려면 POD spec에 `schedulerName` 속성을 이용하면 된다.  

[https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)


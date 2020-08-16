## Rollout & Rollback
새로운 롤아웃은 새로운 Deployment의 수정이 생기면 revison 2라는 형태로 생성된다.  
이것은 변경을 추적할 수 있도록 도와주고 필요할 때 Deployment의 이전 버전으로 롤백할 때 유용하다.  
```shell
$ kubectl rollout status deployment/sample

$ kubectl rollout history deployment/sample
```
Deployment의 전략은 한번에 모든 것을 제거하는 것이 아니라 이전 버전을 하나씩 삭제하면서 새로운 버전을 하나씩 교체하는 방식으로 동작한다.  
이 방식은 애플리케이션이 절대 사용불가한 상태가 되지 않고 매끄럽게 업그레이드 되도록 한다.  

우리가 Deployment를 정의하기 위해 만든 매니페스트 파일에 필요한 수정을 하고 `kubectl apply` 명령으로 적용하면 새로운 버전의 롤아웃이 생성된다.  

- create
	- `kubectl create -f deployment-definition.yaml`
- get
	- `kubectl get deployments`
- update
	- `kubectl apply -f deployment-definition.yaml`
	- `kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1`
- status check
	- `kubectl rollout status deployment/myapp-deployment`
	- `kubectl rollout history deployment/myapp-deployment`
- rollback
	- `kubectl rollout undo deployment/myapp-deployment`


## Application Commands & Arguments
EntryPont는 컨테이너가 시작될 때 실행될 프로그램 명령어와 명령 옵션을 지정하는 Commmand와 유사하다.   

POD를 생성할 때, POD 안에서 동작하는 컨테이너를 위한 command와 args를 정의할 수 있다.   
command를 정의하기 위해서는, POD 안에서 실행되는 컨테이너에 `command` 필드를 포함시킨다.   
command에 대한 인자를 정의하기 위해서는, 구성 파일에 `args` 필드를 포함시킨다.   
정의한 command와 args는 POD가 생성되고 난 이후에는 변경될 수 없다.

매니페스트 파일 안에서 정의하는 command와 args는 컨테이너 도커 이미지가 제공하는 기본 커맨드와 인자들보다 우선시 된다.  
만약 `args`를 정의하고 `command`를 정의하지 않는다면, 기본 command가 새로운 `args`와 함께 사용된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
  restartPolicy: OnFailure
```

컨테이너의 기본 Entrypoint와 Cmd 값을 덮어쓰려고 한다면, 아래의 규칙들이 적용된다.
- 만약 컨테이너를 위한 `command` 값이나 `args` 값을 제공하지 않는다면, 도커 이미지 안에 제공되는 기본 값들이 사용된다.
- 만약 컨테이너를 위한 `command` 값을 제공하고, `args` 값을 제공하지 않는다면, 제공된 `command` 값만이 사용된다. 도커 이미지 안에 정의된 기본 EntryPoint 값과 기본 Cmd 값은 덮어쓰여진다.
- 만약 컨테이너를 위한 `args` 값만 제공한다면, 도커 이미지 안에 정의된 기본 EntryPoint 값이 정의한 args 값들과 함께 실행된다.
- `command` 값과 `args` 값을 동시에 정의한다면, 도커 이미지 안에 정의된 기본 EntryPoint 값과 기본 Cmd 값이 덮어쓰여진다. 

[https://kubernetes.io/ko/docs/tasks/inject-data-application/define-command-argument-container/](https://kubernetes.io/ko/docs/tasks/inject-data-application/define-command-argument-container/)


## Environment Variables
환경 변수를 선언하려면 `ENV` 속성을 사용한다. `ENV` 속성은 배열 타입이다.  
`ENV` 하위 값은 Name과 Value 속성으로 구성된다.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
```

[https://kubernetes.io/ko/docs/tasks/inject-data-application/define-environment-variable-container/](https://kubernetes.io/ko/docs/tasks/inject-data-application/define-environment-variable-container/)

환경 변수를 선언하는 다른 방법은 ConfigMaps과 Secrets을 사용는 것이다.  
이 방식을 사용할 때는 Value를 명시하는 대신 `valueFrom`을 사용해서 ConfigMaps과Secret을 선택한다.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
              name: special-config
              # Specify the key associated with the value
              key: special.how
  restartPolicy: Never
```
[https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data)


## Configuring ConfigMaps
ConfigMaps는 쿠버네티스 상에 설정 관련 데이터를 Key-Value 형태로 저장할 때 사용하는 오브젝트이다.  

```shell
$ kubectl create configmap <configmap-name> --from-literal=<key>=<value> \
                                            --from-literal=<key>=<value>
```

매니페스트 파일을 통해 생성하는 방식도 가능하다.  

```yaml
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

[https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap)


##  Secrets
ConfigMaps은 설정 관련 데이터를 일반적인 텍스트 포맷을 저장한다.  
하지만 Secrets은 비밀스러운 데이터(?)나 비밀번호, 비밀키 같은 민감한 데이터를 저장하기 위해 사용한다.  
근본적으로 Secrets은 Configmaps와 같지만 데이터를 BASE64로 인코딩해서 저장한다는 점에서 차이가 있다.  

```
kubectl create sercret generic <secret-name> --from-literal=<key>=<value>
```

Secrets을 매니페스트 파일로 정의해서 생성할 때는 반드시 데이터를 BASE64로 해시한 뒤 입력해야 한다.  
Secrets을 컨테이너의 환경 변수로 선언하려면 `envFrom` 속성을 사용할 수 있다.  
`envFrom` 속성은 Secrets에 저장된 여러 데이터를 한번에 환경 변수로 전달할 때 유용하다.   

```yaml
envFrom: 
  - secretRef:
      name: app-config
```

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD
```

```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
```

## Multiple containers
POD는 여러 컨테이너를 포함해서 함께 생성되고 같은 생명 주기를 갖는다.  
삭제될 때도 함께 삭제되고, 같은 네트워크와 같은 스토리지 볼륨에 접근할 수 있다.  


## InitContainers
POD의 컨테이너가 실행되기 전에 어떤 선행 과정을 수행해야 할 때가 있다.  
예를 들어 원격 저장소로부터 소스코드나 바이너리 파일을 내려받아 메인 애플리케이션에서 사용할 수 있도록 한다거나, POD가 생성될 때 반드시 1번은 실행되어야 하는 작업이 있을 수 있다.  
이러한 작업을 위해 사용하는 것이 InitContainers 이다.  

POD에 InitContainers를 설정하는 방법은 일반적인 컨테이너 선언하는 방식과 유사하지만 `initContainers` 섹션이 따로 존재한다.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```

POD가 처음 생성될 때 initContainers가 먼저 실행되며, initContainers 실행이 완전히 종료되어야 실제 컨테이너가 실행될 수 있다.  
initContainers도 여러개 설정할 수 있다. 이 때는 InitContainers를 선언한 순서대로 1번씩 실행된다.  
만약 initContainers 중에 어떤 한 개라도 실패하면 쿠버네티스는 해당 POD를 재실행하여 InitContainers 수행이 완전히 성공할 때까지 반복하게 된다.  


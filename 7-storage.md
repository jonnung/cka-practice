## Docker Storage
도커는 Storage drivers와 Volume drivers라는 2가지 컨셉을 갖고 있다.

##### Storage drivers & File system
도커 컨테이너를 실행할 때 호스트 영역과 볼륨을 마운트 할 수 있는 방법은 2가지다.   

1. Volume 마운트
	- `docker volume create data_volume` 명령어를 이용해 새로운 볼륨을 만든다.
	- 이 때 만들어진 볼륨은 `/var/lib/docker/volumes/data_volume` 폴더에 생성된다.
	- 도커 컨테이너에 볼륨을 마운트 하려면 다음과 같은 명령어를 실행한다. 
```shell
docker run -v data_volume:/var/lib/mysql mysql
```
	- 볼륨을 미리 만들지 않고, `docker run`명령어를 실행할 때 볼륨명(아직 없는)을 지정하면 자동으로 볼륨을 생성해준다. 

2. Bind 마운트
	- 호스트에 이미 존재하는 폴더 경로를 도커 컨터이너 내부로 마운트할 때 사용한다. 
```shell
docker run -v /data/mysql:/var/lib/mysql mysql
```

`-v` 옵션으로 볼륨을 마운트하는 방식은 예전 방식이고, 최신 도커 엔진은 `--mount` 옵션을 사용하는 것을 권장한다. 이 옵션을 사용하면 여러 마운트 옵션을 한번에 지정할 수 있다.   

이렇게 도커 컨테이너에 볼륨을 마운트할 수 있도록 해주는 역할을 Storage drivers가 한다.   
Storage drivers 종류로는 `AUFS`, `ZFS`, `BTRFS`, `Device Mapper`, `Overlay`, `Overlay2`가 있다.   

##### Docker Volume
Volume drivers는 볼륨을 컨트롤한다.   
기본적인 볼륨 플러그인은 Local volume plugin이다. 이 플러그인은 도커 호스트의 `/var/lib/docker/volume` 경로에 데이터를 저장하도록 한다.   
다양한 서드파티 Volume drivers가 존재한다.   
예) Azuer file storage | Convoy | DigitalOcean Block Storage | Flocker | gce-docker | GlusterFS | NetApp | RexRay | Portworx | VMware vSphere Storage  

### Container Storage Interface
쿠버네티스에서 사용하는 컨테이너 런타임 연동을 위해 CRI를 제공하고, CRI를 이용해서 구현된 다양한 컨테이너 런타임이 쿠버네티스와 연동될 수 있다. (예: Docker, rkt, cri-o)  
네트워크쪽도 CNI 라는 인터페이스를 통해 다양한 네트워크 플러그인과 연동할 수 있고, 스토리지 관련된 CSI를 통해 여러 스토리지 솔루션을 지원한다.  
쿠버네티스는 POD를 생성할 때 볼륨이 필요한 경우 CSI의 RPC 호출을 통해 볼륨을 요청하고, 반환된 정보를 통해 POD에 볼륨을 마운트하는 방식이다.  

### Volumes in kubernetes
POD 정의에 `volumes`를 정의하여 어떤 타입의 볼륨을 사용할지 정할 수 있다.   
만약 싱글 쿠버네티스 클러스터라면 호스트의 Directory type을 사용할 수도 있지만 멀티 클러스터인 경우 사용하지 않는 것이 좋다.   
그 이유는 POD가 스케줄링 되면서 서로 다른 노드에 배치되는 경우 마운트 될 경로가 매번 달라질 수 있기 때문이다.  
쿠버네티스는 다양한 스토리지 솔루션을 지원한다.   
예) NFS, GlusterFS, Flocker, CephFS, ScaleIO, AWS EBS, Azure Disk, Google Persistent disk  

### Persistent Volumes
Persistent Volume(PV)은 관리자가 클러스터 레벨의 스토리지 볼륨을 제공하기 위해 생성하는 오브젝트이다.   
쿠버네티스에서 실행되는 애플리케이션(사용자)은 Persistent Volume Claim(PVC)을 이용해서 Persistent Volume의 스토리지를 이용할 수 있다.   

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

### Persistent Volume Claim
Persistent Volume Claim(PVC)을 만들면 쿠버네티스는 `request`와 다양한 프로퍼티로 Volume을 지정한다.   
모든 PVC는 단일 PV에 연결되며, 쿠버네티스는 PV에 바인딩 시키기 위해 요청된 Capacity와 Access mode, Volume mode, Storage class 같은 다른 프로퍼티를 활용한다.   
하지만 만약 하나의 Claim에 여러 개가 매칭될 경우 Labels과 Selector를 통해 특정 볼륨을 지정할 수 있다.   
모든 기준이 매칭 된다면 작은 Claim이 큰 볼륨에 연결될 수도 있다.   
Claim과 볼륨은 1:1 관계를 갖고 볼륨의 남은 공간은 다른 Claim에 연결될 수 있다.  
만약 PVC가 연결될 볼륨이 없는 경우 `pending` 상태가 되며, 새로운 볼륨이 생성되면 사용 가능 상태가 된다.   

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessMode:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

위 PVC에서 요청한 용량은 500MB이다. 하지만 매칭된 볼륨에는 1GB가 설정되어 있다. 다른 사용 가능한 볼륨이 없기 때문에 이 볼륨이 연결된다.  

```shell
kubectl get persistentvolumeclaim
```

PVC를 삭제하기 하게 될 때 연결 되었던 볼륨을 어떻게 다룰 것인지에 대한 설정을 할 수 있다.   
이 값은 `persistentVolumeReclaimPolicy` 으로 설정할 수 있으며 기본 값은 `Retain`이다.   
`Retain`은 관리자가 직접 삭제하기 전까지 PV가 그대로 남아있는 설정이다.  
또 다른 옵션으로 `Delete`는 다른 Claim에서 재사용할 수 없게 하거나 자동으로 삭제되게 할 수 있다. 이 PVC가 삭제되는 경우 PV도 자동으로 삭제된다.   
마지막으로 `Recycle` 옵션은 다른 PVC에서 사용되기 전에 볼륨안에 데이터가 삭제된다.   

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

> volumeMounts를 사용하여 "볼륨"강의에서 볼륨을 사용하도록 애플리케이션을 구성하는 방법에 대해 설명했습니다. 이것은 연습 시험과 함께 시험에 충분히 해야 한다.  
> 스토리지 클래스 및 StatefulSet에 대한 추가 주제는 시험 범위를 벗어난다.   


## 볼륨이란?
---
**쿠버네티스 볼륨 = 파드 내 컨테이너들이 데이터를 저장하고 공유할 수 있는 디렉토리**

쿠버네티스 볼륨은 파드 내 컨테이너들이 데이터를 저장하고 공유할 수 있는 디렉토리로, 컨테이너에 디스크 스토리지를 연결하여 데이터를 관리하는 핵심 구성 요소입니다. 파드가 재시작되거나 다른 노드로 재스케줄링되어도 데이터를 유지하며, 기반 스토리지 기술과 애플리케이션 개발자를 분리하는 중요한 역할을 수행합니다.

### 파드의 구성 요소

볼륨은 **파드 스펙에서 컨테이너와 동일하게 정의**됩니다.

```yaml
apiVersion: v1
kind: Pod
spec:
  containers: [...]      # 컨테이너들 정의
  volumes: [...]         # 볼륨들 정의 (같은 레벨!)
```

### 독립적인 오브젝트가 아님

볼륨은 **자체적으로 생성하거나 삭제될 수 없습니다**. 자동차의 엔진처럼, 자동차 없이는 독립적으로 존재할 수 없는 것과 같습니다.

```bash
# ❌ 이런 명령어는 존재하지 않음
kubectl create volume my-volume  # 불가능!

# ✅ 항상 파드의 일부로만 존재
kubectl apply -f pod-with-volume.yaml
```

### 각 컨테이너마다 마운트 필요

볼륨은 **파드 내 모든 컨테이너에서 사용 가능**하지만, 각 컨테이너가 접근하려면 **각각 마운트**되어야 합니다.

```yaml
spec:
  containers:
  - name: container1
    volumeMounts:
    - name: shared-vol
      mountPath: /data      # 이 경로로 접근
  - name: container2
    volumeMounts:
    - name: shared-vol
      mountPath: /shared    # 다른 경로로도 접근 가능
  volumes:
  - name: shared-vol
    emptyDir: {}
```

### Docker와의 차이점

|특성|Docker|Kubernetes|
|---|---|---|
|**관리 단위**|컨테이너별|파드별 (여러 컨테이너 공유)|
|**독립성**|볼륨 독립 생성/삭제 가능|파드에 종속적|
|**복잡성**|상대적으로 단순|더 복잡하지만 기능 풍부|

기본 개념은 비슷하지만, Kubernetes는 **여러 컨테이너를 하나의 파드로 묶어서 관리**하는 점이 가장 큰 차이점입니다.

<br>

## 볼륨의 생명주기
---
### 일반적인 라이프사이클

```
파드 생성 → 볼륨 생성 → 데이터 저장 → 파드 삭제 → 볼륨 삭제
```

### 영구 볼륨의 예외

**`hostPath`나 퍼시스턴트 스토리지는 파드와 독립적으로 데이터를 유지**할 수 있습니다.

|볼륨 타입|파드 삭제|노드 재부팅|노드 교체|클러스터 삭제|
|---|---|---|---|---|
|**emptyDir**|❌ 삭제|❌ 삭제|❌ 삭제|❌ 삭제|
|**hostPath**|✅ 유지|✅ 유지|❌ 삭제|❌ 삭제|
|**NFS**|✅ 유지|✅ 유지|✅ 유지|NFS 서버에 따라|
|**클라우드**|✅ 유지|✅ 유지|✅ 유지|정책에 따라|

<br>

## 내부 vs 외부 저장소
---
### 내부 저장소 (쿠버네티스 종속적)

```yaml
# 쿠버네티스와 운명을 함께하는 저장소
volumes:
- name: temp
  emptyDir: {}           # 노드 임시 공간
- name: host-data
  hostPath:
    path: /mnt/data      # 노드 로컬 파일시스템
```

### 외부 저장소 (독립적 존재)

**"외부 저장소" = 쿠버네티스 밖에 독립적으로 존재하는 저장소**

```yaml
volumes:
- name: nfs-vol
  nfs:
    server: 192.168.1.100   # 독립적 NFS 서버
    path: /exports/data
- name: cloud-disk
  persistentVolumeClaim:
    claimName: aws-ebs-pvc  # AWS EBS 등
```

<br>

## 주요 볼륨 타입들
---
### ✓ emptyDir - 임시 공유 공간

가장 간단한 볼륨 유형으로 빈 디렉터리로 시작합니다. 파드에 실행 중인 애플리케이션은 어떤 파일이든 볼륨에 쓸 수 있으며, 볼륨의 라이프사이클이 파드에 묶여 있으므로 파드가 삭제되면 콘텐츠는 사라집니다.

#### 기본 사용법

```yaml
volumes:
- name: shared-data
  emptyDir: {}
```

#### 메모리 기반 (고성능)

기본적으로 워커 노드의 실제 디스크에 생성되지만, `medium: Memory`를 지정하여 메모리(RAM)를 사용하는 `tmpfs` 파일 시스템으로 생성할 수도 있습니다. 이 경우 I/O 속도는 빠르지만 RAM 용량을 많이 사용하게 되어 대용량 데이터에는 적합하지 않습니다.

```yaml
volumes:
- name: cache-volume
  emptyDir:
    medium: Memory    # RAM을 사용하는 tmpfs
    sizeLimit: 1Gi
```

**사용 사례:**

- 임시 캐시 데이터 저장
- 컨테이너 간 파일 공유
- 스크래치 공간으로 활용
- 체크포인트 파일 저장

### ✓ hostPath - 노드 파일시스템 접근

워커 노드의 파일 시스템을 파드의 디렉터리로 마운트하는 데 사용됩니다. 파드가 종료되어도 콘텐츠가 삭제되지 않으며, 동일 노드에 실행 중인 파드가 `hostPath` 볼륨의 동일 경로를 사용하면 동일한 파일이 표시됩니다.

```yaml
volumes:
- name: host-logs
  hostPath:
    path: /var/log
    type: Directory
```

**hostPath 타입 옵션:**

- `DirectoryOrCreate`: 디렉터리가 없으면 생성
- `Directory`: 기존 디렉터리만 사용
- `FileOrCreate`: 파일이 없으면 생성
- `File`: 기존 파일만 사용
- `Socket`: Unix 소켓
- `CharDevice`, `BlockDevice`: 디바이스 파일

**사용 사례:**

- 노드의 시스템 파일 접근 (로그, 설정)
- DaemonSet에서 노드별 데이터 수집
- 개발/테스트 환경

**주의사항:**

- 파드가 다른 노드로 이동하면 데이터 접근 불가
- 일반적인 애플리케이션 데이터 저장에는 부적합
- 자체 데이터 저장 목적으로 사용하지 않으며, 주로 특정 시스템 레벨의 파드에서 활용


### ✓ nfs - 네트워크 파일 시스템

NFS(Network File System) 공유를 파드에 마운트합니다. 클러스터가 여러 대의 서버로 실행되는 경우 외장 스토리지를 볼륨에 마운트하기 위한 지원 옵션 중 하나이며, 여러 파드가 동시에 읽기/쓰기 접근이 가능하지만 네트워크 기반이므로 성능이 로컬 스토리지보다 떨어질 수 있습니다.

```yaml
volumes:
- name: nfs-storage
  nfs:
    server: 192.168.1.100
    path: /exports/shared
```

**NFS의 특별한 점:**

- ✅ **ReadWriteMany 지원** (여러 Pod에서 동시 읽기/쓰기)
- ✅ **노드 독립적** (어느 노드에서든 접근 가능)
- ❌ **성능 제한** (네트워크 의존적)
- ❌ **단일 장애점** (NFS 서버 다운 시 모든 클라이언트 영향)

**사용 사례:**

- 여러 Pod에서 파일 공유
- 정적 웹 콘텐츠 서빙
- 로그 중앙 수집
- 공유 미디어 파일

### ✓ 클라우드 제공자 전용 스토리지

`gcePersistentDisk`, `awsElasticBlockStore`, `azureDisk` 등은 클라우드 제공자의 전용 스토리지를 마운트하는 데 사용됩니다. 이러한 볼륨은 파드를 삭제하고 재생성해도 데이터가 그대로 유지되는 영구 스토리지이지만, 클라우드 제공자 종속적이므로 이식성이 제한됩니다.

#### AWS EBS

```yaml
volumes:
- name: aws-ebs
  awsElasticBlockStore:
    volumeID: vol-12345678
    fsType: ext4
```

#### Google Persistent Disk

```yaml
volumes:
- name: gcp-pd
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
```

#### Azure Disk

```yaml
volumes:
- name: azure-disk
  azureDisk:
    diskName: myDisk
    diskURI: https://mystorageaccount.blob.core.windows.net/...
```

**클라우드 볼륨의 장점:**

- ✅ **고가용성** (자동 복제, 백업)
- ✅ **확장성** (용량 동적 조정)
- ✅ **노드 이동 가능** (Pod가 다른 노드로 이동해도 접근 가능)

### ✓ 특별한 유형의 볼륨

`configMap`, `secret`, `downwardAPI` 등은 쿠버네티스 리소스나 클러스터 정보를 파드에 노출하는 데 사용됩니다.

#### ConfigMap 볼륨

```yaml
volumes:
- name: config-volume
  configMap:
    name: my-config
    items:
    - key: config.yaml
      path: app-config.yaml
```

#### Secret 볼륨

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: my-secret
    defaultMode: 0400
```

#### DownwardAPI 볼륨

```yaml
volumes:
- name: podinfo
  downwardAPI:
    items:
    - path: "labels"
      fieldRef:
        fieldPath: metadata.labels
    - path: "annotations"
      fieldRef:
        fieldPath: metadata.annotations
```

<br>

## 기반 스토리지 기술과 파드 분리 (PV/PVC/StorageClass)
---
### 쿠버네티스 인프라스트럭처 추상화의 핵심

애플리케이션 개발자가 실제 네트워크 스토리지 인프라스트럭처에 대한 지식 없이도 스토리지를 요청할 수 있도록 **퍼시스턴트볼륨(PV)**, **퍼시스턴트볼륨클레임(PVC)**, **스토리지클래스(StorageClass)** 라는 새로운 리소스가 도입되었습니다. 이는 쿠버네티스의 인프라스트럭처 추상화 아이디어에 부합합니다.

**문제상황:** 개발자가 스토리지를 사용하려면 복잡한 인프라 지식이 필요했습니다.

```yaml
# ❌ 개발자가 알아야 했던 것들
volumes:
- name: storage
  awsElasticBlockStore:
    volumeID: vol-12345678        # AWS 볼륨 ID
    fsType: ext4                  # 파일시스템 타입
```

**해결책:** PV/PVC/StorageClass를 통한 **책임 분리**

#### 인프라 관리자의 역할

```yaml
# StorageClass 정의 (자동화 정책)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
reclaimPolicy: Retain
```

#### 애플리케이션 개발자의 역할

```yaml
# 단순한 스토리지 요청 (PVC)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-storage
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd    # 성능 등급만 지정!
```

### 퍼시스턴트볼륨 (PV) - 실제 스토리지

클러스터 관리자가 설정하는 스토리지 리소스입니다. 클러스터 관리자가 기반 스토리지를 설정하고 PV 리소스를 생성해 쿠버네티스 API 서버에 등록하며, PV 생성 시 용량(capacity), 지원 가능한 접근 모드(accessModes), 해제 정책(persistentVolumeReclaimPolicy) 등을 지정합니다. PV는 파드나 PVC와 달리 네임스페이스에 속하지 않는 클러스터 수준의 리소스입니다.

#### PV 설정 예시

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prod-db-pv
spec:
  capacity:
    storage: 500Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  awsElasticBlockStore:
    volumeID: vol-abcdef123456
    fsType: ext4
```

#### PV 상태
- **Available**: 사용 가능하며 PVC에 바인딩되지 않은 상태
- **Bound**: PVC에 바인딩된 상태
- **Released**: PVC가 삭제되었지만 리소스가 아직 회수되지 않은 상태
- **Failed**: 자동 회수에 실패한 상태
- **Pending**: 동적 프로비저닝 중인 상태

### Persistant Volume Claim (PVC) - 저장소 요청서

개발자가 스토리지를 요청하는 방법입니다. **PVC = 개발자가 쿠버네티스에게 보내는 "스토리지 주문서"**라고 생각하면 됩니다.

클러스터 사용자가 파드에 퍼시스턴트 스토리지를 사용해야 할 때, 최소 크기와 필요한 접근 모드를 명시한 PVC 매니페스트를 생성합니다. 사용자가 PVC를 쿠버네티스 API 서버에 게시하면, 쿠버네티스는 요청을 수용할 만큼 충분히 큰 용량과 적절한 접근 모드를 가진 적절한 PV를 찾아 클레임에 바인딩합니다.

#### 기본 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-storage
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  storageClassName: premium-ssd
```

#### Pod에서 PVC 사용

바인딩된 PVC는 파드 내부의 볼륨 중 하나로 사용될 수 있습니다. 파드는 볼륨에서 이름으로 PVC를 참조하며, PVC에 대한 클레임은 파드를 생성하는 것과 별개의 프로세스이므로, 파드가 재스케줄링되더라도 동일한 PVC가 사용 가능한 상태로 유지됩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /app/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: webapp-storage  # PVC 이름만 지정
```

#### PVC 상태

- **Pending**: 적절한 PV를 찾는 중 또는 생성 중
- **Bound**: PV에 성공적으로 바인딩된 상태
- **Lost**: 바인딩된 PV가 분실된 상태

### 접근 모드 (Access Modes)

PV 및 PVC에서 지정할 수 있는 접근 모드는 다음과 같습니다. 중요한 것은 이 모드들이 파드 수가 아닌 볼륨을 동시에 사용할 수 있는 워커 노드 수와 관련이 있다는 점입니다.

|모드|의미|지원 볼륨 예시|사용 사례|
|---|---|---|---|
|**ReadWriteOnce (RWO)**|하나의 노드만 읽기/쓰기|EBS, PD, hostPath|데이터베이스, 일반 앱|
|**ReadOnlyMany (ROX)**|여러 노드에서 읽기만|NFS, ConfigMap|정적 콘텐츠, 설정 공유|
|**ReadWriteMany (RWX)**|여러 노드에서 읽기/쓰기|NFS, CephFS, EFS|공유 파일시스템, 로그 수집|

### PV/PVC 바인딩 과정

#### 정적 바인딩 (사전 준비된 PV 사용)

```yaml
# 특정 PV에 바인딩 강제하기
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: specific-claim
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  storageClassName: ""    # 동적 프로비저닝 비활성화!
  selector:               # 원하는 PV 조건 지정
    matchLabels:
      special-pv: "true"
```

#### 동적 바인딩 (StorageClass 사용)

```yaml
# StorageClass로 자동 PV 생성
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd  # 자동으로 PV 생성
```

**선택 기준:**

- selector 없는 정적 바인딩: 쿠버네티스가 조건에 맞는 첫 번째 PV 자동 선택 (이름순 정렬)
- selector 있는 정적 바인딩: 추가 조건으로 원하는 PV를 정확히 지정
- 동적 바인딩: StorageClass 정책에 따라 새 PV 자동 생성

### PVC가 볼륨과 파드 간의 "링크"

**PVC = 볼륨(PV)과 파드 간의 "계약서" 또는 "중개자"**

```
Pod ←→ PVC ←→ PV ←→ 실제 스토리지
```

#### 호텔 예약 시스템으로 이해하기

```
🏨 쿠버네티스 클러스터
├─ 🏠 실제 객실들 (PersistentVolumes)
│   ├─ 디럭스룸 #101 (AWS EBS PV)
│   ├─ 스탠다드룸 #102 (NFS PV)  
│   └─ 스위트룸 #201 (Local PV)
│
└─ 📝 객실 예약서 (PersistentVolumeClaims)  
    └─ "비즈니스급 룸 1개 주세요" → 적절한 객실 자동 배정
```

### PV 해제 정책 (PersistentVolumeReclaimPolicy)

PV가 해제될 때 어떤 동작을 해야 할지 쿠버네티스에게 알려줍니다:

- **`Retain` (유지)**: 클레임이 해제되어도 볼륨과 콘텐츠를 유지합니다. 클러스터 관리자가 볼륨을 완전히 비우지 않으면 새로운 클레임에 바인딩할 수 없으며, 수동으로 PV 리소스를 삭제하고 다시 생성해야 재사용이 가능합니다.
- **`Recycle` (재활용)**: 볼륨의 콘텐츠를 삭제하고 볼륨이 다시 클레임될 수 있도록 사용 가능하게 만듭니다. 즉, 여러 번 재사용이 가능합니다.
- **`Delete` (삭제)**: PVC 삭제 시 PV와 기반 스토리지를 모두 삭제합니다.

**주의사항:** `Recycle` 옵션은 GCE 퍼시스턴트 디스크에서 지원되지 않는 등, 볼륨 유형마다 지원 여부가 다를 수 있으므로 정책 확인이 필요합니다.

<br>

## PV의 동적 프로비저닝 (StorageClass)
---
### StorageClass - 스토리지 자동화의 핵심

클러스터 관리자가 실제 스토리지를 미리 프로비저닝하는 대신, 쿠버네티스는 **스토리지클래스(StorageClass)** 리소스를 통해 **PV의 동적 프로비저닝**을 자동으로 수행할 수 있습니다.

**StorageClass = 스토리지 자동 생성 "레시피"**

#### Before: 정적 프로비저닝 (수동, 비효율적) 😱

```bash
# 관리자가 매번 수동으로 해야 하는 일들:
# 1. AWS 콘솔에서 EBS 볼륨 수동 생성
# 2. 볼륨 ID를 PV YAML에 수동 입력
# 3. PV 수동 생성
# 개발자가 100개 스토리지 요청하면 → 관리자가 100번 반복!
```

#### After: 동적 프로비저닝 (자동, 효율적) 🎉

```yaml
# 관리자는 한 번만 StorageClass 정의
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "20000"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

# 개발자가 PVC 요청하면 자동으로 PV 생성!
```

### StorageClass 개념

관리자는 하나 이상의 스토리지클래스 리소스를 생성하여 사용 가능한 스토리지 유형을 정의해야 합니다. 스토리지클래스 리소스는 PVC가 스토리지클래스에 요청할 때 어떤 프로비저너가 PV를 프로비저닝하는 데 사용돼야 할지를 지정하며, 스토리지 클래스에 정의된 파라미터는 프로비저너에 전달되고, 각 프로비저너 플러그인마다 파라미터가 다릅니다.

### 동적 프로비저닝 과정

1. **PVC의 스토리지클래스 요청**: 사용자는 PVC를 생성할 때 `storageClassName` 속성에 스토리지클래스 이름을 참조하여 특정 스토리지클래스를 요청할 수 있습니다.
2. **자동 PV 생성 및 바인딩**: PVC가 생성되면 쿠버네티스는 해당 스토리지클래스에 정의된 프로비저너를 통해 요청된 접근 모드와 스토리지 크기를 기반으로 실제 스토리지를 프로비저닝하고, PV를 생성한 후 PVC에 바인딩합니다.
    
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 10Gi
```

### 동적 프로비저닝 vs 정적 바인딩

#### 동적 프로비저닝 (일반적)

```yaml
spec:
  storageClassName: fast-ssd  # 자동으로 새 PV 생성
```

#### 기본 StorageClass 사용 (자동 할당)

```yaml
spec:
  # storageClassName 필드를 아예 생략
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  # → 기본 StorageClass로 자동 동적 프로비저닝
```

**기본 StorageClass 확인:**

```bash
kubectl get storageclass
# NAME                PROVISIONER   DEFAULT
# standard (default)  aws-ebs       true     ← 이것이 기본값
# fast-ssd           aws-ebs       false
```

**기본 스토리지클래스**: `storageClassName` 속성을 지정하지 않고 PVC를 생성하는 경우, 구글 쿠버네티스 엔진에서는 `pd-standard` 유형의 GCE 퍼시스턴트 디스크가 프로비저닝되는 것처럼 기본(default) 스토리지클래스가 사용될 수 있습니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

#### 정적 바인딩 (특정 PV 사용 강제)

```yaml
spec:
  storageClassName: ""    # 동적 프로비저닝 비활성화!
  selector:               # 원하는 PV 조건 지정
    matchLabels:
      special-pv: "true"
```

**특별한 동작 제어**: 동적 프로비저닝이 아닌 미리 프로비저닝된 PV에 바인딩을 강제하려면 `storageClassName` 속성을 **빈 문자열("")로 지정**해야 합니다. 그렇지 않으면 동적 프로비저너가 새로운 PV를 프로비저닝할 수 있습니다.

### storageClassName 동작 비교

| PVC 설정       | 기본 StorageClass 있음 | 기본 StorageClass 없음 | 결과     |
| ------------ | ------------------ | ------------------ | ------ |
| **필드 생략**    | 기본 SC로 동적 프로비저닝    | 정적 바인딩 시도          | 예측 불가능 |
| **구체적 지정**   | 지정된 SC로 동적 프로비저닝   | 지정된 SC로 동적 프로비저닝   | 예측 가능  |
| **빈 문자열 ""** | 정적 바인딩만            | 정적 바인딩만            | 예측 가능  |

**핵심**:
- `storageClassName: ""`은 "새로 만들지 말고 기존 것만 써"
- 필드 생략 시 기본 StorageClass가 자동으로 할당 (있는 경우)


### 이식성의 핵심 가치

**PVC 이식성**: 다른 클러스터 간에 스토리지클래스 이름을 동일하게 사용하면 PVC 정의를 다른 클러스터로 이식하기 용이합니다. 같은 애플리케이션 코드로 환경별 최적화된 다른 볼륨을 자동 사용할 수 있습니다.

```yaml
# 개발자 코드 (모든 환경에서 동일!)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  storageClassName: standard  # 환경별로 다른 실제 구현
```

**결과:**
- 개발환경: hostPath 사용 (무료, 빠른 개발)
- 스테이징: AWS gp3 사용 (적당한 성능)
- 프로덕션: AWS io2 사용 (최고 성능)

각 환경에서 완전히 독립적인 스토리지를 사용하지만, 개발자는 코드 변경 없이 모든 환경에서 배포가 가능합니다!

### 고급 기능

#### volumeBindingMode (바인딩 모드)

```yaml
# 즉시 바인딩
volumeBindingMode: Immediate

# 지연 바인딩 (권장) - Pod 스케줄링 시 바인딩
volumeBindingMode: WaitForFirstConsumer
```

#### allowVolumeExpansion (볼륨 확장)

```yaml
allowVolumeExpansion: true

# 사용법
kubectl patch pvc my-claim -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

<br>

## 볼륨 스냅샷 및 복제
---
### 볼륨 스냅샷 - 특정 시점 백업

쿠버네티스 1.20부터 안정화된 기능으로, PV의 특정 시점 스냅샷을 생성할 수 있습니다. **특정 시점의 PV 데이터를 복사본으로 저장하는 기능**입니다.

#### VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: fast-snapshot
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  encrypted: "true"
```

#### VolumeSnapshot 생성

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-backup-20241215
spec:
  volumeSnapshotClassName: fast-snapshot
  source:
    persistentVolumeClaimName: mysql-data-pvc
```

#### 스냅샷에서 복구

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-restored-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
  storageClassName: premium-ssd
  dataSource:                           # 스냅샷에서 복구!
    name: mysql-backup-20241215
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

#### 자동 백업 (CronJob)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-daily-backup
spec:
  schedule: "0 2 * * *"              # 매일 새벽 2시
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: bitnami/kubectl
            command: ["/bin/bash", "-c"]
            args:
            - |
              DATE=$(date +%Y%m%d-%H%M%S)
              cat <<EOF | kubectl apply -f -
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: mysql-daily-$DATE
              spec:
                volumeSnapshotClassName: fast-snapshot
                source:
                  persistentVolumeClaimName: data-mysql-0
              EOF
          restartPolicy: OnFailure
```

### 볼륨 복제 - 현재 데이터 복사

기존 PVC에서 새로운 PVC를 생성할 수 있습니다. **기존 PVC의 데이터를 그대로 복사하여 새로운 PVC를 생성**하는 기능입니다.

#### 볼륨 복제 vs 스냅샷 비교

|구분|볼륨 복제|볼륨 스냅샷|
|---|---|---|
|**소스**|살아있는 PVC|PVC의 특정 시점|
|**용도**|현재 데이터의 복사본|백업/복구|
|**독립성**|완전히 독립적인 PVC|읽기 전용 백업|
|**수정**|복제본 수정 가능|스냅샷은 불변|

#### 복제 사용법

```yaml
# 프로덕션 데이터를 개발환경으로 복제
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dev-mysql-data
  namespace: development
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 500Gi
  storageClassName: standard  # 개발용은 저비용 스토리지
  dataSource:
    name: data-postgres-prod-0      # 소스 PVC 이름
    kind: PersistentVolumeClaim
    apiGroup: ""
```

<br>

## 보안 고려사항
---
### 볼륨 보안 컨텍스트

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000              # 볼륨 그룹 권한
  containers:
  - name: app
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: data
      mountPath: /app/data
      readOnly: true           # 읽기 전용 마운트
```

### 볼륨 권한

- `readOnly`: 읽기 전용 마운트
- `subPath`: 볼륨 내 특정 경로만 마운트
- `subPathExpr`: 환경 변수를 사용한 동적 경로

```yaml
volumeMounts:
- name: shared-data
  mountPath: /app/config
  subPath: config
  readOnly: true
```

### 비밀 정보 보호

```yaml
# Secret을 파일로 마운트
volumes:
- name: ssl-certs
  secret:
    secretName: ssl-secret
    defaultMode: 0400        # 읽기 전용
    items:                   # 특정 키만 마운트
    - key: tls.crt
      path: server.crt
    - key: tls.key
      path: server.key
```

<br>

## 모니터링 및 문제 해결
---
### 볼륨 상태 확인

```bash
# PV 상태 확인
kubectl get pv

# PVC 상태 확인  
kubectl get pvc

# 스토리지클래스 확인
kubectl get storageclass

# 자세한 정보 조회
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>
```

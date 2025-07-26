# Amateurs K3s GitOps 프로젝트

K3s 단일 노드 클러스터에서 SpringBoot 애플리케이션과 MySQL Master-Slave, Redis를 운영하는 GitOps 기반 리포지토리입니다.

## 🏗️ 아키텍처 개요

```mermaid
graph TB
    subgraph "외부 접근"
        USER[User]
        TEAM[DBA]
    end
    
    subgraph "K3s 클러스터 (amateurs)"
        subgraph "Ingress Layer"
            NGINX[Nginx Ingress Controller]
            CERT[Cert-Manager]
        end
        
        subgraph "Application Layer"
            SB[SpringBoot App<br/>2 replicas]
        end
        
        subgraph "Data Layer"
            MM[MySQL Master]
            MS[MySQL Slave]
            REDIS[Redis Cache]
        end
        
        subgraph "Storage"
            PVC_M[MySQL Master PVC<br/>bulk-hdd 20Gi]
            PVC_S[MySQL Slave PVC<br/>bulk-hdd 20Gi]
        end
    end
    
    subgraph "외부 서비스"
        MONGO[MongoDB Atlas]
        QDRANT[Qdrant Vector DB]
        GEMINI[Google Gemini AI]
        OPENAI[OpenAI API]
        CloudFlare_R2[CloudFlare_R2]
    end
    
    USER -->|HTTPS| NGINX
    TEAM -->|TCP:30306| MM
    NGINX -->|api.amateurs.co.kr| SB
    SB -->|Write| MM
    SB -->|Read| MS
    SB -->|Cache/Session| REDIS
    MM -->|Replication| MS
    MM --- PVC_M
    MS --- PVC_S
    
    SB -.->|External APIs| MONGO
    SB -.->|External APIs| QDRANT
    SB -.->|External APIs| GEMINI
    SB -.->|External APIs| OPENAI
    SB -.->|External APIs| CloudFlare_R2
```

## 📁 프로젝트 구조

```
AIBE1-FinalProject-Team01-GitOps/
├── core/
│   └── ingress/
│       ├── amateurs-ingress.yaml      # SpringBoot HTTPS 접근
│       ├── mysql-ingress.yaml         # MySQL TCP 접근
│       └── kustomization.yaml
├── applications/
│   ├── namespace.yaml                 # amateurs 네임스페이스
│   ├── mysql/
│   │   ├── master/
│   │   │   ├── mysql-master-pvc.yaml
│   │   │   ├── mysql-master-statefulset.yaml
│   │   │   ├── mysql-master-service.yaml
│   │   │   └── kustomization.yaml
│   │   ├── slave/
│   │   │   ├── mysql-slave-pvc.yaml
│   │   │   ├── mysql-slave-statefulset.yaml
│   │   │   ├── mysql-slave-service.yaml
│   │   │   └── kustomization.yaml
│   │   ├── mysql-master-configmap.yaml
│   │   ├── mysql-slave-configmap.yaml
│   │   └── kustomization.yaml
│   ├── redis/
│   │   ├── redis-deployment.yaml      # 인메모리 캐시
│   │   ├── redis-service.yaml
│   │   └── kustomization.yaml
│   ├── springboot/
│   │   ├── springboot-deployment.yaml
│   │   ├── springboot-service.yaml
│   │   ├── springboot-config.yaml     # 애플리케이션 설정
│   │   └── kustomization.yaml
│   └── kustomization.yaml
└── kustomization.yaml                 # 루트 Kustomize
```

## 🚀 배포된 서비스

### SpringBoot 애플리케이션
- **이미지**: `ghcr.io/prgrms-aibe-devcourse/aibe1-finalproject-team01-be:latest`
- **리소스**: 2 replicas, 1-2Gi 메모리

### MySQL Master-Slave 클러스터
- **Master**: 
  - 읽기/쓰기 담당
  - GTID 기반 복제 설정
- **Slave**: 
  - 읽기 전용
  - 자동 복제 동기화
- **스토리지**: 각각 20Gi bulk-hdd PVC

### Redis Cache
- **용도**: 캐시, 토큰 스토어
- **설정**: 인메모리 전용 (영속성 없음)
- **메모리**: 128MB 데이터, 256MB 컨테이너 제한
- **접근**: ClusterIP (내부 접근만)

## 🔧 사전 요구사항

### K3s 클러스터 설정
```bash
# StorageClass 설정
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: kube-system
data:
  config.json: |-
    {
      "nodePathMap":[
        {
          "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths":["/var/lib/rancher/k3s/storage", "/data/k3s-storage"]
        }
      ]
    }
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
parameters:
  nodePath: /var/lib/rancher/k3s/storage
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: bulk-hdd
provisioner: rancher.io/local-path
parameters:
  nodePath: /data/k3s-storage
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
EOF
```

### 필수 구성요소
- **ArgoCD**: GitOps 배포 관리
- **Nginx Ingress Controller**: HTTP/HTTPS 라우팅
- **Cert-Manager**: SSL 인증서 관리

## 🔐 민감정보 설정

GitOps 보안을 위해 다음 파일들은 서버에서 직접 생성해야 합니다:

### 1. MySQL Secret
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: amateurs
type: Opaque
data:
  mysql-root-password: $(echo -n 'your-root-password' | base64)
  mysql-password: $(echo -n 'your-user-password' | base64)
  mysql-user: $(echo -n 'your-username' | base64)
  mysql-database: $(echo -n 'your-database' | base64)
  replication-user: $(echo -n 'replication' | base64)
  replication-password: $(echo -n 'replication-password' | base64)
EOF
```

### 2. SpringBoot Secret
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: springboot-secret
  namespace: amateurs
type: Opaque
data:
  DB_URL: $(echo -n 'jdbc:mysql://mysql:3306/your-database' | base64)
  DB_USERNAME: $(echo -n 'your-username' | base64)
  DB_PASSWORD: $(echo -n 'your-password' | base64)
  MONGODB_URL: $(echo -n 'mongodb+srv://user:pass@cluster.mongodb.net/db' | base64)
  GIT_CLIENT_ID: $(echo -n 'your-github-client-id' | base64)
  GIT_CLIENT_SECRET: $(echo -n 'your-github-client-secret' | base64)
  KAKAO_CLIENT_ID: $(echo -n 'your-kakao-client-id' | base64)
  KAKAO_CLIENT_SECRET: $(echo -n 'your-kakao-client-secret' | base64)
  GEMINI_API_KEY: $(echo -n 'your-gemini-api-key' | base64)
  OPENAI_API_KEY: $(echo -n 'your-openai-api-key' | base64)
  QDRANT_URL: $(echo -n 'your-qdrant-url' | base64)
  QDRANT_API_KEY: $(echo -n 'your-qdrant-api-key' | base64)
  JWT_SECRET: $(echo -n 'your-jwt-secret' | base64)
  AWS_ACCESS_KEY: $(echo -n 'your-aws-access-key' | base64)
  AWS_SECRET_KEY: $(echo -n 'your-aws-secret-key' | base64)
  S3_BUCKET_NAME: $(echo -n 'your-s3-bucket' | base64)
  GMAIL_USERNAME: $(echo -n 'your-gmail@gmail.com' | base64)
  GMAIL_APP_PASSWORD: $(echo -n 'your-gmail-app-password' | base64)
EOF
```

## 🚀 배포 가이드

### 1. 사전 설정
```bash
# 1. Secret 생성 (위의 민감정보 설정 참조)
# 2. GitOps 저장소 ArgoCD에 등록
```

### 2. ArgoCD 애플리케이션 배포
```bash
# ArgoCD UI 또는 CLI로 애플리케이션 생성
argocd app create amateurs \
  --repo https://github.com/your-org/AIBE1-FinalProject-Team01-GitOps \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace amateurs
```

### 3. MySQL 복제 설정
Master가 실행된 후:
```bash
# Master에서 복제 사용자 생성
kubectl exec -it mysql-master-0 -n amateurs -- mysql -u root -p
```
```sql
CREATE USER 'replication'@'%' IDENTIFIED BY 'replication-password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;
```

Slave에서 복제 시작:
```bash
kubectl exec -it mysql-slave-0 -n amateurs -- mysql -u root -p
```
```sql
CHANGE MASTER TO
  MASTER_HOST='mysql-master',
  MASTER_USER='replication',
  MASTER_PASSWORD='replication-password',
  MASTER_AUTO_POSITION=1;
START SLAVE;
SHOW SLAVE STATUS\G
```

## 🔍 모니터링 및 관리

### 헬스체크
```bash
# 모든 Pod 상태 확인
kubectl get pods -n amateurs

# 특정 서비스 로그 확인
kubectl logs -f deployment/amateurs -n amateurs
kubectl logs -f statefulset/mysql-master -n amateurs
kubectl logs -f deployment/redis -n amateurs
```

### MySQL 복제 상태 확인
```bash
# Slave 복제 상태
kubectl exec mysql-slave-0 -n amateurs -- mysql -u root -p -e "SHOW SLAVE STATUS\G"

# Master 바이너리 로그 위치
kubectl exec mysql-master-0 -n amateurs -- mysql -u root -p -e "SHOW MASTER STATUS"
```

### Redis 상태 확인
```bash
kubectl exec deployment/redis -n amateurs -- redis-cli info memory
kubectl exec deployment/redis -n amateurs -- redis-cli info stats
```

## 🛠️ 트러블슈팅

### 일반적인 문제
1. **Secret 누락**: 민감정보 설정 확인
2. **PVC 마운트 실패**: StorageClass 설정 확인
3. **복제 실패**: 네트워크 정책 및 MySQL 사용자 권한 확인
4. **Ingress 접근 불가**: DNS 설정 및 인증서 상태 확인

### 유용한 명령어
```bash
# 전체 리소스 상태
kubectl get all -n amateurs

# 이벤트 확인
kubectl get events -n amateurs --sort-by='.lastTimestamp'

# 설정 상태 확인
kubectl get configmap,secret -n amateurs
```

---

## 📞 연락처

프로젝트 관련 문의: `daycodingdan@gmail.com`
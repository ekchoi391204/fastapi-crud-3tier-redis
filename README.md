# React + FastAPI + Redis CRUD System

React 프론트엔드, FastAPI API, MySQL 8.0 데이터베이스와 Redis 세션 저장소로 구성된 CRUD 애플리케이션입니다.

## 이미지

| 역할 | 이미지 |
| --- | --- |
| React + Nginx 프론트엔드 | `hifrodo/crud2-front:1.0` |
| FastAPI API | `hifrodo/crud2-api:1.1` |
| MySQL 데이터베이스 | `hifrodo/crud2-sql:1.0` |
| Redis 세션 저장소 | `hifrodo/crud2-redis:1.0` |

## 서비스 연결

- API → MySQL: `db-svc:3306`
- API → Redis: `redis-svc:6379`
- 프론트엔드 Nginx → API: `api-svc:8000`

로그인/회원가입 시 API가 opaque 세션 토큰을 발급하고 실제 세션은 Redis에 TTL과 함께 저장합니다. 프론트엔드는 토큰을 `Authorization: Bearer` 헤더로 전달하며 쿠키는 사용하지 않습니다. 로그아웃하면 Redis 세션이 즉시 삭제됩니다.

## Kubernetes 배포

### 사전 조건

- Kubernetes 클러스터
- `kubectl`
- PVC를 동적으로 생성할 기본(Default) StorageClass

기본 StorageClass는 다음 명령으로 확인합니다.

```bash
kubectl get storageclass
```

`(default)`로 표시되는 StorageClass가 있어야 MySQL과 Redis의 PVC가 자동으로 생성됩니다. 기본 StorageClass가 없다면 `yaml/01-db.yaml`과 `yaml/02-redis.yaml`의 `volumeClaimTemplates`에 사용할 `storageClassName`을 지정해야 합니다.

### 배포

운영 환경에 배포하기 전에 `yaml/01-db.yaml`의 `db-secret` 값을 안전한 값으로 변경합니다.

전체 리소스를 배포합니다.

```bash
kubectl apply -f yaml
```

배포 상태를 확인합니다.

```bash
kubectl get pods,svc,deployment,statefulset,pvc
kubectl rollout status statefulset/db
kubectl rollout status statefulset/redis
kubectl rollout status deployment/api
kubectl rollout status deployment/front
```

Pod가 정상적으로 준비되지 않으면 다음 명령으로 상태와 로그를 확인합니다.

```bash
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
```

### 접속

프론트엔드는 `NodePort 30080`으로 공개됩니다.

```text
http://<NODE_IP>:30080
```

노드 IP는 다음 명령으로 확인할 수 있습니다.

```bash
kubectl get nodes -o wide
```

- 프론트엔드: `http://<NODE_IP>:30080`
- 상태 페이지: `http://<NODE_IP>:30080/status`
- API 상태: `http://<NODE_IP>:30080/health`
- 회원 API: `GET /api/members`
- 초기 아이디: `admin`
- 초기 비밀번호: `frodo1234`

API 문서에 로컬로 접근하려면 API Service를 포트 포워딩합니다.

```bash
kubectl port-forward service/api-svc 8000:8000
```

포트 포워딩 중에는 <http://localhost:8000/docs>에서 API 문서를 확인할 수 있습니다.

### 업데이트

YAML 변경 사항을 다시 적용합니다.

```bash
kubectl apply -f yaml
kubectl rollout status deployment/api
```

API Pod가 반환하는 `server_name`은 실제 요청을 처리한 Pod 이름입니다.

### 삭제

애플리케이션 리소스를 삭제합니다.

```bash
kubectl delete -f yaml
```

StatefulSet을 삭제해도 MySQL과 Redis의 PVC는 데이터 보호를 위해 유지될 수 있습니다. PVC까지 삭제하면 저장된 데이터도 복구할 수 없으므로 필요한 경우에만 별도로 삭제해야 합니다.

## 개별 이미지 빌드 및 push

```powershell
docker build -f db/Dockerfile -t hifrodo/crud2-sql:1.0 db
docker build -f app/Dockerfile -t hifrodo/crud2-api:1.1 .
docker build -f frontend/Dockerfile -t hifrodo/crud2-front:1.0 frontend
docker build -f redis/Dockerfile -t hifrodo/crud2-redis:1.0 redis

docker push hifrodo/crud2-sql:1.0
docker push hifrodo/crud2-api:1.1
docker push hifrodo/crud2-front:1.0
docker push hifrodo/crud2-redis:1.0
```

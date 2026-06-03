# Shoong 🛵

**MSA 기반 음식 배달 서비스를 Kubernetes 위에서 GitOps로 배포·운영하는 DevOps 프로젝트**

주문 → 조리 → 배달 → 알림으로 이어지는 배달 도메인을 마이크로서비스로 나누고, <br />
그 아래 **인프라(IaC) → 이미지 빌드(CI) → 배포(GitOps/CD) → 관측성(Observability)** 까지의 파이프라인을 직접 구성했습니다.

---

## 아키텍처

![DevOps Architecture](https://raw.githubusercontent.com/shoong-delivery/.github/main/profile/img/devops-architecture.png)

- **3 AZ** 구성의 VPC — Public / Private / DB 서브넷 분리
- 트래픽 진입: **Route53 → CloudFront → ALB → Istio Ingress → 서비스**
- **EKS** 워커 노드는 Private Subnet, **RDS(PostgreSQL)** 는 DB Subnet에 격리
- 운영 접근은 공인 IP 없는 **Bastion + SSM Session Manager**, Private 구간 AWS 호출은 **VPC Endpoint** 경유
- 설정·시크릿은 **SSM Parameter Store / Secrets Manager** → **External Secrets Operator** 로 클러스터에 주입

<details>
<summary><strong>K8s 클러스터 토폴로지 · CI/CD 파이프라인</strong></summary>

![K8s Topology](https://raw.githubusercontent.com/shoong-delivery/.github/main/profile/img/k8s-topology.png)

![CI/CD](https://raw.githubusercontent.com/shoong-delivery/.github/main/profile/img/cicd.png)

</details>

---

## 레포지토리

**플랫폼 / 인프라**

| 레포                                                                    | 역할                                               |
| ----------------------------------------------------------------------- | -------------------------------------------------- |
| [shoong-terraform](https://github.com/shoong-delivery/shoong-terraform) | AWS 인프라 프로비저닝 (IaC) — VPC·EKS·RDS·ECR·OIDC |
| [shoong-gitops](https://github.com/shoong-delivery/shoong-gitops)       | ArgoCD 앱 정의 / Helm 차트 / 관측성 스택 (CD)      |

**애플리케이션**

| 레포                                                                                  | 역할                                       | 포트 |
| ------------------------------------------------------------------------------------- | ------------------------------------------ | ---- |
| [shoong-order-api](https://github.com/shoong-delivery/shoong-order-api)               | 주문 서비스 (흐름의 시작점)                | 3001 |
| [shoong-kitchen-api](https://github.com/shoong-delivery/shoong-kitchen-api)           | 주방 서비스                                | 3002 |
| [shoong-delivery-api](https://github.com/shoong-delivery/shoong-delivery-api)         | 배달 서비스                                | 3003 |
| [shoong-notification-api](https://github.com/shoong-delivery/shoong-notification-api) | 알림 서비스                                | 3004 |
| [shoong-batch](https://github.com/shoong-delivery/shoong-batch)                       | 적체 자동완료 · 오래된 주문 정리 (CronJob) | -    |
| [shoong-frontend](https://github.com/shoong-delivery/shoong-frontend)                 | 프론트엔드 (React SPA)                     | -    |

---

## 동작 흐름

```
shoong-terraform ──(EKS / ECR / RDS / OIDC / SSM 프로비저닝)──┐
                                                             ▼
  앱 레포 ── GitHub Actions(OIDC) ── 빌드 → Trivy 스캔 → ECR push
                                                             │
                                       (envs/{env}/{app}.yaml image.tag 갱신)
                                                             ▼
                                                     shoong-gitops
                                          ArgoCD가 감지 → Helm 차트로 EKS 배포
```

- 앱 레포 CI가 이미지를 빌드해 ECR에 올리고, GitOps 레포의 이미지 태그를 갱신
- ArgoCD(App of Apps)가 변경을 감지해 자동 동기화 (prune + selfHeal)
- 프론트엔드는 정적 빌드 → **S3 + CloudFront** 로 별도 배포

---

## 기술 스택

| 영역                          | 사용 기술                                                                                     |
| ----------------------------- | --------------------------------------------------------------------------------------------- |
| **IaC**                       | Terraform (모듈화, S3 원격 상태 + DynamoDB Lock)                                              |
| **컨테이너 / 오케스트레이션** | Docker, Kubernetes (EKS)                                                                      |
| **CD / GitOps**               | ArgoCD (App of Apps, Multi-source), Helm                                                      |
| **CI**                        | GitHub Actions (OIDC 인증, Trivy 이미지 스캔)                                                 |
| **서비스 메시**               | Istio (Gateway / VirtualService / DestinationRule 서킷브레이커), Kiali                        |
| **관측성**                    | Prometheus · Grafana · Alertmanager / Loki · Promtail(로그) / Tempo · OpenTelemetry(트레이스) |
| **시크릿**                    | External Secrets Operator (SSM Parameter Store / Secrets Manager)                             |
| **부하 테스트**               | k6 (CronJob)                                                                                  |
| **애플리케이션**              | Node.js · TypeScript · Express · Prisma / React · Vite                                        |
| **DB**                        | PostgreSQL (RDS)                                                                              |

---

## 주요 설계 포인트

- **IRSA** 로 Pod 단위 IAM 권한 최소화 (노드 공유 권한 제거)
- **App of Apps + Multi-source** — 공통 Helm 차트 1개로 6개 앱 운영, 차이는 환경별 values로만 표현
- **상태 머신 기반 주문 흐름** — 서비스 간 연쇄 호출 + 멱등성 가드로 중복/재시도 안전성 확보
- **풀 스택 관측성** — 메트릭·로그·트레이스 + 비즈니스 대시보드(주문/조리/배달 단계별 지표)
- **비용 최적화** — 개발 환경 일일 destroy/apply 자동화 (DNS·레지스트리 등 영속 리소스는 보존)

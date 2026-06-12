# 샌드박스 인프라 구축 리뷰 — 콘솔 수동 재현

> **목적** — 샌드박스 AWS 계정에 콘솔(ClickOps)로 직접 구축하며 동작·연결을 검증. 이후 전체 삭제 → Terraform 재현 X. 콘솔에서 직접 만들며 학습
---

## 1. 최종 아키텍처

```
                        [Cloudflare DNS — timora.ai.kr zone]
                          api-ksy / admin-ksy  🟠 프록시   media-ksy ⚪ DNS만
                                   │                          │
              브라우저 ─HTTPS(CF Universal SSL)─ Cloudflare   │
                                   │ HTTPS(ACM 서울)          │ HTTPS(ACM 버지니아)
                                   ▼                          ▼
                            ALB :443 (호스트 룰)         CloudFront ─OAC─ S3(비공개)
                          ┌────────┴─────────┐
                 host=admin-ksy          default
                          ▼                  ▼
                   [backoffice :8090]   [api :8080/8081]──SvcConnect──[spicedb :50051]
                          │                  │                            │
                          └───── RDS MariaDB / ElastiCache Valkey / OpenSearch(+Nori)
                                       (private data subnets)

  CloudWatch:  알람 2개 설정 → SNS 토픽 (구독 미설정)
```

## 2. 구축 완료 요약

| 영역 | 내용 |
|---|---|
| 네트워크 | VPC(10.64.0.0/16), 서브넷 5(2AZ), IGW, NAT(영역별[zonal]), 라우트 테이블 3개 |
| 보안그룹 | 7개(api·backoffice·spicedb·rds·cache·search·alb) — 생성 후, 규칙 연결 |
| IAM/ECR | `ksy-ecsTaskExecutionRole`, task role(S3·SSM·CW), ECR 3repo |
| 데이터 | RDS MariaDB 11.8 / Valkey 8.2(전송·저장암호화) / OpenSearch 2.19(+**Nori 플러그인**) |
| ECS | 클러스터, Service Connect, task def 3|
| ALB | :443 HTTPS(ACM, TLS13-1-2-2021-06) + 호스트 룰(admin-ksy→bkoffice / default→api), SG는 Cloudflare IP 대역(15+7)만 |
| S3/CloudFront | 비공개 버킷 + OAC + 버킷정책, 별칭 media-ksy + 버지니아 인증서 |
| ACM | SAN 3개(api-ksy/admin-ksy/media-ksy) × 2리전(서울=ALB, 버지니아=CloudFront), DNS 검증 |
| Cloudflare | 검증 CNAME 3, DKIM CNAME 3(⚪), 트래픽 레코드 3(api/admin 🟠, media ⚪) |
| SES | 도메인 identity `ksy.timora.ai.kr` + SMTP 유저(`ksy-ses-smtp-user`, SendEmail/SendRawEmail)|
| CloudWatch | SNS 토픽 + 알람 2(JVM heap, HikariCP) — **토픽 구독(Slack/이메일) 미설정** |

## 3. Phase 별 구축 상세

### Phase 1 — VPC / 네트워크 ✅
- VPC `ksy-timora-sbx-apne2-vpc` = `vpc-0cd033a289c509048`, dev 와 동일. dev=10.64, prod=172.16 으로 환경별 대역 분리 — 피어링 시 충돌 방지
- 서브넷 5개 (이름↔실제 AZ 정렬 확인 완료)
  - public az1 `10.64.1.0/24`(2a) / public az2 `10.64.2.0/24`(2b)
  - private-app az1 `10.64.11.0/24`(2a)
  - private-data az1 `10.64.21.0/24`(2a) / az2 `10.64.22.0/24`(2b)
- IGW + **NAT GW(Zonal, public-az1)** + EIP 1개
- 라우트 테이블 3개 — public→IGW / private-app→NAT / private-data→NAT

### Phase 2 — 보안그룹 7개 ✅
- `api / backoffice / spicedb / rds / cache / search / alb` "SG 먼저, 규칙 나중"
- 아웃바운드는 기본(전체 허용) 유지
- 최종 인바운드 규칙
  ```
  외부 → Cloudflare → [ALB]        alb-sg:        443 ← Cloudflare IP 대역
           [ALB] → [api]           api-sg:        8080·8081 ← alb-sg  (default 룰 — 트래픽·헬스체크)
           [ALB] → [backoffice]    backoffice-sg: 8090 ← alb-sg      (admin-ksy 호스트 룰)
           [api] → [spicedb]       spicedb-sg:    50051 ← api-sg     (gRPC 권한 체크)
           [api] → [RDS]           rds-sg:        3306 ← api-sg      (앱 DB)
           [api] → [Valkey]        cache-sg:      6379 ← api-sg      (캐시·토큰)
           [api] → [OpenSearch]    search-sg:     80 ← api-sg        (검색 — api 만 사용)
           [spicedb] → [RDS]       rds-sg:        3306 ← spicedb-sg  (권한 데이터 저장)
           [backoffice] → [RDS]    rds-sg:        3306 ← backoffice-sg
           [backoffice] → [Valkey] cache-sg:      6379 ← backoffice-sg           
  관리자 → [bastion] → [RDS]       rds-sg:        3306 ← bastion-sg  (DB 생성·점검용 뒷문)
  ```

### Phase 3 — IAM + ECR ✅
- **`ksy-ecsTaskExecutionRole`** (execution role) — **ECS 인프라가 컨테이너를 "올릴 때" 쓰는 신분증.** 정책 2개:
  - `AmazonECSTaskExecutionRolePolicy`(AWS 관리형) → ECR 이미지 pull + 컨테이너 로그를 로그 그룹에 쓰기
  - `ksy-timora-ecs-ssm-access`(직접 생성) → SSM `/timora/*` 읽기+복호화 — spicedb 비밀값(preshared key·DB URI) 주입용
- **`ksy-timora-sbx-apne2-ecs-task-role`** (task role) — **컨테이너 안의 앱 코드가 "실행 중" 쓰는 신분증**
  - `s3-media-bucket-access` → 미디어 버킷 읽기/쓰기/삭제 (게시물 이미지)
  - `ssm-spring-config-read` → SSM `/spring-boot/*` 읽기 — api 가 부팅 때 설정 ~30개 셀프 로드
  - `cloudwatch-put-metric-data` → micrometer 지표 push (알람 2개의 데이터 원천)
- aws쪽이 쓰는 권한과 컨테이너 안 앱이 쓰는 권한 분리.
- **ECR repo 3개**: `timora/ksy-sbx-{api,backoffice,spicedb}` — 서비스별 컨테이너 이미지 창고. 직접 docker push 로 넣음

### Phase 4 — 데이터 레이어 ✅
| 리소스 | 스펙 | 
|--------|------|
| RDS MariaDB | 11.8.5, db.t3.micro, gp3 20GiB, 암호화 ON, AZ 미지정(No preference)이라 AWS 가 **2b(private-data-az2)** 에 배치 |
| ElastiCache Valkey | 8.2, cache.t3.micro, 노드1, **전송중·저장시 암호화 ON** — AZ 미지정(No preference), AWS 가 **2b** 에 배치 (RDS 와 동일) |
| OpenSearch | 2.19, t3.small.search, 1-AZ 1노드, **암호화 전부 OFF**, **+ Nori 플러그인** |

### SSM Parameter Store ✅
- `/spring-boot/ksy_sns_sandbox/` 환경변수명(~33개)

### RDS DB 생성 (bastion 경유) ✅
- RDS 는 private → **bastion EC2** (`i-0e480ed7a7d16177d`, Amazon Linux 2023, t3.micro, **private-app-az1**(10.64.11.202, 공인 IP x)인바운드 0)
  - SSM agent 아웃바운드가 **NAT 경유**라 공인 IP 없이도 Session Manager 접속 가능
  - SSM 서비스가 중계 역할: 사용자 - ssm 서비스 - vpc(gw- nat gw(public) - bastion ec2(private))
- **SSM 포트포워딩** ( 로컬 13306→RDS 3306) → IntelliJ 로 접속 후 `CREATE DATABASE mydatabase / spicedb` (utf8mb4)

### Phase 5 — ACM ✅
- **SAN 3개(api-ksy / admin-ksy / media-ksy) 인증서 1장 × 2리전** — 서울(ALB용), 버지니아(CloudFront용), DNS 검증, 둘 다 ISSUED
- 도메인 체계: `*-ksy.timora.ai.kr` **1단계 서브도메인** (Cloudflare Universal SSL 이 1단계까지만 커버)
- cloudflare 에서 인증서 생성 후 aws 에서 가져오기 방식보다 acm 에서 인증서 직접 생성 

### Phase 6 — ALB ✅ (HTTP 임시 → HTTPS 완성)
- 대상 그룹 — api-tg(Fargate 배포 시 마다 ip 변경 됨), 8080, 헬스체크 **8081** `/trafficReady`) + bkoffice-tg(8090)
- 리스너 :443 HTTPS(ACM, `ELBSecurityPolicy-TLS13-1-2-2021-06`) + **호스트 기반 룰**(admin-ksy→bkoffice / default→api)
- alb-sg 443 은 **Cloudflare IP 대역(IPv4 15 + IPv6 7)만** 허용 (운영 패턴)

### Phase 7 — ECS ✅ (서비스 3개 RUNNING+HEALTHY)
- ECR push: spicedb, api, backoffice
- 클러스터 `ksy-…-ecs-cluster`
- 로그 그룹 3개 `/ecs/ksy-…-{api,backoffice,spicedb}-task-definition` (7일)
- 태스크 정의 3개 (JSON 등록)
  - spicedb:`serve`, 50051, secrets(SSM 2개), grpc_health_probe 헬스체크
  - api: 8080·8081
  - backoffice: 8090 
- 서비스 3개 — spicedb / api / backoffice

### Phase 8~10 — CloudFront / Cloudflare·SES / CloudWatch ✅(부분)
- **CloudFront** — 비공개 S3 + OAC + 버킷정책, 별칭 `media-ksy.timora.ai.kr` + 버지니아 인증서 → 200 확인
- **Cloudflare** — CNAME 3, DKIM CNAME 3(⚪ DNS only), 트래픽 레코드 3 (api/admin 🟠 프록시, media ⚪)
- **SES** — identity `ksy.timora.ai.kr`(운영 도메인 명의 충돌 회피) + Easy DKIM + SMTP 유저
- **CloudWatch** — SNS 토픽 + 알람 2개(JVM heap, HikariCP)

# 리소스 정리 체크리스트

- 리전: 아시아 태평양(서울) `ap-northeast-2`
- 정리 시점: 동료 평가 3회 종료 후 (평가 중에는 외부 접속 확인을 위해 서비스 유지)
- 작업 계정: 1~8단계는 IAM 사용자 `b3-1-user`, 9~10단계는 루트 (b3-1-user는 결제·IAM 권한 없음)

## 생성한 리소스 (생성 시점부터 추적)

| 리소스 | 이름 / ID | 생성 | 정리 완료 |
|---|---|---|---|
| EC2 Instance | b3-1-web / i-042e5d2245b4cb5bd (t3.micro, Ubuntu 24.04) | 2026-11-22 | ✅ 2026-11-22 |
| EBS Volume | b3-1-web 루트 볼륨 (gp3 8GiB, 종료 시 삭제) | 2026-11-22 | ✅ 2026-11-22 |
| Key Pair | b3-1-key (RSA, .pem) | 2026-11-22 | ✅ 2026-11-22 |
| Security Group | b3-1-web-sg / sg-08cd8b3db6b800ad0 | 2026-11-22 | ✅ 2026-11-22 |
| Internet Gateway | b3-1-igw / igw-0122eeef13978f085 | 2026-11-22 | ✅ 2026-11-22 |
| Subnet | b3-1-public-subnet / subnet-064ec64c25abfcc17 | 2026-11-22 | ✅ 2026-11-22 |
| Route Table | b3-1-public-rt / rtb-0677ed61585ee7ae7 | 2026-11-22 | ✅ 2026-11-22 |
| VPC | b3-1-vpc / vpc-0280e8ef9cc471065 | 2026-11-22 | ✅ 2026-11-22 |
| Elastic IP | 사용 안 함 (자동 할당 퍼블릭 IP 사용) | - | 해당 없음 |
| NAT Gateway | 사용 안 함 | - | 해당 없음 |
| ELB/ALB | 사용 안 함 | - | 해당 없음 |
| RDS | 사용 안 함 | - | 해당 없음 |

## 정리 순서와 이유

**원칙: 과금이 큰 것부터, 그리고 다른 리소스가 의존하는 것(바깥쪽)은 나중에.**
VPC 안에 서브넷이, 서브넷 안에 인스턴스가 있으므로 **안쪽(인스턴스)부터 바깥쪽(VPC) 순서**로 지운다. 순서가 틀리면 `DependencyViolation`(사용 중) 오류로 삭제되지 않는다.

| 순서 | 대상 | 과금 | 이 순서인 이유 |
|---|---|---|---|
| 1 | EC2 인스턴스 | **시간당 과금** (가장 큼) | 과금을 가장 먼저 멈춤. 다른 리소스(SG, 서브넷)가 인스턴스에 묶여 있음 |
| 2 | EBS 볼륨 | **GB당 월 과금** | 인스턴스가 종료돼야 분리됨. 남은 미사용 볼륨도 과금 |
| 3 | Elastic IP | 미연결 시에도 과금 | 해당 없음 (확인만) |
| 4 | Key Pair | 없음 | 정리 차원 |
| 5 | Security Group | 없음 | 인스턴스(네트워크 인터페이스)가 사라져야 삭제 가능 |
| 6 | Internet Gateway | 없음 | VPC에서 먼저 분리(Detach)해야 삭제 가능 |
| 7 | Subnet → Route Table | 없음 | 서브넷을 지우면 라우팅 테이블 연결이 풀려 삭제 가능 |
| 8 | VPC | 없음 | 안의 리소스가 모두 사라져야 삭제 가능 |

> 퍼블릭 IPv4는 인스턴스에 자동 할당된 것이라 인스턴스 종료와 함께 반납된다.

## 정리 절차 (콘솔)

### 1~8단계: `b3-1-user`로 로그인, 리전 = 서울

1. [x] **EC2 인스턴스 종료**
   - EC2 → 인스턴스 → `b3-1-web` 선택 → **인스턴스 상태** → **인스턴스 종료(삭제)** → 확인
   - 상태가 `종료 중` → **`종료됨`** 으로 바뀔 때까지 대기 (1~2분)
   - 종료된 인스턴스는 약 1시간 동안 목록에 회색으로 남아 있다가 사라짐 (과금 없음)
   - 📸 `cleanup-01-ec2-terminated.png`

2. [x] **EBS 볼륨 확인**
   - EC2 → **볼륨** → 목록이 **비어 있는지** 확인 (루트 볼륨은 "종료 시 삭제"가 기본이라 자동 삭제됨)
   - 남아 있는 `사용 가능` 상태 볼륨이 있으면 선택 → **작업 → 볼륨 삭제**
   - 📸 `cleanup-02-ebs-empty.png`

3. [x] **Elastic IP 확인**
   - EC2 → **탄력적 IP** → 목록이 비어 있는지 확인
   - 📸 `cleanup-03-eip-empty.png`

4. [x] **키 페어 삭제**
   - EC2 → **키 페어** → `b3-1-key` 선택 → **작업 → 삭제**
   - 노트북의 사용자 폴더 `.ssh\b3-1-key.pem` 파일도 삭제

5. [x] **보안 그룹 삭제**
   - EC2(또는 VPC) → **보안 그룹** → `b3-1-web-sg` 선택 → **작업 → 보안 그룹 삭제**
   - "연결된 인스턴스가 있어 삭제할 수 없음"이 나오면 1단계 인스턴스가 완전히 `종료됨`인지 확인 후 재시도
   - VPC를 만들 때 자동 생성된 `default` 보안 그룹은 직접 삭제할 수 없음 (8단계 VPC 삭제 시 함께 삭제)

6. [x] **인터넷 게이트웨이 분리 후 삭제**
   - VPC → **인터넷 게이트웨이** → `b3-1-igw` 선택 → **작업 → VPC에서 분리** → 확인
   - 상태가 `Detached` 가 되면 → **작업 → 인터넷 게이트웨이 삭제**

7. [x] **서브넷 → 라우팅 테이블 삭제**
   - VPC → **서브넷** → `b3-1-public-subnet` → **작업 → 서브넷 삭제**
   - VPC → **라우팅 테이블** → `b3-1-public-rt` → **작업 → 라우팅 테이블 삭제**
   - VPC를 만들 때 생긴 기본(main) 라우팅 테이블은 직접 삭제할 수 없음 (8단계에서 함께 삭제)
   - ⚠️ 실제 정리 중 라우팅 테이블을 서브넷보다 먼저 삭제하려다 **"라우팅 테이블에는 연결이 있으며 삭제할 수 없습니다"** 오류 발생 → 서브넷을 먼저 삭제해 연결을 해제한 뒤 진행 (📸 `cleanup-05-rt-dependency-error.png`)

8. [x] **VPC 삭제**
   - VPC → **내 VPC** → `b3-1-vpc` → **작업 → VPC 삭제** → 확인 문구 입력
   - 남은 리소스가 있으면 삭제 화면에 목록으로 표시됨
   - 📸 `cleanup-07-vpc-deleted.png` (서울 리전 VPC 목록에 `b3-1-vpc` 없음. AWS 기본 VPC는 원래 있던 것이므로 남아 있어도 됨)

### 9~10단계: 루트로 로그인

9. [x] **결제 확인**
   - 결제 및 비용 관리 → **청구서** 또는 **Cost Explorer** → 서울 리전 EC2·VPC 항목이 더 늘어나지 않는지 확인
   - 비용 데이터는 하루 몇 번만 갱신되므로 **다음 날 한 번 더** 확인
   - 📸 `cleanup-09-billing.png`

10. [x] **계정 정리 (선택)**
    - IAM → 사용자 `b3-1-user` 삭제, 정책 `codyssey-b3-1-least-privilege` 삭제
    - 예산 `codyssey-b3-1`: 다음 실습에도 쓸 계정이면 유지 (과금 감시 역할)
    - 더 이상 AWS를 쓰지 않을 계획이면 계정 해지도 고려

## 정리 증빙

정리 일시: 2026-11-22 (동료 평가 3회 완료 후)

| 단계 | 결과 | 증빙 |
|---|---|---|
| EC2 종료 | `b3-1-web` 상태 **종료됨**, 퍼블릭 IP 반납 | [cleanup-01](images/cleanup-01-ec2-terminated.png) |
| EBS | "현재 이 리전에 볼륨이 없습니다" | [cleanup-02](images/cleanup-02-ebs-empty.png) |
| Elastic IP | "이 리전에서 탄력적 IP 주소를 찾을 수 없음" | [cleanup-03](images/cleanup-03-eip-empty.png) |
| 키 페어 | "1개의 키 페어 삭제 완료", 로컬 `.pem` 삭제 | [cleanup-04](images/cleanup-04-keypair-deleted.png) |
| 라우팅 테이블 의존성 오류 | 서브넷 연결이 남아 삭제 거부 → 순서 조정 | [cleanup-05](images/cleanup-05-rt-dependency-error.png) |
| Security Group / IGW / Subnet | 삭제 및 분리 완료, 남은 서브넷은 AWS 기본 VPC 소속 | [cleanup-06](images/cleanup-06-subnet-deleted.png) |
| VPC | "b3-1-vpc 및 1 기타 리소스를 삭제했습니다", 기본 VPC(172.31.0.0/16)만 남음 | [cleanup-07](images/cleanup-07-vpc-deleted.png) |
| Route Table | 기본 VPC의 main 테이블만 남음 | [cleanup-08](images/cleanup-08-rt-deleted.png) |
| Billing | 11월 예상 총합계 **USD 0.00** (정리 직후 기준) | [cleanup-09](images/cleanup-09-billing.png) |
| IAM (선택) | 사용자 `b3-1-user`, 정책 `codyssey-b3-1-least-privilege` 삭제 | [cleanup-10](images/cleanup-10-iam-user-deleted.png), [cleanup-11](images/cleanup-11-iam-policy-deleted.png) |

- 비용 데이터는 최대 하루 늦게 반영되므로 다음 날 Billing을 다시 확인한다.
- 남아 있는 기본 VPC, 기본 서브넷, 기본 IGW, 기본 보안 그룹은 계정 생성 시 AWS가 만든 것으로 과금되지 않는다.

## 계정 관련 메모

- 이 계정은 **유료 플랜**(이전 계정 이력으로 무료 플랜 대상 아님)이라 프리 티어 한도를 넘으면 바로 과금된다.
- 유지 비용(평가 기간): 인스턴스 약 0.013 USD/h + 퍼블릭 IPv4 약 0.005 USD/h + EBS 8GiB ≈ **하루 약 0.45 USD**
- 예산 알림: 월 2 USD 초과(실제 85%/100%, 예상 100%) 시 이메일

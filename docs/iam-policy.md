# IAM 최소권한 정책 설명

- 정책 이름: `codyssey-b3-1-least-privilege`
- 정책 원본: [iam-policy.json](iam-policy.json)
- 연결 대상: IAM 사용자 `b3-1-user` (AdministratorAccess 미부여)
- 루트 계정은 IAM 사용자 생성, MFA, 예산 알림 설정에만 사용하고 실습은 모두 IAM 사용자로 진행

## Security Group vs IAM

| 구분 | Security Group | IAM |
|---|---|---|
| 무엇을 막나 | **네트워크 트래픽** (어떤 IP가 어떤 포트로 들어올 수 있나) | **AWS 작업** (누가 어떤 리소스를 만들고 지울 수 있나) |
| 적용 위치 | EC2 인스턴스(네트워크 인터페이스) | 사용자 / 역할 |
| 예시 | 80은 전체 허용, 22는 내 IP만 | EC2·VPC만 만들 수 있고 S3·RDS는 불가 |

## 정책 구성 (Statement별)

| Sid | 허용 내용 | 제한 조건 | 이유 |
|---|---|---|---|
| ReadOnlyEc2AndVpc | EC2/VPC 조회 | 서울 리전만 | 콘솔 화면이 목록을 불러오려면 조회 권한이 필요 |
| ManageNetwork | VPC, Subnet, IGW, Route Table 생성·연결·삭제 | 서울 리전만 | 네트워크 구성과 정리(삭제) |
| ManageSecurityGroups | 보안 그룹과 인바운드/아웃바운드 규칙 관리 | 서울 리전만 | 80/22 포트 규칙 설정 |
| LaunchOnlyMicroInstances | 인스턴스 생성 | **t2.micro / t3.micro만** | 비싼 인스턴스 실수 생성 방지 |
| LaunchWithSmallVolumesOnly | 인스턴스 생성 시 EBS 볼륨 | **10GiB 이하만** | 불필요한 스토리지 증설 방지 |
| LaunchSupportingResources | 인스턴스 생성에 쓰이는 AMI, 서브넷, ENI, 보안 그룹, 키페어 | 서울 리전 ARN만 | RunInstances는 여러 리소스를 함께 사용하므로 각각 허용 필요 |
| ManageInstancesKeysTagsAndAddresses | 시작/중지/종료, 키페어, 태그, EBS 삭제, Elastic IP | 서울 리전만 | 운영과 정리(Terminate, Release) |
| ReadPublicAmiParameters | AWS 공개 AMI 정보 조회 | `/aws/service/*` 공개 파라미터만 | 인스턴스 생성 화면에서 최신 AMI 조회 |

## 의도적으로 허용하지 않은 것

- S3, RDS, Lambda, IAM 등 **실습과 무관한 서비스 전체**
- NAT Gateway, Load Balancer, VPC Peering 등 **과금이 큰 네트워크 리소스**
- 서울 이외 **다른 리전**의 모든 작업
- micro 이외 **인스턴스 타입**, 10GiB 초과 **볼륨**
- 액세스 키 발급 (콘솔 + SSH만 사용)

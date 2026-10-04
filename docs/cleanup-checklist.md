# 리소스 정리 체크리스트 (OCI)

실습 종료 후 아래 순서대로 삭제하고, 각 항목에 확인 근거를 남깁니다. (의존 관계의 안쪽부터)

## 필수 5종 (평가 기준: EC2 / EBS / EIP / IGW / VPC 정리)

| 순서 | AWS 대응 | OCI 리소스 | 이름 | 삭제 확인 방법 | 완료 | 근거 |
|---|---|---|---|---|---|---|
| 1 | EC2 | Compute Instance | `b31-web-01` | Compute > Instances 상태 `Terminated` | ☐ | `<스크린샷/시각>` |
| 2 | EBS | Boot Volume | `b31-web-01 (Boot Volume)` | Terminate 시 부트 볼륨 삭제 체크, Boot Volumes 목록 비어 있음 | ☐ | |
| 3 | EIP | Reserved Public IP | `b31-eip` | Networking > Reserved Public IPs 비어 있음 | ☐ | |
| 4 | IGW | Internet Gateway | `b31-igw` | VCN > Internet Gateways 비어 있음 | ☐ | |
| 5 | VPC | VCN | `b31-vcn` | Networking > VCN 목록에서 제거 | ☐ | |

## 추가 항목

| 순서 | 항목 | 확인 방법 | 완료 | 근거 |
|---|---|---|---|---|
| 6 | Subnet, Route Table, Security List, NSG | 각 목록에서 `b31-*` 항목 삭제 (VCN 삭제 전 완료) | ☐ | |
| 7 | Block Volume (만들었다면) | Block Volumes 목록 비어 있음 | ☐ | |
| 8 | NAT Gateway / Load Balancer 등 | 만들지 않았다면 N/A | ☐ | |
| 9 | 남은 리소스 조회 | Tenancy Explorer 또는 검색에서 태그 `project=b3-1` 결과 없음 | ☐ | `<스크린샷>` |
| 10 | 과금 확인 | Billing > Cost Analysis, 태그/Compartment 필터로 비용 항목 없음 | ☐ | `<스크린샷>` |
| 11 | IAM 정리 (선택) | 실습용 사용자, 그룹, Policy 삭제 또는 비활성화 | ☐ | |

## 추적 기준
- 이름 규칙: 모든 리소스에 `b31-` 접두사
- 태그: `project=b3-1`, `env=lab` (생성 시 모든 리소스에 부여)
- 삭제 후 하루 뒤 Cost Analysis를 한 번 더 확인

## 정리 순서 (삭제 순서)
Instance → Boot/Block Volume → Reserved Public IP → Route Table 연결 해제 → Internet Gateway → Subnet / NSG / Security List → VCN

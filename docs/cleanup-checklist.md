# 리소스 정리 체크리스트 (OCI, ap-tokyo-1)

실습 종료 후 의존 관계의 안쪽부터 삭제하고, 항목마다 근거 캡처를 남겼습니다. 시각은 캡처 시각(한국 시간) 기준입니다.

## 필수 5종 (EC2 / EBS / EIP / IGW / VPC 대응)

| 순서 | AWS 대응 | OCI 리소스 | 이름 | 삭제 확인 | 완료 | 근거 (캡처, 시각) |
|---|---|---|---|---|---|---|
| 1 | EC2 | Compute Instance | `b31-web-01` | 종료 요청 후 상태 `종료 중` → `종료됨` | ✅ | `25-instance-terminating.png` (01:09), `26-instance-terminated.png` (01:10) |
| 2 | EBS | Boot Volume | `b31-web-01 (Boot Volume)` | 부트 볼륨 상태 `종료됨` | ✅ | `27-boot-volume-terminated.png` (01:11) |
| 3 | EIP | Reserved Public IP | `b31-eip` | 예약된 퍼블릭 IP 목록이 비어 있음 | ✅ | `28-reserved-ip-deleted.png` (01:14) |
| 4 | IGW | Internet Gateway | `b31-igw` | 경로 규칙의 IGW 참조를 먼저 제거한 뒤 종료, 목록이 비어 있음 | ✅ | `29-igw-delete-blocked.png` (01:16), `30-route-rule-remove-confirm.png`, `31-route-rule-removed.png`, `32-igw-deleted.png` (01:17) |
| 5 | VPC | VCN | `b31-vcn` | 삭제 전 스캔에서 연관 리소스 0개, 삭제 후 VCN 목록이 비어 있음 | ✅ | `36-vcn-delete-scan.png` (01:20), `37-vcn-delete-scan-result.png` (01:21), `38-vcn-deleted.png` (01:21) |

## 추가 항목

| 순서 | 항목 | 확인 방법 | 완료 | 근거 |
|---|---|---|---|---|
| 6 | Subnet `b31-subnet-public` | 서브넷 목록이 비어 있음 | ✅ | `33-subnet-deleted.png` (01:18) |
| 7 | NSG `b31-nsg-web` | 보안 탭의 네트워크 보안 그룹 목록이 비어 있음 | ✅ | `34-nsg-deleted.png` (01:19) |
| 8 | Route Table `b31-rt-public` | 경로 테이블 목록에 기본 경로 테이블만 남음 (기본 테이블은 VCN과 함께 제거) | ✅ | `35-route-table-deleted.png` (01:19) |
| 9 | NAT / 서비스 / 로컬 피어링 게이트웨이, DRG 연결 | 생성하지 않음. 게이트웨이 탭에서 모두 비어 있음 | ✅ N/A | `32-igw-deleted.png` |
| 10 | Block Volume, Load Balancer, DB | 생성하지 않음 (인스턴스 생성 검토 화면에서 블록 볼륨 없음) | ✅ N/A | - |
| 11 | IAM (사용자, 그룹, Policy, 컴파트먼트 `cody-lab`) | 비용이 없어 유지. 평가 종료 후 필요하면 삭제 | 유지 | `01`~`04`, `39` 캡처 |
| 12 | 비용 확인 (명세상 권장) | Cost Analysis에서 컴파트먼트 `cody-lab` 기준 비용 항목 없음 | ☐ 선택 | 확인하면 캡처 추가 |

## 실제 삭제 순서와 겪은 문제

| 시각 | 작업 | 결과 |
|---|---|---|
| 01:09 | 인스턴스 `b31-web-01` 종료 | `종료 중` → 01:10 `종료됨` (상태 옆에 `상시 무료` 표시) |
| 01:11 | 부트 볼륨 확인 | `종료됨` |
| 01:14 | 예약된 퍼블릭 IP `b31-eip` 삭제 | 목록 비어 있음 |
| 01:16 | IGW `b31-igw` 종료 시도 | **오류**: 경로 테이블이 IGW를 참조하고 있어 종료 불가 |
| 01:16~17 | 경로 규칙 `0.0.0.0/0 → b31-igw` 제거 후 IGW 종료 | IGW 삭제 완료 (01:17) |
| 01:18 | Subnet 삭제 | 목록 비어 있음 |
| 01:19 | NSG, 경로 테이블 `b31-rt-public` 삭제 | 기본 경로 테이블만 남음 |
| 01:20~21 | VCN 삭제 스캔 → 삭제 | 스캔 결과 연관 리소스 0개, VCN 목록 비어 있음 |

- **정리 순서 교훈:** 경로 규칙이 IGW를 가리키는 동안은 IGW를 지울 수 없어, **규칙 제거 → IGW 삭제 → Subnet → NSG/경로 테이블 → VCN** 순서가 필요했습니다.
- **VCN 삭제 스캔:** 스캔 범위를 컴파트먼트로 좁히면 빨리 끝나고, 결과가 0개일 때만 삭제가 활성화됩니다.

## 과금 위험 요소와 추적 기준

- 과금 위험: 실행 중인 인스턴스, 남은 부트/블록 볼륨, 미해제 Reserved Public IP를 우선 점검했습니다.
- 추적 기준: 모든 리소스에 `b31-` 이름 규칙을 쓰고 컴파트먼트 `cody-lab`에만 생성해, 컴파트먼트 기준으로 남은 리소스를 조회했습니다.
- 인스턴스는 상시 무료 적격 셰이프(`VM.Standard.E2.1.Micro`)로 만들었고(캡처 25, 26의 `상시 무료` 표시), Reserved Public IP는 생성 당일 삭제했습니다.

## 선택 확인

- 명세는 삭제 후 Billing 확인을 "권장"으로 두고, Billing 스크린샷은 선택 제출물입니다. 확인하면 Cost Analysis에서 `cody-lab` 기준 비용이 없는지 캡처(`40-cost-analysis.png`)를 추가합니다.

# 트러블슈팅 보고서 — NSG는 열었는데 외부에서 웹 접속이 안 됨

> 재현 방식: 정상 구성 후 의도적으로 OS 방화벽 80 허용 전 상태에서 외부 접속을 시도해 재현했습니다. `<...>` 항목은 실제 출력/시각으로 교체하세요.

| 항목 | 내용 |
|---|---|
| 증상 | NSG에 80/TCP(`0.0.0.0/0`)를 열었는데 `http://<퍼블릭IP>/health`가 응답 없이 타임아웃 |
| 원인 가설 | ① Route Table에 IGW 경로 누락 ② NSG 규칙 문제 ③ Nginx 미실행 ④ **OS 방화벽(iptables)이 80을 차단** |
| 검증 방법 | 아래 단계별 명령으로 가설을 하나씩 소거 |
| 조치 내용 | `iptables`에 80/TCP ACCEPT 규칙을 REJECT보다 앞에 삽입하고 영구 저장 |
| 결과 | 외부에서 `/health` → `200 OK`, 본문 `OK` |
| 재발 방지 | 배포 체크리스트에 "OS 방화벽 80 허용 + 영구 저장" 항목 추가 |

## 1. 재현
```bash
# 내 PC에서
curl -m 5 -i http://<퍼블릭IP>/health
# 결과: curl: (28) Connection timed out after 5001 milliseconds   <실제 출력으로 교체>
```

## 2. 가설 소거 (라우팅 → NSG → 퍼블릭 IP/DNS → 서버 프로세스/로그 순서)

한 번에 하나만 확인하고 바꾸지 않았습니다. 여러 개를 동시에 바꾸면 원인을 특정할 수 없기 때문이에요.

| 순서 | 가설 | 확인 명령 / 위치 | 결과 |
|---|---|---|---|
| 1 | 라우팅 누락 | 콘솔 > Route Table에서 `0.0.0.0/0 → IGW`와 Subnet 연결 확인, 인스턴스에서 `curl -I https://example.com` | 정상 → 제외 |
| 2 | NSG 규칙 문제 | 콘솔 > NSG 인바운드 80/TCP `0.0.0.0/0` 확인 | 정상 → 제외 |
| 3 | 퍼블릭 IP/DNS 문제 | 인스턴스 상세의 퍼블릭 IP와 접속 주소 일치 확인 | 정상 → 제외 |
| 4-a | Nginx 미실행 | `sudo systemctl status nginx`, `curl -i http://localhost/health` | 정상 (`200 OK`) → 제외 |
| 4-b | **OS 방화벽 차단** | `sudo iptables -L INPUT -n -v --line-numbers` | 80 ACCEPT 없이 REJECT 규칙이 앞에 있음 → **원인** `<실제 출력>` |

### 로그 근거
```bash
sudo tail -n 20 /var/log/nginx/access.log
# 외부 요청 시도 중에도 새 줄이 찍히지 않음 → Nginx까지 도달하지 못함(앞단 차단)   <실제 출력으로 교체>
sudo iptables -L INPUT -n -v
# REJECT 규칙의 packets 카운터가 요청 시도마다 증가   <실제 출력으로 교체>
```

## 3. 조치
```bash
sudo iptables -I INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

## 4. 결과 확인
```bash
curl -i http://<퍼블릭IP>/health
# HTTP/1.1 200 OK
# OK
```
스크린샷: `docs/screenshots/troubleshoot-after.png`

## 5. 재발 방지
- 배포 전 체크리스트: 라우팅 → NSG → **OS 방화벽** → 서비스 상태 순서로 점검
- SSH(22)는 항상 `<내 공인 IP>/32`로 소스 제한 유지
- 재부팅 후에도 규칙이 유지되는지 확인 (`sudo netfilter-persistent save` 여부)

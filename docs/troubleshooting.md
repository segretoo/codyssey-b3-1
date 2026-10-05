# 트러블슈팅 보고서 — NSG는 열려 있는데 외부 웹 접속이 타임아웃

> 환경: OCI `ap-tokyo-1`, 인스턴스 `b31-web-01` (Ubuntu 22.04, `VM.Standard.E2.1.Micro`), Reserved Public IP `168.110.46.76`, 2026-10-05(UTC) 작업. 실제로 발생한 문제를 그대로 기록했습니다.

## 요약

| 항목 | 내용 |
|---|---|
| 증상 | Nginx를 설치하고 NSG에 80/TCP(`0.0.0.0/0`)를 열었는데, 집 PC에서 `curl -m 5 -i http://168.110.46.76/health`가 `Connection timed out` |
| 원인 가설 | ① 라우팅 누락 ② NSG 규칙 문제 ③ 퍼블릭 IP/DNS 문제 ④ Nginx 미실행 ⑤ **OS 방화벽(iptables)이 80을 차단** |
| 검증 방법 | 라우팅 → NSG → 퍼블릭 IP/DNS → 서버 프로세스/로그 순서로 하나씩 소거 (아래 표) |
| 조치 내용 | `iptables` INPUT 체인 맨 앞에 80/TCP ACCEPT 규칙을 삽입하고 `netfilter-persistent save`로 저장 |
| 결과 | 외부에서 `HTTP/1.1 200 OK`, `Content-Type: text/plain`, 본문 `OK` 확인 |
| 재발 방지 | 배포 체크리스트에 "OS 방화벽 80 허용 + 영구 저장" 추가, 점검 순서 고정 |

## 1. 증상 (재현)

```bash
# 집 PC에서
curl -m 5 -i http://168.110.46.76/health
# curl: (28) Connection timed out after 5006 milliseconds
```

- 캡처: `docs/screenshots/18-external-timeout.png`
- 로그: `docs/logs/02-nginx-setup-and-timeout.log`

## 2. 가설과 검증 (라우팅 → NSG → 퍼블릭 IP/DNS → 서버 프로세스/로그)

한 번에 하나만 확인하고 바꾸지 않았습니다. 여러 개를 동시에 바꾸면 원인을 특정할 수 없기 때문입니다.

| 순서 | 가설 | 검증 | 결과 |
|---|---|---|---|
| 1 | 라우팅 누락 | 같은 경로로 SSH(22) 접속 성공, 인스턴스에서 `curl -I https://example.com` → `HTTP/2 200`, 경로 테이블 `0.0.0.0/0 → b31-igw` 확인 | 정상 → 제외 |
| 2 | NSG 규칙 문제 | NSG 인바운드 80/TCP ← `0.0.0.0/0`, VNIC에 `b31-nsg-web` 연결 확인 | 정상 → 제외 |
| 3 | 퍼블릭 IP/DNS 문제 | Reserved Public IP `168.110.46.76` (예약됨), 같은 IP로 SSH 접속 성공 | 정상 → 제외 |
| 4-a | Nginx 미실행 | `systemctl status nginx` → `active (running)`, `curl -I http://localhost` → `200 OK` | 정상 → 제외 |
| 4-b | **OS 방화벽 차단** | `sudo iptables -L INPUT -n -v --line-numbers` | 80 허용 규칙 없이 `REJECT`가 있음 → **원인** |

- 관련 캡처: `06-route-table.png`, `14-nsg-rules-updated.png`, `16-vnic-detail-reserved-ip.png`, `19-iptables-and-access-log-before.png`

## 3. 로그 근거

**iptables (조치 전)**

```
num  pkts bytes target  prot opt in  out  source     destination
1   43916  62M  ACCEPT  all  --  *   *    0.0.0.0/0  0.0.0.0/0   state RELATED,ESTABLISHED
2       0     0 ACCEPT  icmp --  *   *    0.0.0.0/0  0.0.0.0/0
3    1074  107K ACCEPT  all  --  lo  *    0.0.0.0/0  0.0.0.0/0
4       2   104 ACCEPT  tcp  --  *   *    0.0.0.0/0  0.0.0.0/0   state NEW tcp dpt:22
5      45  2696 REJECT  all  --  *   *    0.0.0.0/0  0.0.0.0/0   reject-with icmp-host-prohibited
```

- 22번(SSH)만 허용돼 있고, 80번 허용 규칙이 없는 채로 전체 `REJECT`가 이어집니다.
- `REJECT` 규칙에 45개 패킷이 걸려 있어, 허용되지 않은 접속이 이 규칙에서 차단되고 있음을 보여줍니다.

**Nginx 접근 로그**

```bash
sudo tail -n 5 /var/log/nginx/access.log
# 127.0.0.1 - - [05/Oct/2026:15:39:54 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.81.0"
```

- 기록된 요청은 인스턴스 내부에서 내가 실행한 `curl localhost`뿐이고, 외부 요청 기록이 없습니다. 요청이 Nginx까지 도달하지 못했다는 근거입니다.

## 4. 조치

```bash
sudo iptables -I INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
sudo DEBIAN_FRONTEND=noninteractive apt install -y iptables-persistent
sudo netfilter-persistent save
sudo iptables -L INPUT -n -v --line-numbers
```

**iptables (조치 후)**

```
1       0     0 ACCEPT  tcp  --  *   *   0.0.0.0/0  0.0.0.0/0   tcp dpt:80 state NEW
2   43975   62M ACCEPT  all  --  *   *   0.0.0.0/0  0.0.0.0/0   state RELATED,ESTABLISHED
3       0     0 ACCEPT  icmp --  *   *   0.0.0.0/0  0.0.0.0/0
4    1074  107K ACCEPT  all  --  lo  *   0.0.0.0/0  0.0.0.0/0
5       2   104 ACCEPT  tcp  --  *   *   0.0.0.0/0  0.0.0.0/0   state NEW tcp dpt:22
6      45  2696 REJECT  all  --  *   *   0.0.0.0/0  0.0.0.0/0   reject-with icmp-host-prohibited
```

- 80/TCP ACCEPT가 `REJECT`보다 앞(1번)에 위치합니다. 규칙은 위에서부터 순서대로 평가되므로 반드시 `REJECT`보다 앞에 있어야 합니다.
- `netfilter-persistent save`가 `ip4tables`/`ip6tables` 저장을 실행해 규칙이 영구 저장됩니다.
- 캡처: `docs/screenshots/20-iptables-after.png`

## 5. 결과

```bash
curl -m 5 -i http://168.110.46.76/health
# HTTP/1.1 200 OK
# Server: nginx/1.18.0 (Ubuntu)
# Content-Type: text/plain
# Content-Length: 3
#
# OK
```

- 캡처: `docs/screenshots/22-external-health-ok.png`

## 6. 추가 이슈: `/health` 설정이 적용되지 않음 (붙여넣기 오류)

| 항목 | 내용 |
|---|---|
| 증상 | `/health` 설정 블록을 붙여넣었는데 `[200~sudo: command not found`가 나옴 |
| 원인 | 터미널의 붙여넣기 모드(bracketed paste)가 제어 문자(`[200~`)를 명령어에 섞어 넣어, 첫 줄이 `sudo`가 아닌 문자열로 읽힘. 설정 파일은 바뀌지 않았고 뒤의 `nginx -t`는 기존 설정을 검사해서 성공으로 보임 |
| 조치 | `bind 'set enable-bracketed-paste off'`로 붙여넣기 모드를 끈 뒤 같은 블록을 다시 실행 |
| 결과 | `nginx -t`가 `syntax is ok` / `test is successful`, 외부 `/health`가 `200 OK`와 본문 `OK`를 반환 |

- 캡처: `docs/screenshots/21-health-config-nginx-test.png`
- 실패 시점의 터미널 기록: `docs/logs/02-nginx-setup-and-timeout.log`

## 7. 재발 방지

- 점검 순서를 **라우팅 → NSG → 퍼블릭 IP/DNS → 서버 프로세스/로그(OS 방화벽 포함)**로 고정하고, 한 번에 하나씩만 바꿉니다.
- 배포 체크리스트에 "**OS 방화벽 80 허용 + 영구 저장**"을 추가합니다. NSG에서 열었는데도 접속이 안 되면 OS 방화벽을 의심합니다.
- 방화벽 규칙을 추가한 뒤에는 `iptables -L INPUT -n --line-numbers`로 `ACCEPT`가 `REJECT`보다 앞에 있는지 확인합니다.
- SSH(22)는 항상 내 공인 IP(`/32`)로만 허용합니다. 장소가 바뀌면 IP도 바뀌므로 NSG의 22번 소스를 현재 IP로 갱신합니다.
- 긴 설정 블록은 붙여넣기 전에 붙여넣기 모드 문제를 점검하고, 적용 후 `nginx -t`와 `curl localhost/health`로 반드시 확인합니다.

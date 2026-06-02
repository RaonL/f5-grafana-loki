# F5 Grafana + Loki + Promtail 모니터링 스택

> **WSL2 + Docker Desktop** 환경에서 F5 BIG-IP WAF 로그를 실시간으로 모니터링합니다.

## 전체 구성도

```
┌─ Proxmox (Mini PC) ───────────────────────┐
│  F5 BIG-IP (192.168.137.125)              │
│    └─ ASM 로그 → syslog (UDP 1514) ───────┼──────┐
└────────────────────────────────────────────┘      │
                                                    ▼
┌─ 노트북 (WSL2 + Docker Desktop) ───────────────────┐
│                                                      │
│  Promtail (port 1514)                                │
│    └─ syslog 수신 → 로그 파싱                        │
│         │                                            │
│         ▼                                            │
│  Loki (port 3100)                                    │
│    └─ 로그 저장/검색                                  │
│         │                                            │
│         ▼                                            │
│  Grafana (port 3000)                                 │
│    └─ http://localhost:3000                           │
│       ├─ ID: admin / PW: admin                       │
│       ├─ Loki Data Source 자동 연결                   │
│       └─ F5 AWAF 대시보드 자동 임포트                │
└──────────────────────────────────────────────────────┘
```

---

## 1️⃣ WSL2 + Docker Desktop 설치 (처음 1회)

### Windows PowerShell (관리자 권한)에서 실행

```powershell
# 1. WSL2 활성화
wsl --install -d Ubuntu

# 2. 재부팅 후 Ubuntu 초기 설정 (계정/비밀번호 생성)

# 3. Docker Desktop 설치
#    https://docs.docker.com/desktop/setup/install/windows-install/
#    → 다운로드 후 설치
#    → Settings > Resources > WSL Integration > "Ubuntu" ON

# 4. WSL2 기본 버전 확인
wsl -l -v
#    → Ubuntu가 2로 표시되어야 함
```

### Ubuntu(WSL2) 터미널에서 실행

```bash
# Ubuntu 업데이트
sudo apt update && sudo apt upgrade -y

# Docker Compose 확인
docker compose version
```

---

## 2️⃣ 프로젝트 준비

```bash
# WSL2 Ubuntu에서 클론
cd ~
git clone https://github.com/RaonL/f5-grafana-loki.git
cd f5-grafana-loki

# 디렉토리 구조 확인
ls -la
# ├── docker-compose.yml
# ├── .gitignore
# ├── loki/loki-config.yml
# ├── promtail/promtail-config.yml
# └── grafana/
#     ├── datasources/datasource.yml
#     └── dashboards/
#         ├── dashboard.yml
#         └── f5_awaf_dashboard.json
```

---

## 3️⃣ 모니터링 스택 실행

```bash
cd ~/f5-grafana-loki

# 컨테이너 실행
docker compose up -d

# 상태 확인
docker compose ps
# NAME            IMAGE                    PORTS
# f5-grafana      grafana/grafana:11.6.1  0.0.0.0:3000->3000/tcp
# f5-loki         grafana/loki:3.7.2      0.0.0.0:3100->3100/tcp
# f5-promtail     grafana/promtail:3.7.2  0.0.0.0:1514->1514/tcp,udp

# 로그 확인
docker compose logs -f
```

---

## 4️⃣ Grafana 접속 확인

브라우저에서 접속:
- **URL**: http://localhost:3000
- **ID**: admin
- **PW**: admin

**자동 설정 확인:**
1. ✅ Loki Data Source가 이미 등록되어 있어야 함
2. ✅ "F5 AWAF - 실시간 WAF 모니터링" 대시보드가 자동 임포트됨

---

## 5️⃣ BIG-IP → Promtail syslog 연결

> WSL2는 NAT 네트워크이므로 **Windows 방화벽 포트포워딩**이 필요합니다.

### WSL2 IP 확인 + 포트포워딩 설정

```powershell
# Windows PowerShell (관리자 권한)

# WSL2 IP 확인
wsl -- ip addr show eth0 | findstr "inet "
# → 예: 172.27.64.1

# Windows → WSL2 포트포워딩
netsh interface portproxy add v4tov4 listenport=1514 listenaddress=0.0.0.0 connectport=1514 connectaddress=172.27.64.1

# Windows 방화벽 인바운드 규칙 추가 (1514번 포트)
netsh advfirewall firewall add rule name="Promtail 1514" dir=in action=allow protocol=TCP localport=1514
netsh advfirewall firewall add rule name="Promtail 1514 UDP" dir=in action=allow protocol=UDP localport=1514

# 포트포워딩 확인
netsh interface portproxy show all
```

### 노트북 IP 확인

```powershell
ipconfig
# → 무선 LAN 어댑터 Wi-Fi IP 확인 (예: 192.168.137.x)
```

### BIG-IP에서 syslog 설정 (CLI)

```bash
# BIG-IP SSH 접속
ssh admin@192.168.137.125

# 원격 syslog 설정 (노트북 IP가 192.168.137.50 이라고 가정)
tmsh create sys syslog remote-servers { notebook-mon { host 192.168.137.50 port 1514 } }

# 설정 저장
tmsh save sys config
```

### BIG-IP에서 syslog 설정 (웹 UI)
1. **BIG-IP 웹 UI 접속**: `https://192.168.137.125`
2. **System → Logs → Configuration → Remote Logging**
3. **Remote Servers** → **Add**
   - Name: `notebook-mon`
   - Host: `[노트북 IP]`
   - Port: `1514`
   - Protocol: `UDP`
4. **Update** → **Save**

---

## 6️⃣ 테스트 및 확인

```bash
# Promtail 로그 확인 (syslog 수신 여부)
docker compose logs promtail | tail -20

# Promtail 상태 및 syslog 수신 카운터 확인
curl -s http://localhost:9080/ready
curl -s http://localhost:9080/metrics | grep promtail_syslog_target_entries_total

# Grafana 대시보드 열기
# 브라우저 → http://localhost:3000
# → "F5 AWAF - 실시간 WAF 모니터링" 대시보드
```

### 직접 테스트 로그 보내보기 (BIG-IP 없이도 가능)

```bash
echo "<14>1 $(date -u +%Y-%m-%dT%H:%M:%SZ) bigip asm - - attack_type=\"SQL Injection\",ip_client=\"192.168.1.100\",method=\"GET\",policy_name=\"/Common/test_policy\",request_status=\"blocked\",response_code=\"403\",sig_names=\"SQL Injection Attempt\",support_id=\"123456789\",uri=\"/test?id=1\",violations=\"VIOL_ATTACK_SIGNATURE\",violation_rating=\"5\"" | nc -u localhost 1514
```

---

## 7️⃣ 유용한 명령어

```bash
# 서비스 중지
docker compose down

# 서비스 재시작
docker compose restart

# 로그 실시간 보기
docker compose logs -f promtail

# Loki에 저장된 로그 개수 확인
curl -s "http://localhost:3100/loki/api/v1/query?query=count_over_time({job=\"f5-awaf\"}[1h])" | jq

# Loki에 생성된 label 확인
curl -s "http://localhost:3100/loki/api/v1/labels" | jq
curl -s "http://localhost:3100/loki/api/v1/label/job/values" | jq

# 모든 데이터 초기화
docker compose down -v
```

---

## 8️⃣ 문제 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| Grafana 3000 접속 안 됨 | Docker 미실행 | `docker compose up -d` 실행 |
| Promtail 로그 없음 | BIG-IP syslog 미도달 | ① Windows 방화벽 확인 ② `netsh portproxy` 확인 ③ BIG-IP syslog 설정 확인 |
| `expecting a version value in the range 1-999` | Promtail이 RFC5424로 파싱하지만 BIG-IP가 RFC3164 syslog를 전송 | `promtail-config.yml`의 `syslog_format: rfc3164` 확인 후 `docker compose restart promtail` |
| Loki 연결 오류 | Loki 컨테이너 미실행 | `docker compose logs loki` 확인 |
| 대시보드 안 보임 | 프로비저닝 실패 | `docker compose restart grafana` |
| WSL2 IP 변경됨 | WSL2 재시작 | `netsh interface portproxy` 재설정 |
| 포트 충돌 | 다른 프로그램 사용 중 | `docker-compose.yml`에서 포트 변경 (예: 3001:3000) |

---

## 📁 프로젝트 구조

```
f5-grafana-loki/
├── docker-compose.yml              # Grafana + Loki + Promtail 통합 실행
├── .gitignore
├── README.md
├── loki/
│   └── loki-config.yml             # Loki 설정 (30일 보존, WSL2 최적화)
├── promtail/
│   └── promtail-config.yml         # Promtail 설정 (syslog 수신, ASM 로그 파싱)
└── grafana/
    ├── datasources/
    │   └── datasource.yml          # Loki Data Source 자동 등록
    └── dashboards/
        ├── dashboard.yml           # 대시보드 프로비저닝 설정
        └── f5_awaf_dashboard.json  # F5 AWAF 대시보드 (통계/추세/Top10/로그)
```

---

## 📊 대시보드 패널

| 패널 | 설명 |
|------|------|
| 🛡️ 총 탐지 건수 (24시간) | 임계치 기반 컬러 표시 (녹색/주황/빨강) |
| 🚫 차단 건수 (24시간) | 차단된 요청 수 |
| ⚠️ 탐지율 (%) | 차단 / 전체 * 100 |
| 🌐 활성 공격자 IP | 최근 1시간 고유 IP 수 |
| 📊 공격 추세 (시간별) | 전체/차단/Alarm 시계열 그래프 |
| 🔝 Top 10 공격 유형 | 가장 많이 탐지된 공격 유형 (가로 막대) |
| 🔝 Top 10 공격자 IP | 가장 활동적인 IP (가로 막대) |
| 📋 최근 탐지 로그 | 실시간 로그 뷰어 |

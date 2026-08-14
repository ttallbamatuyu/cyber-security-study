# Day 01 — 네트워크 기초: IP, Port, Router, Gateway

> 작성일: 2026-08-14  
> 권장 학습 시간: 2시간~3시간

오늘은 명령어를 많이 외우는 것보다 **내 Ubuntu가 인터넷의 다른 컴퓨터와 어떻게 통신하는지 머릿속에 그림을 만드는 것**이 목표입니다.

LLM API, RAG 서버, 데이터베이스, AI Agent의 Tool 호출도 모두 네트워크 위에서 동작합니다. 오늘 익히는 IP, Port, Router, Gateway는 이후 HTTP, Burp Suite, API 보안, SSRF와 AI Agent 보안을 이해하는 기초가 됩니다.

## 오늘의 학습 목표

- [ ] 네트워크가 무엇인지 설명할 수 있다.
- [ ] Client와 Server의 역할을 구분할 수 있다.
- [ ] IPv4 주소의 형태와 역할을 설명할 수 있다.
- [ ] Private IP, Public IP, Loopback 주소를 구분할 수 있다.
- [ ] IP 주소와 MAC 주소의 차이를 설명할 수 있다.
- [ ] Port가 필요한 이유와 `IP:Port`의 의미를 설명할 수 있다.
- [ ] Router, Default Gateway, NAT의 역할을 설명할 수 있다.
- [ ] `ip addr`, `ip route`, `ping`, `ss`를 사용할 수 있다.
- [ ] 명령어 결과에서 내 IP, Gateway, Listening Port를 찾을 수 있다.
- [ ] 서비스가 어느 네트워크 주소에 노출되어 있는지 기본적으로 판단할 수 있다.

---

## 1. 네트워크란?

네트워크는 여러 컴퓨터와 장치가 서로 데이터를 주고받을 수 있도록 연결된 구조입니다.

집에서 노트북, 스마트폰, 태블릿, TV가 같은 Wi-Fi 공유기에 연결되어 있다면 이 장치들은 하나의 로컬 네트워크에 속합니다.

```text
노트북 ─────┐
스마트폰 ───┤
태블릿 ─────┼── 공유기 ─── 인터넷
TV ─────────┘
```

인터넷도 특별히 분리된 기술이 아니라 수많은 작은 네트워크가 서로 연결된 거대한 네트워크입니다.

```text
내 Ubuntu
    │
    ▼
집 또는 회사의 공유기
    │
    ▼
인터넷 서비스 제공자(ISP)
    │
    ▼
인터넷
    │
    ▼
Web / API / LLM Server
```

네트워크 통신에는 대략 다음 정보가 필요합니다.

- 데이터를 보내는 출발지
- 데이터를 받을 목적지
- 목적지까지 가는 경로
- 목적지 컴퓨터에서 사용할 서비스
- 통신할 때 따를 규칙인 Protocol

장치 또는 인터페이스를 찾는 데 사용하는 것이 **IP 주소**이고, 장치 안의 서비스를 구분하는 것이 **Port**입니다.

---

## 2. Client와 Server

### Client

Client는 서비스를 요청하는 쪽입니다.

예:

- Chrome, Firefox 같은 웹 브라우저
- 스마트폰 애플리케이션
- `curl` 같은 명령줄 도구
- 외부 API를 호출하는 백엔드
- LLM API를 호출하는 AI 애플리케이션

### Server

Server는 요청을 받아 서비스를 제공하는 쪽입니다.

예:

- Web Server
- API Server
- Database Server
- SSH Server
- LLM Inference Server
- RAG Search Server
- AI Agent가 사용하는 Tool Server

```text
Client
  │
  │ Request
  ▼
Server
  │
  │ Response
  ▼
Client
```

### 역할은 고정되지 않는다

한 컴퓨터가 항상 Client이거나 항상 Server인 것은 아닙니다.

백엔드는 브라우저의 요청을 받을 때는 Server이지만, 외부 LLM API에 요청을 보낼 때는 Client가 됩니다.

```text
사용자 브라우저 ──요청──> 백엔드 서버 ──요청──> LLM 서버
     Client            Server + Client          Server
```

Client와 Server는 장치의 종류가 아니라 **특정 통신에서 담당하는 역할**입니다.

---

## 3. IP 주소

IP는 `Internet Protocol`의 약자입니다.

IP 주소는 네트워크에서 통신 대상을 식별하고 데이터가 목적지까지 전달될 수 있도록 사용하는 주소입니다.

```text
Source IP      보내는 쪽 주소
Destination IP 받는 쪽 주소
```

예:

```text
192.168.10.24
203.0.113.10
```

> `203.0.113.0/24`는 문서와 교육용으로 예약된 주소 대역입니다. 위 주소는 실제 서버 주소가 아닙니다.

### 컴퓨터 한 대에 IP는 하나뿐일까?

컴퓨터에는 여러 네트워크 인터페이스가 있을 수 있습니다.

- 유선 Ethernet 인터페이스
- Wi-Fi 인터페이스
- VPN 인터페이스
- Docker 가상 네트워크
- Loopback 인터페이스

각 인터페이스에 서로 다른 IP 주소가 할당될 수 있으므로 컴퓨터 한 대에 IP 주소가 반드시 하나만 있는 것은 아닙니다.

IP 주소는 DHCP, VPN, 네트워크 이동 등에 따라 바뀔 수도 있습니다. 따라서 IP 하나만으로 특정 사용자나 장치를 영구적으로 식별할 수 있다고 생각하면 안 됩니다.

---

## 4. IPv4와 CIDR

IPv4 주소는 총 32bit입니다. 사람이 읽기 쉽게 8bit씩 네 부분으로 나누어 10진수로 표시합니다.

```text
192  .  168  .  10  .  24
8bit   8bit    8bit   8bit
```

8bit로 표현할 수 있는 값은 `0`부터 `255`까지입니다.

`ip addr` 결과에서는 주소 뒤에 `/24` 같은 표기가 붙을 수 있습니다.

```text
192.168.10.24/24
```

- `192.168.10.24`: 인터페이스에 할당된 IPv4 주소
- `/24`: 앞의 24bit가 네트워크 부분이라는 뜻
- `/24`: 일반적으로 Subnet Mask `255.255.255.0`과 대응

이 예의 네트워크 대역은 보통 다음과 같이 표현합니다.

```text
192.168.10.0/24
```

`/24`는 Port 번호가 아니라 주소가 속한 네트워크 범위를 나타내는 CIDR 표기입니다. Subnet 계산은 이후에 더 자세히 학습합니다.

---

## 5. Private IP와 Public IP

### Private IP

Private IP는 집, 회사, 학교, 가상 머신처럼 내부 네트워크에서 사용하는 주소입니다.

대표적인 IPv4 Private 주소 범위:

| 주소 범위 | CIDR |
| --- | --- |
| `10.0.0.0` ~ `10.255.255.255` | `10.0.0.0/8` |
| `172.16.0.0` ~ `172.31.255.255` | `172.16.0.0/12` |
| `192.168.0.0` ~ `192.168.255.255` | `192.168.0.0/16` |

처음에는 다음 형태를 기억하면 됩니다.

```text
10.x.x.x
172.16.x.x ~ 172.31.x.x
192.168.x.x
```

서로 다른 집에서 같은 `192.168.0.10`을 사용해도 두 내부 네트워크가 분리되어 있으므로 문제가 생기지 않습니다.

Private IP는 일반적으로 인터넷에서 직접 Routing되지 않습니다. 인터넷에 나갈 때는 보통 Router의 NAT 기능을 거칩니다.

### Public IP

Public IP는 인터넷에서 식별되고 Routing될 수 있는 주소입니다. 보통 ISP나 Cloud 사업자가 할당합니다.

```text
노트북 192.168.0.10 ─┐
휴대폰 192.168.0.11 ─┼── 공유기 ── Public IP ── Internet
태블릿 192.168.0.12 ─┘
```

| 구분 | Public IP | Private IP |
| --- | --- | --- |
| 사용 범위 | 인터넷 | 내부 네트워크 |
| 인터넷 Routing | 가능 | 일반적으로 직접 불가능 |
| 중복 사용 | 인터넷 전체에서 고유하게 관리 | 서로 다른 내부망에서 중복 가능 |
| 일반적인 할당 | ISP, Cloud 사업자 | 공유기, DHCP Server, 관리자 |

> Private IP를 사용한다고 자동으로 안전해지는 것은 아닙니다. 같은 내부망의 침해된 장치, 잘못된 VPN, Port Forwarding, Cloud 설정 때문에 접근될 수 있습니다.

---

## 6. Loopback과 localhost

Loopback은 컴퓨터가 자기 자신과 통신할 때 사용하는 특별한 네트워크입니다.

IPv4에서는 `127.0.0.0/8` 대역이 Loopback 용도로 예약되어 있고, 가장 자주 보는 주소는 `127.0.0.1`입니다.

```text
localhost → 127.0.0.1 → 현재 컴퓨터 자신
```

예:

```text
127.0.0.1:8000
```

이는 현재 컴퓨터의 Loopback 주소에서 8000번 Port를 사용하는 서비스라는 뜻입니다.

### `127.0.0.1`과 `0.0.0.0`

- `127.0.0.1:8000`: Loopback에서만 연결을 받음
- `0.0.0.0:8000`: 모든 IPv4 인터페이스에서 연결을 받도록 대기

`0.0.0.0`은 원격 접속에 사용하는 실제 목적지 주소가 아니라 Server가 모든 로컬 IPv4 주소에서 요청을 받겠다는 의미로 자주 사용됩니다.

`0.0.0.0`에 Binding했다고 인터넷에서 반드시 접속 가능한 것은 아닙니다. Routing, Firewall, NAT, Port Forwarding, Cloud 보안 정책도 영향을 줍니다.

IPv6에서는 모든 인터페이스를 나타내는 형태로 `[::]`를 볼 수 있습니다.

---

## 7. MAC 주소

MAC 주소는 같은 로컬 네트워크 구간에서 네트워크 인터페이스를 식별하는 데 사용하는 주소입니다.

보통 48bit를 16진수 여섯 부분으로 표시합니다.

```text
02:42:ac:11:00:02
```

| 구분 | IP 주소 | MAC 주소 |
| --- | --- | --- |
| 주요 역할 | 네트워크 간 주소 지정과 Routing | 로컬 구간의 인터페이스 식별 |
| 예시 | `192.168.10.24` | `02:42:ac:11:00:02` |
| 사용 범위 | 여러 네트워크를 거친 통신 | 주로 같은 로컬 네트워크 |
| 변경 가능성 | DHCP, VPN 등에 따라 바뀔 수 있음 | 가상화, Randomization, Spoofing 가능 |

IPv4 환경에서는 ARP를 사용해 같은 네트워크에 있는 IP와 MAC의 관계를 확인합니다.

MAC 주소가 원격 웹 서버까지 그대로 전달되는 것은 아닙니다. Router를 지날 때마다 각 로컬 구간에 필요한 Link 계층 정보가 달라집니다.

---

## 8. Port

한 컴퓨터에서는 여러 네트워크 서비스가 동시에 실행될 수 있습니다.

```text
한 Server
├── SSH Server
├── Web Server
├── Database Server
└── API Server
```

IP만으로는 어느 서비스와 통신할지 구분할 수 없습니다. 이때 사용하는 논리적인 번호가 Port입니다.

```text
IP 주소 = 어느 컴퓨터 또는 인터페이스인가?
Port     = 그 안의 어느 서비스인가?
```

### Port 범위

TCP와 UDP의 Port 번호는 `0`부터 `65535`까지입니다.

| 범위 | 일반적인 분류 |
| ---: | --- |
| `0` ~ `1023` | Well-known Port |
| `1024` ~ `49151` | Registered Port |
| `49152` ~ `65535` | Dynamic/Ephemeral Port |

Client는 Server에 연결할 때 운영체제가 임시 Port를 자동으로 선택하는 경우가 많습니다.

```text
Client 192.168.10.24:53122
              │
              ▼
Server 198.51.100.20:443
```

- Server는 HTTPS Port `443`에서 요청을 기다립니다.
- Client의 `53122`는 이 연결을 위해 임시로 선택된 Port일 수 있습니다.
- `198.51.100.0/24`는 문서용 주소 대역입니다.

### 자주 보는 Port

| Port | 일반적인 서비스 | 주로 사용하는 전송 Protocol |
| ---: | --- | --- |
| `22` | SSH | TCP |
| `53` | DNS | UDP 또는 TCP |
| `80` | HTTP | TCP |
| `443` | HTTPS | TCP, QUIC 환경에서는 UDP |
| `3306` | MySQL | TCP |
| `5432` | PostgreSQL | TCP |
| `8000`, `8080` | 개발용 Web Server에서 자주 사용 | 주로 TCP |

Port 번호는 관례입니다. Port만 보고 실제 프로그램을 확정하면 안 됩니다. 다른 Port에서 서비스를 실행할 수도 있고, 같은 Port를 전혀 다른 프로그램이 사용할 수도 있습니다.

---

## 9. `IP:Port` 읽기

```text
192.168.10.24:22
```

이는 `192.168.10.24`라는 IP 주소의 `22`번 Port를 사용하는 서비스라는 뜻입니다.

URL에서는 다음과 같이 볼 수 있습니다.

```text
http://127.0.0.1:8000
```

```text
http        → Application Protocol
127.0.0.1   → Destination IP
8000        → Destination Port
```

실제 Socket은 IP와 Port뿐 아니라 TCP 또는 UDP도 함께 구분합니다. 예를 들어 TCP 53번 Port와 UDP 53번 Port는 서로 다른 Socket입니다.

---

## 10. Router와 Gateway

### Router

Router는 서로 다른 네트워크 사이에서 Packet이 이동할 경로를 결정하고 전달하는 장치입니다.

```text
Network A
192.168.10.0/24
       │
       ▼
     Router
       │
       ▼
Network B / Internet
```

가정용 공유기는 보통 다음 기능을 함께 제공합니다.

- Wi-Fi Access Point
- Switch
- Router
- DHCP Server
- NAT
- 기본 Firewall

이 기능들은 개념적으로 서로 다른 역할입니다.

### Gateway

Gateway는 현재 네트워크에서 다른 네트워크로 나갈 때 사용하는 출구입니다.

```text
내 Ubuntu IP:     192.168.10.24
Default Gateway: 192.168.10.1
Network:         192.168.10.0/24
```

같은 네트워크의 `192.168.10.50`에는 로컬 네트워크에서 통신할 수 있지만, 외부 주소로 보낼 때는 보통 Default Gateway에 먼저 전달합니다.

```text
내 Ubuntu
192.168.10.24
      │
      ▼
Default Gateway
192.168.10.1
      │
      ▼
Internet
```

Router는 네트워크 사이에서 Packet을 전달하는 장치이고, Gateway는 현재 장치가 다른 네트워크로 나갈 때 바라보는 다음 출구라는 관점의 차이가 있습니다. 집에서는 공유기가 두 역할을 함께 하므로 비슷하게 느껴집니다.

---

## 11. NAT

NAT는 `Network Address Translation`의 약자입니다. 내부 Private 주소와 외부 Public 주소 사이를 변환합니다.

```text
노트북 192.168.10.24 ─┐
휴대폰 192.168.10.25 ─┼── Router/NAT ── Public IP ── Internet
태블릿 192.168.10.26 ─┘
```

가정에서는 여러 장치가 하나의 Public IP를 공유하는 경우가 많습니다. Router는 내부 주소와 Port의 조합을 외부 주소와 Port로 변환하고, 응답을 어느 내부 장치에 전달해야 하는지 기록합니다.

원리를 위한 예:

```text
변환 전
192.168.10.24:53122 → 198.51.100.20:443

변환 후
203.0.113.10:40001 → 198.51.100.20:443
```

위 주소들은 모두 문서용 예시입니다.

외부에서 내부 Server로 새 연결을 시작하려면 Port Forwarding, VPN, Reverse Proxy, Tunnel 같은 추가 구성이 필요한 경우가 많습니다.

### NAT와 Firewall은 다르다

- NAT: 주소 또는 Port를 변환
- Firewall: 정책에 따라 통신을 허용하거나 차단

가정용 공유기가 두 기능을 함께 제공할 수 있지만 NAT가 있다는 사실만으로 안전이 완전히 보장되지는 않습니다.

---

## 12. 웹사이트 접속 흐름

브라우저에서 HTTPS 웹사이트에 접속하면 대략 다음 과정이 일어납니다.

```text
Domain 입력
   ↓
DNS로 Server IP 확인
   ↓
Routing Table로 경로 결정
   ↓
외부 목적지라면 Default Gateway로 전달
   ↓
필요하면 Router가 NAT 수행
   ↓
Server Port에 TCP 또는 QUIC 연결
   ↓
TLS로 암호화 통신 설정
   ↓
HTTP Request / Response
```

오늘은 이 흐름 중 IP, Port, Gateway, Router, NAT에 집중합니다. DNS, TCP, HTTP, TLS는 다음 Day에서 자세히 학습합니다.

---

## 13. 오늘의 명령어

| 명령어 | 확인할 내용 |
| --- | --- |
| `ip addr` | Interface, IP 주소, MAC 주소 |
| `ip -br addr` | Interface와 주소를 짧은 형식으로 확인 |
| `ip route` | Routing Table과 Default Gateway |
| `ping -c 4 대상` | 기본적인 IP 도달 가능성과 왕복 시간 |
| `ss -tuln` | TCP/UDP Listening Socket과 Port |
| `sudo ss -ltnp` | TCP Listening Port와 사용 Process |

명령어 출력은 Ubuntu, WSL, 가상 머신, VPN, Docker 환경에 따라 달라집니다. 아래 출력은 실제 사용자의 결과가 아니라 **읽는 방법을 설명하기 위한 가상 예시**입니다.

---

## 14. `ip addr` — Interface와 IP 확인

```bash
ip addr
```

짧게 확인:

```bash
ip -br addr
```

가상 출력 예:

```text
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    link/loopback 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo

2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    link/ether 02:42:ac:11:00:02
    inet 192.168.10.24/24 scope global dynamic ens33
```

### 출력 해석

- `lo`: Loopback Interface
- `127.0.0.1/8`: 자기 자신과 통신하는 IPv4 Loopback 주소
- `ens33`: 유선 또는 가상 Network Interface 이름의 예
- `link/ether` 뒤의 값: MAC 주소
- `inet 192.168.10.24/24`: IPv4 주소와 CIDR
- `dynamic`: DHCP 등으로 동적으로 할당된 주소일 가능성

환경에 따라 `eth0`, `enp0s3`, `ens33`, `wlan0`, `wlp2s0` 같은 이름을 볼 수 있습니다.

### 직접 확인할 것

1. Loopback Interface 이름
2. 실제 통신에 사용하는 Interface 이름
3. IPv4 주소
4. Private IP 범위 포함 여부
5. CIDR 접두사 길이
6. MAC 주소
7. VPN이나 Docker Interface 존재 여부

---

## 15. `ip route` — Gateway와 Routing 확인

```bash
ip route
```

가상 출력 예:

```text
default via 192.168.10.1 dev ens33 proto dhcp src 192.168.10.24 metric 100
192.168.10.0/24 dev ens33 proto kernel scope link src 192.168.10.24
```

### 기본 경로

```text
default via 192.168.10.1 dev ens33
```

- `default`: 더 구체적인 경로가 없을 때 사용하는 기본 경로
- `via 192.168.10.1`: 다음으로 전달할 Default Gateway
- `dev ens33`: 사용할 Interface

### 직접 연결된 네트워크

```text
192.168.10.0/24 dev ens33
```

`192.168.10.0/24` 네트워크가 `ens33` Interface에 직접 연결되어 있다는 뜻입니다.

Wi-Fi, Ethernet, VPN을 동시에 사용하면 여러 경로가 보일 수 있습니다. 더 구체적인 경로, CIDR, Metric, VPN 정책 등이 경로 선택에 영향을 줍니다.

---

## 16. `ping` — 기본 도달 가능성 확인

`ping`은 일반적으로 ICMP Echo Request와 Echo Reply를 사용해 대상의 응답과 왕복 시간을 확인합니다.

### Loopback 확인

```bash
ping -c 4 127.0.0.1
```

### Default Gateway 확인

먼저 `ip route`에서 Gateway를 찾은 뒤 자신의 값으로 실행합니다.

```bash
ping -c 4 <기본_게이트웨이_IP>
```

### Domain 확인

```bash
ping -c 4 example.com
```

`example.com`은 문서와 테스트 용도로 예약된 Domain입니다.

### 결과에서 볼 것

- 응답이 도착했는가?
- Packet Loss는 얼마인가?
- 왕복 시간은 어느 정도인가?
- Domain을 IP 주소로 변환했는가?

### 해석 시 주의

`ping` 성공이 Web Server, SSH, 애플리케이션의 정상 동작을 보장하지 않습니다.

반대로 `ping` 실패가 대상 컴퓨터의 장애를 확정하지도 않습니다. 대상이나 중간 Firewall이 ICMP를 차단할 수 있기 때문입니다.

Linux에서 횟수 제한 없이 `ping`을 실행했다면 `Ctrl+C`로 중지할 수 있습니다.

---

## 17. `ss` — Socket과 Port 확인

`ss`는 현재 시스템의 Network Socket 상태를 확인하는 도구입니다.

```bash
ss -tuln
```

| 옵션 | 의미 |
| --- | --- |
| `-t` | TCP Socket |
| `-u` | UDP Socket |
| `-l` | Listening Socket |
| `-n` | Service 이름 대신 숫자 주소와 Port 표시 |

TCP 연결:

```bash
ss -tn
```

UDP Socket:

```bash
ss -lun
```

Process 정보와 함께 확인:

```bash
sudo ss -ltnp
```

`-p`는 Socket을 사용하는 Process를 표시합니다. 일부 정보에는 관리자 권한이 필요할 수 있습니다.

가상 출력 예:

```text
Netid State  Local Address:Port  Peer Address:Port
tcp   LISTEN 127.0.0.1:8000     0.0.0.0:*
tcp   LISTEN 0.0.0.0:22         0.0.0.0:*
udp   UNCONN 127.0.0.53:53      0.0.0.0:*
```

### `127.0.0.1:8000`

Loopback의 TCP 8000번 Port에서 연결을 기다리는 예입니다. 기본적으로 같은 컴퓨터 안에서 접근합니다.

### `0.0.0.0:22`

모든 IPv4 Interface의 TCP 22번 Port에서 연결을 기다리는 예입니다.

다른 장치의 실제 접근 가능 여부는 Routing, OS Firewall, NAT, Port Forwarding, Cloud Security Group, VPN 정책, 서비스 접근 제어에 따라 달라집니다.

### 상태

- `LISTEN`: 새로운 TCP 연결을 기다림
- `ESTAB`: TCP 연결이 성립됨
- `UNCONN`: UDP처럼 연결을 맺지 않는 Socket에서 볼 수 있음

Listening Port가 있다는 것은 로컬 프로그램이 요청을 받을 준비를 했다는 뜻이지, 곧바로 인터넷에 공개되었다는 뜻은 아닙니다.

---

## 18. 실습 1 — 내 네트워크 정보 조사

```bash
ip addr
ip route
```

아래 표를 자신의 결과로 채웁니다. 공개 저장소에는 실제 Public IP, Hostname, VPN 주소와 원격 연결 상대를 필요한 만큼 마스킹합니다.

| 항목 | 확인한 값 |
| --- | --- |
| Loopback Interface | |
| 인터넷 통신에 사용하는 Interface | |
| Private IPv4 주소 | |
| CIDR 접두사 | |
| MAC 주소 | |
| Default Gateway | |
| 직접 연결된 Network | |
| VPN 또는 가상 Interface | |

완료 기준:

- [ ] `ip addr`에서 IPv4 주소를 찾았다.
- [ ] Public 또는 Private 범위를 구분했다.
- [ ] `ip route`에서 Default Gateway를 찾았다.
- [ ] 실제 통신에 사용하는 Interface를 확인했다.

---

## 19. 실습 2 — 구간별 연결 확인

```bash
ping -c 4 127.0.0.1
ping -c 4 <기본_게이트웨이_IP>
ping -c 4 example.com
```

| 대상 | 성공 여부 | Packet Loss | 해석 |
| --- | --- | ---: | --- |
| `127.0.0.1` | | | |
| Default Gateway | | | |
| `example.com` | | | |

응답하지 않는 대상이 있어도 즉시 장애라고 단정하지 말고 ICMP 차단 가능성을 함께 기록합니다.

---

## 20. 실습 3 — Listening Port 조사

```bash
ss -tuln
```

필요하면:

```bash
sudo ss -ltnp
```

표시된 Socket 가운데 최대 세 개를 골라 기록합니다.

| Protocol | Local Address | Port | 상태 | 노출 범위 추정 |
| --- | --- | ---: | --- | --- |
| | | | | |
| | | | | |
| | | | | |

노출 범위의 1차 판단:

- `127.0.0.1`: 로컬 전용
- 특정 Private IP: 해당 Interface에서 대기
- `0.0.0.0`: 모든 IPv4 Interface에서 대기
- `[::]`: 모든 IPv6 Interface에서 대기

최종 접근 가능 여부는 Firewall, Routing, NAT 등과 함께 확인해야 합니다.

---

## 21. 실습 4 — Local Web Server 관찰

> 자신이 관리하는 Ubuntu 환경에서만 수행합니다. 현재 디렉터리의 파일이 Web으로 제공될 수 있으므로 개인 정보나 비밀 파일이 있는 곳에서는 실행하지 않습니다.

빈 연습 디렉터리에서 Local 전용 Web Server를 실행합니다.

```bash
mkdir -p ~/network-practice/day01
cd ~/network-practice/day01
python3 -m http.server 8000 --bind 127.0.0.1
```

다른 Terminal에서 확인:

```bash
ss -ltn
```

다음 형태의 항목을 찾습니다.

```text
127.0.0.1:8000
```

Browser에서 접속:

```text
http://127.0.0.1:8000
```

실습을 마치면 Server를 실행한 Terminal에서 `Ctrl+C`로 종료합니다.

관찰할 내용:

1. Server 실행 전후 `ss -ltn` 결과는 어떻게 달라졌는가?
2. 왜 `127.0.0.1`에 Binding했는가?
3. `0.0.0.0`에 Binding한다면 노출 범위가 어떻게 달라질 수 있는가?
4. 종료 후 8000번 Listening 항목이 사라졌는가?

---

## 22. 실습 5 — 내 네트워크 구조 그리기

확인한 정보를 다음 형식으로 그립니다.

```text
[내 Ubuntu]
IP: <Private_IP>
Interface: <이름>
        │
        ▼
[Default Gateway]
IP: <Gateway_IP>
        │
        ▼
[NAT / ISP]
        │
        ▼
[Internet]
        │
        ▼
[Web / API / LLM Server]
Port: 443 등
```

Virtual Machine, WSL, Docker, VPN을 사용한다면 중간 Network 계층이 추가될 수 있습니다. 직접 확인한 정보와 추정한 정보를 구분해 표시합니다.

---

## 23. 네트워크 문제 확인 순서

```text
1. Interface가 존재하고 활성화되어 있는가?
                    ↓
2. IP 주소가 할당되어 있는가?
                    ↓
3. Default Gateway와 Route가 있는가?
                    ↓
4. Loopback과 Gateway에 도달할 수 있는가?
                    ↓
5. 외부 목적지까지 도달할 수 있는가?
                    ↓
6. 필요한 서비스가 올바른 주소와 Port에서 대기하는가?
```

| 확인 대상 | 명령어 |
| --- | --- |
| Interface, IP, MAC | `ip addr` |
| Gateway와 경로 | `ip route` |
| 기본 도달 가능성 | `ping` |
| 로컬 Socket과 Port | `ss -tuln` |

예를 들어 `ping`은 성공하지만 Web 페이지가 열리지 않는다면 Web Server, TCP Port, Firewall, Application, Proxy, TLS 설정을 추가로 확인해야 합니다. ICMP 응답과 Web 서비스의 정상 동작은 서로 다른 문제입니다.

---

## 24. AI 보안과의 연결

### LLM API도 네트워크 서비스다

AI 애플리케이션이 외부 LLM API를 호출할 때 Domain, IP, HTTPS Port, DNS, Routing, TLS, HTTP가 모두 사용됩니다.

API Key가 안전해도 내부 관리 Server가 불필요한 Interface에 노출되어 있거나 Network 접근 제어가 잘못되면 위험할 수 있습니다.

### RAG는 여러 서비스의 연결이다

```text
사용자
  │
  ▼
Web Application
  │
  ├── LLM API
  ├── Embedding API
  ├── Vector Database
  ├── Document Storage
  └── Internal Search Service
```

각 연결에는 IP, Port, 인증, 암호화, 접근 제어가 필요합니다.

Vector Database나 관리 API를 `0.0.0.0`에 Binding하고 Firewall까지 열어 두면 의도하지 않은 Network에 노출될 수 있습니다.

### AI Agent는 외부 요청을 보낼 수 있다

Web, Email, Calendar, Database Tool을 사용하는 Agent에서는 다음을 확인해야 합니다.

- Agent가 어느 목적지에 연결할 수 있는가?
- 내부 전용 서비스에도 접근할 수 있는가?
- 외부로 민감한 데이터를 보낼 수 있는가?
- 허용되지 않은 Port나 Protocol을 사용할 수 있는가?
- Network Log에서 출발지와 목적지를 추적할 수 있는가?

이 내용은 이후 학습할 SSRF, Data Leakage, Tool 오용, Network 접근 제어와 연결됩니다.

### Container의 localhost

Docker 같은 Container 환경에서 `127.0.0.1`은 Host가 아니라 현재 Container 자신을 의미합니다.

AI Application과 Database가 서로 다른 Container에 있다면 `localhost`만으로 연결되지 않을 수 있습니다. Container Network와 Service 이름을 별도로 이해해야 합니다.

### Port 변경만으로는 안전해지지 않는다

SSH를 22번이 아닌 다른 Port로 옮겨도 다음 보안 통제를 대신할 수 없습니다.

- 강력한 인증
- 최소 권한
- Firewall
- 접근 허용 목록
- 보안 Update
- Log와 Monitoring

---

## 25. 자주 하는 오해

### 오해 1 — 컴퓨터 한 대에는 IP가 하나뿐이다

유선, Wi-Fi, VPN, 가상 Network, Loopback Interface마다 다른 주소를 가질 수 있습니다.

### 오해 2 — IP 주소는 사람을 영구 식별한다

DHCP, NAT, VPN, 이동통신망 때문에 IP는 바뀌거나 여러 사용자가 공유할 수 있습니다.

### 오해 3 — Private IP면 공격받지 않는다

내부 장치, VPN, Port Forwarding, Cloud 설정 오류 등을 통해 접근될 수 있습니다.

### 오해 4 — NAT는 Firewall이다

NAT는 주소 변환이고 Firewall은 허용·차단 정책을 집행합니다.

### 오해 5 — `ping` 실패는 Server가 꺼졌다는 뜻이다

ICMP가 차단되어도 Web이나 SSH 서비스는 정상일 수 있습니다.

### 오해 6 — Listening Port는 인터넷에 공개된 Port다

실제 접근 가능성은 Binding 주소, Routing, Firewall, NAT 설정을 함께 봐야 합니다.

### 오해 7 — Port 번호만 보면 Program을 확정할 수 있다

Port는 관례이며 Service는 다른 Port에서도 실행될 수 있습니다.

---

## 26. 자가 테스트

답을 보기 전에 자신의 말로 설명합니다.

1. Network에서 IP 주소와 Port는 각각 무엇을 구분하는가?
2. Client와 Server 역할은 항상 고정되어 있는가?
3. `192.168.10.24`는 Public IP인가, Private IP인가?
4. `127.0.0.1`은 어떤 용도로 사용하는가?
5. `192.168.10.24/24`에서 `/24`는 무엇을 뜻하는가?
6. IP 주소와 MAC 주소의 차이는 무엇인가?
7. Router와 Default Gateway의 역할은 무엇인가?
8. NAT와 Firewall의 차이는 무엇인가?
9. `127.0.0.1:8000`과 `0.0.0.0:8000`의 차이는 무엇인가?
10. `ping` 실패만으로 Server 장애를 확정할 수 없는 이유는 무엇인가?
11. `ss -tuln`은 무엇을 보여 주는가?
12. Listening Port가 보이면 인터넷에 공개된 것인가?
13. AI Agent의 Network 접근 범위를 제한해야 하는 이유는 무엇인가?

<details>
<summary>자가 테스트 답안 보기</summary>

1. IP는 네트워크의 목적지 장치 또는 Interface를, Port는 그 안의 서비스를 구분한다.
2. 고정되지 않는다. 특정 통신에서 요청하는 쪽이 Client이고 제공하는 쪽이 Server다.
3. `192.168.0.0/16`에 포함되므로 Private IP다.
4. 컴퓨터가 자기 자신과 통신할 때 사용하는 Loopback 주소다.
5. 앞의 24bit가 Network 부분이라는 CIDR 표기다.
6. IP는 Network 간 주소 지정과 Routing에, MAC은 주로 로컬 구간의 Interface 식별에 사용된다.
7. Router는 Network 사이에서 Packet을 전달하고, Default Gateway는 외부로 나갈 때 사용하는 기본 출구다.
8. NAT는 주소나 Port를 변환하고, Firewall은 정책에 따라 통신을 허용하거나 차단한다.
9. `127.0.0.1`은 Loopback에서만, `0.0.0.0`은 모든 IPv4 Interface에서 연결을 기다린다.
10. Firewall이 ICMP를 차단해도 Server의 다른 서비스는 정상일 수 있기 때문이다.
11. TCP와 UDP의 Listening Socket을 숫자 주소와 Port 형태로 보여 준다.
12. 아니다. Binding, Routing, Firewall, NAT 등을 함께 확인해야 한다.
13. 내부 서비스 무단 접근과 외부 Data 유출 위험을 줄이기 위해서다.

</details>

---

## 27. 오늘 반드시 기억할 핵심

```text
Network = 여러 장치가 데이터를 주고받도록 연결된 구조

Client = 서비스를 요청하는 쪽
Server = 서비스를 제공하는 쪽

IP 주소 = 어느 컴퓨터 또는 Interface와 통신할지 나타내는 주소
MAC 주소 = 로컬 Network 구간의 Interface 식별 주소
Port = 한 장치 안의 서비스를 구분하는 번호

127.0.0.1 = 자기 자신을 가리키는 Loopback 주소
0.0.0.0 Binding = 모든 IPv4 Interface에서 연결 대기

Router = 서로 다른 Network 사이에서 Packet 전달
Default Gateway = 외부 Network로 나가는 기본 출구
NAT = Private 주소와 Public 주소/Port 사이의 변환
Firewall = 정책에 따라 Network 통신 허용 또는 차단
```

핵심 명령어:

```bash
ip addr
ip route
ping -c 4 127.0.0.1
ss -tuln
```

> **IP 주소는 어느 컴퓨터 또는 인터페이스와 통신할지 나타내고, Port는 그 안의 어느 서비스와 통신할지 나타냅니다.**

---

## 28. 학습 기록

실습 후 직접 작성합니다. 실제 Public IP, Hostname, VPN 주소처럼 공개가 불필요한 정보는 마스킹합니다.

```text
오늘 새롭게 알게 된 개념:

IP 주소와 Port의 차이를 내 말로 설명:

내 환경에서 확인한 Network Interface:

Default Gateway가 하는 일:

127.0.0.1과 0.0.0.0의 차이:

ss 명령어로 확인한 내용:

AI 보안과 연결되는 부분:

실행 중 발생한 오류:

오류를 해결한 방법:

아직 이해가 부족한 부분:

다음 학습에서 확인하고 싶은 질문:
```


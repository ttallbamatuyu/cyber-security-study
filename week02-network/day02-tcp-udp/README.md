# Day 02 — TCP와 UDP: 연결, 신뢰성, 속도

> 작성일: 2026-08-15  
> 학습 주제: TCP, UDP, 3-Way Handshake, Socket 상태 확인

Day 01에서 IP가 컴퓨터 또는 네트워크 인터페이스를 찾는 주소이고 Port가 그 안의 서비스를 구분하는 번호라는 것을 배웠다.

오늘은 그 IP와 Port 사이에서 실제 데이터를 어떤 방식으로 전달하는지 공부했다. 처음에는 TCP는 느리고 UDP는 빠르다는 정도로만 알고 있었지만, 정리하면서 두 Protocol의 핵심 차이는 단순한 속도가 아니라 **연결 상태와 신뢰성을 누가 책임지는가**에 있다는 것을 이해했다.

이 문서는 강의 내용을 그대로 옮긴 것이 아니라, 내가 이해한 내용을 다시 설명하고 실습할 때 확인할 기준을 정리한 학습 기록이다.

## 오늘 확인한 목표

- [o] TCP와 UDP가 Transport Layer Protocol이라는 것을 이해했다.
- [o] TCP가 연결 지향적이라는 말의 의미를 설명할 수 있다.
- [o] TCP의 3-Way Handshake 흐름을 설명할 수 있다.
- [o] TCP가 순서, 재전송, 흐름 제어를 어떻게 다루는지 큰 그림을 이해했다.
- [o] UDP가 연결 과정 없이 Datagram을 전달한다는 것을 이해했다.
- [o] UDP가 무조건 TCP보다 빠른 것은 아니라는 점을 이해했다.
- [o] TCP와 UDP를 사용하는 대표적인 서비스를 구분할 수 있다.
- [o] `ss`로 TCP 연결과 UDP Socket을 확인하는 방법을 정리했다.
- [o] `nc`를 이용한 로컬 TCP/UDP 통신 실습 방법을 정리했다.
- [o] TCP와 UDP의 차이가 보안에 어떤 영향을 주는지 생각해 보았다.

---

## 1. 내가 이해한 전체 그림

Application이 Network로 데이터를 보낼 때 IP만으로는 충분하지 않다.

IP는 목적지 장치까지 찾아가는 역할을 하고, TCP 또는 UDP는 Application의 데이터를 어떤 방식으로 전달할지 정한다.

```text
Application
HTTP / DNS / SSH / Game / LLM API
            │
            ▼
Transport
TCP 또는 UDP
            │
            ▼
Internet
IP
            │
            ▼
Network Interface
Ethernet / Wi-Fi
```

내가 기억할 구분은 다음과 같다.

```text
IP       = 어느 장치로 보낼 것인가?
Port     = 그 장치의 어느 서비스로 보낼 것인가?
TCP/UDP  = 데이터를 어떤 방식으로 전달할 것인가?
```

TCP와 UDP는 모두 Port 번호를 사용한다. 같은 53번 Port라도 `TCP 53`과 `UDP 53`은 서로 다른 Socket으로 취급된다.

---

## 2. TCP를 내가 이해한 방식

TCP는 `Transmission Control Protocol`의 약자다.

나는 TCP를 **상대방과 먼저 연결 상태를 만든 뒤, 데이터가 빠지거나 순서가 바뀌었는지 확인하면서 전달하는 방식**으로 이해했다.

TCP의 핵심 특징을 내 말로 정리하면 다음과 같다.

- 데이터를 보내기 전에 연결을 설정한다.
- 보낸 데이터가 도착했는지 ACK로 확인한다.
- 손실된 것으로 판단한 데이터는 다시 보낸다.
- 데이터의 순서를 맞춘다.
- 수신자가 감당할 수 있는 양을 고려한다.
- Network 혼잡 상태를 고려해 전송량을 조절한다.
- Application에는 연속된 Byte Stream처럼 데이터를 제공한다.

### 연결 지향이라는 말

연결 지향이라는 말이 실제 전용 선을 하나 만든다는 뜻은 아니다.

두 장치가 Sequence Number와 현재 상태를 관리하면서 서로 통신할 준비가 되었음을 확인한다는 의미로 이해했다.

```text
Client 상태                         Server 상태
연결 전                             연결 대기
   │                                   │
   └──── Handshake로 상태 확인 ────────┘
                   │
                   ▼
              데이터 통신
```

TCP는 Client와 Server 양쪽에서 연결 상태를 기억한다. 그래서 운영체제에서 `ESTAB`, `SYN-SENT`, `TIME-WAIT` 같은 상태를 확인할 수 있다.

---

## 3. TCP 3-Way Handshake

TCP는 일반적으로 데이터를 주고받기 전에 세 단계의 Handshake를 진행한다.

```text
Client                                      Server
  │                                            │
  │  1. SYN                                    │
  ├───────────────────────────────────────────>│
  │                                            │
  │  2. SYN + ACK                              │
  │<───────────────────────────────────────────┤
  │                                            │
  │  3. ACK                                    │
  ├───────────────────────────────────────────>│
  │                                            │
  │              연결 성립                    │
```

### 1단계 — SYN

Client가 Server에 연결하고 싶다는 의미로 SYN을 보낸다.

```text
Client: 연결을 시작하고 싶다.
```

이때 Client는 자신이 사용할 초기 Sequence Number도 전달한다.

### 2단계 — SYN + ACK

Server는 Client의 요청을 받았다는 ACK와 자신도 연결 준비가 되었다는 SYN을 함께 보낸다.

```text
Server: 요청을 받았고 나도 준비되었다.
```

### 3단계 — ACK

Client가 Server의 SYN을 확인했다는 ACK를 보낸다.

```text
Client: Server의 응답도 확인했다.
```

이 과정이 끝나면 양쪽은 서로의 통신 준비 상태와 초기 Sequence Number를 확인한 상태가 된다.

### 왜 세 번일까?

처음에는 단순히 서로 한 번씩 응답하면 두 번이면 충분하지 않을까 생각했다.

하지만 TCP는 Client의 송신 가능 여부만 확인하는 것이 아니라, 양쪽이 서로의 요청과 응답을 모두 확인해야 한다.

```text
1. Server가 Client의 SYN을 받음
2. Client가 Server의 SYN과 ACK를 받음
3. Server가 Client의 마지막 ACK를 받음
```

마지막 ACK까지 받아야 Server도 자신의 응답이 Client에 도착했다는 것을 알 수 있다.

---

## 4. Sequence Number와 ACK

TCP는 Byte Stream의 위치를 나타내는 Sequence Number를 사용한다.

개념을 단순화하면 다음과 같이 이해할 수 있다.

```text
보낸 데이터: 1번, 2번, 3번
받은 쪽:     1번과 2번까지 받음
ACK:         다음에는 3번을 보내 달라고 알림
```

실제 TCP의 ACK는 일반적으로 **다음에 받기를 기대하는 Sequence Number**를 나타낸다.

### 순서 보장

Packet은 Network 상황에 따라 다른 경로를 지나거나 도착 순서가 달라질 수 있다.

TCP는 Sequence Number를 이용해 원래 순서로 정리한 뒤 Application에 전달한다.

```text
Network에서 도착: 1 → 3 → 2
Application에 전달: 1 → 2 → 3
```

### 재전송

보낸 데이터에 대한 ACK가 일정 시간 안에 오지 않거나 손실이 감지되면 TCP는 데이터를 다시 보낼 수 있다.

이 기능 때문에 Application이 Packet 손실 처리 전부를 직접 구현하지 않아도 된다.

### 중복 처리

재전송 때문에 같은 데이터가 중복 도착할 수도 있다. TCP는 Sequence Number를 이용해 중복된 Byte를 구분한다.

---

## 5. 흐름 제어와 혼잡 제어

처음에는 두 개념이 같은 것처럼 느껴졌지만 대상이 다르다.

### 흐름 제어

흐름 제어는 **수신자가 처리할 수 있는 속도**를 고려하는 것이다.

```text
Sender ── 너무 많은 데이터 ──> Receiver Buffer 부족
```

수신자는 자신이 받을 수 있는 Window 크기를 알리고, 송신자는 그 범위를 고려해 데이터를 보낸다.

### 혼잡 제어

혼잡 제어는 **Network 전체의 혼잡 상태**를 고려하는 것이다.

```text
많은 장치가 동시에 전송
          ↓
Router와 Network 경로 혼잡
          ↓
지연과 Packet 손실 증가
```

TCP는 손실과 지연 같은 신호를 이용해 전송량을 조절한다.

내가 기억할 문장:

```text
흐름 제어 = 받는 장치가 감당할 수 있는가?
혼잡 제어 = Network 경로가 감당할 수 있는가?
```

---

## 6. TCP가 보장하지 않는 것

TCP가 신뢰성 있는 Protocol이라고 해서 모든 것을 보장하는 것은 아니다.

TCP가 주로 제공하는 것은 Byte의 전달, 순서, 중복 처리, 재전송이다.

TCP만으로는 다음을 보장하지 않는다.

- Application의 요청이 올바른 의미를 가지는지
- Server가 요청 처리를 성공했는지
- 사용자가 인증된 사람인지
- 전송 내용이 암호화되었는지
- Application의 취약점이 없는지

예를 들어 TCP 연결이 성공해도 HTTP 응답은 `500 Internal Server Error`일 수 있다.

또한 TCP 자체는 암호화 Protocol이 아니다. HTTPS에서는 TCP 위에 TLS를 사용해 기밀성과 무결성을 보호한다. HTTP/3은 TCP 대신 UDP 기반 QUIC를 사용하지만 암호화 기능을 Protocol에 포함한다.

---

## 7. TCP 연결 종료

TCP 연결을 정상적으로 종료할 때는 일반적으로 FIN과 ACK가 오간다.

```text
한쪽                                          반대쪽
 │  FIN                                          │
 ├──────────────────────────────────────────────>│
 │  ACK                                          │
 │<──────────────────────────────────────────────┤
 │  FIN                                          │
 │<──────────────────────────────────────────────┤
 │  ACK                                          │
 ├──────────────────────────────────────────────>│
```

이를 흔히 4-Way Handshake라고 부른다. 실제 Packet은 상황에 따라 FIN과 ACK가 함께 전달되는 등 형태가 달라질 수 있다.

### FIN과 RST

- `FIN`: 남은 데이터를 정리하며 정상적으로 연결을 닫을 때 사용
- `RST`: 연결을 즉시 중단하거나 예상하지 못한 연결을 거절할 때 사용

### TIME-WAIT

연결을 먼저 종료한 쪽에서 `TIME-WAIT` 상태를 볼 수 있다.

늦게 도착한 Packet이 다음 연결과 섞이는 것을 막고 마지막 ACK가 손실되었을 때 다시 처리할 시간을 확보하는 상태라고 이해했다.

`TIME-WAIT`이 보인다고 무조건 오류는 아니다.

---

## 8. UDP를 내가 이해한 방식

UDP는 `User Datagram Protocol`의 약자다.

나는 UDP를 **상대방과 연결 상태를 먼저 만들지 않고 각각의 Datagram을 독립적으로 보내는 단순한 전달 방식**으로 이해했다.

UDP의 특징:

- 3-Way Handshake가 없다.
- 연결 상태를 계속 관리하지 않는다.
- 전달 성공을 Protocol 자체에서 확인하지 않는다.
- 순서를 보장하지 않는다.
- 재전송을 기본 제공하지 않는다.
- 중복 Datagram이 도착할 수 있다.
- TCP보다 Header와 상태 관리 부담이 작다.
- Message 단위인 Datagram 경계를 유지한다.

```text
Sender
  │
  ├── Datagram 1 ───────────────────> Receiver
  ├── Datagram 2 ──────X 손실
  └── Datagram 3 ───────────────────> Receiver
```

UDP는 두 번째 Datagram이 손실되어도 스스로 다시 보내지 않는다. 필요한 경우 Application이 재전송, 순서, 오류 복구 기능을 구현해야 한다.

---

## 9. UDP를 사용하는 이유

UDP가 신뢰성을 기본 제공하지 않는다고 해서 좋지 않은 Protocol은 아니다.

다음과 같이 최신 정보가 더 중요하거나 Application이 전달 방식을 직접 제어해야 하는 상황에서 유용하다.

- 실시간 음성·영상
- 온라인 Game
- DNS Query
- 실시간 상태 정보
- 일부 VPN
- QUIC와 HTTP/3

예를 들어 실시간 음성 통화에서 오래된 음성 Packet을 늦게 재전송하는 것보다 일부를 포기하고 최신 음성을 계속 전달하는 편이 자연스러울 수 있다.

다만 **UDP는 항상 TCP보다 빠르다**고 단정하면 안 된다.

실제 속도와 지연은 Network 품질, Application 구현, 암호화, 재전송 전략, 혼잡 제어 방식에 따라 달라진다.

QUIC는 UDP 위에서 연결 관리, 신뢰성, 혼잡 제어, 암호화 같은 기능을 구현한다. UDP를 사용한다고 신뢰성 기능을 전혀 사용할 수 없는 것은 아니다.

---

## 10. TCP와 UDP 비교

| 비교 항목 | TCP | UDP |
| --- | --- | --- |
| 연결 설정 | 3-Way Handshake | 별도 Handshake 없음 |
| 연결 상태 관리 | 함 | 기본적으로 하지 않음 |
| 전달 확인 | ACK 사용 | 기본 제공하지 않음 |
| 순서 보장 | 제공 | 제공하지 않음 |
| 재전송 | 제공 | 제공하지 않음 |
| 데이터 형태 | Byte Stream | Datagram |
| Header와 관리 부담 | 상대적으로 큼 | 상대적으로 작음 |
| 대표 용도 | SSH, HTTP/1.1, HTTP/2, Email | DNS, 실시간 통신, QUIC |

### 내가 정리한 선택 기준

```text
정확한 순서와 빠짐없는 전달이 중요
                 ↓
               TCP

낮은 지연과 개별 Message 전달이 중요
Application이 손실 처리를 직접 결정
                 ↓
               UDP
```

Protocol 선택은 단순히 어느 쪽이 더 좋다는 문제가 아니라 Application이 필요한 특성에 따라 달라진다.

---

## 11. 대표적인 사용 예

| Service 또는 Protocol | 일반적인 Transport | 내가 이해한 이유 |
| --- | --- | --- |
| SSH | TCP | 명령과 출력의 정확한 순서가 중요 |
| HTTP/1.1, HTTP/2 | TCP | 요청과 응답 데이터를 신뢰성 있게 전달 |
| HTTPS | TCP + TLS가 일반적 | 전달 신뢰성과 암호화 필요 |
| HTTP/3 | UDP 기반 QUIC | 낮은 지연과 독립 Stream 처리 |
| DNS | UDP 또는 TCP | 일반 Query는 UDP가 많지만 큰 응답 등에는 TCP 사용 |
| Email | TCP | Message의 정확한 전달이 중요 |
| 실시간 음성·영상 | UDP를 자주 사용 | 오래된 Packet 재전송보다 낮은 지연이 중요할 수 있음 |

### DNS는 무조건 UDP일까?

DNS는 일반적인 작은 Query에서 UDP 53번 Port를 많이 사용한다.

하지만 응답이 크거나 잘렸을 때, Zone Transfer를 수행할 때 등에는 TCP를 사용할 수 있다.

따라서 `DNS = UDP`라고만 외우기보다 **DNS는 상황에 따라 UDP와 TCP를 모두 사용한다**고 기억하는 것이 정확하다.

---

## 12. Socket과 5-Tuple

Network 연결을 구분할 때 다음 다섯 정보를 함께 볼 수 있다.

```text
Protocol
Source IP
Source Port
Destination IP
Destination Port
```

이를 5-Tuple이라고 부른다.

예:

```text
TCP
192.168.10.24:53122
       ↓
198.51.100.20:443
```

같은 Server의 443번 Port에 여러 Client가 동시에 접속할 수 있는 이유는 각 연결의 Source IP와 Source Port 조합이 다르기 때문이다.

```text
Client A 192.168.10.24:53122 ─┐
Client B 192.168.10.25:49810 ─┼──> Server 198.51.100.20:443
Client C 192.168.10.26:61003 ─┘
```

위 Public IP 대역은 문서 작성을 위한 예시다.

---

## 13. `ss`로 TCP 상태 확인

오늘 정리한 핵심 명령어:

| 명령어 | 내가 확인할 내용 |
| --- | --- |
| `ss -tn` | 현재 TCP Socket |
| `ss -tan` | Listening을 포함한 모든 TCP Socket |
| `ss -ltn` | TCP Listening Port |
| `ss -tn state established` | 연결이 성립된 TCP Socket |
| `ss -un` | UDP Socket |
| `ss -lun` | UDP Socket과 Local Port |
| `ss -s` | Protocol별 Socket 요약 |

### TCP 연결 확인

```bash
ss -tn
```

### TCP Listening Port 확인

```bash
ss -ltn
```

### 모든 TCP 상태 확인

```bash
ss -tan
```

### UDP Socket 확인

```bash
ss -un
ss -lun
```

### 요약 확인

```bash
ss -s
```

Process 정보까지 보고 싶을 때:

```bash
sudo ss -ltnp
```

`-p`는 Socket을 사용하는 Process를 표시한다. 일부 정보에는 관리자 권한이 필요할 수 있다.

---

## 14. `ss` 결과를 읽는 기준

가상 예시:

```text
State   Local Address:Port       Peer Address:Port
LISTEN  127.0.0.1:9000          0.0.0.0:*
ESTAB   127.0.0.1:9000          127.0.0.1:53214
```

이 출력은 형식을 설명하기 위한 예시이며 내 실제 환경의 결과가 아니다.

### LISTEN

```text
127.0.0.1:9000
```

내 컴퓨터의 Loopback 9000번 Port에서 TCP 연결을 기다리고 있다는 의미다.

### ESTAB

```text
127.0.0.1:9000 ↔ 127.0.0.1:53214
```

TCP 연결이 성립되어 양쪽 Socket이 통신 중이라는 의미다.

### 자주 볼 수 있는 TCP 상태

| 상태 | 내가 이해한 의미 |
| --- | --- |
| `LISTEN` | Server가 새 연결을 기다림 |
| `SYN-SENT` | SYN을 보내고 응답을 기다림 |
| `SYN-RECV` | SYN을 받고 마지막 ACK를 기다림 |
| `ESTAB` | 연결이 성립됨 |
| `FIN-WAIT-1/2` | 연결 종료를 진행 중 |
| `CLOSE-WAIT` | 상대의 종료 요청을 받고 Application 종료를 기다림 |
| `TIME-WAIT` | 늦은 Packet과 마지막 ACK 처리를 위해 잠시 유지 |

상태 하나만 보고 장애를 단정하지 않고 Application Log와 시간에 따른 변화를 함께 확인해야 한다.

---

## 15. 로컬 TCP 통신 실습 정리

`nc`는 Netcat으로, TCP 또는 UDP 연결을 만들고 데이터를 주고받을 수 있는 도구다.

Ubuntu 환경에 따라 Package와 Option이 다를 수 있으므로 먼저 도움말을 확인한다.

```bash
nc -h
```

실습은 내 컴퓨터의 Loopback에서만 진행하도록 정리했다.

### Terminal A — TCP Server

```bash
nc -lv 9000
```

### Terminal B — TCP Client

```bash
nc 127.0.0.1 9000
```

연결 후 양쪽 Terminal에서 Text를 입력해 전달되는지 확인한다.

### Terminal C — Socket 상태

```bash
ss -tn
ss -ltn
```

### 내가 확인할 항목

- 연결 전에는 9000번 Port가 `LISTEN` 상태인가?
- Client가 연결하면 `ESTAB` 상태가 생기는가?
- Server와 Client의 Local/Peer Port는 어떻게 표시되는가?
- Client 쪽에는 임시 Port가 할당되는가?
- 한쪽을 종료하면 상태가 어떻게 변하는가?
- 잠시 `TIME-WAIT` 상태가 보이는가?

실습 종료는 각 Terminal에서 `Ctrl+C`를 사용한다.

---

## 16. 로컬 UDP 통신 실습 정리

UDP도 Loopback에서만 실습하도록 정리했다.

### Terminal A — UDP 수신

```bash
nc -ulv 9001
```

### Terminal B — UDP 송신

```bash
nc -u 127.0.0.1 9001
```

### Terminal C — UDP Socket 확인

```bash
ss -lun
```

### TCP 실습과 비교할 점

- UDP에는 3-Way Handshake가 보이지 않는다.
- TCP와 같은 `ESTAB` 상태가 기본적으로 보이지 않는다.
- Text는 전달될 수 있지만 Protocol 자체의 ACK와 재전송은 없다.
- 수신 Program이 실행되지 않아도 송신 쪽에서 즉시 오류를 알지 못할 수 있다.
- Netcat 구현에 따라 종료와 출력 동작이 다를 수 있다.

이 실습의 목적은 UDP가 아무것도 못 한다는 것을 보여 주는 것이 아니라, TCP처럼 연결 상태와 전달 확인을 기본 제공하지 않는다는 것을 확인하는 것이다.

---

## 17. Packet을 관찰할 때 찾을 것

Packet Capture는 내 컴퓨터 또는 명시적으로 허가받은 환경에서만 수행한다.

Loopback의 TCP 9000번 Port를 관찰하는 예:

```bash
sudo tcpdump -i lo -nn 'tcp port 9000'
```

UDP 9001번 Port를 관찰하는 예:

```bash
sudo tcpdump -i lo -nn 'udp port 9001'
```

### TCP에서 찾을 것

```text
SYN
SYN, ACK
ACK
PSH, ACK
FIN, ACK
```

### UDP에서 찾을 것

```text
Handshake 없이 개별 UDP Datagram 전달
```

Wireshark를 사용할 때의 Display Filter:

```text
tcp.port == 9000
udp.port == 9001
```

실제 Interface 이름은 환경에 따라 `lo`가 아닐 수 있으므로 Capture 전에 확인해야 한다.

---

## 18. 실습 기록표

실제 명령 결과를 그대로 공개하기보다 필요한 정보만 마스킹해 기록한다.

| 확인 항목 | TCP 실습에서 본 것 | UDP 실습에서 본 것 |
| --- | --- | --- |
| 사용한 Port | | |
| Handshake | | |
| `ss` 상태 | | |
| 임시 Port | | |
| 데이터 단위 | | |
| 종료 후 상태 | | |
| 새롭게 알게 된 점 | | |

내부 Hostname, Public IP, VPN 주소, 원격 연결 상대처럼 공개할 필요가 없는 정보는 가린다.

---

## 19. 보안과 연결해서 생각한 점

### TCP SYN Flood

TCP Server는 SYN을 받고 마지막 ACK를 기다리는 동안 연결 정보를 일정 시간 관리한다.

공격자가 많은 SYN을 보내고 Handshake를 끝내지 않으면 Server의 연결 대기 자원을 소모시킬 수 있다. 이를 SYN Flood라고 한다.

```text
다수의 SYN
    ↓
Server의 Half-open 연결 증가
    ↓
정상 Client 연결 방해 가능
```

보안 장비, SYN Cookie, Rate Limit, 적절한 Queue 설정 같은 방어가 사용될 수 있다.

### UDP Spoofing과 Reflection

UDP는 TCP와 같은 Handshake가 없으므로 Source IP가 위조된 Datagram을 악용하는 공격과 연결되기 쉽다.

공격자는 피해자의 IP를 Source로 위조해 공개 UDP Server에 요청을 보내고, 응답이 피해자에게 향하게 만들 수 있다.

요청보다 응답이 훨씬 큰 Service라면 증폭 효과가 생길 수 있다.

```text
위조된 작은 요청
      ↓
공개 UDP Service
      ↓
더 큰 응답이 피해자에게 전달
```

DNS, NTP 등 UDP Service를 운영할 때 외부 공개 범위와 증폭 가능성을 점검해야 한다.

### Listening Port는 공격 표면이다

`ss -ltn`과 `ss -lun`으로 확인되는 Socket은 Local Program이 요청을 받을 준비를 한 상태다.

불필요한 Service가 모든 Interface에 Binding되어 있다면 공격 표면이 넓어진다.

확인할 내용:

- 이 Service가 꼭 필요한가?
- `127.0.0.1`에만 Binding해도 되는가?
- `0.0.0.0` 또는 `[::]`에 열려 있는가?
- Firewall에서 필요한 Source만 허용하는가?
- 인증과 암호화가 적용되어 있는가?
- 최신 보안 Update가 적용되어 있는가?

### TCP와 UDP는 암호화 여부를 뜻하지 않는다

TCP를 쓴다고 자동으로 안전하거나 암호화되는 것은 아니다.

UDP를 쓴다고 자동으로 위험한 것도 아니다.

보안은 TLS, DTLS, QUIC, Application 인증, 권한, Firewall 같은 추가 계층과 함께 판단해야 한다.

---

## 20. AI 보안과의 연결

### LLM API 통신

일반적인 LLM API는 HTTPS를 사용한다.

HTTP/1.1과 HTTP/2에서는 보통 TCP 443 위에서 TLS를 사용하고, HTTP/3 환경에서는 UDP 443 기반 QUIC를 사용할 수 있다.

```text
AI Application
      │
      │ HTTPS
      ▼
LLM API Server
```

TCP 연결 성공은 API 인증 성공을 의미하지 않는다. 연결 이후에도 TLS 인증서 검증, API Key, 권한, Rate Limit, 요청 내용 검증이 필요하다.

### Streaming Response

LLM의 Streaming Response에서는 연결이 유지되는 동안 Token이 계속 전달된다.

Network 지연, 연결 종료, Timeout, 재시도 정책이 사용자 경험과 비용에 영향을 줄 수 있다.

재시도를 잘못 구현하면 같은 요청이 중복 실행되거나 Agent Tool이 두 번 호출되는 문제가 생길 수 있다는 점도 기억해야 한다.

### AI Agent와 UDP/TCP 접근

AI Agent가 Browser, Database, 내부 API 같은 Tool을 사용할 수 있다면 Agent가 연결할 수 있는 IP, Port, Protocol 범위를 제한해야 한다.

```text
Prompt Injection
      ↓
Agent의 Network 요청
      ↓
내부 Service 또는 외부 목적지 접근
```

Network 접근 제어가 없으면 SSRF, 내부망 탐색, Data 유출 같은 위험으로 이어질 수 있다.

### Log를 볼 때

TCP/UDP Log를 분석할 때는 다음 정보를 함께 확인해야 한다.

- Source와 Destination IP
- Source와 Destination Port
- Protocol
- TCP State 또는 연결 시간
- 전송량
- NAT 또는 Proxy 여부
- Application 인증 정보
- 시간대와 요청 ID

Network 정보만으로 사용자의 의도나 Application 요청 내용을 전부 알 수는 없으므로 다른 Log와 함께 분석해야 한다.

---

## 21. 공부하면서 바로잡은 오해

### 오해 1 — TCP는 무조건 느리고 UDP는 무조건 빠르다

TCP는 연결과 신뢰성 관리 부담이 있지만 실제 성능은 Network와 Application 구현에 따라 달라진다. QUIC처럼 UDP 위에 다양한 기능을 구현할 수도 있다.

### 오해 2 — UDP는 데이터가 항상 손실된다

UDP가 손실을 보장한다는 뜻이 아니다. 전달 보장을 Protocol 자체에서 제공하지 않는다는 뜻이다.

### 오해 3 — TCP 연결에 성공하면 Service가 정상이다

Transport 연결만 성공한 것이다. Application 오류, 인증 실패, 권한 문제는 별도로 발생할 수 있다.

### 오해 4 — DNS는 항상 UDP다

일반 Query는 UDP를 많이 사용하지만 상황에 따라 TCP도 사용한다.

### 오해 5 — TCP는 암호화된다

TCP 자체는 암호화 Protocol이 아니다. HTTPS에서는 TLS가 암호화를 담당한다.

### 오해 6 — `TIME-WAIT`은 무조건 장애다

정상적인 연결 종료 과정에서도 볼 수 있는 상태다. 비정상적으로 많거나 오래 지속될 때 원인과 영향을 추가로 확인한다.

### 오해 7 — UDP에는 연결이라는 개념을 전혀 사용할 수 없다

UDP Protocol 자체에는 TCP 같은 Handshake와 연결 상태가 없지만, 운영체제 API나 Application이 특정 Peer를 지정하고 자체 상태를 관리할 수 있다.

---

## 22. 자가 테스트

답을 보기 전에 내 말로 설명해 본다.

1. TCP와 UDP는 Network 구조에서 어떤 역할을 하는가?
2. TCP가 연결 지향적이라는 말은 무엇을 뜻하는가?
3. 3-Way Handshake의 세 단계는 무엇인가?
4. TCP에서 Sequence Number와 ACK는 왜 필요한가?
5. 흐름 제어와 혼잡 제어의 차이는 무엇인가?
6. TCP가 Application 처리 성공과 암호화까지 보장하는가?
7. UDP가 순서와 재전송을 보장하지 않는 이유는 무엇인가?
8. UDP가 유용한 상황은 무엇인가?
9. DNS는 TCP와 UDP 중 무엇을 사용하는가?
10. Socket을 구분하는 5-Tuple은 무엇인가?
11. `LISTEN`과 `ESTAB`은 각각 무엇을 의미하는가?
12. `ss -ltn`과 `ss -lun`의 차이는 무엇인가?
13. SYN Flood는 TCP의 어떤 특징을 악용하는가?
14. UDP Reflection은 왜 가능한가?
15. LLM API의 TCP 연결 성공과 API 인증 성공은 같은 의미인가?

<details>
<summary>내가 정리한 답</summary>

1. IP와 Port를 사용하는 Application 데이터를 어떤 방식으로 전달할지 정하는 Transport Layer Protocol이다.
2. 통신 전에 Handshake를 하고 양쪽이 Sequence Number와 연결 상태를 관리한다는 뜻이다.
3. SYN → SYN+ACK → ACK다.
4. Byte의 순서, 전달 확인, 중복 처리, 재전송을 위해 필요하다.
5. 흐름 제어는 수신자의 처리 능력, 혼잡 제어는 Network 경로의 혼잡 상태를 고려한다.
6. 아니다. Application 성공, 인증, 암호화는 별도의 계층에서 처리해야 한다.
7. UDP는 연결 상태와 전달 확인을 기본 제공하지 않고 Datagram을 독립적으로 전달하기 때문이다.
8. 낮은 지연이 중요하거나 Application이 손실과 재전송 정책을 직접 정해야 하는 실시간 통신 등에 유용하다.
9. 상황에 따라 둘 다 사용한다.
10. Protocol, Source IP, Source Port, Destination IP, Destination Port다.
11. `LISTEN`은 새 TCP 연결을 기다리는 상태이고 `ESTAB`은 연결이 성립된 상태다.
12. 전자는 TCP Listening Socket, 후자는 UDP Socket과 Local Port를 확인한다.
13. Server가 SYN을 받은 뒤 마지막 ACK를 기다리며 연결 정보를 관리하는 특성을 악용한다.
14. TCP 같은 Handshake가 없어 위조된 Source IP를 이용한 요청과 응답 반사가 가능하기 때문이다.
15. 아니다. 연결 후에도 TLS, API Key, 권한 검사가 따로 필요하다.

</details>

---

## 23. 오늘의 핵심 요약

```text
TCP
- 연결 전에 3-Way Handshake
- Sequence Number와 ACK 사용
- 순서, 재전송, 중복 처리 제공
- 흐름 제어와 혼잡 제어
- Byte Stream

UDP
- 별도 Handshake 없음
- 전달 확인, 순서, 재전송을 기본 제공하지 않음
- 개별 Datagram 전달
- 상태와 Header 부담이 상대적으로 작음
- 필요한 신뢰성은 Application이 구현 가능
```

내가 반드시 기억할 흐름:

```text
TCP 연결
SYN → SYN+ACK → ACK

TCP 정상 종료의 일반적인 형태
FIN → ACK → FIN → ACK
```

핵심 명령어:

```bash
ss -tn
ss -ltn
ss -un
ss -lun
ss -s
```

> **TCP와 UDP의 차이는 단순히 느림과 빠름이 아니라, 연결 상태·순서·재전송 같은 신뢰성 기능을 Transport Protocol이 담당하는지에 있다.**

---

## 24. 오늘의 학습 기록

아래 내용은 실습 후 내 실제 결과와 생각으로 채운다.

```text
오늘 새롭게 이해한 내용:

TCP를 내 말로 설명:

UDP를 내 말로 설명:

3-Way Handshake를 내 말로 설명:

TCP와 UDP의 가장 중요한 차이:

ss에서 확인한 TCP 상태:

ss에서 확인한 UDP Socket:

nc TCP 실습에서 관찰한 내용:

nc UDP 실습에서 관찰한 내용:

TCP와 UDP가 AI 보안과 연결되는 부분:

처음에 잘못 알고 있었던 내용:

실습 중 발생한 오류:

오류를 해결한 방법:

추가로 공부할 질문:
```


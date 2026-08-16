# Day 03 — DNS: 도메인 이름 해석 과정

> 작성일: 2026-08-16  
> 학습 주제: DNS 구조, 이름 해석 과정, Record, Cache, `dig`, `nslookup`

Day 01에서는 IP 주소와 Port를 배웠고, Day 02에서는 TCP와 UDP가 데이터를 전달하는 방식을 배웠다.

오늘은 Browser에 입력한 Domain 이름이 실제 통신에 필요한 IP 주소로 어떻게 바뀌는지 공부했다. 처음에는 DNS를 단순한 인터넷 전화번호부라고 생각했지만, 정리하면서 DNS가 여러 계층의 Server, Cache, Record와 보안 기능이 함께 동작하는 분산 Database라는 것을 이해했다.

## 오늘 확인한 목표

- [o] DNS가 필요한 이유를 내 말로 설명할 수 있다.
- [o] Domain 이름의 계층 구조를 읽을 수 있다.
- [o] Stub Resolver, Recursive Resolver, Root, TLD, Authoritative Server의 역할을 구분할 수 있다.
- [o] Recursive Query와 Iterative Query의 차이를 설명할 수 있다.
- [o] A, AAAA, CNAME, MX, NS, TXT, PTR, SOA Record의 용도를 구분할 수 있다.
- [o] TTL과 Cache가 DNS 응답에 미치는 영향을 설명할 수 있다.
- [o] DNS가 UDP와 TCP 53번 Port를 모두 사용할 수 있다는 것을 이해했다.
- [o] `dig`와 `nslookup`의 기본 결과를 읽는 방법을 정리했다.
- [o] `/etc/hosts`, `/etc/resolv.conf`, `resolvectl`의 역할을 구분했다.
- [o] DNSSEC, DoT, DoH가 각각 무엇을 보호하는지 정리했다.
- [o] DNS가 AI Application과 Agent 보안에 어떻게 연결되는지 생각해 보았다.

---

## 1. 내가 이해한 DNS

DNS는 `Domain Name System`의 약자다.

사람은 `example.com` 같은 이름을 기억하기 쉽지만, Network 통신에는 IP 주소가 필요하다.

DNS는 Domain 이름에 연결된 IP 주소와 여러 정보를 찾아 주는 분산 Database다.

```text
사람이 입력한 이름
example.com
      │
      │ DNS Query
      ▼
DNS가 찾아 준 주소
192.0.2.10 같은 설명용 IPv4 주소
      │
      ▼
실제 Network 연결
```

`192.0.2.0/24`는 문서 작성용으로 예약된 주소 대역이다. 실제 DNS 결과는 변경될 수 있으므로 직접 `dig`로 확인해야 한다.

내가 기억할 핵심:

```text
DNS = Domain 이름과 IP 주소만 연결하는 단일 Server가 아니라
      여러 종류의 Resource Record를 계층적으로 관리하는 분산 System
```

DNS는 IP 주소 외에도 Mail Server, Name Server, 별칭, 인증과 정책에 사용하는 Text 등 다양한 정보를 제공한다.

---

## 2. Domain 이름의 계층 구조

Domain 이름은 오른쪽에서 왼쪽으로 더 구체적인 범위를 나타낸다.

```text
www.example.com.
│      │     │
│      │     └── Root
│      └──────── Top-Level Domain: com
└─────────────── example 아래의 Host 또는 Subdomain: www
```

조금 더 정확히 나누면 다음과 같다.

```text
.
└── com
    └── example
        └── www
```

### Root

Domain 이름 맨 오른쪽의 마지막 점 `.`은 DNS Root를 뜻한다.

일반적으로 Browser에서는 마지막 점을 생략해 `www.example.com`으로 입력한다.

`www.example.com.`처럼 Root까지 표시한 전체 이름을 FQDN, 즉 `Fully Qualified Domain Name`이라고 부를 수 있다.

### TLD

`com`, `org`, `net`, `kr` 같은 최상위 Domain을 TLD라고 한다.

### Registered Domain과 Subdomain

`example.com`은 `com` 아래에 등록된 Domain의 예이고, `www.example.com`의 `www`는 그 아래 Label이다.

실제 등록 가능 단위는 Public Suffix 정책의 영향도 받으므로 마지막 두 Label이 언제나 등록 Domain이라고 단정하면 안 된다. 예를 들어 국가 Domain 구조는 더 여러 단계일 수 있다.

### Domain과 URL은 다르다

```text
https://api.example.com:443/v1/chat?stream=true
```

각 부분을 나누면:

```text
https              Protocol
api.example.com    Hostname
443                Port
/v1/chat           Path
?stream=true       Query String
```

DNS가 주로 해석하는 대상은 URL 전체가 아니라 Hostname 부분이다.

---

## 3. DNS 이름 해석에 참여하는 구성 요소

DNS Query 하나에도 여러 구성 요소가 참여할 수 있다.

### Application과 Local Cache

Browser나 Application이 자체 DNS Cache를 사용할 수 있다.

같은 이름을 다시 요청할 때 Cache가 유효하면 Network Query 없이 결과를 사용할 수 있다.

### Stub Resolver

운영체제에서 Application의 이름 해석 요청을 받아 설정된 DNS Resolver로 전달하는 구성 요소다.

Application은 보통 Root Server부터 직접 찾아가지 않고 Local Stub Resolver에 이름 해석을 요청한다.

### Recursive Resolver

Client를 대신해 최종 답을 찾는 DNS Server다.

집의 공유기, ISP, 회사, 운영체제, Public DNS Service 등이 Recursive Resolver 역할을 할 수 있다.

Recursive Resolver는 Cache를 확인하고 답이 없으면 DNS 계층을 따라 필요한 Server에 Query하거나, 설정된 다른 Upstream Resolver에 Forwarding할 수 있다.

### Root Name Server

Root Server는 모든 Domain의 최종 IP를 직접 저장하는 Server가 아니다.

`com`, `org`, `kr` 같은 TLD를 담당하는 Name Server 정보를 안내한다.

### TLD Name Server

TLD Server는 해당 TLD 아래 Domain의 Authoritative Name Server가 어디인지 알려 준다.

예를 들어 `com` TLD Server는 `example.com`의 Authoritative Server 정보를 안내할 수 있다.

### Authoritative Name Server

특정 DNS Zone의 공식 Record를 제공하는 Server다.

`example.com` Zone의 A, AAAA, MX 같은 최종 정보를 가지고 응답한다.

내가 기억할 역할:

```text
Stub Resolver       = 내 컴퓨터에서 Query를 전달
Recursive Resolver  = 나를 대신해 답을 찾고 Cache
Root Server         = TLD Server 방향 안내
TLD Server          = Authoritative Server 방향 안내
Authoritative       = Zone의 공식 Record 제공
```

---

## 4. DNS 이름 해석 흐름

Cache에 답이 없고 Recursive Resolver가 DNS 계층을 직접 조회한다고 가정했을 때의 기본 흐름을 다음과 같이 정리했다.

```text
1. Browser가 example.com의 주소를 요청
             │
             ▼
2. OS Stub Resolver가 Recursive Resolver에 Query
             │
             ▼
3. Recursive Resolver가 Root Server에 질문
   "com은 어디에 있는가?"
             │
             ▼
4. Root Server가 com TLD Server 정보를 안내
             │
             ▼
5. Resolver가 com TLD Server에 질문
   "example.com은 어디에서 관리하는가?"
             │
             ▼
6. TLD Server가 Authoritative Server를 안내
             │
             ▼
7. Resolver가 Authoritative Server에 최종 Record 요청
             │
             ▼
8. Resolver가 답을 Cache하고 Client에 반환
             │
             ▼
9. Browser가 받은 IP와 TCP 또는 QUIC 연결 시작
```

실제 환경에서는 Browser Cache, OS Cache, Local Resolver, 공유기, 회사 DNS, CDN과 CNAME Chain 등이 추가되어 흐름이 달라질 수 있다. Recursive Resolver가 Root부터 직접 조회하지 않고 다른 Resolver에 Forwarding하는 구성도 있다.

Cache에 유효한 답이 있다면 Root부터 다시 조회하지 않고 바로 응답할 수 있다.

---

## 5. Recursive Query와 Iterative Query

처음에는 두 용어가 모두 여러 Server를 순서대로 조회한다는 뜻처럼 느껴졌다.

내가 이해한 차이는 **누가 최종 답을 끝까지 찾아야 하는가**다.

### Recursive Query

Client가 Resolver에 최종 답 또는 오류를 요청하는 방식이다.

```text
Client:
"example.com의 최종 주소를 찾아서 알려 달라."
```

Recursive Resolver가 Client 대신 필요한 조회를 수행한다.

### Iterative Query

질문을 받은 DNS Server가 최종 답을 가지고 있지 않으면 더 가까운 Server 정보를 Referral로 알려 주는 방식이다.

```text
Resolver → Root:
"example.com을 아는가?"

Root:
"최종 답은 없지만 com Server에 물어보라."
```

Recursive Resolver는 Root, TLD, Authoritative Server의 Referral을 따라가며 답을 찾을 수 있다.

```text
Client → Recursive Resolver: 최종 답을 요청

Recursive Resolver → DNS 계층:
Referral을 따라 단계적으로 조회
```

모든 DNS 통신이 항상 이 그림과 정확히 같은 것은 아니지만, 초보 단계에서는 Client와 Recursive Resolver의 역할을 구분하는 데 유용하다.

---

## 6. Resource Record

DNS에 저장되는 개별 정보 단위를 Resource Record라고 한다.

Record는 이름, Type, Class, TTL, Data 같은 정보를 가진다.

```text
Name        TTL   Class   Type   Data
example.com 300   IN      A      192.0.2.10
```

`192.0.2.0/24`는 문서 작성용으로 예약된 주소 대역이다.

### A Record

Hostname을 IPv4 주소와 연결한다.

```text
example.com.  300  IN  A  192.0.2.10
```

### AAAA Record

Hostname을 IPv6 주소와 연결한다.

```text
example.com.  300  IN  AAAA  2001:db8::10
```

`2001:db8::/32`는 문서 작성용 IPv6 대역이다.

AAAA는 A가 네 번이라는 이름이며 IPv4 A Record의 주소 길이보다 IPv6 주소가 네 배 길다는 배경에서 만들어진 이름이다.

### CNAME Record

한 이름을 다른 정식 이름의 별칭으로 연결한다.

```text
www.example.com.  300  IN  CNAME  web.example.net.
```

CNAME Data는 IP 주소가 아니라 다른 Domain 이름이다. Resolver는 그 대상 이름의 A 또는 AAAA Record를 추가로 찾아야 할 수 있다.

CNAME을 가진 이름은 DNSSEC 관련 Record 같은 예외를 제외하면 일반적으로 다른 Record Data와 함께 사용하지 않는다.

### MX Record

Domain의 Mail Exchange Server를 지정한다.

```text
example.com.  300  IN  MX  10 mail1.example.com.
example.com.  300  IN  MX  20 mail2.example.com.
```

앞의 숫자는 Preference 값이며 일반적으로 더 작은 값이 우선된다.

MX Record의 대상은 Hostname이며, 실제 연결에는 그 이름의 A 또는 AAAA 조회가 추가로 필요할 수 있다.

MX 대상은 Alias인 CNAME이 아니라 주소 Record를 가진 정식 Hostname이어야 한다.

### NS Record

Zone을 담당하는 Authoritative Name Server를 나타낸다.

```text
example.com.  300  IN  NS  ns1.example.net.
```

상위 Zone의 Delegation과 하위 Zone의 NS Record가 함께 이름 해석 구조를 만든다.

NS 대상 역시 CNAME이 아니라 주소를 찾을 수 있는 정식 Hostname이어야 한다.

### TXT Record

문자열 정보를 저장한다.

Email 발신 정책, Domain 소유 확인, Service 설정 등 여러 용도로 사용된다.

```text
example.com.  300  IN  TXT  "verification=example-value"
```

TXT Record에 내용이 있다는 사실만으로 그 내용이 안전하거나 신뢰할 수 있다고 단정할 수 없다.

### PTR Record

IP 주소에서 Hostname을 찾는 Reverse DNS에 사용한다.

IPv4 Reverse DNS는 `in-addr.arpa` 아래의 역순 이름을 사용한다.

```text
192.0.2.10
      ↓
10.2.0.192.in-addr.arpa.
      ↓
host.example.com.
```

PTR 결과와 Forward A/AAAA 결과가 항상 서로 일치한다고 보장할 수는 없다.

### SOA Record

`Start of Authority`의 약자로 Zone의 기본 관리 정보를 담는다.

대표적으로 다음 정보가 포함된다.

- Primary Name Server
- 관리 책임자 Mailbox를 표현한 이름
- Zone Serial
- Refresh, Retry, Expire 관련 Timer
- Negative Caching에 사용하는 값

Zone 변경 시 Secondary Server가 새 Version을 판단하는 데 Serial이 중요하다.

### 추가로 볼 수 있는 Record

| Type | 용도 |
| --- | --- |
| `SRV` | Service의 Hostname과 Port 위치 표현 |
| `CAA` | 인증서 발급을 허용할 CA 정책 표현 |
| `DNSKEY` | DNSSEC 공개 Key |
| `DS` | 상위 Zone에서 하위 Zone의 DNSSEC 신뢰 연결 |
| `RRSIG` | DNSSEC 서명 |

---

## 7. TTL과 Cache

TTL은 `Time To Live`의 약자다.

DNS 응답을 Cache에 얼마 동안 보관할 수 있는지를 초 단위로 나타낸다.

```text
example.com.  300  IN  A  192.0.2.10
              │
              └── 최대 300초 동안 Cache 가능
```

### Cache의 장점

- 같은 Query를 반복하는 시간을 줄인다.
- Root, TLD, Authoritative Server의 부하를 줄인다.
- 전체 이름 해석 속도를 높인다.

### TTL이 길 때

- Query 수와 부하를 줄일 수 있다.
- Record 변경 후 이전 값이 더 오래 남을 수 있다.

### TTL이 짧을 때

- 변경 사항이 상대적으로 빨리 반영될 수 있다.
- Query 수와 Resolver 부하가 증가할 수 있다.

TTL이 끝나기 전에 무조건 새 Record를 받을 수 있는 것은 아니다. Browser, Application, OS, Resolver가 각자 Cache 정책을 가질 수 있다.

일반적으로 TTL이 만료되면 원본 Data를 다시 확인하지만, 장애 상황 같은 제한된 조건에서는 Resolver가 만료된 Stale Answer를 잠시 제공하도록 구성될 수도 있다.

### Negative Caching

존재하지 않는 이름이라는 NXDOMAIN 응답이나 요청한 Type의 Data가 없다는 결과도 일정 시간 Cache될 수 있다.

따라서 DNS Record를 새로 만든 직후에도 이전의 실패 결과가 Cache에 남아 잠시 조회되지 않을 수 있다.

내가 기억할 문장:

```text
DNS 변경은 저장한 즉시 전 세계에서 동시에 바뀌는 것이 아니다.
여러 Cache와 TTL 때문에 이전 응답이 일정 시간 남을 수 있다.
```

---

## 8. DNS는 UDP일까 TCP일까?

Day 02에서 DNS가 UDP를 많이 사용한다고 배웠지만 DNS는 UDP와 TCP를 모두 사용할 수 있다.

### UDP 53

일반적인 작은 DNS Query와 Response에서 UDP 53번 Port를 많이 사용한다.

UDP는 별도 3-Way Handshake가 없어 Query와 Response의 부담이 작다.

### TCP 53

DNS는 TCP 53번 Port도 사용한다.

대표적인 경우:

- UDP 응답의 TC Flag가 설정되어 잘렸을 때 TCP로 다시 조회
- 큰 DNS Message를 안정적으로 전달할 때
- Zone Transfer
- Policy 또는 Network 환경이 TCP를 요구할 때

```text
UDP Query
   ↓
Response가 잘림: TC = 1
   ↓
Client가 TCP로 다시 Query 가능
```

EDNS를 사용하면 전통적인 작은 UDP 크기보다 큰 DNS Message를 협상할 수 있지만, Fragmentation과 Network 정책 문제는 여전히 고려해야 한다.

`DNS = UDP`라고만 외우지 않고 다음처럼 기억하기로 했다.

```text
DNS는 UDP와 TCP 53을 모두 사용한다.
일반 Query는 UDP가 많고 필요하면 TCP를 사용할 수 있다.
```

---

## 9. Local 이름 해석 순서

Linux Application이 이름을 찾을 때 반드시 DNS만 사용하는 것은 아니다.

`/etc/nsswitch.conf`의 `hosts` 항목은 이름을 어떤 Source와 순서로 찾을지 결정하는 데 사용된다.

예시 형태:

```text
hosts: files dns
```

환경에 따라 `resolve`, `mdns`, `myhostname` 같은 항목이 추가될 수 있다.

### `/etc/hosts`

Local에서 이름과 주소를 직접 연결하는 File이다.

```text
127.0.0.1 localhost
```

`/etc/hosts`의 내용은 일반 DNS Record가 아니며 해당 컴퓨터의 Local 이름 해석에 영향을 준다.

실습 목적 없이 System Entry를 수정하지 않고, 변경 전에는 원본과 영향 범위를 확인해야 한다.

### `/etc/resolv.conf`

Resolver가 사용할 Name Server와 Search Domain 관련 설정을 확인할 수 있다.

```bash
cat /etc/resolv.conf
```

Ubuntu에서 이 File은 systemd-resolved 또는 Network 관리 도구가 생성한 Symbolic Link일 수 있다.

직접 수정하면 Network 재연결 후 덮어써질 수 있으므로 먼저 실제 관리 주체를 확인해야 한다.

### systemd-resolved

Ubuntu 환경에서는 systemd-resolved가 Local DNS Stub과 Cache를 제공할 수 있다.

`/etc/resolv.conf`에 다음과 비슷한 주소가 보일 수 있다.

```text
nameserver 127.0.0.53
```

`127.0.0.53`은 Public DNS Server가 아니라 현재 컴퓨터의 Local Stub 주소다.

실제 Upstream DNS 설정 확인:

```bash
resolvectl status
```

이름 조회:

```bash
resolvectl query example.com
```

DNS Protocol만 지정해 비교하고 싶을 때:

```bash
resolvectl query --protocol=dns example.com
```

기본 `resolvectl query`는 환경에 따라 DNS 외에 LLMNR 또는 mDNS 같은 Protocol을 사용할 수도 있으므로 출력의 Protocol을 함께 확인한다.

모든 Ubuntu 환경이 systemd-resolved를 사용하는 것은 아니므로 Command가 없거나 구성이 다를 수 있다.

---

## 10. System Resolver 기준으로 확인하기

`getent`는 Application이 사용하는 Name Service Switch 경로에 가까운 결과를 확인할 때 유용하다.

```bash
getent hosts example.com
```

IPv4와 IPv6를 포함한 주소 확인:

```bash
getent ahosts example.com
```

`dig` 결과와 `getent` 결과가 다를 수 있는 이유:

- `/etc/hosts` 영향
- NSS Module
- Local Cache
- Search Domain
- mDNS 또는 다른 이름 Service
- Application 또는 Resolver 설정 차이

DNS Server의 응답만 확인하려는 것과 실제 Application의 이름 해석 경로를 확인하려는 것은 목적이 다르다.

---

## 11. `dig` 기본 사용

`dig`는 DNS Query와 Response의 자세한 정보를 확인할 수 있는 도구다.

기본 조회:

```bash
dig example.com
```

Record Type 지정:

```bash
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com SOA
```

짧은 결과:

```bash
dig +short example.com
```

TCP 사용:

```bash
dig +tcp example.com
```

Reverse DNS:

```bash
dig -x 192.0.2.10
```

특정 Resolver에 직접 Query:

```bash
dig @<resolver_IP> example.com
```

임의의 Public Resolver를 사용하면 Query 정보가 그 운영자에게 전달될 수 있으므로 목적과 Privacy를 확인한 뒤 사용한다.

---

## 12. `dig` 결과 읽기

다음은 형식을 설명하기 위한 단순화된 가상 예시다.

```text
; <<>> DiG <<>> example.com A
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1

;; QUESTION SECTION:
;example.com.          IN  A

;; ANSWER SECTION:
example.com.       300 IN  A  192.0.2.10

;; Query time: 20 msec
;; SERVER: 127.0.0.53#53
```

실제 `example.com` 결과가 위 주소와 같다는 뜻은 아니다.

### HEADER

```text
status: NOERROR
```

DNS 처리 결과 Code를 보여 준다.

### Flags

자주 볼 수 있는 Flag:

| Flag | 내가 이해한 의미 |
| --- | --- |
| `qr` | 이 Message가 Response임 |
| `rd` | Client가 Recursion Desired를 요청 |
| `ra` | Server가 Recursion Available을 알림 |
| `aa` | Authoritative Answer |
| `tc` | Message가 잘렸음 |
| `ad` | 검증 Resolver가 DNSSEC Data를 검증했다고 알리는 용도로 사용 |

Flag는 Query한 Server와 설정에 따라 다르게 보일 수 있다.

### QUESTION SECTION

내가 어떤 Name과 Record Type을 요청했는지 보여 준다.

```text
example.com.  IN  A
```

### ANSWER SECTION

Query에 대한 Answer Record를 보여 준다.

```text
example.com.  300  IN  A  192.0.2.10
```

여기서 `300`은 TTL이다.

### AUTHORITY SECTION

답과 관련된 Authoritative Server 또는 Zone 정보를 포함할 수 있다.

### ADDITIONAL SECTION

추가로 필요한 주소 정보나 EDNS 관련 정보가 포함될 수 있다.

### SERVER

실제로 응답을 받은 Resolver의 IP와 Port를 보여 준다.

```text
SERVER: 127.0.0.53#53
```

이 경우 Local systemd-resolved Stub에 Query했을 가능성이 있다.

---

## 13. DNS Response Code

### NOERROR

Query가 Protocol 수준에서 정상 처리되었다는 뜻이다.

NOERROR라고 해서 Answer Section에 반드시 요청한 Record가 있다는 뜻은 아니다. 이름은 존재하지만 요청한 Type의 Data가 없는 NODATA 결과일 수 있다.

### NXDOMAIN

요청한 Domain 이름이 존재하지 않는다는 응답이다.

오타, 아직 생성되지 않은 이름, 잘못된 Search Domain 등을 확인한다.

### SERVFAIL

Server가 Query를 처리하지 못했다는 뜻이다.

가능한 원인:

- Upstream Server 문제
- DNSSEC Validation 실패
- 일시적인 Network 문제
- Authoritative Server 응답 실패
- Resolver 내부 오류

SERVFAIL 하나만으로 정확한 원인을 확정할 수 없다.

### REFUSED

Server가 Policy상 Query 처리를 거부했다는 뜻이다.

예를 들어 외부 Client의 Recursive Query를 허용하지 않는 Authoritative Server에서 볼 수 있다.

### Timeout

응답이 오지 않았다는 의미다.

가능한 원인:

- UDP 또는 TCP 53 차단
- 잘못된 Resolver 주소
- Network 연결 문제
- Server 장애
- VPN 또는 Firewall Policy

UDP만 확인하지 말고 필요하면 TCP Query도 비교한다.

---

## 14. `nslookup` 사용

기본 조회:

```bash
nslookup example.com
```

Record Type 지정:

```bash
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com
```

특정 Resolver 사용:

```bash
nslookup example.com <resolver_IP>
```

`nslookup`은 빠르게 기본 결과를 확인하기 편하고, `dig`는 Header, Flag, Section, TTL 등 더 자세한 DNS 정보를 분석하기 편했다.

도구의 출력만 외우기보다 내가 어떤 Resolver에 어떤 Record Type을 Query했는지 확인하는 것이 중요하다.

---

## 15. `dig +trace`로 계층 관찰

`dig +trace`는 `dig` 자체가 Root부터 Iterative Query를 보내며 Delegation을 따라가는 흐름을 관찰하는 데 사용할 수 있다.

```bash
dig +trace example.com
```

확인할 흐름:

```text
Root NS
   ↓
com TLD NS
   ↓
example.com Authoritative NS
   ↓
최종 Record
```

주의할 점:

- 일반 Recursive 조회와 동일한 Cache 경로를 보여 주는 것은 아니다.
- 여러 DNS Server에 직접 Query를 보낼 수 있다.
- 회사, VPN, Firewall 환경에서 외부 DNS가 차단되면 실패할 수 있다.
- DNSSEC 관련 추가 Record 때문에 출력이 길어질 수 있다.

허가된 Network에서 학습 목적으로만 사용하고 실제 Domain 정보는 공개 기록에 필요한 만큼만 남긴다.

---

## 16. Day 03 실습 순서

### 실습 1 — Local Resolver 설정 확인

```bash
cat /etc/resolv.conf
resolvectl status
```

내가 확인할 것:

- `/etc/resolv.conf`가 Symbolic Link인가?
- `nameserver`는 어떤 주소인가?
- `127.0.0.53` Local Stub을 사용하는가?
- Interface별 DNS Server가 따로 설정되어 있는가?
- Search Domain이 설정되어 있는가?

### 실습 2 — System 이름 해석 확인

```bash
getent hosts example.com
getent ahosts example.com
```

확인할 것:

- IPv4와 IPv6 중 무엇이 보이는가?
- 같은 주소가 여러 줄 보이는 이유는 무엇인가?
- `dig` 결과와 차이가 있는가?

### 실습 3 — 기본 DNS Record 조회

```bash
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com SOA
```

확인할 것:

- Response Code
- Query한 Record Type
- Answer Section
- TTL
- 응답한 Server
- Query Time

### 실습 4 — UDP와 TCP 비교

```bash
dig example.com
dig +tcp example.com
```

확인할 것:

- 두 Query가 모두 성공하는가?
- 출력의 Server와 Answer가 같은가?
- Query Time 차이가 있는가?
- Network Policy에 따라 어느 쪽이 실패하는가?

### 실습 5 — Reverse DNS

문서용 주소로 Command 형식만 확인:

```bash
dig -x 192.0.2.10
```

실제 PTR Record가 없을 수 있다. PTR이 없다는 사실이 해당 IP가 사용되지 않는다는 뜻은 아니다.

### 실습 6 — 계층 구조 확인

```bash
dig +trace example.com
```

Root, TLD, Authoritative Server가 어떤 순서로 나타나는지 확인한다.

---

## 17. 실습 기록표

실제 결과에서 공개할 필요가 없는 내부 Resolver, VPN Domain, 회사 Search Domain, Hostname은 마스킹한다.

| 확인 항목 | 내가 본 결과 | 내가 이해한 의미 |
| --- | --- | --- |
| Local DNS Stub | | |
| Upstream Resolver | | |
| A Record | | |
| AAAA Record | | |
| MX Record | | |
| NS Record | | |
| SOA Serial | | |
| TTL | | |
| UDP Query | | |
| TCP Query | | |
| Reverse DNS | | |

명령 출력 전체를 그대로 붙이기보다 핵심 줄과 해석을 함께 기록한다.

---

## 18. DNS 문제 확인 순서

```text
1. IP Network 자체가 연결되어 있는가?
              ↓
2. Resolver 주소가 설정되어 있는가?
              ↓
3. /etc/hosts 또는 NSS가 영향을 주는가?
              ↓
4. 기본 DNS Query가 응답하는가?
              ↓
5. UDP와 TCP 53 중 어느 쪽이 동작하는가?
              ↓
6. Response Code는 무엇인가?
              ↓
7. Authoritative Record와 Delegation이 정상인가?
              ↓
8. Cache와 TTL 영향이 남아 있는가?
```

### IP는 되는데 Domain만 실패

예를 들어 허가된 목적지에 IP로는 연결할 수 있지만 이름으로는 실패한다면 DNS 설정이나 Resolver 문제를 의심할 수 있다.

하지만 `ping` 결과만으로 Service 전체 상태를 단정하지 않는다.

### 내 컴퓨터에서만 실패

확인할 항목:

- Browser 또는 Application Cache
- `/etc/hosts`
- `/etc/nsswitch.conf`
- `/etc/resolv.conf`
- systemd-resolved 상태
- VPN과 Proxy 설정

### 여러 환경에서 모두 실패

확인할 항목:

- Authoritative Server
- NS Delegation
- Record 오타
- Zone Serial과 배포 상태
- DNSSEC Validation
- TTL과 Negative Cache

---

## 19. DNS Cache Poisoning과 Spoofing

공격자가 Resolver Cache에 잘못된 DNS 정보를 저장시키면 사용자가 정상 Domain을 입력해도 공격자가 원하는 주소로 연결될 수 있다.

```text
정상 Query
example.com
      │
      ▼
오염된 DNS Cache
      │
      ▼
공격자가 지정한 주소
```

현대 DNS는 Query ID, Source Port Randomization, Bailiwick 확인, DNSSEC Validation 같은 방법으로 위험을 줄인다.

하지만 DNSSEC를 사용하지 않는 Zone과 검증하지 않는 Resolver에서는 서명 기반 검증을 기대할 수 없다.

HTTPS를 올바르게 사용하고 Certificate를 검증하면 DNS가 잘못된 주소를 반환하더라도 공격자가 정상 Domain의 유효한 Certificate를 제시하지 못하는 경우 연결이 차단될 수 있다.

따라서 DNS 보안과 TLS 검증은 서로 대체하는 것이 아니라 함께 필요한 계층이다.

---

## 20. DNSSEC

DNSSEC는 DNS RRset에 Digital Signature를 연결해 응답 Data의 출처와 무결성을 검증할 수 있게 한다.

주요 Record:

- `DNSKEY`
- `DS`
- `RRSIG`
- `NSEC` 또는 `NSEC3`

상위 Zone의 DS Record와 하위 Zone의 DNSKEY를 통해 Root부터 신뢰 사슬을 만들 수 있다.

### DNSSEC가 보호하는 것

- 서명된 DNS Data가 권한 있는 Zone에서 온 것인지 검증
- 전송 중 Data가 변경되지 않았는지 검증
- 서명된 Zone에서 이름이나 Record가 존재하지 않는다는 응답을 검증
- 검증 Resolver가 위조된 서명 Data를 거부하도록 지원

### DNSSEC가 보호하지 않는 것

- DNS Query 이름의 기밀성
- DNS Server의 가용성
- 최종 Web Service의 Application 보안
- Domain 자체가 선한 운영자 소유인지 여부
- TLS Certificate와 HTTP 내용

내가 기억할 문장:

```text
DNSSEC = 서명된 DNS Data의 출처와 무결성 검증
DNS Traffic 암호화 기능은 아님
```

```bash
dig +dnssec example.com
```

`dig +dnssec`는 DNSSEC 관련 Record를 요청하는 DO Bit를 설정하는 것이며, 이 명령만 실행했다고 독립적인 서명 검증이 끝난 것은 아니다. 실제 검증 여부는 검증 Resolver의 상태를 확인하거나 `delv` 같은 검증 도구로 별도로 확인해야 한다.

---

## 21. DoT와 DoH

일반 DNS Query는 Network에서 평문으로 관찰될 수 있다.

DoT와 DoH는 Client와 Resolver 사이 DNS Traffic을 암호화하는 방법이다.

### DoT

`DNS over TLS`다.

일반적으로 TCP 853번 Port에서 TLS를 사용한다.

### DoH

`DNS over HTTPS`다.

DNS Message를 HTTPS로 전달하며 일반적으로 443번 Port를 사용한다.

### DNSSEC와 차이

```text
DNSSEC = DNS Data의 서명 검증
DoT/DoH = Client와 Resolver 사이 전송 암호화
```

DoT나 DoH를 쓴다고 Resolver 운영자가 Query를 볼 수 없는 것은 아니다.

또한 Client에서 Recursive Resolver까지 암호화되더라도 Resolver 이후 Authoritative Server까지의 모든 구간이 자동으로 암호화된다는 뜻은 아니다.

어떤 Resolver를 신뢰할 것인지, Logging 정책이 무엇인지도 함께 확인해야 한다.

---

## 22. DNS Reflection과 Amplification

UDP는 TCP 같은 Handshake가 없으므로 Source IP Spoofing을 이용한 Reflection 공격에 악용될 수 있다.

```text
공격자
피해자 IP로 Source 위조
      │
      ▼
Open Recursive Resolver
      │
      ▼
피해자에게 DNS Response 전달
```

작은 Query에 비해 큰 Response가 생성되면 Amplification 효과가 생긴다.

방어 관점에서 확인할 내용:

- 외부에 불필요한 Recursive Query를 허용하지 않기
- Source Address Spoofing 방지
- Response Rate Limiting
- 비정상적으로 큰 Query와 Response 감시
- DNS Software Update

학습 과정에서 허가받지 않은 DNS Server를 대상으로 부하 Test를 하지 않는다.

---

## 23. DNS Tunneling

DNS Query의 QNAME이나 여러 Resource Record의 Data에 값을 나누어 넣어 Command 전달이나 Data 유출 통로로 악용할 수 있다. TXT는 악용될 수 있는 한 가지 예일 뿐이다.

```text
encoded-data.attacker.example
```

DNS는 많은 Network에서 기본적으로 허용되기 때문에 우회 Channel로 악용될 수 있다.

탐지할 때 볼 수 있는 신호:

- 비정상적으로 긴 Label
- 높은 Entropy의 무작위 문자열
- 특정 Domain으로 반복되는 많은 Query
- TXT Record의 비정상 사용
- 일정한 주기의 Query
- Client별 평소와 다른 DNS 양

긴 Domain이나 TXT Query가 모두 공격은 아니므로 Application과 업무 맥락을 함께 분석해야 한다.

---

## 24. DNS Rebinding

DNS Rebinding은 같은 Domain의 DNS 응답을 바꾸어 Browser나 Server가 처음에는 외부 주소를 보고 허용한 뒤 나중에는 내부 또는 Local 주소로 연결되도록 유도하는 공격 기법이다.

```text
첫 번째 DNS 응답
attacker.example → Public IP

이후 DNS 응답
attacker.example → 127.0.0.1 또는 Private IP
```

이는 위조된 응답을 Cache에 넣는 Cache Poisoning과 다른 공격이다. 공격자가 자기 Zone에서 정상적으로 서명한 응답을 바꾸는 경우라면 DNSSEC만으로 Rebinding을 막을 수 없다.

방어할 때 Domain 문자열만 검사해서는 부족할 수 있다.

- 실제 연결 직전 해석된 모든 IP를 검사
- Loopback, Private, Link-local, Metadata 주소 차단
- IPv4와 IPv6를 모두 검사
- Redirect 이후 목적지를 다시 검사
- 허용한 Protocol과 Port만 사용
- HTTP Client가 실제로 연결한 주소를 Log
- DNS 응답 변경과 TTL을 고려

정상적인 CDN과 Load Balancing도 여러 IP를 반환할 수 있으므로 무조건 하나의 IP로 고정하는 방식에는 운영상 문제가 생길 수 있다. 보안 Policy와 정상 Service 동작을 함께 설계해야 한다.

---

## 25. 다른 DNS 관련 위험

### DNS Hijacking

Router, Local 설정, Resolver 또는 Domain 계정이 침해되어 DNS 응답이나 설정이 공격자가 원하는 방향으로 바뀌는 상황이다.

### 잘못된 `/etc/hosts`

Local Hosts File이 변경되면 DNS Server가 정상이어도 해당 컴퓨터만 잘못된 주소로 연결될 수 있다.

### Subdomain Takeover

DNS Record가 삭제된 Cloud Resource나 외부 Service를 계속 가리키고, 다른 사람이 그 Resource를 다시 등록할 수 있다면 Subdomain 탈취 위험이 생길 수 있다.

### 유사 Domain과 IDN

사람이 비슷하게 보이는 문자나 오타 Domain에 속을 수 있다.

DNS가 정상적으로 해석된다는 사실은 그 Domain이 원래 의도한 안전한 Service라는 뜻이 아니다.

---

## 26. AI 보안과 DNS의 연결

### LLM API도 먼저 DNS를 사용한다

AI Application이 LLM API Domain에 연결할 때도 먼저 DNS로 주소를 찾는다.

```text
AI Application
      │
      │ DNS Query
      ▼
LLM API Domain의 IP
      │
      │ TCP+TLS 또는 QUIC
      ▼
LLM API Server
```

DNS가 잘못된 주소를 반환해도 올바른 TLS Certificate 검증이 적용되면 단순한 Server 위장은 차단될 수 있다.

그래서 DNS 결과뿐 아니라 TLS 검증, API 인증, 권한도 함께 확인해야 한다.

### AI Agent와 SSRF

AI Agent가 URL을 받아 Web Page나 API에 접속할 수 있다면 공격자가 DNS를 이용해 내부 Network 접근을 유도할 수 있다.

```text
사용자 또는 외부 문서
        │
        ▼
AI Agent가 URL 호출
        │
        ▼
DNS가 내부 IP를 반환
        │
        ▼
내부 Service 접근 위험
```

Agent의 Network Tool에서는 다음을 확인해야 한다.

- Domain 문자열 Allowlist만 믿지 않기
- 해석된 모든 IPv4와 IPv6 주소 검사
- Redirect마다 목적지 재검사
- Localhost, Private, Link-local, Cloud Metadata 차단
- 허용된 Port와 Protocol 제한
- Proxy가 실제로 연결한 주소 확인
- DNS Query와 Tool 호출을 같은 Request ID로 기록

### RAG와 외부 문서 수집

RAG Pipeline이 외부 URL에서 문서를 가져오면 DNS가 가리키는 목적지가 바뀔 수 있다.

수집기는 외부 공개 주소만 허용하고 내부 Service, 관리 Interface, Metadata Endpoint로 연결되지 않도록 Egress Policy를 적용해야 한다.

### DNS Log와 AI Security Monitoring

DNS Log에서 확인할 수 있는 정보:

- 어떤 Client가 어떤 Domain을 조회했는가
- Query Type
- Response Code
- 반환된 주소
- Query 빈도
- Resolver와 응답 시간

DNS Log만으로 실제 HTTP Path, Prompt, 전송 Data를 알 수는 없다. Proxy, Application, Authentication Log와 함께 분석해야 한다.

### Domain은 신원이 아니다

Domain 이름이 그럴듯하다는 사실만으로 Service를 신뢰하지 않는다.

TLS Certificate, Application 인증, 소유권, 권한, Supply Chain을 함께 확인해야 한다.

---

## 27. 공부하면서 바로잡은 오해

### 오해 1 — DNS는 Domain을 IP로만 바꾼다

DNS는 Mail Server, Name Server, Alias, Policy Text 등 여러 Record를 제공한다.

### 오해 2 — 내 컴퓨터가 매번 Root Server에 직접 Query한다

일반적으로 Client는 Recursive Resolver에 요청하고 Cache가 있으면 Root까지 조회하지 않는다.

### 오해 3 — DNS는 UDP만 사용한다

UDP와 TCP 53을 모두 사용한다.

### 오해 4 — TTL이 끝나면 전 세계 DNS가 동시에 바뀐다

Browser, OS, Resolver 등 여러 Cache와 정책 때문에 반영 시점이 다를 수 있다.

### 오해 5 — NXDOMAIN은 Network 연결이 끊겼다는 뜻이다

Resolver가 요청한 이름이 존재하지 않는다고 응답한 것이다. Timeout과 다르다.

### 오해 6 — NOERROR면 반드시 IP가 있다

이름은 존재하지만 요청한 Record Type이 없어 Answer가 비어 있을 수 있다.

### 오해 7 — DNSSEC는 DNS를 암호화한다

DNSSEC는 서명된 Data의 출처와 무결성을 검증하며 Query 기밀성은 제공하지 않는다.

### 오해 8 — DoH를 사용하면 DNS가 완전히 익명이다

전송 구간은 암호화되지만 선택한 Resolver는 Query를 처리하며 Logging 정책의 영향을 받는다.

### 오해 9 — Domain이 정상 해석되면 안전한 Site다

DNS는 이름에 대한 Record를 제공할 뿐 Domain 운영자의 신뢰성과 Application 안전을 보장하지 않는다.

### 오해 10 — PTR이 없으면 사용하지 않는 IP다

Reverse DNS가 설정되지 않은 IP도 정상적으로 사용될 수 있다.

---

## 28. 자가 테스트

답을 보기 전에 내 말로 설명해 본다.

1. DNS가 필요한 이유는 무엇인가?
2. Domain 이름은 어느 방향으로 계층을 읽는가?
3. Stub Resolver와 Recursive Resolver의 차이는 무엇인가?
4. Root, TLD, Authoritative Server는 각각 무엇을 알려 주는가?
5. Recursive Query와 Iterative Query의 차이는 무엇인가?
6. A와 AAAA Record의 차이는 무엇인가?
7. CNAME은 IP 주소를 직접 저장하는가?
8. MX의 Preference 값은 일반적으로 어느 값이 우선인가?
9. PTR Record는 무엇에 사용하는가?
10. TTL은 무엇을 의미하는가?
11. DNS가 TCP 53을 사용할 수 있는 경우는 무엇인가?
12. `NOERROR`인데 Answer가 비어 있을 수 있는가?
13. `127.0.0.53`은 어떤 주소일 수 있는가?
14. `dig`와 `getent` 결과가 다를 수 있는 이유는 무엇인가?
15. DNSSEC와 DoT/DoH의 보호 대상은 어떻게 다른가?
16. DNS Reflection은 UDP의 어떤 특징을 악용하는가?
17. DNS Rebinding 방어에서 Domain 문자열 검사만으로 부족한 이유는 무엇인가?
18. AI Agent의 URL Tool에서 DNS 결과를 어떻게 검증해야 하는가?

<details>
<summary>내가 정리한 답</summary>

1. 사람이 사용하는 Domain 이름에 연결된 IP와 여러 Service 정보를 찾기 위해 필요하다.
2. 오른쪽 Root와 TLD에서 왼쪽의 더 구체적인 Label 방향으로 계층이 이어진다.
3. Stub Resolver는 Local 요청을 전달하고 Recursive Resolver는 Client 대신 최종 답을 찾고 Cache한다.
4. Root는 TLD 방향, TLD는 Authoritative Server 방향, Authoritative Server는 Zone의 공식 Record를 제공한다.
5. Recursive Query는 최종 답을 요청하고, Iterative Query는 다음에 물어볼 Server 정보를 Referral로 받을 수 있다.
6. A는 IPv4, AAAA는 IPv6 주소를 제공한다.
7. 아니다. 다른 정식 Domain 이름을 가리킨다.
8. 일반적으로 더 작은 Preference 값이 우선한다.
9. IP 주소에서 Hostname을 찾는 Reverse DNS에 사용한다.
10. DNS 응답을 Cache할 수 있는 시간을 초 단위로 나타낸다.
11. UDP 응답이 잘렸을 때, 큰 Message, Zone Transfer 등에서 사용할 수 있다.
12. 있다. 이름은 존재하지만 요청한 Type의 Data가 없는 경우다.
13. systemd-resolved가 제공하는 Local DNS Stub 주소일 수 있다.
14. `/etc/hosts`, NSS, Local Cache, mDNS, Search Domain과 Resolver 설정이 다를 수 있기 때문이다.
15. DNSSEC는 서명된 Data의 출처와 무결성을 검증하고, DoT/DoH는 Client와 Resolver 사이 전송을 암호화한다.
16. Handshake 없이 Source IP를 위조한 Query를 보낼 수 있는 점을 악용한다.
17. 같은 Domain이 이후 내부 IP를 반환할 수 있고 Redirect나 여러 주소가 존재할 수 있기 때문이다.
18. 실제 연결 직전 모든 IPv4/IPv6 주소와 Redirect 목적지를 다시 검사하고 내부·Local·Metadata 주소를 차단해야 한다.

</details>

---

## 29. 오늘의 핵심 요약

```text
DNS
- Domain 이름에 연결된 여러 Resource Record를 관리하는 분산 System
- Client는 일반적으로 Recursive Resolver에 Query
- Resolver는 Cache, 상위 Resolver 또는 Root → TLD → Authoritative 경로로 답을 찾음

주요 Record
A      IPv4
AAAA   IPv6
CNAME  다른 이름의 Alias
MX     Mail Server
NS     Authoritative Name Server
TXT    Text Data
PTR    Reverse DNS
SOA    Zone 관리 정보

전송
UDP 53을 많이 사용
필요하면 TCP 53도 사용

보안
DNSSEC = 서명된 DNS Data의 출처·무결성 검증
DoT/DoH = Client와 Resolver 사이 전송 암호화
```

핵심 명령어:

```bash
cat /etc/resolv.conf
resolvectl status
getent hosts example.com
dig example.com
dig example.com MX
dig +tcp example.com
dig +trace example.com
nslookup example.com
```

> **DNS는 이름을 단순히 IP로 바꾸는 한 대의 Server가 아니라, 여러 계층과 Cache가 Resource Record를 찾아 주는 분산 System이다.**

---

## 30. 오늘의 학습 기록

DNS를 처음에는 Domain 이름을 IP 주소로 바꾸는 한 대의 Server라고 생각했다. 오늘 정리하면서 실제로는 Stub Resolver와 Recursive Resolver가 여러 계층의 Name Server, Cache, Resource Record를 이용해 필요한 답을 찾는 분산 System이라는 점을 알게 되었다.

가장 헷갈렸던 부분은 Root와 TLD Server가 최종 IP를 직접 알려 주는 것이 아니라 다음에 물어볼 Authoritative Server의 방향을 Referral로 안내한다는 점이었다. Client는 보통 이 과정을 직접 수행하지 않고 Recursive Resolver에 최종 답을 요청한다.

TTL은 DNS 변경이 전 세계에 퍼지는 시간을 뜻하는 값이 아니라 Resolver가 Record를 Cache할 수 있는 시간이다. TTL이 끝나도 모든 Cache가 동시에 바뀌는 것은 아니며, NXDOMAIN 같은 실패 결과도 Cache될 수 있다는 점을 함께 기억하기로 했다.

DNS는 일반적인 작은 Query에서 UDP 53을 많이 사용하지만 TCP 53도 정식 전송 방식이다. UDP 응답이 잘렸거나 Message가 큰 경우, Zone Transfer 같은 상황에서는 TCP를 사용할 수 있으므로 `DNS는 UDP만 사용한다`고 외우면 안 된다.

`dig`는 지정한 DNS Server의 Response, TTL, Flag, Section을 자세히 보는 데 유용하고, `getent`는 `/etc/hosts`와 NSS를 포함해 Application에 더 가까운 이름 해석 결과를 확인하는 데 유용하다고 정리했다. 그래서 두 명령의 결과가 다르다고 해서 바로 오류라고 판단하지 않는다.

보안 부분에서는 DNSSEC와 DoH를 구분하는 것이 중요했다. DNSSEC는 서명된 DNS Data의 출처와 무결성을 검증하고, DoH와 DoT는 Client와 Resolver 사이의 전송 구간을 암호화한다. 서로 해결하는 문제가 다르기 때문에 하나가 다른 하나를 대신하지 않는다.

AI Agent나 RAG 수집기가 외부 URL에 접속할 때는 Domain 문자열만 허용 목록과 비교하면 DNS Rebinding이나 SSRF를 충분히 막지 못할 수 있다. 실제 연결 직전에 해석된 IPv4·IPv6 주소와 Redirect 목적지를 다시 검사하고, 내부·Loopback·Link-local·Cloud Metadata 주소를 차단해야 한다.

---

## 검증에 참고한 공식 문서

- [RFC 1034 — Domain Names: Concepts and Facilities](https://www.rfc-editor.org/rfc/rfc1034)
- [RFC 1035 — Domain Names: Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
- [RFC 2181 — Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181)
- [RFC 2308 — Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308)
- [RFC 3596 — DNS Extensions to Support IPv6](https://www.rfc-editor.org/rfc/rfc3596)
- [RFC 7766 — DNS Transport over TCP](https://www.rfc-editor.org/rfc/rfc7766)
- [RFC 9499 — DNS Terminology](https://www.rfc-editor.org/rfc/rfc9499)
- [RFC 8767 — Serving Stale Data to Improve DNS Resiliency](https://www.rfc-editor.org/rfc/rfc8767)
- [RFC 4033 — DNS Security Introduction and Requirements](https://www.rfc-editor.org/rfc/rfc4033)
- [RFC 7858 — DNS over TLS](https://www.rfc-editor.org/rfc/rfc7858)
- [RFC 8484 — DNS Queries over HTTPS](https://www.rfc-editor.org/rfc/rfc8484)
- [RFC 5452 — Measures for Making DNS More Resilient against Forged Answers](https://www.rfc-editor.org/rfc/rfc5452)
- [RFC 5358 — Preventing Use of Recursive Nameservers in Reflector Attacks](https://www.rfc-editor.org/rfc/rfc5358)
- [RFC 9526 — Simple Provisioning of Public Names for Residential Networks](https://www.rfc-editor.org/rfc/rfc9526)
- [IANA — Domain Name System Parameters](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml)
- [IANA — Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [systemd-resolved 공식 설명서](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html)
- [Ubuntu `resolvectl` 매뉴얼](https://manpages.ubuntu.com/manpages/jammy/man1/resolvectl.1.html)
- [Ubuntu `dig` 매뉴얼](https://manpages.ubuntu.com/manpages/jammy/man1/dig.1.html)
- [Ubuntu `delv` 매뉴얼](https://manpages.ubuntu.com/manpages/jammy/man1/delv.1.html)


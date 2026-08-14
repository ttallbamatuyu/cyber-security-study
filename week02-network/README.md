# Week 02 — 네트워크 기초와 HTTP/HTTPS

## 이번 주 목표

인터넷 통신의 기초 구조를 이해하고, Ubuntu 명령어와 Wireshark를 사용해 DNS → TCP → TLS → HTTP 흐름을 직접 관찰합니다. 이후 웹 보안, API 보안, LLM API와 AI Agent 통신을 분석할 수 있는 기반을 만드는 주차입니다.

## 일차별 계획

| Day | 주제 | 완료 기준 |
| --- | --- | --- |
| [Day 01](day01-network-basics/README.md) | 네트워크 기초 | IP와 Port, Router와 Gateway를 설명하고 `ip`, `ping`, `ss` 결과를 읽을 수 있다. |
| Day 02 | TCP와 UDP | 두 프로토콜의 차이와 TCP 3-Way Handshake를 설명할 수 있다. |
| Day 03 | DNS | 도메인이 IP 주소로 변환되는 과정을 설명하고 `dig`, `nslookup`을 사용할 수 있다. |
| Day 04 | HTTP | HTTP Request/Response, Method, Header, Status Code를 읽고 `curl`로 요청할 수 있다. |
| Day 05 | HTTPS와 TLS | HTTP와 HTTPS의 차이, 인증서와 TLS의 역할을 설명할 수 있다. |
| Day 06 | 패킷 분석 | Wireshark로 DNS, TCP, HTTP 패킷을 찾아 기본 필드를 확인할 수 있다. |
| Day 07 | 종합 복습 | 웹사이트 접속 과정인 DNS → TCP → TLS → HTTP 흐름을 자신의 말로 설명할 수 있다. |

## 핵심 명령어

```bash
ip addr
ip route
ping
ss -tuln
dig
nslookup
curl
openssl
```

## AI 보안과의 연결

LLM 애플리케이션, RAG 서버, AI Agent의 Tool 호출도 대부분 네트워크와 Web API를 통해 이루어집니다. IP와 Port, HTTP/HTTPS, DNS, Listening 상태를 읽을 수 있어야 다음 단계에서 API 노출, 잘못된 바인딩, SSRF, 인증·인가 문제와 같은 공격면을 이해할 수 있습니다.

## 기록 규칙

- 명령어 출력은 그대로 복사하기보다 중요한 줄과 해석을 함께 적는다.
- 사용자 이름, Public IP, 내부 호스트 이름 등 공개가 불필요한 정보는 마스킹한다.
- 예제 IP와 실제 환경의 IP를 구분해 기록한다.
- 패킷 캡처와 포트 확인은 본인 장비 또는 명시적으로 허가받은 학습 환경에서만 수행한다.
- 실습 완료 후 각 Day 문서의 체크리스트와 학습 기록을 갱신한다.


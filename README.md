# Cyber Security Study

화이트해킹과 정보보안 학습 과정을 주차별·일차별로 기록하는 저장소입니다.

명령어를 단순히 암기하기보다 **개념 이해 → 직접 실습 → 문제 해결 → 학습 기록** 순서로 공부합니다.

ChatGPT 5.6 sol을 통해 학습하고 기록하였고 학습 기록 밑에 제가 스스로 적은 답변들을 섞어서 올렸으며 실습은 모두 다 직접 진행해보았습니다.

## 학습 원칙

- 명령어는 직접 입력하고 실행 결과를 관찰한다.
- 모르는 옵션은 먼저 `--help`와 `man`으로 확인한다.
- 실습 과정, 막힌 부분, 해결 방법을 함께 기록한다.
- 워게임 비밀번호, API 키, 개인 정보와 같은 민감 정보는 커밋하지 않는다.
- 허가받은 환경과 학습용 시스템에서만 보안 실습을 진행한다.

## Week 01 — Ubuntu/Linux 터미널 기초

### 학습 목표

- Ubuntu 터미널을 실행할 수 있다.
- 현재 사용자와 현재 위치를 확인할 수 있다.
- 숨김 파일을 포함한 파일 목록을 볼 수 있다.
- 절대경로와 상대경로를 설명할 수 있다.
- 디렉터리를 생성하고 이동할 수 있다.
- 파일을 생성·복사·이동·삭제할 수 있다.
- 파일 내용을 확인할 수 있다.
- `>`와 `>>`의 차이를 설명할 수 있다.
- `man`과 `--help`로 사용법을 찾을 수 있다.
- SSH의 목적을 설명하고 원격 서버에 접속할 수 있다.
- OverTheWire Bandit Level 0~5에 도전한다.
- 학습 내용을 Markdown 문서로 기록한다.

### 일차별 구성

| Day | 주제 | 핵심 내용 |
| --- | --- | --- |
| [Day 01](week01-linux/day01-terminal-basics/README.md) | 터미널 기초 | Linux, Ubuntu, GUI/CLI, Terminal, Shell, `whoami`, `pwd`, `ls` |
| [Day 02](week01-linux/day02-path-directory/README.md) | 경로와 디렉터리 | 절대경로, 상대경로, `cd`, `mkdir`, `rmdir` |
| [Day 03](week01-linux/day03-file-management/README.md) | 파일 관리 | `touch`, `cp`, `mv`, `rm` |
| [Day 04](week01-linux/day04-file-content-redirection/README.md) | 파일 내용과 리다이렉션 | `cat`, `less`, `head`, `tail`, `>`, `>>` |
| [Day 05](week01-linux/day05-help-ssh/README.md) | 도움말과 SSH | `man`, `--help`, SSH |
| [Day 06](week01-linux/day06-bandit-00-05/README.md) | Bandit 실습 | OverTheWire Bandit Level 0~5 |
| [Day 07](week01-linux/day07-review/README.md) | 복습과 기록 | 종합 미션, 자가 점검, 학습 기록 정리 |

자세한 주간 계획은 [Week 01 안내](week01-linux/README.md)에서 확인할 수 있습니다.

## Week 02 — 네트워크 기초와 HTTP/HTTPS

### 학습 목표

- 네트워크, 클라이언트, 서버의 관계를 설명할 수 있다.
- IP 주소와 Port가 각각 무엇을 식별하는지 설명할 수 있다.
- Private IP, Public IP, Loopback 주소를 구분할 수 있다.
- Router와 Default Gateway의 역할을 설명할 수 있다.
- TCP와 UDP, DNS, HTTP와 HTTPS의 핵심 차이를 이해한다.
- Ubuntu에서 기본 네트워크 상태와 Listening Port를 확인할 수 있다.
- 브라우저가 웹 서버에 접속할 때의 흐름을 설명할 수 있다.

### 일차별 구성

| Day | 주제 | 핵심 내용 |
| --- | --- | --- |
| [Day 01](week02-network/day01-network-basics/README.md) | 네트워크 기초 | Network, Client/Server, IP, Port, Router, Gateway, `ip`, `ping`, `ss` |
| [Day 02](week02-network/day02-tcp-udp/README.md) | TCP와 UDP | 연결 방식, 신뢰성, 3-Way Handshake, `ss`, `nc` |
| [Day 03](week02-network/day03-dns/README.md) | DNS | 도메인 이름 해석, `dig`, `nslookup` |
| Day 04 | HTTP | Request/Response, Method, Header, Status Code, `curl` |
| Day 05 | HTTPS와 TLS | 암호화, 인증서, TLS Handshake, `openssl` |
| Day 06 | 패킷 분석 | Wireshark로 DNS, TCP, HTTP 패킷 관찰 |
| Day 07 | 종합 복습 | DNS → TCP → TLS → HTTP 연결 흐름 정리 |

자세한 주간 계획은 [Week 02 안내](week02-network/README.md)에서 확인할 수 있습니다.

## 저장소 구조

```text
cyber-security-study/
├── README.md
├── week01-linux/
│   ├── README.md
│   ├── day01-terminal-basics/
│   │   └── README.md
│   ├── day02-path-directory/
│   │   └── README.md
│   ├── day03-file-management/
│   │   └── README.md
│   ├── day04-file-content-redirection/
│   │   └── README.md
│   ├── day05-help-ssh/
│   │   └── README.md
│   ├── day06-bandit-00-05/
│   │   └── README.md
│   └── day07-review/
│       └── README.md
└── week02-network/
    ├── README.md
    ├── day01-network-basics/
    │   └── README.md
    ├── day02-tcp-udp/
    │   └── README.md
    └── day03-dns/
        └── README.md
```

## 진행 상황

- [x] Week 01 구조 생성
- [x] Day 01 학습 내용 정리
- [x] Day 02 학습 및 기록
- [x] Day 03 학습 및 기록
- [ ] Day 04 학습 및 기록
- [ ] Day 05 학습 및 기록
- [ ] Day 06 Bandit Level 0~5
- [ ] Day 07 종합 복습
- [x] Week 02 구조 생성
- [x] Week 02 Day 01 네트워크 기초 정리
- [x] Week 02 Day 02 TCP와 UDP 정리
- [x] Week 02 Day 03 DNS 정리
- [ ] Week 02 Day 04~07 학습 및 기록



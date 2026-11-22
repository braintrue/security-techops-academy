# Day 06. TCP Connection & Packet Analysis

## 1. 학습 목표

- TCP 연결 상태와 주요 플래그 이해
- `ss`를 이용한 소켓·포트·프로세스 확인
- `tcpdump`를 이용한 SSH 패킷 캡처
- TCP SYN 및 SYN-ACK 패킷 분석
- HTTPS 통신에서 TCP, TLS, HTTP의 역할 구분

## 2. 실습 환경

| 항목 | 환경 |
|---|---|
| Host OS | macOS |
| Host Device | MacBook Air M2 (8GB) |
| Virtualization | UTM |
| Guest OS | Ubuntu Server ARM64 |
| VM CPU | 2 vCPU |
| VM Memory | 약 1.6 GiB |
| Network Interface | enp0s1 |
| Ubuntu IP | 192.168.64.3 |
| Mac IP | 192.168.64.1 |
| SSH Port | TCP 22 |

Mac 터미널에서 SSH로 Ubuntu 서버에 접속해 실습했다.

## 3. TCP 연결 상태 확인

### 3-1. 전체 TCP 연결

```bash
ss -tan
```

SSH 포트 22의 LISTEN 상태와 SSH 접속에 따른 ESTAB 상태를 확인했다.

- LISTEN: 연결 요청 대기
- ESTAB: 연결 수립
- TIME-WAIT: 연결 종료 후 일정 시간 유지

### 3-2. 포트 및 프로세스 확인

```bash
sudo ss -tulpen
```

확인한 주요 항목:

- SSH: TCP 22
- systemd-resolved: DNS 관련 포트 53
- chronyd: UDP 323
- systemd-networkd: UDP 68

SSH 포트에서는 `sshd`와 `systemd`의 Socket Activation 구조도 확인했다.

### 3-3. 소켓 통계

```bash
ss -s
```

실습 당시 주요 출력:

```text
Total: 187
TCP: 7 (estab 3, closed 0, orphaned 0, timewait 0)
```

개별 연결 조회와 전체 소켓 통계 조회의 목적이 다르다는 점을 확인했다.

## 4. tcpdump 패킷 캡처

### 4-1. 버전 확인

```bash
tcpdump --version
```

확인 결과: tcpdump 4.99.6

### 4-2. 기존 SSH 트래픽 캡처

```bash
sudo tcpdump -i enp0s1 -nn 'tcp port 22'
```

- `-i enp0s1`: 캡처 인터페이스 지정
- `-nn`: 주소와 포트를 숫자로 표시
- `tcp port 22`: SSH TCP 트래픽 필터링

기존에 수립된 SSH 연결에서 주로 다음 패킷을 확인했다.

- `[.]`: ACK
- `[P.]`: PSH + ACK

이는 새 연결의 Handshake가 아니라 기존 SSH 세션의 데이터 교환을 관찰한 결과다.

## 5. TCP SYN 및 SYN-ACK 분석

Mac에서 새로운 SSH 연결을 생성한 뒤 Ubuntu에서 SYN 비트가 설정된 패킷을 필터링했다.

```bash
sudo tcpdump -i enp0s1 -nn -vv 'tcp port 22 and host 192.168.64.1 and (tcp[tcpflags] & tcp-syn != 0)'
```

실제 캡처 결과:

```text
192.168.64.1.50357 > 192.168.64.3.22:
Flags [SEW], seq 3105961995

192.168.64.3.22 > 192.168.64.1.50357:
Flags [S.E], seq 2636560138, ack 3105961996
```

### 패킷 해석

| 순서 | 방향 | 관찰 결과 | 의미 |
|---|---|---|---|
| 1 | Mac → Ubuntu | `[SEW]` | SYN 및 ECN 관련 플래그 |
| 2 | Ubuntu → Mac | `[S.E]` | SYN + ACK 및 ECN-Echo |

첫 번째 패킷의 클라이언트 초기 시퀀스 번호는 `3105961995`이고, 서버 응답의 ACK 번호는 `3105961996`이다. 즉, 클라이언트의 초기 시퀀스 번호에 1을 더한 값으로 응답했다.

### 세 번째 ACK가 캡처되지 않은 이유

TCP 3-Way Handshake의 일반적인 순서는 다음과 같다.

1. SYN: 클라이언트의 연결 요청
2. SYN-ACK: 서버의 응답
3. ACK: 클라이언트의 최종 확인

이번 캡처 필터에는 다음 조건이 포함되어 있었다.

```text
tcp[tcpflags] & tcp-syn != 0
```

따라서 SYN 비트가 없는 세 번째 ACK 패킷은 필터링 대상에서 제외된다. 이번 실습에서는 SYN과 SYN-ACK를 직접 관찰했으며, 마지막 ACK는 직접 캡처하지 않았다.

## 6. HTTPS 연결 과정 확인

### 실행 명령어

```bash
curl -v https://ubuntu.com
```

실습에서 확인한 결과:

| 항목 | 결과 |
|---|---|
| IPv6 연결 | `Network is unreachable` |
| IPv4 연결 | `185.125.190.21:443` 연결 성공 |
| TLS | TLS 1.3 |
| Cipher | `TLS_AES_256_GCM_SHA384` |
| 인증서 | 검증 성공 |
| ALPN | HTTP/2 (`h2`) |
| HTTP 응답 | 200 |

IPv6 연결 시도는 실패했지만 IPv4 연결에 성공해 HTTPS 응답을 받았다. 이 결과만으로 공격이나 보안 침해를 의미하지는 않는다.

TCP 연결 이후 TLS 협상이 수행되고, 그 위에서 HTTP/2 통신이 이루어지는 흐름을 확인했다.

## 7. 보안 및 운영 관점

- 예상하지 못한 LISTEN 포트가 있는지 점검한다.
- ESTAB 연결의 출발지·목적지와 프로세스를 확인한다.
- TCP 플래그를 통해 연결 수립 및 데이터 교환 흐름을 분석한다.
- 패킷 필터 조건이 분석 결과에 어떤 영향을 주는지 확인한다.
- HTTPS 연결에서 TLS 버전과 인증서 검증 결과를 확인한다.

## 8. 학습 정리

| 명령어 | 목적 |
|---|---|
| `ss -tan` | TCP 연결 상태 확인 |
| `sudo ss -tulpen` | 포트 및 프로세스 확인 |
| `ss -s` | 소켓 상태 요약 |
| `tcpdump` | 네트워크 패킷 캡처 |
| `curl -v` | HTTP/TLS 연결 과정 확인 |

### 핵심 정리

- TCP 연결 수립은 SYN → SYN-ACK → ACK 순서로 진행된다.
- `ss`는 소켓 상태를 확인하고 `tcpdump`는 실제 패킷을 관찰한다.
- 패킷 필터 조건에 따라 특정 패킷이 캡처 결과에서 제외될 수 있다.
- HTTPS는 TCP 연결 위에서 TLS 암호화 통신을 수행한다.

**Day 06: TCP Connection & Packet Analysis — Completed**

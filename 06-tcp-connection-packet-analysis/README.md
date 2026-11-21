
# Day 06. TCP Connection & Packet Analysis

## 1. 학습 목표

TCP 연결 상태와 패킷 흐름을 관찰하고, SSH 통신 과정에서 TCP Flags와 데이터 교환을 분석한다.

- TCP 3-Way Handshake 이해
- TCP 연결 상태 및 포트 구조 이해
- tcpdump를 이용한 패킷 캡처
- TCP Flags 분석
- SSH 통신의 네트워크 흐름 확인

## 2. 실습 환경

| 항목 | 환경 |
|---|---|
| Host | MacBook Air M2 |
| Virtualization | UTM |
| Guest OS | Ubuntu Server ARM64 |
| CPU | 2 vCPU |
| RAM | 약 1.6 GiB |
| Network Interface | enp0s1 |
| SSH Server | 192.168.64.3 |
| SSH Port | TCP 22 |

## 3. TCP 연결 구조

TCP 3-Way Handshake는 다음 순서로 진행된다.

1. SYN: 클라이언트가 연결 요청
2. SYN-ACK: 서버가 요청에 응답
3. ACK: 클라이언트가 응답을 확인

주요 TCP Flags:

| Flag | 의미 |
|---|---|
| SYN | 연결 요청 |
| ACK | 수신 확인 |
| PSH | 수신 데이터를 상위 계층으로 전달하도록 요청 |
| FIN | 정상적인 연결 종료 요청 |
| RST | 연결 초기화 |

## 4. 실습 명령어

### 4-1. 패킷 캡처 인터페이스 확인

```bash
sudo tcpdump -D
```

확인된 인터페이스:
- enp0s1
- any
- lo
- bluetooth-monitor
- nflog
- nfqueue

### 4-2. TCP 패킷 캡처

```bash
sudo tcpdump -i enp0s1 tcp -nn
```

SSH 포트만 필터링:

```bash
sudo tcpdump -i enp0s1 'tcp port 22' -nn
```

옵션 설명:

| 옵션 | 의미 |
|---|---|
| -i enp0s1 | 캡처 인터페이스 지정 |
| tcp | TCP 패킷 필터링 |
| port 22 | SSH 포트 필터링 |
| -nn | IP 주소와 포트를 숫자로 표시 |

## 5. 실습 결과

실제 tcpdump 출력에서 다음 통신을 확인했다.

| 항목 | 관찰 결과 |
|---|---|
| 서버 주소 | 192.168.64.3:22 |
| 상대방 주소 | 192.168.64.1:49523 |
| 프로토콜 | TCP |
| 관찰된 Flags | ACK, PSH-ACK |
| 데이터 교환 | 확인 |
| ACK 패킷 | 확인 |

### 주요 패킷 분석

**1. ACK 패킷**

```text
Flags [.]
```

ACK 플래그가 설정된 패킷이다. TCP 통신에서 상대방으로부터 받은 데이터의 수신을 확인하는 데 사용된다.

**2. PSH-ACK 패킷**

```text
Flags [P.]
```

PSH와 ACK 플래그가 함께 설정된 패킷이다. PSH는 수신 데이터를 상위 계층으로 전달하도록 요청하는 의미를 가진다.

**3. TCP 데이터 길이**

캡처 화면에서 다음과 같은 데이터 길이를 관찰했다.

- length 44
- length 68
- length 108
- length 148
- length 264
- length 628
- length 1156

반면 `length 0`인 패킷은 TCP 페이로드가 없는 패킷이다.

## 6. 보안 관점 분석

이번 실습에서 확인한 내용:

- SSH 서버와 상대방 사이의 양방향 TCP 통신
- TCP ACK를 통한 수신 확인
- PSH-ACK를 통한 데이터 전달
- TCP 페이로드가 포함된 패킷과 페이로드가 없는 패킷의 차이

주의할 점:

이번 캡처 화면에는 주로 ACK와 PSH-ACK가 관찰됐다. 따라서 TCP 연결 수립 이후의 데이터 교환을 확인한 것으로 해석했다.

SYN → SYN-ACK → ACK의 3-Way Handshake 전체 과정은 제출된 캡처 화면에서 확인되지 않았다.

또한 이번 결과만으로 비정상 접근이나 공격 여부를 판단할 수는 없다.

## 7. 학습 정리

이번 실습을 통해 tcpdump로 SSH 통신을 캡처하고 TCP Flags, IP 주소, 포트 번호, 데이터 길이를 확인했다.

특히 TCP 패킷을 단순히 수집하는 것에서 나아가 각 패킷의 Flags와 통신 방향을 기준으로 연결 상태를 해석하는 방법을 학습했다.

## 8. 다음 학습

Day 07에서는 네트워크 트래픽 분석을 확장해 DNS 요청 및 응답, 포트 기반 서비스 식별, 네트워크 통신 로그 분석을 학습할 예정이다.


# Architecture

## 1. 목적

외부 고정 IP가 제한된 학교 네트워크 환경에서 하나의 Proxmox Host를 이용해 여러 VM을 운영하면서 다음 요구사항을 만족하기 위한 구조입니다.

- VM별 독립 SSH 접근
- 여러 웹 서비스 운영
- 외부 IP 추가 할당 최소화
- VM을 사설망에 분리
- 서비스 추가가 쉬운 구조
- 장애 발생 시 계층별 진단 가능

## 2. 논리 구성

```text
[External User]
      │
      │ School Fixed IP
      ▼
[Proxmox Host]
      │
      ├─ vmbr0 : External Network
      │
      ├─ nftables
      │    ├─ DNAT
      │    ├─ MASQUERADE
      │    └─ Port Forwarding
      │
      └─ vmbr1 : 10.10.10.1/24
            │
            ├─ 10.10.10.10 Reverse Proxy
            ├─ 10.10.10.11 Backend VM
            ├─ 10.10.10.12 Backend VM
            ├─ 10.10.10.13 Backend VM
            ├─ 10.10.10.14 Backend VM
            └─ 10.10.10.15 Backend VM
```

## 3. SSH Flow

```text
Client
  │
  │ TCP 10012
  ▼
School Fixed IP
  │
  ▼
vmbr0
  │
  │ nftables PREROUTING / DNAT
  ▼
vmbr1
  │
  ▼
10.10.10.12:22
```

외부에서 들어온 SSH 연결은 Proxmox Host에서 목적지 주소를 변환한 뒤 내부 VM으로 전달합니다.

## 4. Web Flow

```text
Client
  │
  │ TCP 80 / 443
  ▼
School Fixed IP
  │
  ▼
Proxmox nftables DNAT
  │
  ▼
10.10.10.10
Nginx Reverse Proxy
  │
  ├─ /service-a → 10.10.10.11
  ├─ /service-b → 10.10.10.12
  └─ Host/Domain → Backend VM
```

## 5. 계층별 역할

| Layer | 구성 | 역할 |
|---|---|---|
| L2 | Proxmox Linux Bridge | 물리/가상 인터페이스 연결 |
| L3 | 10.10.10.0/24, IP Forwarding | 내부/외부 네트워크 간 패킷 전달 |
| L3/L4 | NAT, MASQUERADE | 내부 VM의 외부 통신 |
| L4 | DNAT, Port Forwarding | 외부 포트를 VM의 SSH로 전달 |
| L7 | Nginx Reverse Proxy | HTTP 요청을 URL 기준으로 Backend에 전달 |

## 6. 설계상 장점

### 외부 IP 절약

각 VM에 공인 IP를 부여하지 않고 하나의 고정 IP를 여러 VM이 공유할 수 있습니다.

### VM 격리

VM은 사설망에 위치하고 외부에서 직접 접근하지 않습니다.

### 서비스 확장

새 VM을 추가할 때 내부 IP와 필요한 NAT/Proxy 규칙만 추가하면 됩니다.

### 문제 분리

장애를 다음과 같이 계층별로 나눌 수 있습니다.

```text
Physical / External
        ↓
vmbr0
        ↓
NAT / DNAT
        ↓
vmbr1
        ↓
VM OS
        ↓
Application
```

## 7. 포트폴리오 표현 시 주의

실제 구축 범위와 확장 설계를 구분합니다.

**구축 경험으로 표현 가능한 범위**
- Proxmox 가상 네트워크
- 사설망 분리
- NAT / DNAT
- VM별 SSH 포트포워딩
- Nginx Reverse Proxy
- Path 기반 라우팅

**설계/확장 방향으로 표현하는 범위**
- 전체 서비스의 서브도메인 자동 할당
- Wildcard DNS 기반 사용자별 LXC 서비스
- 완전한 HTTPS/Certbot 자동화

확인되지 않은 항목은 운영 완료로 표현하지 않는 것이 좋습니다.

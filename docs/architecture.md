# Architecture

## 1. 목적

학교에서 외부 고정 IP를 하나만 할당받아 VM마다 별도의 외부 IP를 부여하기 어려웠습니다. 이 IP 하나를 최대한 공유하기 위해 VM에는 각각 사설 IP를 부여하고, Proxmox Host를 외부 진입점으로 사용했습니다. 하나의 Proxmox Host에서 여러 VM을 운영하면서 다음 요구사항을 만족하기 위한 구조입니다.

- VM별 독립 SSH 접근
- 하나의 DuckDNS 주소에서 URL 경로에 따라 VM별 웹 서비스 연결
- 외부 고정 IP 1개와 웹 진입 포트 80/443 공유
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

사용자는 같은 DuckDNS 주소의 `/service-a/`, `/service-b/` 같은 경로로 서비스에 접근했습니다. DuckDNS 이름은 학교에서 할당받은 외부 고정 IP로 연결하고, 그 IP의 80/443 포트로 들어온 연결은 DNAT를 통해 Nginx Reverse Proxy VM으로 전달했습니다.

```text
Client
  │
  │ <DUCKDNS_NAME>.duckdns.org/서비스경로/
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
  ├─ /service-a/ → 10.10.10.11:80
  └─ /service-b/ → 10.10.10.12:80
```

DuckDNS 이름과 서비스 경로는 공개용 예시입니다. **VM 선택은 Nginx가 URL Path를 기준으로 수행**합니다. 예를 들어 같은 주소의 `/service-a/` 요청은 VM A로, `/service-b/` 요청은 VM B로 전달하므로 각 VM에 외부 IP를 추가 할당하지 않아도 됩니다.

| 단계 | 역할 |
|---|---|
| DuckDNS | 도메인 이름을 학교 외부 고정 IP 1개에 연결 |
| Proxmox nftables DNAT | 외부 80/443 연결을 Reverse Proxy VM의 80/443으로 전달 |
| Nginx Reverse Proxy | 요청의 URL 경로를 확인해 해당 Backend VM의 서비스로 전달 |
| Backend VM | 사설 IP에서 각 웹 서비스를 실행 |

HTTPS 요청의 경로 분기는 TLS 종료 후 HTTP 요청을 처리하는 단계에서 이루어집니다. 443 포트를 Reverse Proxy로 전달하는 구성과 인증서 발급·갱신 자동화는 구분하며, 자동화 완료 여부는 이 문서에서 확정하지 않습니다. Host/서브도메인별 분리는 실제 Path 기반 구성과 구분되는 확장 설계입니다.

## 5. 계층별 역할

| Layer | 구성 | 역할 |
|---|---|---|
| L2 | Proxmox Linux Bridge | 물리/가상 인터페이스 연결 |
| L3 | 10.10.10.0/24, IP Forwarding | 내부/외부 네트워크 간 패킷 전달 |
| L3/L4 | NAT, MASQUERADE | 내부 VM의 외부 통신 |
| L4 | DNAT, Port Forwarding | 외부 포트를 VM의 SSH로 전달 |
| L7 | Nginx Reverse Proxy | 같은 DuckDNS 주소의 HTTP 요청을 URL Path에 따라 Backend에 전달 |

## 6. 설계상 장점

### 외부 IP 절약

학교에서 할당받은 고정 IP 하나를 여러 VM이 공유합니다. 웹 서비스는 같은 DuckDNS 주소와 80/443 포트를 사용하면서 경로로 구분하므로, VM별 외부 IP를 추가로 확보하지 않고도 여러 서비스를 제공할 수 있습니다.

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

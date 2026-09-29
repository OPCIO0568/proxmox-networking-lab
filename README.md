# Proxmox Virtual Network & Reverse Proxy

학교에서 할당받은 **외부 고정 IP 1개를 여러 VM이 공유**하기 위해 구성했던 **Proxmox 가상 네트워크 + nftables NAT/DNAT + SSH 포트포워딩 + Nginx Reverse Proxy** 구조를 포트폴리오 형태로 정리한 저장소입니다. 웹 서비스는 하나의 DuckDNS 주소에서 URL 경로에 따라 각 VM으로 연결했습니다.

> 공개 포트폴리오용 문서이므로 실제 학교 외부 IP, 사용자 계정, 현재 운영 중인 민감한 서비스 주소는 제외했습니다.

![Architecture](assets/proxmox-network-architecture.svg)

## 프로젝트 개요

학교에서 외부 고정 IP를 하나만 할당해 주었기 때문에 VM마다 별도의 외부 IP를 부여하기 어려웠습니다. 따라서 추가 외부 IP 없이 **하나의 IP를 최대한 공유하면서 여러 VM의 서비스를 외부에 제공하는 것**이 구축의 출발점이었습니다. 각 VM에는 서로 다른 사설 IP를 부여하고, 외부 접속은 Proxmox Host의 고정 IP 하나로 모았습니다.

웹 서비스에는 **DuckDNS 주소 + URL 경로** 방식을 적용했습니다. 같은 DuckDNS 주소의 80/443 포트로 들어온 요청을 Reverse Proxy VM으로 전달하고, Nginx가 주소 뒤의 `/service-a/`, `/service-b/` 같은 경로를 기준으로 서비스가 실행 중인 VM을 선택하도록 구성했습니다. 사용자는 VM별 외부 IP나 별도 웹 포트를 기억할 필요 없이 같은 주소에서 경로만 바꾸어 서비스에 접근할 수 있었습니다. SSH 접속은 별도의 외부 포트 번호로 VM을 구분했습니다.

이를 해결하기 위해 네트워크를 다음 계층으로 나누어 구성했습니다.

```text
L2  : Proxmox Linux Bridge
L3  : Private Network / IP Forwarding
L3/4: NAT / MASQUERADE / DNAT
L4  : SSH Port Forwarding
L7  : Nginx Reverse Proxy
```

핵심은 **외부 고정 IP 1개를 Proxmox Host의 진입점으로 사용하고, 내부 VM은 사설망으로 분리한 뒤 목적에 따라 포트포워딩 또는 Reverse Proxy로 전달하는 구조**입니다.

---

## 전체 구조

```mermaid
flowchart TB
    U["External User"] -->|"SSH / HTTP(S)"| PUB["School Fixed IP"]

    subgraph PVE["Proxmox Host"]
        B0["vmbr0<br/>External Network"]
        NAT["nftables<br/>DNAT / MASQUERADE"]
        B1["vmbr1<br/>10.10.10.1/24"]
        B0 --> NAT --> B1
    end

    PUB --> B0

    B1 --> RP["10.10.10.10<br/>Nginx Reverse Proxy"]
    B1 --> VM11["10.10.10.11<br/>VM"]
    B1 --> VM12["10.10.10.12<br/>VM"]
    B1 --> VM13["10.10.10.13<br/>VM"]
    B1 --> VM14["10.10.10.14<br/>VM"]
    B1 --> VM15["10.10.10.15<br/>VM"]

    RP -->|"Path Routing"| VM11
    RP -->|"Path Routing"| VM12
    RP -->|"Path Routing"| VM15

    PUB -. "TCP 10010 → :22" .-> RP
    PUB -. "TCP 10011 → :22" .-> VM11
    PUB -. "TCP 10012 → :22" .-> VM12
    PUB -. "TCP 10013 → :22" .-> VM13
    PUB -. "TCP 10014 → :22" .-> VM14
```

### 트래픽 분리 방식

| 목적 | 외부 요청 | 처리 방식 | 내부 목적지 |
|---|---|---|---|
| SSH | 고정 IP + 개별 고포트 | nftables DNAT | 각 VM의 TCP 22 |
| Web | DuckDNS 주소 + 80/443 + URL 경로 | DNAT → Nginx Reverse Proxy | 요청 Path에 맞는 Backend VM |

SSH는 **포트 번호**로 VM을 구분하고, 실제 구성한 웹 서비스는 **하나의 DuckDNS 주소 뒤의 URL Path**로 서비스를 구분하도록 역할을 나누었습니다. Host/서브도메인 기반 분리는 아래의 확장 설계로 구분했습니다.

---

## Proxmox 가상 네트워크

외부 네트워크와 내부 VM 네트워크를 Linux Bridge로 분리했습니다.

```text
vmbr0
 └─ 학교 외부 네트워크
    └─ Proxmox Host의 고정 IP

vmbr1
 └─ 10.10.10.0/24 Private Network
    ├─ 10.10.10.10  Reverse Proxy
    ├─ 10.10.10.11  Backend VM
    ├─ 10.10.10.12  Backend VM
    ├─ 10.10.10.13  Backend VM
    ├─ 10.10.10.14  Backend VM
    └─ 10.10.10.15  Backend VM
```

VM을 외부망에 직접 노출하지 않고 사설망에 배치하여 **Proxmox Host를 네트워크 진입점으로 통일**했습니다.

---

## NAT / IP Forwarding

내부 VM이 Proxmox Host를 통해 외부 네트워크와 통신하도록 Linux IP Forwarding을 사용했습니다.

```bash
sysctl net.ipv4.ip_forward
```

활성화 상태 예시:

```text
net.ipv4.ip_forward = 1
```

내부 VM의 외부 통신에는 `MASQUERADE`를 적용했습니다.

```nft
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 10.10.10.0/24 oifname "vmbr0" masquerade
    }
}
```

흐름:

```text
10.10.10.11
    ↓
vmbr1
    ↓
Proxmox Host
    ↓ MASQUERADE
vmbr0
    ↓
External Network
```

---

## SSH 포트포워딩

외부 IP가 하나이기 때문에 외부의 서로 다른 포트를 각 VM의 SSH 22번 포트에 매핑했습니다.

```text
External :10010 → 10.10.10.10:22
External :10011 → 10.10.10.11:22
External :10012 → 10.10.10.12:22
External :10013 → 10.10.10.13:22
External :10014 → 10.10.10.14:22
```

개념적인 nftables 구성:

```nft
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        iifname "vmbr0" tcp dport 10010 dnat to 10.10.10.10:22
        iifname "vmbr0" tcp dport 10011 dnat to 10.10.10.11:22
        iifname "vmbr0" tcp dport 10012 dnat to 10.10.10.12:22
        iifname "vmbr0" tcp dport 10013 dnat to 10.10.10.13:22
        iifname "vmbr0" tcp dport 10014 dnat to 10.10.10.14:22
    }
}
```

외부에서는 같은 IP를 사용하되 포트만 구분하여 원하는 VM에 접근할 수 있습니다.

```bash
ssh -p 10011 user@<SCHOOL_FIXED_IP>
ssh -p 10012 user@<SCHOOL_FIXED_IP>
```

---

## Web 트래픽과 Reverse Proxy

웹 서비스는 서비스마다 외부 포트를 다르게 노출하지 않고 80/443을 Reverse Proxy VM으로 집중시켰습니다.

```text
External :80  → 10.10.10.10:80
External :443 → 10.10.10.10:443
```

개념적인 DNAT 규칙:

```nft
iifname "vmbr0" tcp dport 80  dnat to 10.10.10.10:80
iifname "vmbr0" tcp dport 443 dnat to 10.10.10.10:443
```

Reverse Proxy VM의 Nginx가 요청의 URL 경로를 확인한 뒤 실제 서비스가 실행 중인 Backend VM으로 전달하도록 구성했습니다. 여기서 DuckDNS는 이름을 외부 고정 IP로 연결하고, nftables는 80/443 연결을 Reverse Proxy VM으로 전달하며, Nginx가 `/경로`를 읽어 목적지 서비스를 선택합니다.

443으로 들어오는 HTTPS 요청의 경로를 Nginx에서 분기하려면 TLS를 종료한 뒤 HTTP 요청을 처리해야 합니다. 443 포트 전달과 HTTPS 인증서 발급·갱신 자동화는 별개의 항목이며, 인증서 자동화의 완료 여부는 기존 문서와 같이 확인된 구축 범위에 포함하지 않습니다.

---

## Path 기반 Reverse Proxy

실제 구축에서는 **`<DUCKDNS_NAME>.duckdns.org/서비스경로/`** 형태로 접근하도록 했습니다. 같은 DuckDNS 주소와 외부 고정 IP를 사용하면서, 슬래시 뒤의 경로에 따라 서로 다른 VM의 웹 서비스로 요청을 전달했습니다.

아래의 DuckDNS 이름과 서비스 경로는 공개용으로 일반화한 예시입니다.

```text
http://<DUCKDNS_NAME>.duckdns.org/service-a/
        ↓
10.10.10.11:80

http://<DUCKDNS_NAME>.duckdns.org/service-b/
        ↓
10.10.10.12:80
```

예시:

```nginx
server {
    listen 80;

    location /service-a/ {
        proxy_pass http://10.10.10.11:80/;
    }

    location /service-b/ {
        proxy_pass http://10.10.10.12:80/;
    }
}
```

이 구성으로 여러 VM의 웹 서비스가 **외부 고정 IP 1개와 공통 80/443 진입점**을 공유하도록 했습니다. DuckDNS 주소 자체가 VM을 구분하는 것이 아니라, Reverse Proxy의 Nginx `location` 설정이 URL 경로를 각 VM의 서비스에 연결합니다. 위 설정은 HTTP 경로 분기를 보여 주는 예시이며, 실제 도메인과 서비스명은 공개하지 않았습니다.

---

## Host 기반 Reverse Proxy

서브도메인을 기준으로 Backend를 나누는 방식은 여러 서비스를 더 명확하게 분리하기 위한 **확장 설계**로 정리했습니다.

```text
service-a.example.com
        ↓
Nginx Reverse Proxy
        ↓
10.10.10.11

service-b.example.com
        ↓
Nginx Reverse Proxy
        ↓
10.10.10.12
```

예시:

```nginx
server {
    listen 80;
    server_name service-a.example.com;

    location / {
        proxy_pass http://10.10.10.11:80;
    }
}

server {
    listen 80;
    server_name service-b.example.com;

    location / {
        proxy_pass http://10.10.10.12:80;
    }
}
```

> Host/Subdomain 기반 분리는 설계 및 확장 방향으로 기록하며, 실제 학교 서버에서 모든 서브도메인 구성이 운영 완료되었다고 과장하지 않습니다.

---

## 장애 분석 방식

접속 문제가 발생하면 전체 시스템을 한 번에 보지 않고 패킷 경로를 기준으로 범위를 좁혔습니다.

```text
External Network
      ↓
Proxmox vmbr0
      ↓
nftables NAT / DNAT
      ↓
vmbr1
      ↓
VM Network
      ↓
SSH / Nginx / Application
```

확인 순서:

```bash
# VM 서비스
ss -lntp
systemctl status ssh

# 내부 네트워크
ping 10.10.10.11
ssh user@10.10.10.11

# Routing
ip addr
ip route
sysctl net.ipv4.ip_forward

# NAT / DNAT
sudo nft list ruleset

# Reverse Proxy
sudo nginx -t
systemctl status nginx
curl -I http://10.10.10.11
```

이 방식으로 문제를 **외부 네트워크 → NAT → Bridge → VM → Application** 단계로 나누어 확인했습니다.

---

## 기술 스택

`Proxmox VE` `Linux` `Linux Bridge` `TCP/IP` `Private Network` `IP Forwarding` `nftables` `NAT` `DNAT` `MASQUERADE` `SSH` `Nginx` `Reverse Proxy`

---

## 결과

- 학교에서 할당받은 외부 고정 IP 1개를 여러 VM의 SSH 및 웹 서비스가 공유하는 구조 구성
- `vmbr0` 외부망과 `vmbr1` 사설망 분리
- `10.10.10.0/24` Private Network 구성
- nftables DNAT 기반 VM별 SSH 포트포워딩
- MASQUERADE 기반 내부 VM의 외부 통신 구성
- 80/443 트래픽을 Reverse Proxy VM으로 집중
- 하나의 DuckDNS 주소 뒤의 URL Path에 따라 Nginx가 VM별 웹 서비스로 요청을 분기
- VM 추가 시 사설 IP와 포트 규칙을 추가하는 방식으로 확장 가능한 구조 확보
- 패킷 이동 경로를 네트워크 계층별로 나누어 장애를 분석하는 경험 확보

---

## 배운 점

이 경험을 통해 단순히 “포트가 열려 있는지”만 보는 것이 아니라 **패킷이 실제 서비스까지 어떤 경로를 거치는지 이해해야 네트워크 문제를 정확히 찾을 수 있다는 점**을 배웠습니다.

또한 가상화 환경에서는 VM 생성 자체보다 외부망과 내부망의 분리, Routing, NAT, 포트포워딩, Reverse Proxy까지 함께 설계해야 실제 다중 사용자 서비스 환경으로 사용할 수 있다는 것을 경험했습니다.

---

## 상세 문서

- [Architecture](docs/architecture.md)
- [Configuration Examples](docs/config-examples.md)

---

## 공개 범위 주의사항

- 실제 학교 외부 고정 IP는 공개하지 않음
- 사용자 계정 및 인증정보는 공개하지 않음
- 현재 사용 중인 서비스 도메인/Endpoint는 일반화
- HTTPS 인증서 자동화는 완료 여부가 확실하지 않아 구축 완료 사항으로 기재하지 않음
- Host/Subdomain 기반 Reverse Proxy는 확장 설계와 실제 구축 범위를 구분하여 작성

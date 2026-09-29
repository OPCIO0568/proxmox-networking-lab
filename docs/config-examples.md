# Configuration Examples

> 아래 설정은 당시 구성 구조를 설명하기 위해 공개용으로 일반화한 예시입니다. 실제 운영 서버에 그대로 적용하기 전에 인터페이스명, IP 대역, 방화벽 정책을 확인해야 합니다.

## 1. IP Forwarding

일시 확인:

```bash
sysctl net.ipv4.ip_forward
```

활성화 예시:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

영구 설정 예시:

```text
# /etc/sysctl.d/99-ip-forward.conf
net.ipv4.ip_forward=1
```

적용:

```bash
sudo sysctl --system
```

## 2. nftables NAT

```nft
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # SSH
        iifname "vmbr0" tcp dport 10010 dnat to 10.10.10.10:22
        iifname "vmbr0" tcp dport 10011 dnat to 10.10.10.11:22
        iifname "vmbr0" tcp dport 10012 dnat to 10.10.10.12:22
        iifname "vmbr0" tcp dport 10013 dnat to 10.10.10.13:22
        iifname "vmbr0" tcp dport 10014 dnat to 10.10.10.14:22

        # Reverse Proxy
        iifname "vmbr0" tcp dport 80  dnat to 10.10.10.10:80
        iifname "vmbr0" tcp dport 443 dnat to 10.10.10.10:443
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        ip saddr 10.10.10.0/24 oifname "vmbr0" masquerade
    }
}
```

현재 규칙 확인:

```bash
sudo nft list ruleset
```

## 3. SSH

외부 접속 예시:

```bash
ssh -p 10011 user@<SCHOOL_FIXED_IP>
ssh -p 10012 user@<SCHOOL_FIXED_IP>
```

내부에서 직접 테스트:

```bash
ssh user@10.10.10.11
```

## 4. Nginx Path Reverse Proxy

```nginx
server {
    listen 80;
    server_name _;

    location /service-a/ {
        proxy_pass http://10.10.10.11:80/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /service-b/ {
        proxy_pass http://10.10.10.12:80/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

확인:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 5. Host 기반 Reverse Proxy 예시

이 구성은 서비스 분리를 위한 확장 설계 예시입니다.

```nginx
server {
    listen 80;
    server_name service-a.example.com;

    location / {
        proxy_pass http://10.10.10.11:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 6. 트러블슈팅 체크리스트

### Proxmox Host

```bash
ip addr
ip route
bridge link
sysctl net.ipv4.ip_forward
sudo nft list ruleset
```

### VM

```bash
ip addr
ip route
ss -lntp
systemctl status ssh
```

### Reverse Proxy VM

```bash
sudo nginx -t
systemctl status nginx
ss -lntp
curl -I http://10.10.10.11
curl -I http://10.10.10.12
```

### 확인 순서

```text
1. 외부 IP까지 도달하는가?
2. Proxmox vmbr0로 패킷이 들어오는가?
3. DNAT 규칙이 맞는가?
4. vmbr1에서 목적지 VM까지 도달하는가?
5. VM의 서비스가 해당 포트에서 LISTEN 중인가?
6. Reverse Proxy라면 Backend에 직접 접근 가능한가?
7. Nginx 설정과 요청 Host/Path가 일치하는가?
```

## 7. 실제 서버에서 확인할 명령

과거 구성값을 다시 검증할 때는 다음 순서로 확인하면 됩니다.

```bash
sudo nft list ruleset
cat /etc/network/interfaces
sudo cat /etc/nftables.conf
sysctl net.ipv4.ip_forward
```

Reverse Proxy VM:

```bash
sudo nginx -T
```

이 명령들을 이용하면 현재 운영 상태와 과거 기록의 차이를 다시 검증할 수 있습니다.

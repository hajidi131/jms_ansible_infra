# jms.infra — 코드형 온프레미스 인프라 컬렉션

Ansible Collection + Role 구조로 DNS · WEB(+Tomcat) · FTP · MAIL 4대 서버를
**코드 한 번**으로 구축 · 배포 · 원복하는 인프라 자동화 프로젝트입니다.


#### 사전 환경설정 스크립트(필수!!!!!!!!!!)####
# ------------------------------------------------------------------
# 1. 환경 변수 설정 (여기에 실제 root 비밀번호를 입력하세요)
# ------------------------------------------------------------------
ROOT_PASS="여기에_실제_ROOT_비밀번호_입력"
ANSIBLE_PASS="ansible"

# 2. 제어 노드에 SSH 키가 없으면 생성
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 3. 자동화를 위한 sshpass 패키지 설치 (CentOS/RHEL/Rocky 기반 OS 기준)
sudo dnf install -y epel-release && sudo dnf install -y sshpass

# 4. 대상 서버 리스트 순회하며 한 줄 명령어로 실행
for IP in 192.168.30.10 192.168.30.11 192.168.30.12 192.168.30.13; do
    echo "==================== 작업 중: $IP ===================="
    
    # Known hosts 에러 방지
    ssh-keyscan -H "$IP" >> ~/.ssh/known_hosts 2>/dev/null
    
    # [A] root로 접속하여 유저 생성, 비밀번호 설정, sudoers 권한 부여를 단일 명령어로 처리
    sshpass -p "$ROOT_PASS" ssh -o StrictHostKeyChecking=no root@$IP "id -u ansible &>/dev/null || useradd -m -s /bin/bash ansible; echo 'ansible:$ANSIBLE_PASS' | chpasswd; grep -q '^ansible' /etc/sudoers || echo 'ansible ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers"
    
    # [B] 생성된 ansible 계정으로 SSH 키 배포
    sshpass -p "$ANSIBLE_PASS" ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_ed25519.pub ansible@$IP
    
    echo "[완료] $IP 서버 설정 완료"
done






---

## 📌 구성도

```
                         [ 제어 노드 : myDNS ]
                          ansible-playbook
                                 │
           ┌─────────────────────┼──────────────────┬──────────────────┐
           │                     │                  │                  │
      [myDNS]             [ansible-myWEB]   [ansible-myFTP]   [ansible-myMAIL]
    192.168.30.10          192.168.30.11     192.168.30.12     192.168.30.13
     BIND (named)         Apache + Tomcat      vsftpd         Postfix + Dovecot
       DNS (53)            HTTP (80/443)        FTP (21)       SMTP (25)
                          ┌──────────────┐   PASV (40000~)    POP3 (110)
                          │   Apache     │                     IMAP (143)
                          │  mod_proxy   │
                          │     AJP      │
                          │  Tomcat 9    │  ← 127.0.0.1 전용
                          │  (8080/8009) │    외부 직접 접근 불가
                          └──────────────┘
```

| 호스트명 | IP | OS | 서비스 | 포트 |
|----------|-----|----|--------|------|
| myDNS | 192.168.30.10 | CentOS Stream 9 | BIND (named) | 53 |
| ansible-myWEB | 192.168.30.11 | CentOS Stream 9 | Apache httpd + Tomcat 9 | 80, 443 |
| ansible-myFTP | 192.168.30.12 | CentOS Stream 9 | vsftpd | 21, 40000-40100 |
| ansible-myMAIL | 192.168.30.13 | CentOS Stream 9 | Postfix + Dovecot | 25, 110, 143, 465, 587, 993, 995 |

---

## 📁 디렉토리 구조

```
jms_infra/
├── ansible.cfg                  # Ansible 동작 설정
├── inventory/
│   ├── hosts                    # 대상 서버 목록 및 그룹
│   ├── group_vars/
│   │   └── all.yml              # 전체 공통 변수 (도메인, IP, 호스트 목록)
│   └── host_vars/
│       ├── myDNS.yml            # host_role: dns
│       ├── ansible-myWEB.yml    # host_role: web
│       ├── ansible-myFTP.yml    # host_role: ftp
│       └── ansible-myMAIL.yml   # host_role: mail
├── site.yml                     # 전체 구축 마스터 플레이북
├── dns.yml                      # DNS 서버만 단독 구축
├── web.yml                      # WEB + Tomcat 단독 구축
├── ftp.yml                      # FTP 서버 단독 구축
├── mail.yml                     # MAIL 서버 단독 구축
├── t.yml                        # 테스트용 임시 플레이북 (common 역할 검증)
├── restore.yml                  # 전체 원복 (서비스 중지 + 패키지 제거)
├── requirements.yml             # 외부 컬렉션 의존성 (ansible.posix >= 1.5.0)
├── secret.yml                   # 민감 변수 (ansible-vault 암호화 권장)
├── vault-pass                   # vault 암호 파일 (git 커밋 금지)
├── .gitignore
└── collections/
    └── ansible_collections/
        └── jms/
            └── infra/                    # ★ 배포 단위 컬렉션
                ├── galaxy.yml            # 컬렉션 메타데이터 (namespace: jms, name: infra)
                ├── meta/
                │   └── runtime.yml       # 최소 Ansible 버전 선언
                └── roles/
                    ├── common/           # 전 서버 공통 (/etc/hosts, 기본 패키지)
                    │   ├── defaults/main.yml
                    │   ├── tasks/main.yml
                    │   ├── templates/hosts.j2
                    │   └── meta/main.yml
                    ├── myfirewall/       # 방화벽 공유 역할 (종속성으로만 호출)
                    │   ├── defaults/main.yml
                    │   ├── tasks/main.yml
                    │   └── meta/main.yml
                    ├── mydns/            # BIND DNS
                    │   ├── defaults/main.yml
                    │   ├── vars/main.yml
                    │   ├── tasks/main.yml
                    │   ├── handlers/main.yml
                    │   ├── templates/
                    │   │   ├── named.conf.j2
                    │   │   ├── named.rfc1912.zones.j2
                    │   │   ├── forward.zone.j2
                    │   │   └── reverse.zone.j2
                    │   └── meta/main.yml
                    ├── myweb/            # Apache + Tomcat 9
                    │   ├── defaults/main.yml
                    │   ├── tasks/
                    │   │   ├── main.yml
                    │   │   └── tomcat.yml   # Tomcat 설치/AJP 연동 (include_tasks)
                    │   ├── handlers/main.yml
                    │   ├── files/index.html
                    │   ├── templates/
                    │   │   ├── vhost.conf.j2
                    │   │   ├── server.xml.j2
                    │   │   ├── tomcat.service.j2
                    │   │   └── index.jsp.j2
                    │   └── meta/main.yml
                    ├── myftp/            # vsftpd
                    │   ├── defaults/main.yml
                    │   ├── vars/main.yml
                    │   ├── tasks/main.yml
                    │   ├── handlers/main.yml
                    │   ├── files/banner.txt
                    │   ├── templates/vsftpd.conf.j2
                    │   └── meta/main.yml
                    └── mymail/           # Postfix + Dovecot
                        ├── defaults/main.yml
                        ├── vars/main.yml
                        ├── tasks/main.yml
                        ├── handlers/main.yml
                        ├── templates/main.cf.j2
                        └── meta/main.yml
```

---

## 🔧 역할(Role) 구성

| 역할 | 서비스 | 개방 포트 (myfirewall) | 주요 특징 |
|------|--------|------------------------|-----------|
| `common` | /etc/hosts, 기본 패키지 | — | 전 서버 공통, 가장 먼저 실행 |
| `myfirewall` | firewalld | — | 공유 역할, 직접 호출 안 함 |
| `mydns` | BIND (named) | dns (53) | named-checkconf/checkzone 사전 검증 |
| `myweb` | Apache + Tomcat 9 | http (80), https (443) | mod_proxy_ajp, SELinux 자동 설정 |
| `myftp` | vsftpd | ftp (21) | 로컬 사용자 + chroot + PASV |
| `mymail` | Postfix + Dovecot | smtp, smtps, submission, pop3, pop3s, imap, imaps | SASL 인증 연동 |

### 핵심 패턴 — 종속성 (meta/main.yml)

```
ansible-playbook site.yml 실행
        ↓
mydns 역할 시작 전 → meta/main.yml 확인
        ↓
myfirewall 먼저 자동 실행 (fw_svcs: [dns] 전달 → 53번 포트 개방)
        ↓
mydns 실행 (BIND 설치/설정/기동)
```

```yaml
# roles/mydns/meta/main.yml
dependencies:
  - role: jms.infra.myfirewall
    fw_svcs: [dns]          # myfirewall 에 전달 → firewalld 에서 dns(53) 개방
```

같은 패턴이 myweb(http/https), myftp(ftp), mymail(smtp/pop3/imap 등) 에도 동일하게 적용됨.

---

## ⚙️ 주요 변수

### inventory/group_vars/all.yml

| 변수 | 현재 설정값 | 설명 |
|------|------------|------|
| `domain_name` | `jms.com` | 전체 인프라 도메인 — 이 한 줄만 바꾸면 전체 반영 |
| `dns_server_ip` | `192.168.30.10` | 네임서버 IP |
| `internal_network` | `192.168.30.0/24` | 내부 네트워크 대역 (Postfix mynetworks) |
| `reverse_zone` | `30.168.192.in-addr.arpa` | 역방향 DNS 존 |
| `infra_hosts` | 4대 목록 | DNS 존 파일과 /etc/hosts 가 이 목록을 공유 |

> `domain_name` 변경 시 존 파일 · /etc/hosts · vhost.conf · postfix main.cf 가 모두 자동으로 따라 바뀜.
> 단, 변경 후에는 `roles/mydns/vars/main.yml` 의 `zone_serial` 숫자를 반드시 올려야 함.

### roles/myweb/defaults/main.yml (Tomcat 관련)

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `tomcat_enabled` | `true` | `false` 로 바꾸면 Tomcat 설치 건너뜀 |
| `tomcat_version` | `9.0.118` | Tomcat 버전 |
| `tomcat_proxy_path` | `/app` | 이 경로 요청을 Tomcat 으로 전달 |
| `tomcat_ajp_port` | `8009` | AJP 포트 (127.0.0.1 전용, 외부 비노출) |
| `tomcat_http_port` | `8080` | HTTP 포트 (127.0.0.1 전용, 외부 비노출) |

> Tomcat 설치 시 관리 호스트가 인터넷(archive.apache.org)에 접근 가능해야 함.
> 폐쇄망이면 `roles/myweb/files/` 에 tar.gz 미리 넣고 `get_url` → `copy` 로 변경 필요.

---

## 🚀 사용 방법

### 0. 사전 준비

**관리 호스트 4대 각각에서 (root 로):**
```bash
useradd ansible
echo 'soldesk1.' | passwd --stdin ansible
echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible
```

**제어 노드에서:**
```bash
# Ansible 설치
sudo dnf install -y ansible

# SSH 공개키 배포 (비번 한 번만 입력)
ssh-keygen -t ed25519
for ip in 10 11 12 13; do
  ssh-copy-id ansible@192.168.30.$ip
done

# 외부 컬렉션 설치 (firewalld, seboolean 모듈)
ansible-galaxy collection install -r requirements.yml
```

### 1. 연결 확인

```bash
ansible all -m ping
# 4대 모두 "pong" 이 나와야 다음 단계 진행
```

### 2. 구축 실행 (항상 이 3단계 순서)

```bash
ansible-playbook site.yml --syntax-check   # ① 문법 검사
ansible-playbook site.yml --check          # ② 미리보기 (실제 변경 없음)
ansible-playbook site.yml                   # ③ 실제 구축
```

### 3. 개별 서비스만 구축

```bash
ansible-playbook dns.yml    # DNS 만
ansible-playbook web.yml    # WEB + Tomcat 만
ansible-playbook ftp.yml    # FTP 만
ansible-playbook mail.yml   # MAIL 만

# 특정 서버만 실행
ansible-playbook site.yml --limit web
```

---

## ✅ 검증

```bash
# DNS (내 DNS 서버가 응답하는지 확인)
nslookup www.jms.com          # Server: 192.168.30.10, Address: 192.168.30.11
nslookup mail.jms.com         # 192.168.30.13
dig @192.168.30.10 -x 192.168.30.13 +short  # → mail.jms.com.

# DNS 가 1순위인지 확인
cat /etc/resolv.conf          # nameserver 192.168.30.10 이 첫 번째여야 함

# WEB — Apache 정적 페이지
curl http://www.jms.com

# WEB — Apache → Tomcat 동적 페이지
curl http://www.jms.com/app/  # "Hello from Apache Tomcat" + 서버시각 출력

# FTP
ftp 192.168.30.12             # user01 / soldesk1. 로그인 후 put 테스트

# MAIL
echo "test" | mail -s "hello" user02@jms.com
# user02 로 로그인 후 mail 명령으로 수신 확인
```

### 멱등성 확인 (최종 관문)

```bash
ansible-playbook site.yml     # 두 번째 실행
# 전 태스크 changed=0 이면 완성
```

---

## 📦 컬렉션 빌드 & Galaxy 배포

```bash
cd collections/ansible_collections/jms/infra

# 배포 단위 패키지 생성
ansible-galaxy collection build
# → jms-infra-1.0.0.tar.gz 생성

# Ansible Galaxy 업로드 (https://galaxy.ansible.com 에서 API 키 발급)
ansible-galaxy collection publish jms-infra-1.0.0.tar.gz --api-key=발급받은API키
```

---

## 🛠️ 트러블슈팅

실습 중 실제로 겪은 에러 목록입니다.

| 증상 | 원인 | 해결 |
|------|------|------|
| `ping` 실패 | SSH 키 / ansible 계정 / sudo 미설정 | 사전 준비 0단계 재확인 |
| `ansible.posix.firewalld` 모듈 없음 | ansible.posix 미설치 | `ansible-galaxy collection install -r requirements.yml` |
| `repo BaseOS` 다운로드 실패 | CD-ROM 미마운트 또는 저장소 URL 오류 | `mount /dev/cdrom /mnt/cdrom` 또는 인터넷 저장소로 교체 |
| `named.conf` validate 에러 (`...` unknown option) | 템플릿에 생략 표시(`...`)가 그대로 들어감 | `vi` 로 직접 편집하여 완전한 내용 작성 |
| `validate must contain %s` | validate 명령에 `%s` 누락 | `httpd -t` → `httpd -t -f %s` |
| `No MPM loaded` (httpd validate) | vhost.conf 단독 문법 검사 불가 | `validate:` 줄 제거 |
| named 가 안 뜸 | 존 파일 문법 오류 | `named-checkzone jms.com /var/named/example.zone` |
| `ftp_home_dir` SELinux 없음 | CentOS 9 불리언 이름 변경 | `ftp_home_dir` → `ftpd_full_access` |
| `/app/` 502 에러 | SELinux 차단 또는 Tomcat 미기동 | `setsebool -P httpd_can_network_connect on` |
| Tomcat 다운로드 실패 (`Network is unreachable`) | 관리 호스트 인터넷 불가 | 제어 노드에서 `wget` 후 `files/` 에 넣고 `copy` 모듈로 변경 |
| 존 변경 후 반영 안 됨 | zone_serial 미증가 | `roles/mydns/vars/main.yml` 의 `zone_serial` 숫자 증가 |
| `nslookup` 이 8.8.8.8 응답 | resolv.conf DNS 순서 문제 | `nmcli con mod eth0 ipv4.dns "192.168.30.10 8.8.8.8"` |
| `shell` 모듈 파싱 에러 | 명령에 줄바꿈 포함 | 한 줄로 작성 |
| `changed` 가 계속 뜸 | 비멱등 태스크 | `lineinfile` 의 `regexp` 또는 `creates:` 점검 |

---

## 📋 환경

| 항목 | 버전 |
|------|------|
| OS | CentOS Stream 9 |
| Ansible | 2.14+ |
| ansible.posix | 1.5.0+ |
| BIND | 9.x |
| Apache httpd | 2.4.x |
| Tomcat | 9.0.118 |
| Java | OpenJDK 17 |
| vsftpd | 3.x |
| Postfix | 3.x |
| Dovecot | 2.x |

---

## 📝 라이선스

MIT License

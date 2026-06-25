# myftp

jms_infra 컬렉션의 myftp 역할입니다.

## 설명
vsftpd FTP 서버 구축. 로컬 사용자 인증 + chroot + PASV 모드.

## 사용법

```yaml
- hosts: 대상그룹
  roles:
    - hajidi131.jms_infra.myftp
```

# myweb

jms_infra 컬렉션의 myweb 역할입니다.

## 설명
Apache httpd + Tomcat 9 구축. mod_proxy_ajp 로 /app 경로 연동.

## 사용법

```yaml
- hosts: 대상그룹
  roles:
    - hajidi131.jms_infra.myweb
```

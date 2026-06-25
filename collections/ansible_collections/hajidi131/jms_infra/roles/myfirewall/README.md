# myfirewall

jms_infra 컬렉션의 myfirewall 역할입니다.

## 설명
firewalld 설치/기동 및 포트 개방. 다른 역할의 종속성으로 호출됨.

## 사용법

```yaml
- hosts: 대상그룹
  roles:
    - hajidi131.jms_infra.myfirewall
```

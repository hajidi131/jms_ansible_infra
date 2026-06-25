# mymail

jms_infra 컬렉션의 mymail 역할입니다.

## 설명
Postfix(SMTP) + Dovecot(POP3/IMAP) 구축. SASL 인증 연동.

## 사용법

```yaml
- hosts: 대상그룹
  roles:
    - hajidi131.jms_infra.mymail
```

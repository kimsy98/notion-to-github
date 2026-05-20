# SSM 터널링&RDS, OpenSearch 접근 방법

> 날짜: 2026-05-20
> 원본 노션: [링크](https://www.notion.so/SSM-RDS-OpenSearch-365edfd5a476804fbab1feb39f1fffd5)

---

보안을 위해 cloudshell 접근 권한 X

## ssm 터널링을 통해 접근(ssh 대체)

### AWS SSM(System Manager 또는 Simple System Manager)

![Image](https://prod-files-secure.s3.us-west-2.amazonaws.com/9a7eb97d-889f-4e17-b799-95d6f957652d/b83d6e98-6799-4545-a42d-df2246ded16b/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466VLQ5ZUR4%2F20260520%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260520T175609Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjECoaCXVzLXdlc3QtMiJGMEQCIAb%2Fq4mfzkvU%2FpJGiPdaMMjWQWswXD%2F9ixCBnDvCfRB1AiAICEN7FLDN9PAKTuEHatqD54I%2FffopMqWBKMXW3MfCUiqIBAjy%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F8BEAAaDDYzNzQyMzE4MzgwNSIMniHiMpkJgRbw7MkhKtwDPEc%2Bb0M0k4JdZbItiXurXycG6uKMUbxyFLKm8zviCye3BjQ6mFPetn3Apw%2BrjfXbBqjlM7Yw1QtEpe7kReKWv59mdQMlhLrfIqfxxM%2BL1aNJX8hzm%2Bzq%2FRo%2FunlhAxFflDUcheQwKGM046IpZ6bBn%2FCfUwXilh0DT4YYJOzCRo13jFBMkC7lU%2BDNMHCBGu%2BOZ20h2cDgY1cj78Q1Utn6XYYpJLU9z6uLqJEE1AXS0ePUYzsmG%2FD6RQdgUYPCKT9V8GdXlJFbrpMTkPxFBOfeIZrAd7TUhsvogq8HBrKDxe3G2fja%2F%2BzkkqFMqbhvF%2BqHIbg1TuOFGp4wqcST%2BSEb3u33sRi3akji8SlZGbpL%2FwlEBGJZDoE6P2nC%2FlctaJgh3wLu8RFVtcPp7AArAMNY0Cq6NH13myUeRFLP3bx2tmKrUbrM7%2Fnvc2qBvYO9XdfZvUJKb%2FHSe0R900uBF809sqD14imCPBQowwkpeHMpiiUqymYN3a3qVsa%2B9FStmSj3MiyqZnhu3PYyIoMIx%2FLlrZx%2BogZR93IrhEm5w4thBvouBT6ej%2FKT5U4Ou2NWAosAKr7W7VMJYSUhY38xy33YiZYjbG%2BEMKDevCyHdat4VIJAc%2FmnhBT%2FzxdHFFEwq9q30AY6pgG0A0PF9Qwnr2Td0KDl%2FSS%2F51rmlxZ5gDxSKR4oU0WIqenP6z7NLsY4P5d0dzfUHCx8Pvh3jv5riwZApHoqwwX0zvYEXcV5YKoAwAdmZ%2FR6J%2F2VsCUXSXV5ZnJZoBz%2FDKLxnrwP0KvWVepTB5wPJ3YNUk6W8tdHPIGcNeliHIqg0T42mtDyaVa%2BqwMEkgOwKM5XO%2BdmDO6efH1h7AcvF7ZfhaQ85AIi&X-Amz-Signature=2e5a8a186d373164a91d18ce4659388e14700f1c5f53ac13633ddfab8061c246&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)

로컬 PC와 원격 리소스(RDS 등) 사이에 암호화된 통로를 만드는 방식

- 핵심: 외부에서 22번 포트(SSH)로 들어오는 게 아니라, 리소스(EC2 등)에 설치된 SSM Agent가 AWS 백엔드 서비스와 양방향 통신을 맺고 있는 것을 이용
- 비유: SSH가 건물 정문(포트)을 부수고 들어가는 방식이라면 SSM 터널링은 건물의 환기구(SSM Agent의 외부 통신)를 통해 몰래 밧줄을 내려보내 연결하는 방식
### 방식 A: 기존 Bastion Host를 '경유'하는 방식 (가장 흔함)

지금 작성하신 아키텍처 다이어그램처럼, 이미 Public Subnet에 있는 Bastion(EC2)을 SSM 통로로 삼는 방식입니다.

- 과정: 내 로컬 ➡️ SSM Session Manager ➡️ Bastion(EC2) ➡️ RDS
- 장점: 별도의 인프라를 추가할 필요가 없고, 이미 보안 검증이 끝난 Bastion만 활용하면 됩니다.
### 방식 B: 'Application Server'를 직접 경유하는 방식 (최신 트렌드 SSM Agent 방식)

Bastion Host가 아예 없는 환경이라면, 그냥 Private Subnet에 있는 Spring Boot 서버(API1)를 경유지로 씁니다.

- 과정: 내 로컬 ➡️ SSM Session Manager ➡️ API1(Spring Boot) ➡️ RDS
- 장점: Bastion Host라는 관리 대상(비용, 보안 패치 등)을 아예 없앨 수 있습니다.
- 주의: 앱 서버의 SSM Agent가 RDS 포트로의 접근 권한(보안 그룹)을 가지고 있어야 합니다.
### 방식 C: SSM 포트 포워딩 + 직접 연결 (VPC Endpoint 활용)

사실 위 두 방식(A, B)은 모두 '특정 EC2 인스턴스'를 경유합니다. 하지만 아예 EC2가 필요 없는 방식도 있습니다. AWS PrivateLink를 이용하는 방식입니다.

- 과정: 내 로컬 ➡️ Client VPN 또는 VPC Endpoint ➡️ RDS
- 특징: SSM 터널링은 아니지만, 사실상 터널링과 같은 효과를 내며 가장 정석적인 기업용 접속 방식입니다. SSM Agent나 EC2 경유 없이 네트워크 레벨에서 RDS를 로컬망처럼 쓰는 것이죠.
### 요약

결론적으로 현재 아키텍처에서는:

1. 가장 쉬운 길: 아까 성공했던 것처럼 i-0a2520ffef7beab91(Bastion Host)를 경유지로 사용하는 방식 (방식 A).
1. 더 깔끔한 길: Bastion Host가 보안상 위험하거나 관리가 귀찮다면, 아예 Bastion을 없애고 Back Office나 API1 서버를 경유지로 지정해서 같은 명령어를 실행 (방식 B).


### SSH vs SSM

### SSO(Single Sign-On)

기존 방식 :  개발자마다 Access Key와 Secret Key를 발급받아 로컬 PC에 저장해두고 사용

SSO 방식 : 영구적인 키 발급 X, 브라우저를 통해 회사 계정으로 로그인 후 인증 완료 시 AWS 측에서 자동 만료되는 임시 토큰을 로컬 PC에 발급




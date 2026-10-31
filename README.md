# b3-1 - AWS 웹 서비스 인프라 구축

VPC로 격리된 네트워크를 구성하고, EC2에 Nginx를 배포해 외부에서 접속 가능한 웹 서비스를 구축한다. (리전: ap-northeast-2)

## 구성
VPC(10.0.0.0/16) · Public Subnet(10.0.1.0/24) · Internet Gateway · Route Table(0.0.0.0/0 → IGW) · Security Group(80: 전체, 22: 내 IP) · EC2(Nginx)

## 외부 접속 증빙
- 선택한 방식: **(A 또는 B 중 택 1 기재)**
- 접속 정보: `http://<퍼블릭IP>` (또는 `http://<퍼블릭IP>/health`)
- 결과 스크린샷:

<img width="1222" height="412" alt="외부 접속 증빙" src="https://github.com/user-attachments/assets/7b042ec1-fdae-4b4f-8eda-624cdb06b48b" />

## 문서
| 제출물 | 위치 |
|---|---|
| 아키텍처 다이어그램 | [docs/architecture.md](docs/architecture.md) (+ `docs/architecture.png`) |
| 트러블슈팅 보고서 | [docs/troubleshooting.md](docs/troubleshooting.md) |
| 리소스 정리 체크리스트 | [docs/cleanup-checklist.md](docs/cleanup-checklist.md) |

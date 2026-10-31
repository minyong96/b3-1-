# 트러블슈팅 보고서 (장애 시나리오 사전 분석)

> **안내:** 이 보고서는 실습 중 실제로 발생한 장애 기록이 아니라, 구축한 환경(VPC / Public Subnet / IGW / Route Table / Security Group / EC2+Nginx)에서 **발생할 수 있는 오류를 가정하고 로그·증상을 근거로 원인을 좁혀 가는 과정을 분석한 문서**이다. 실제 재현 및 스크린샷은 포함하지 않는다.

## 분석 대상 환경

| 구성 | 값 |
|---|---|
| 리전 | ap-northeast-2 |
| VPC / Subnet | 10.0.0.0/16 / 10.0.1.0/24 (Public) |
| Route Table | 0.0.0.0/0 → Internet Gateway |
| Security Group | Inbound 80: 0.0.0.0/0, 22: 내 IP |
| EC2 | Nginx (80) |

---

## 사례 1. 외부에서 http://<퍼블릭IP> 접속이 안 됨 (SG 문제 가정)

| 항목 | 내용 |
|---|---|
| 증상 | 브라우저에서 접속 시 계속 로딩되다가 `ERR_CONNECTION_TIMED_OUT`. `curl -m 5 http://<퍼블릭IP>`도 응답 없이 timeout. 단, SSH 접속과 인스턴스 내부의 `curl -I http://localhost`는 `200 OK` |
| 원인 가설 | ① SG 인바운드에 80 포트 규칙이 없거나 소스가 잘못됨 ② Route Table에 `0.0.0.0/0 → IGW` 없음 ③ 퍼블릭 IP 미할당 ④ Nginx 미실행 |
| 검증 방법 | - 내부 `curl localhost`가 200이므로 ④(Nginx)는 제외<br>- 콘솔에서 인스턴스에 Public IPv4가 있는지 확인 → 있으면 ③ 제외<br>- SSH는 되므로 IGW 경로와 22 규칙은 정상 → ②는 가능성 낮음<br>- SG 인바운드 규칙 확인 → 80 규칙이 없거나 소스가 `0.0.0.0/0`이 아니면 ① 확정 |
| 조치 내용 | SG 인바운드에 `HTTP / TCP 80 / 0.0.0.0/0` 규칙 추가 |
| 결과 | 브라우저에서 Nginx 기본 페이지 표시, `curl -I http://<퍼블릭IP>` → `200 OK` |
| 재발 방지 | 배포 후 체크리스트에 "내부 `curl localhost` + 외부 접속 둘 다 확인" 추가. SG는 "필요 포트 허용 여부"를 규칙 목록으로 점검 |

**핵심 판단 근거:** 내부에서는 되고 외부에서만 안 되면 웹 서버가 아니라 네트워크 경로(SG/라우팅/퍼블릭 IP) 문제이다. SG가 막으면 패킷이 버려져 응답이 없으므로 **timeout**으로 나타난다.

---

## 사례 2. 외부 접속 시 즉시 "연결 거부" (Nginx 중단 가정)

| 항목 | 내용 |
|---|---|
| 증상 | 브라우저에서 `ERR_CONNECTION_REFUSED`가 즉시 표시. `curl http://<퍼블릭IP>` → `Connection refused` (사례 1과 달리 timeout이 아님) |
| 원인 가설 | ① Nginx 프로세스가 중지됨 ② Nginx가 80이 아닌 다른 포트에서 listen ③ 인스턴스 OS 방화벽(ufw 등)이 차단 |
| 검증 방법 | - SSH 접속 후 `sudo systemctl status nginx`로 상태 확인<br>- `sudo ss -tlnp \| grep :80`로 80 포트 listen 여부 확인<br>- `sudo tail -n 50 /var/log/nginx/error.log`로 오류 확인<br>- `curl -I http://localhost`로 내부 응답 확인 |
| 조치 내용 | `sudo systemctl start nginx` 후 `sudo systemctl enable nginx`로 재부팅 시 자동 실행 설정 (설정 파일 오류가 원인이면 `sudo nginx -t`로 문법 검사 후 수정) |
| 결과 | `systemctl status nginx`가 `active (running)`, 외부 접속 시 `200 OK` |
| 재발 방지 | Nginx를 `enable`로 등록. 배포 직후 `systemctl is-active nginx`와 `/health` 응답을 점검 |

**핵심 판단 근거:** `refused`는 패킷이 인스턴스까지 도달했지만 해당 포트를 받는 프로세스가 없다는 뜻이므로 SG/라우팅은 정상이다. timeout과 refused를 구분하면 원인 계층을 빠르게 좁힐 수 있다.

---

## 두 사례의 증상 비교

| 구분 | 사례 1 (SG 80 미허용) | 사례 2 (Nginx 중단) |
|---|---|---|
| 외부 증상 | timeout (무응답) | connection refused (즉시 거부) |
| 인스턴스 내부 `curl localhost` | 200 OK | 실패 |
| 문제 계층 | 네트워크 경로 (SG) | 애플리케이션 (프로세스) |
| 우선 확인 | SG 인바운드 규칙 | `systemctl status nginx` |

## 일반 점검 순서 (외부 접속 불가 시)

1. 인스턴스 내부에서 `curl -I http://localhost` → 웹 서버 정상 여부
2. 인스턴스에 Public IPv4가 있는지
3. Route Table에 `0.0.0.0/0 → IGW`가 있고 서브넷에 연결돼 있는지
4. SG 인바운드에 80(및 SSH용 22) 규칙이 올바른 소스로 있는지
5. 증상이 timeout이면 네트워크 경로, refused면 프로세스/포트 문제로 구분

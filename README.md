# week8_practice · AWS EC2 배포 실습

5주차의 **캡스톤 기획 도우미**를 EC2 Ubuntu에서 실행합니다. 프로젝트 검색·소개글 분량 검사·기본 연결 확인 기능은 그대로이며, **Compose 설정은 완성되어 있습니다. 파일을 수정하거나 복사할 필요가 없습니다.**

## 1. 프로젝트 받기

EC2 Ubuntu에 PuTTY로 접속한 뒤 실행합니다.

```bash
cd ~
git clone https://github.com/Mok2Lee/week8_practice.git
cd week8_practice
```

본인 GitHub로 Fork했다면 clone 주소의 `Mok2Lee`를 본인 계정명으로 바꿉니다. 이미 받은 폴더가 있다면 해당 폴더로 이동합니다.

## 2. 바로 실행

```bash
sudo docker compose up -d --build
sudo docker compose ps
curl -i http://localhost:8080/api/health
```

**확인 결과:** `api`는 `healthy`, `nginx`는 실행 상태이며, 연결 확인 응답은 `200`과 `{"status":"ok"}`입니다. 처음 실행할 때 이미지와 Python 라이브러리를 내려받습니다. Ubuntu에 Python이나 Flask를 별도로 설치하지 않습니다.

Docker를 일반 사용자 권한으로 실행하도록 준비했다면 `sudo`를 생략할 수 있습니다. 이후 Docker 명령도 같은 방식으로 사용합니다.

## 3. 내 PC에서 웹 페이지 확인

1. **EC2 → 인스턴스 선택 → 보안 탭 → 보안 그룹 링크**로 이동합니다.
2. **인바운드 규칙 편집 → 규칙 추가**에서 사용자 지정 TCP, 포트 **8080**, 소스 **내 IP** 또는 승인된 강의실 공인 IP 범위를 입력하고 저장합니다.
3. **아웃바운드 규칙**에서 외부 다운로드에 필요한 **HTTPS 443·HTTP 80** 허용 여부를 확인합니다. 기본 모든 트래픽 허용 규칙이 있으면 이미 허용된 상태입니다.
4. EC2 콘솔에서 현재 퍼블릭 IPv4 주소를 확인하고, 내 PC 브라우저에서 `http://EC2퍼블릭IP:8080`으로 접속합니다.

**웹 접속은 인바운드 8080, 외부 다운로드는 아웃바운드 80·443**입니다. 기존 SSH 규칙은 유지합니다. 화면별 설정과 실패 시 확인 순서는 [보안 그룹 실습 안내](docs/security-group.md)를 참고합니다.

| 확인할 기능 | 입력 | 정상 결과 |
|---|---|---|
| 기본 연결 확인 | 화면 위 `/api/health` 링크 | `status: ok` |
| 프로젝트 검색 | 분야 웹 · 검색어 예약 | 교내 공간 예약 1개 |
| 소개글 분량 검사 | 짧은 글과 100~300자 글 | 글자 수·단어 수·권장 분량 충족 여부 |

퍼블릭 IP와 인터넷 게이트웨이로 향하는 라우팅 경로가 있는 EC2를 사용합니다. 컨테이너의 Flask 5000번 포트는 외부에 공개하지 않습니다.

## 4. 로그 확인과 종료

```bash
sudo docker compose logs --tail=30 api nginx
sudo docker compose down
```

실습을 마치면 Compose를 종료하고, 다음 시간까지 사용하지 않을 EC2를 콘솔에서 **중지(Stop)** 를 선택합니다.

## 구성 확인

```text
내 PC → EC2:8080 → nginx:80 → api:5000
```

| 파일 | 역할 |
|---|---|
| `compose.yaml` | 두 이미지 빌드·실행, 8080 포트 연결 |
| `api/app.py` | 5주차와 동일한 Flask 기능 |
| `api/Dockerfile` | Python 실행 환경과 Flask 설치 |
| `nginx/nginx.conf` | 화면 제공, `/api/` 요청을 `api:5000`으로 전달 |
| `nginx/html/` | 기존 HTML·JavaScript·CSS 화면 |

실습 이미지: `week8_01`, `week8_02` · 컨테이너: `week8_api01`, `week8_web01` · 네트워크: `week8_net01`

nginx는 API의 연결 확인이 완료된 뒤 시작합니다. `compose.starter.yaml`이나 TODO 항목은 사용하지 않습니다.

## 실행이 안 될 때

| 증상 | 확인할 내용 |
|---|---|
| `compose` 명령이 없음 | 아래 Ubuntu Compose 설치 안내 |
| Docker 서버 연결 실패 | `sudo systemctl status docker`, 중지 상태라면 `sudo systemctl start docker` |
| 8080 포트 사용 중 | 이전 실습 컨테이너가 실행 중인지 `sudo docker ps`로 확인 |
| 이름이 이미 사용 중 | 같은 week8 실습이 다른 폴더에서 실행 중인지 확인 |
| EC2 안에서는 되지만 PC에서는 안 됨 | 현재 퍼블릭 IP → 인터넷 경로 → 보안 그룹의 8080·허용 IP |
| API가 unhealthy이거나 nginx가 실행되지 않음 | `sudo docker compose logs --tail=50 api nginx` |

Ubuntu 패키지의 `docker.io`로 설치한 환경에서 Compose가 없을 때:

```bash
sudo apt update
sudo apt install -y docker-compose-v2
docker compose version
```

Docker 공식 저장소의 `docker-ce`를 사용하는 경우에는 Compose 패키지 이름이 `docker-compose-plugin`입니다.

## 다음 단계: ECS Fargate

Compose는 위의 `nginx.conf`를 사용합니다. 같은 Linux 태스크 안에 nginx와 API를 함께 배치하는 ECS 실습용 파일도 준비되어 있습니다.

- `nginx/nginx.ecs.conf`: 내부 API 주소가 `127.0.0.1:5000`
- `nginx/Dockerfile.ecs`: 위 설정을 포함해 nginx 이미지를 빌드

ECR 업로드부터 태스크 생성·접속·종료까지는 [ECS 배포 실습 안내](docs/ecs-deploy.md)를 따릅니다. 태스크 정의 JSON은 저장소 루트의 [task-definition.template.json](task-definition.template.json)을 사용합니다.

| AWS 실습 자원 | 이름 |
|---|---|
| ECR 저장소 | `week8-api`, `week8-web` |
| ECS 클러스터 | `week8-cluster` |
| 태스크 정의 패밀리 | `week8-ecs` |
| ECS 서비스 | `week8-service` |
| CloudWatch 로그 그룹 | `/ecs/week8` |

EC2 Compose를 실행할 때는 JSON이나 ECS용 파일을 수정할 필요가 없습니다. ECS 태스크 정의를 등록할 때만 JSON의 `ACCOUNT_ID` 전체와 승인된 실행 역할 ARN을 본인 수업 계정에 맞춥니다.

---

기존 수업의 캡스톤 기획 도우미 기능을 유지한 8주차 배포용 프로젝트입니다.

이 프로젝트는 수업용 Flask 개발 서버를 사용합니다.

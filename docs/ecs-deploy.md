# week8 · ECR 업로드와 ECS Fargate 배포

명령은 **EC2 Ubuntu의 같은 Bash 터미널**에서 실행합니다. AWS 콘솔과 외부 웹 확인은 본인 PC에서 진행합니다. EC2 Compose 실행은 [README](../README.md)를 따릅니다.

## 사용할 이름

| 구분 | 이름 |
|---|---|
| GitHub 저장소·작업 폴더 | `week8_practice` |
| Compose 이미지 | API `week8_01`, 웹 `week8_02` |
| Compose 컨테이너 | `week8_api01`, `week8_web01` |
| Compose 네트워크 | `week8_net01` |
| ECR 저장소·ECS용 로컬 이미지 | `week8-api`, `week8-web` |
| 이미지 태그 | `lesson-v1`, 선택 재배포 `lesson-v2` |
| ECS 클러스터 | `week8-cluster` |
| 태스크 정의 패밀리 | `week8-ecs` |
| ECS 서비스 | `week8-service` |
| CloudWatch 로그 그룹 | `/ecs/week8` |
| IAM 역할 | `week8-role` 하나를 EC2와 ECS에서 공통 사용 |
| 리전 | 본인 콘솔에서 사용하는 리전 확인 후 입력 |

Compose의 `week8_01/02`는 EC2 실행용 이미지입니다. ECS용 빌드에서는 위 ECR 저장소에 맞춰 `week8-api/web:lesson-v1`이라는 태그를 별도로 사용합니다.

## 1. 공통 역할 생성과 EC2 연결

PC의 **IAM → 역할 → 역할 생성**에서 진행합니다.

1. 신뢰할 수 있는 엔터티: **AWS 서비스**, 사용 사례: **EC2**.
2. 권한 정책에서 아래 두 정책을 각각 검색하고 체크합니다. 첫 정책을 선택한 뒤 검색어만 바꿉니다.
   - `AmazonEC2ContainerRegistryPowerUser`: ECR 로그인과 이미지 업로드·다운로드.
   - `AmazonECSTaskExecutionRolePolicy`: ECS의 이미지 다운로드와 로그 전송.
3. 역할 이름을 **`week8-role`**로 생성합니다.
4. 역할의 **신뢰 관계 → 신뢰 정책 편집**에서 [week8-role-trust.json](week8-role-trust.json) 전체를 붙여 넣고 저장합니다. EC2와 ECS 태스크가 같은 역할을 사용하도록 지정합니다.
5. **EC2 → 인스턴스 → 본인 인스턴스 → 작업 → 보안 → IAM 역할 수정**에서 `week8-role`을 선택하고 저장합니다.

역할은 하나입니다. EC2에서는 이미지 업로드에, ECS에서는 이미지 받기와 로그 전송에 사용합니다. 이 수업 앱은 AWS 서비스를 직접 호출하지 않으므로 **태스크 역할은 비워 두고, 태스크 실행 역할에 `week8-role`을 지정**합니다. 콘솔 로그인 사용자는 리소스 생성과 이 역할을 ECS에 전달할 권한(`iam:PassRole`)이 필요합니다.

### AWS CLI 설치·인증 확인

EC2 Ubuntu의 PuTTY 터미널에서 실행합니다. `aws --version`이 정상 출력되면 설치는 생략합니다. 아래 설치는 **x86_64 EC2** 기준입니다.

```bash
aws --version
uname -m
```

`aws: command not found`이면 설치합니다.

```bash
sudo apt update
sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp/week8-awscli
sudo /tmp/week8-awscli/aws/install
aws --version
aws sts get-caller-identity --no-cli-pager
```

`Arn`에 **`assumed-role/week8-role/`**이 표시되면 역할 연결 성공입니다. 액세스 키를 입력하는 `aws configure`는 필요하지 않습니다. SSH 접속 키는 서버 접속용이고, IAM 역할은 AWS 서비스 사용 권한입니다.

## 2. ECR 저장소 생성과 로그인

PC 콘솔의 현재 리전을 확인합니다. 예: 서울 `ap-northeast-2`, 시드니 `ap-southeast-2`. **ECR·ECS·CloudWatch를 같은 리전에서 만듭니다.**

1. **ECR → 프라이빗 레지스트리 → 리포지토리 → 리포지토리 생성**.
2. `week8-api`를 생성합니다. 태그 변경 가능, AES-256 기본값을 사용합니다.
3. 같은 방법으로 `week8-web`을 생성합니다. 이미 있으면 재생성하지 않습니다.

EC2의 같은 Bash 터미널에서 아래 `REGION` 값을 **본인 콘솔의 리전 코드**로 바꿔 실행합니다. 계정 ID는 자동으로 읽습니다.

```bash
REGION=ap-northeast-2
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"
aws ecr get-login-password --region "$REGION" |
  sudo docker login --username AWS --password-stdin "$REGISTRY"
```

**`Login Succeeded`**를 확인합니다. 새 터미널을 열었다면 위 세 변수부터 다시 설정합니다. ECR 저장소 생성은 앞 단계의 콘솔에서 완료합니다.

## 3. ECS용 이미지 빌드

ECS용 nginx 파일은 이미 저장소에 있습니다. 복사하거나 Vim으로 수정할 필요가 없습니다.

```bash
cd ~/week8_practice
sudo docker build --platform linux/amd64 -t week8-api:lesson-v1 ./api
sudo docker build --platform linux/amd64 -f nginx/Dockerfile.ecs -t week8-web:lesson-v1 ./nginx
sudo docker image inspect week8-api:lesson-v1 --format '{{.Architecture}}'
sudo docker image inspect week8-web:lesson-v1 --format '{{.Architecture}}'
```

두 이미지 모두 `amd64`인지 확인합니다. Compose의 nginx 설정은 `api:5000`, 같은 Linux 태스크에 넣을 ECS 설정은 `127.0.0.1:5000`입니다. ECS용 빌드에서 `-f Dockerfile.ecs`를 빠뜨리지 않습니다.

## 4. ECR 업로드

```bash
sudo docker tag week8-api:lesson-v1 "$REGISTRY/week8-api:lesson-v1"
sudo docker push "$REGISTRY/week8-api:lesson-v1"
sudo docker tag week8-web:lesson-v1 "$REGISTRY/week8-web:lesson-v1"
sudo docker push "$REGISTRY/week8-web:lesson-v1"

echo "$REGISTRY/week8-api:lesson-v1"
echo "$REGISTRY/week8-web:lesson-v1"
```

PC의 ECR 콘솔에서 **두 저장소의 `lesson-v1` 이미지**를 확인합니다. 태스크 정의에는 계정·리전·태그가 포함된 전체 Image URI를 사용합니다.

## 5. 로그 그룹과 클러스터 준비

PC AWS 콘솔에서 앞서 ECR을 만든 **동일 리전**을 사용합니다.

1. **CloudWatch → 로그 → 로그 그룹 → 로그 그룹 생성**으로 이동합니다.
2. 이름을 **`/ecs/week8`**로 지정합니다. 이미 있으면 그 그룹을 사용합니다.
3. 보존 기간을 수업 기준인 **1일**로 지정합니다.
4. **ECS → 클러스터 → 클러스터 생성**으로 이동합니다.
5. 이름 **`week8-cluster`**, 실행 용량 **Fargate**로 생성합니다.

`AmazonECSTaskExecutionRolePolicy`에는 로그 그룹 생성 권한이 없으므로, 그룹은 태스크 실행 전에 준비합니다.

## 6. JSON으로 태스크 정의 등록

PC에서 [task-definition.template.json](../task-definition.template.json)을 받아 편집합니다. **EC2에서 Compose를 실행할 때는 이 파일을 수정하지 않습니다.**

1. `ACCOUNT_ID` **세 곳 전체**를 본인 AWS 계정 ID로 변경합니다.
2. `AWS_REGION` **네 곳 전체**를 본인 리전 코드로 변경합니다. 실행 역할은 앞서 만든 **`week8-role`**입니다.
3. 두 `image` 값이 ECR의 `week8-api:lesson-v1`, `week8-web:lesson-v1` 전체 주소와 같은지 확인합니다.
4. ECS의 **태스크 정의 생성 → JSON 방식**에서 전체 내용을 붙여 넣고 등록합니다.
5. 패밀리 **`week8-ecs`**와 등록된 revision 번호를 확인합니다.

포함된 설정: Linux/X86_64, Fargate, 0.5 vCPU·1 GB, api 5000·nginx 80, `/ecs/week8` 로그, api 내부 상태 검사. Task role은 사용하지 않습니다. 같은 설정을 다시 입력하거나 healthCheck를 중복 추가하지 않습니다.

## 7. Service 생성

태스크 정의 **`week8-ecs` → 방금 만든 개정 → 배포 → 서비스 생성**으로 이동합니다.

| 항목 | 설정 |
|---|---|
| 실행 방식 | Fargate |
| 태스크 정의 | `week8-ecs`의 방금 등록한 revision |
| 서비스 이름 | `week8-service` |
| 원하는 태스크 수 | 1 |
| 네트워크 | 본인 EC2와 같은 VPC의 퍼블릭 서브넷 |
| 퍼블릭 IP 자동 할당 | 켜기 |
| 보안 그룹 인바운드 | TCP 80, 본인/강의실의 승인된 공인 IP 범위 |
| 보안 그룹 아웃바운드 | HTTPS TCP 443, 대상 `0.0.0.0/0` 허용 여부 확인 |
| 로드 밸런서·자동 확장 | 이번 실습에서는 추가하지 않음 |

기본 아웃바운드 **모든 트래픽 허용** 규칙이 있으면 HTTPS도 이미 허용되므로 같은 규칙을 추가할 필요가 없습니다. 제한된 그룹이면 HTTPS 443 규칙을 추가합니다. 이는 ECR 이미지 받기·CloudWatch 로그 전송에 사용됩니다. HTTP 패키지 저장소도 사용할 EC2의 아웃바운드 설명은 [보안 그룹 안내](security-group.md)를 참고합니다.

**새 보안 그룹 생성**을 선택하고 이름을 `week8-ecs-sg`로 지정합니다. 인바운드는 **HTTP(80)**, 소스는 **사용자 지정**, 값은 **본인 공인 IP/32**입니다. 공인 IP는 본인 PC에서 https://checkip.amazonaws.com 으로 확인합니다. 학내 공인 IP가 바뀌면 규칙도 수정합니다.

서브넷 라우팅 테이블에 `0.0.0.0/0 → 인터넷 게이트웨이` 경로가 있어야 합니다. EC2와 Fargate 태스크의 보안 그룹은 서로 다를 수 있으므로 실제 연결된 그룹을 확인합니다.

## 8. 현재 Task의 웹·로그 확인

1. Service의 **desired=1·running=1**, api·nginx 실행 상태를 확인합니다.
2. 현재 태스크의 **네트워킹 → 퍼블릭 IP**를 확인합니다.
3. PC 브라우저에서 `http://현재Task퍼블릭IP/`로 접속합니다. **`:8080`은 붙이지 않습니다.**
4. health·프로젝트 검색·소개글 분량 검사를 확인합니다.
5. 태스크의 로그에서 api·nginx를 선택하거나 CloudWatch의 **`/ecs/week8`**을 엽니다.

PC PowerShell에서 `TASK_IP`를 실제 태스크 IP로 바꿉니다.

```powershell
curl.exe -i http://TASK_IP/api/health
```

상태 코드 200·`status: ok`와 요청 로그를 확인합니다. RUNNING 표시만으로 웹 접속 성공을 판단하지 않습니다.

## 9. 종료

1. **`week8-service`의 원하는 태스크 수를 0**으로 변경합니다.
2. 실행 태스크가 0개인지 확인합니다. 태스크만 중지하면 Service가 다시 실행할 수 있습니다.
3. EC2에서 Compose를 종료합니다.

```bash
cd ~/week8_practice
sudo docker compose down
```

4. 사용하지 않을 EC2를 중지합니다. 수업용 이미지·로그·디스크는 교수자의 보관 기준에 따릅니다.

## 선택 시연·복습: 웹 이미지 새 버전 배포

EC2의 `nginx/html/index.html`을 수정한 뒤 실행합니다. 같은 터미널의 `REGISTRY` 변수를 사용합니다.

```bash
cd ~/week8_practice
sudo docker build --platform linux/amd64 -f nginx/Dockerfile.ecs -t week8-web:lesson-v2 ./nginx
sudo docker tag week8-web:lesson-v2 "$REGISTRY/week8-web:lesson-v2"
sudo docker push "$REGISTRY/week8-web:lesson-v2"
```

`week8-ecs`의 새 revision에서 nginx 이미지 태그만 `lesson-v2`로 변경하고 api는 `lesson-v1`을 유지합니다. `week8-service`에 새 revision을 적용하고 **새 태스크의 IP**에서 확인합니다. 시연 후 원하는 태스크 수를 다시 0으로 변경합니다.

## 참고

- [AWS 보안 그룹의 인바운드·아웃바운드와 응답 허용](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Fargate 네트워크와 같은 태스크의 localhost 통신](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)
- [태스크 실행 역할](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)

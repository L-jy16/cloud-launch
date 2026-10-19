<!-- @format -->

# Cloud Launch — PULSE 챗봇 AWS 배포 가이드

이 문서는 기존 `ai_chatbot` FastAPI 앱을 서울 리전 EC2에 배포하고, 클라우드 실습 과제의 증빙을 준비하는 절차입니다. `cloud-launch`는 문서 폴더이며 배포 대상은 저장소 전체입니다. 아래 명령은 Ubuntu 24.04 LTS, 서버 사용자 `ubuntu`, 저장소 경로 `/home/ubuntu/ai_chatbot`을 기준으로 합니다.

> 현재 상태: 배포 안내 작성 완료. AWS 리소스 생성·실제 배포·외부 접속 검증·리소스 삭제는 아직 수행하지 않았습니다. 아래 기록란은 실제 확인 후 작성하세요.

## 1. 배포 구조

```text
외부 브라우저 / curl
  → EC2 퍼블릭 IPv4:80
  → Internet Gateway / VPC 라우팅 / Security Group
  → Public Subnet의 EC2
      Nginx :80 → Uvicorn 127.0.0.1:8000 → FastAPI / SQLite
                                              → 외부 AI·네이버 API
```

논리 구성 예시이며 제출용 다이어그램은 별도 PNG 또는 PDF로 작성합니다. VPC 안에 Subnet, 그 안에 EC2를 표시하고 SG와 IGW, 요청 방향도 표시하세요.

- 공개 포트: HTTP 80, 내 공인 IP에서만 SSH 22
- 앱 포트 8000: EC2 내부 루프백에서만 수신
- 외부 접속 검증: **B 방식**, `GET http://<퍼블릭IP>/health`
- 정상 응답: HTTP 200, `{"status":"ok"}`
- Docker·HTTPS는 선택 과제입니다. HTTP 단계에서는 실습용 계정과 데이터만 사용하고, 실제 사용자 운영 전에는 HTTPS를 적용하세요.

## 2. AWS 계정·권한 준비

1. 실습용 IAM 사용자/Role로 로그인합니다. 루트 계정을 실습에 사용하지 않습니다.
2. 리전을 **서울(ap-northeast-2)**로 고정합니다.
3. 계정의 무료 사용 조건, 크레딧 잔액과 만료일을 확인합니다. micro 인스턴스라고 반드시 무료인 것은 아닙니다.
4. IAM 권한은 EC2·VPC·Subnet·Route Table·IGW·Security Group·키페어·볼륨의 실습에 필요한 조회/생성/연결/태그/수정/삭제 작업으로 제한합니다. `AdministratorAccess`나 무관한 S3/RDS 권한을 부여하지 않습니다. 실제 정책의 Action·Resource·Condition과 적용 근거를 기록하세요. 기존 관리자에게 계정과 실습 권한을 준비받을 수 있습니다.
5. 리소스에 `Project=cloud-launch` 태그를 붙이고 ID를 기록합니다. IAM은 AWS 작업 권한, SG는 인스턴스 네트워크 접근을 제어합니다.

## 3. 네트워크 생성 — AWS 콘솔

VPC 콘솔에서 아래 순서로 직접 생성합니다. 이 구성에는 NAT Gateway, ALB, RDS가 필요하지 않습니다.

| 순서 | 리소스           | 예시 설정                                                               |
| ---- | ---------------- | ----------------------------------------------------------------------- |
| 1    | VPC              | 이름 `cloud-launch-vpc`, IPv4 CIDR `10.0.0.0/16`, DNS resolution 활성화 |
| 2    | Subnet           | 위 VPC, 서울 AZ 1개, 이름 `cloud-launch-public`, CIDR `10.0.1.0/24`     |
| 3    | Internet Gateway | `cloud-launch-igw` 생성 후 위 VPC에 연결                                |
| 4    | Route Table      | 위 VPC에 `cloud-launch-public-rt` 생성                                  |
| 5    | 경로             | 기존 local 경로 유지, `0.0.0.0/0`의 대상에 위 IGW 추가                  |
| 6    | 서브넷 연결      | Route Table의 명시적 서브넷 연결에 위 Public Subnet 지정                |

기본 NACL은 그대로 사용합니다. 직접 제한했다면 응답 트래픽까지 고려해야 합니다.

## 4. 보안 그룹과 EC2 생성

EC2 콘솔에서 다음 설정으로 인스턴스를 시작합니다.

- 이름: `cloud-launch-web`
- AMI: Ubuntu Server 24.04 LTS, x86_64
- 유형: 계정에서 무료 혜택 적용을 확인한 micro 인스턴스 1대
- 스토리지: EBS 8~10GiB, 루트 볼륨의 종료 시 삭제 여부 확인
- 키페어: 생성 후 `.pem` 파일 안전하게 보관, Git에 추가하지 않기
- 네트워크: 직접 만든 VPC와 Public Subnet
- 퍼블릭 IPv4 자동 할당: 활성화
- Security Group: 해당 VPC에 `cloud-launch-web-sg` 생성

인바운드 규칙:

| 유형 | 포트   | 소스                                   |
| ---- | ------ | -------------------------------------- |
| HTTP | TCP 80 | `0.0.0.0/0`                            |
| SSH  | TCP 22 | 콘솔의 내 IP 선택: `<내 공인 IPv4>/32` |

8000 포트와 전체 포트 개방 규칙은 추가하지 않습니다. 기본 아웃바운드 허용을 유지해 패키지 설치와 외부 API 통신을 가능하게 합니다. 인스턴스 상태 검사를 통과하면 퍼블릭 IPv4를 기록합니다.

## 5. SSH 접속 — 내 컴퓨터

`<...>` 부분은 실제 값으로 바꾸고 실행합니다.

```bash
chmod 400 "/키파일/경로/cloud-launch.pem"
ssh -i "/키파일/경로/cloud-launch.pem" ubuntu@<퍼블릭IP>
```

접속이 안 되면 퍼블릭 IP, SSH 사용자 `ubuntu`, 키페어, SG의 내 공인 IP, 서브넷의 Route Table 연결을 확인합니다. 내 네트워크가 바뀌면 SG의 SSH 소스도 갱신합니다.

## 6. 앱 설치 — EC2 터미널

```bash
sudo apt update
sudo apt install -y git python3 python3-venv nginx curl
cd /home/ubuntu
git clone https://github.com/L-jy16/ai_chatbot.git
cd /home/ubuntu/ai_chatbot
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m pip check
```

저장소가 비공개라면 본인에게 허용된 Git 인증 방식을 사용하세요. 인증 토큰을 URL이나 README에 적지 않습니다. 이미 clone한 폴더가 있다면 다시 clone하지 말고 기존 폴더에서 작업합니다.

환경 변수는 서버에서 별도로 준비합니다. 로컬 `.env`나 개인 DB를 Git에 올리지 않습니다.

```bash
cd /home/ubuntu/ai_chatbot
# .env가 없을 때만 예제를 복사
cp -n .env.example .env
chmod 600 .env
.venv/bin/python -c 'import secrets; print(secrets.token_hex(32))'
nano .env
```

생성한 난수를 `SECRET_KEY`에 넣습니다. 실제 AI 응답을 쓰려면 `LLM_API_KEY`, 뉴스·검색 추이를 쓰려면 `NAVER_CLIENT_ID`와 `NAVER_CLIENT_SECRET`도 입력합니다. 나머지 기본값은 `.env.example`을 참고합니다. 키가 없는 상태의 health 성공은 AI 연결 성공을 뜻하지 않습니다.

## 7. FastAPI 상시 실행 — systemd

기존 `deploy/ai-chatbot.service`는 8000 포트를 모든 인터페이스에 바인딩합니다. 이번 구성에서는 아래처럼 **127.0.0.1**로 제한한 서비스 파일을 사용합니다.

```bash
sudo tee /etc/systemd/system/ai-chatbot.service > /dev/null <<'SERVICE'
[Unit]
Description=PULSE FastAPI chatbot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/ai_chatbot
ExecStart=/home/ubuntu/ai_chatbot/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 1
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
SERVICE
sudo systemctl daemon-reload
sudo systemctl enable --now ai-chatbot
sudo systemctl status ai-chatbot --no-pager
curl -i http://127.0.0.1:8000/health
```

`200 OK`와 `{"status":"ok"}`를 확인합니다. 앱은 작업 폴더의 `.env`를 읽고, 기본 설정에서 같은 폴더에 SQLite DB를 만듭니다. SQLite 사용을 고려해 worker는 1개로 둡니다.

## 8. Nginx로 HTTP 80 연결

```bash
sudo tee /etc/nginx/sites-available/cloud-launch > /dev/null <<'NGINX'
server {
    listen 80 default_server;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
NGINX
sudo ln -sfn /etc/nginx/sites-available/cloud-launch /etc/nginx/sites-enabled/cloud-launch
# 이 실습용 신규 서버의 기본 사이트만 비활성화
sudo unlink /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl reload nginx
curl -i http://localhost/health
curl -I http://localhost
curl -I https://example.com
```

`default` 링크가 이미 없다면 unlink 단계는 생략합니다. `nginx -t`가 성공한 경우에만 reload합니다. `/health`는 200이 나와야 합니다. 비로그인 상태의 `/`는 로그인 화면으로 리다이렉트될 수 있습니다. 과제의 localhost 200 검증은 `curl -I http://localhost/login`으로도 별도 기록하세요.

UFW를 이미 활성화했다면 먼저 현재 SSH 접속을 유지할 수 있도록 허용 규칙을 추가하고 HTTP를 허용합니다.

```bash
sudo ufw status
# active일 때만, 실제 내 IP로 바꿔 실행
sudo ufw allow from <내공인IP> to any port 22 proto tcp
sudo ufw allow 80/tcp
```

## 9. 외부 접속 검증 — 내 컴퓨터

EC2 내부가 아닌 **내 컴퓨터 터미널**에서 실행합니다.

```bash
curl -i http://<퍼블릭IP>/health
```

예상 결과:

```text
HTTP/1.1 200 OK
...
{"status":"ok"}
```

명령의 URL과 응답이 함께 보이는 화면을 캡처합니다. 브라우저에서 `http://<퍼블릭IP>/login`도 확인하고 실습용 계정으로 회원가입·로그인·채팅을 점검합니다. `https://` 자동 전환 시 이 단계에서는 주소를 `http://`로 입력하세요.

| 제출 기록              | 실제 확인 후 입력                       |
| ---------------------- | --------------------------------------- |
| 검증 방식              | B: GET /health                          |
| 접속 URL               | 미검증 — 배포 후 입력                   |
| 검증 시각(KST)         | 미검증                                  |
| HTTP 코드/본문         | 미검증                                  |
| 증빙 이미지            | 배포 후 `docs/external-access.png` 추가 |
| IAM 정책/최소권한 근거 | 적용 후 기록                            |

## 10. 문제 해결과 로그

```bash
sudo journalctl -u ai-chatbot -n 100 --no-pager
sudo systemctl status nginx --no-pager
sudo nginx -t
sudo tail -n 50 /var/log/nginx/error.log
sudo ss -lntp
```

| 증상                     | 검증 순서                                                   |
| ------------------------ | ----------------------------------------------------------- |
| SSH 시간 초과            | 공인 IP → SG 22의 내 IP → IGW 경로 → 서브넷 연결            |
| 외부 HTTP 시간 초과      | SG 80 → 공인 IP → Route Table → UFW → Nginx 상태            |
| 502 Bad Gateway          | 내부 `127.0.0.1:8000/health` → 앱 서비스 로그 → .env·의존성 |
| Nginx 기본 페이지만 표시 | 기본 사이트 링크 → cloud-launch 링크 → 설정 검사·reload     |
| health 성공, AI 실패     | LLM 키·게이트웨이 접근·앱 로그, 네이버 인증은 별도 점검     |

`docs/troubleshooting.md`에 실제 1건 이상을 다음 형식으로 기록합니다. 예시를 실제 해결 이력처럼 제출하지 마세요.

```text
증상:
원인 가설:
검증 명령/로그와 관찰 결과:
조치:
조치 후 결과:
재발 방지:
```

## 11. 업데이트

변경 전 필요한 DB와 설정은 접근 제한된 위치에 백업합니다. `.env`와 `app.db`는 업데이트 중 덮어쓰지 않습니다.

```bash
cd /home/ubuntu/ai_chatbot
git pull --ff-only
.venv/bin/python -m pip install -r requirements.txt
sudo systemctl restart ai-chatbot
curl -i http://localhost/health
```

## 12. 제출 파일 구성

`cloud-launch` 폴더를 제출 기준 폴더로 사용합니다. 아래 docs 파일은 실제 실습 후 작성해야 하며 이 README만으로 제출이 완료되지는 않습니다.

```text
cloud-launch/
├── README.md
└── docs/
    ├── architecture.png          # 또는 architecture.pdf
    ├── external-access.png       # 외부 curl의 URL·200·본문 증빙
    ├── troubleshooting.md        # 실제 문제 해결 1건 이상
    └── cleanup-checklist.md      # 종료/삭제 결과와 근거
```

네트워크·SG·IAM 설정과 내부 curl 결과도 캡처하면 구성과 최소권한 설명의 근거로 사용할 수 있습니다. 키·토큰·쿠키가 포함된 화면은 제출하지 않습니다.

## 13. 실습 종료 후 정리

증빙 확보 후 **이 실습에서 만든 리소스만** 정리하고 `docs/cleanup-checklist.md`에 리소스 ID, 완료 시각, 확인 근거를 남깁니다. 서비스를 정리하면 기록한 URL은 더 이상 접속되지 않으므로 README에 종료 시각도 기록하세요.

- [ ] EC2: Stop이 아닌 **Terminate**, 최종 상태 확인
- [ ] EBS: 종료 시 삭제 여부 확인, 남은 실습 볼륨 삭제
- [ ] Elastic IP: 할당한 경우 Release, 없으면 미생성 기록
- [ ] 실습 스냅샷: 생성했다면 필요 여부 확인 후 정리
- [ ] NAT Gateway/ALB/RDS: 생성했다면 삭제, 없으면 미생성 기록
- [ ] 네트워크 인터페이스 등 VPC 삭제를 막는 의존성 확인
- [ ] Internet Gateway: VPC에서 Detach 후 삭제
- [ ] 실습 Subnet·사용자 지정 Route Table·SG 정리 후 VPC 삭제
- [ ] 실습 전용 키페어와 불필요한 IAM 자격증명 정리
- [ ] Billing/사용량 확인 — 반영 지연이 있을 수 있어 잔존 리소스 목록도 함께 확인

공유 IAM·기존 VPC 등 다른 실습이나 사용자가 쓰는 리소스는 정리 대상에서 제외합니다.

## 공식 참고 자료

- [AWS: Internet Gateway와 퍼블릭 서브넷의 인터넷 연결](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [AWS: Free Tier 계정별 체계 변경 안내](https://aws.amazon.com/blogs/aws/aws-free-tier-update-new-customers-can-get-started-and-explore-aws-with-up-to-200-in-credits/)
- [Nginx: proxy_pass 및 proxy_read_timeout](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

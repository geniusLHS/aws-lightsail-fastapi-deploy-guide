# AWS Lightsail을 이용한 FastAPI 배포 로드맵

## 0. 작업 일정

| 날짜  | 해당 단계 | 작업 내용                      | 예상 소요 시간 |
| ----- | --------- | ------------------------------ | -------------- |
| 1일차 | 1, 2      | AWS Lightsail 생성 후 SSH 연결 | 3시간          |
| 2일차 | 3         | 프로젝트 환경 설정             | 2시간          |
| 3일차 | 4, 5      | FastAPI 코드 백그라운드 실행   | 2시간          |
| 4일차 | 6         | Nginx 설정 및 실행             | 1시간          |
| 5일차 | 7, 8      | HTTPS 적용 후 최종 테스트      | 3시간          |

## 1. AWS Lightsail 인스턴스 생성

- https://lightsail.aws.amazon.com/ 접속 후 인스턴스 생성
- Ubuntu 22.04 LTS 선택
  > 최신 버전인 24.04 LTS는 아직 불안정
- 가장 저렴한 크기 선택 (월별 $5)

## 2. Lightsail 인스턴스에 SSH 접속

- AWS에서 `.pem` 형식의 SSH 키 다운로드
- VSCode에서 `Remote-SSH` 확장 프로그램 설치
- Remote-SSH 설정 파일에 해당 서버 정보 추가

  - 내 경우 설정 파일은 `C:/Users/<user>/.ssh/config` 경로에, SSH 키 파일은 `C:/Users/<user>/.ssh/lightsail-key.pem` 경로에 위치
  - 설정 파일에 아래와 같이 추가
    ```
    Host snuclsai-lightsail
        HostName <퍼블릭 IPv4 주소>
        User ubuntu
        IdentityFile <다운로드한 .pem 키 경로>
    ```

- VSCode 왼쪽 하단 "Open a Remote Window" 버튼 눌러 연결

  - 대부분의 오류는 인터넷 검색으로 해결 가능
  - 그래도 안된다면 노트북 껐다가 다시 켜볼 것
  - 그래도 안된다면 lightsail 인스턴스 재부팅 해볼 것 (많은 경우 이거면 해결됨)

- 필수로 알아야 하는 linux 명령어
  > ```bash
  > ls         # 현재 디렉토리의 파일 목록 보기
  > pwd        # 현재 디렉토리의 경로 출력
  > cd DIR     # 다른 디렉토리로 이동 (예: cd /home, cd /..)
  > mkdir DIR  # 새로운 디렉토리 생성 (예: mkdir new_folder)
  > rm FILE    # 파일 삭제 (예: rm test.txt)
  > rmdir DIR  # 빈 디렉토리 삭제
  > rm -r DIR  # 비어있지 않은 디렉토리 삭제
  > ```

## 3. 프로젝트 환경 설정

### (1) 필수 패키지 업데이트 및 설치

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3
sudo apt install -y python3-pip
sudo apt install -y python3-venv
```

> 서버 성능이 충분하지 않아 한번에 설치할 경우 컴퓨터가 멈추는 일이 있음.
> 인내심을 가지고 하나씩 천천히 설치 해보자.

### (2) 프로젝트 디렉토리 생성 및 환경 설정

- 프로젝트 디렉토리 생성 및 가상 환경 설정

  ```bash
  mkdir ~/snuclsai && cd ~/snuclsai
  python3 -m venv .venv
  source .venv/bin/activate
  ```

  > - 이 때 터미널에서 `(venv)` 라는 표시가 나타나면 가상 환경에 접속된 상태이다.
  > - VSCode에서 다른 디렉토리를 갔다가 다시 돌아오거나 하면 `(venv)` 표시가 사라질 수 있는데, 이 때는 다시 `source .venv/bin/activate` 명령어를 실행해야 한다.

- 필수 패키지 설치
  ```bash
  pip install fastapi uvicorn
  pip install pydantic
  ```

## 4. FastAPI 코드 작성

- 우선 간단한 예시 코드를 사용하자. 다음의 코드를 `/home/ubuntu/snuclsai/main.py` 경로에 생성한다.

  ```python
  from fastapi import FastAPI
  from pydantic import BaseModel

  class Item(BaseModel):
      message: str

  app = FastAPI()

  @app.get("/")
  def read_root():
      return {"message": "Hello, FastAPI!"}

  @app.post("/test")
  def send_back(item: Item):
      return {"yousaid": item.message}
  ```

- 실제로는 아래와 같이 원격 저장소에 연결하여 다운로드 해야 한다. 이 작업은 백엔드 코드가 완성된 후 진행할 것임.

  1. 지금까지 작업한 코드를 github에 업로드

     - 로컬에 있는 디렉토리를 github와 연결하고, `git push` 명령어로 업로드
     - `GIthub Desktop` 프로그램 사용 권장

  2. lightsail 인스턴스에서 코드 다운로드

     ```bash
     git clone https://github.com/snuclsgpt/snu-cls-ai-back.git .  # 현재 디렉토리에 다운로드
     # git pull origin main                                        # 코드 업데이트 시 사용
     ```

     > 참고로 우리가 만든 저장소는 private으로 설정되어 있기 때문에 로그인 등의 인증 절차가 필요하다.

## 5. FastAPI 백그라운드 실행 (Systemd)

### (1) `snuclsai.service` 생성

```bash
sudo nano /etc/systemd/system/snuclsai.service
```

```ini
[Unit]
Description=SNUCLSAI Application
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/snuclsai
ExecStart=/home/ubuntu/snuclsai/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

> 작성 후 Ctrl+O (저장) → Enter (파일명 확인 및 저장) → Ctrl+X (종료)

### (2) 백그라운드에서 서비스 시작

```bash
sudo systemctl daemon-reload
sudo systemctl enable snuclsai
sudo systemctl start snuclsai
sudo systemctl status snuclsai
```

- 여기까지 왔으면 서버 컴퓨터 내부에서의 요청은 처리할 수 있는 상태가 되었다.

  다음 코드를 실행해서 잘 작동하는지 테스트 해보자. (**서버 컴퓨터** 터미널에서 실행)

  ```bash
  # GET 요청
  curl http://localhost:8000 -X GET
  # 올바른 응답: {"message": "Hello, FastAPI!"}

  # POST 요청
  curl http://localhost:8000/test -X POST -H "Content-Type: application/json" -d '{"message": "YAHO~~"}'
  # 출바른 응답: {"yousaid": "YAHO~~"}
  ```

## 6. Nginx 설치 및 설정

- 이제 서버 컴퓨터 외부에서의 요청도 처리할 수 있도록 해보자.

### (1) Nginx 설치

```bash
sudo apt install nginx
```

### (2) Nginx 설정 파일 생성

```bash
sudo nano /etc/nginx/sites-available/snuclsai
```

```nginx
server {
    listen 80;
    server_name <퍼블릭 IPv4 주소>;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### (3) 설정 적용

```bash
sudo ln -s /etc/nginx/sites-available/snuclsai /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

- 이제 서버 컴퓨터 외부에서도 요청을 보낼 수 있다.

  다음 코드를 실행해서 잘 작동하는지 테스트 해보자. (**로컬 컴퓨터** 터미널에서 실행)

  - linux

    ```bash
    # GET 요청
    curl http://<퍼블릭 IPv4 주소> -X GET
    # 올바른 응답: {"message": "Hello, FastAPI!"}

    # POST 요청
    curl http://<퍼블릭 IPv4 주소>/test -X POST -H "Content-Type: application/json" -d '{"message": "YAHO~~"}'
    # 올바른 응답: {"yousaid": "YAHO~~"}
    ```

  - windows (powershell)

    ```powershell
    # GET 요청
    curl http://<퍼블릭 IPv4 주소>
    # 올바른 응답: {"message": "Hello, FastAPI!"}

    # POST 요청
    Invoke-WebRequest -Uri "http://<퍼블릭 IPv4 주소>/test" `
      -Method Post `
      -Headers @{"Content-Type"="application/json"} `
      -Body '{"message": "YAHO~~"}'
    # 올바른 응답: {"yousaid": "YAHO~~"}
    ```

## 7. HTTPS 적용

- 지금은 http 프로토콜만 사용 가능하다. 인증서를 적용하여 https 프로토콜을 사용할 수 있도록 바꾸자.

  > https의 s는 secure의 약자이다. 만약 http 프로토콜을 사용하면 데이터가 암호화되지 않은 상태로 전송되므로 해커가 쉽게 가로챌 수 있음.

- 만약 도메인을 가지고 있다면 cerbot을 사용하여 매우 쉽게 인증서를 발급할 수 있다.

  우리는 도메인이 없으므로 직접 인증서를 발급받아 적용시켜보자.

### (1) 인증서 발급

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -newkey rsa:4096 -keyout /etc/nginx/ssl/key.pem -out /etc/nginx/ssl/cert.pem -days 365 -nodes
```

- 위 명령어를 실행시 나오는 질문은 눈치껏 작성하면 된다.

  - Country Name (2 letter code): `KR`
  - State or Province Name: `Seoul`
  - Locality Name: `Seoul`
  - Organization Name: `SNUCLSGPTTF`
  - Organizational Unit Name: `BE`
  - Common Name (e.g. server FQDN or YOUR name): `<서버 퍼블릭 IP>`
  - Email Address: `<이메일 주소>`

> `-days 365` 옵션은 인증서 만료 기간을 365일로 설정한다. 이 기간이 지나면 인증서를 재발급 받아야 한다. 아래에서 자동 갱신 코드를 작성할 것이다.

### (2) Nginx 설정 파일 수정

```bash
sudo nano /etc/nginx/sites-available/snuclsai
```

```nginx
server {
    listen 80;
    server_name <퍼블릭 IPv4 주소>;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name <퍼블릭 IPv4 주소>;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### (3) Nginx 설정 적용

```bash
sudo nginx -t
sudo systemctl restart nginx
```

### (4) 방화벽 설정

1. [lightsail.aws.amazon.com](https://lightsail.aws.amazon.com)에서 인스턴스 페이지 접속
2. 네트워킹 탭에서 IPv4 방화벽에 HTTPS 규칙 추가

### (5) 인증서 자동 갱신 설정

1. 인증서 자동 갱신 스크립트 생성

   ```bash
   sudo nano /etc/cron.monthly/renew_ssl.sh
   ```

   ```sh
   #!/bin/bash
   openssl req -x509 -newkey rsa:4096 -keyout /etc/nginx/ssl/key.pem -out /etc/nginx/ssl/cert.pem -days 365 -nodes -subj "/C=KR/ST=Seoul/L=Seoul/O=SNUCLSGPTTF/OU=BE/CN=<퍼블릭 IPv4 주소>"
   systemctl restart nginx
   ```

2. 스크립트 실행 권한 부여

   ```bash
   sudo chmod +x /etc/cron.monthly/renew_ssl.sh
   ```

## 8. 최종 테스트

- 브라우저에 `https://<퍼블릭 IPv4 주소>` 접속

  - 기본적으로 브라우저를 통한 접속은 GET 요청이다. 즉, `/` 엔드포인트에 GET 요청을 보낸 것이다.

  > - 이 때 브라우저에서 "안전하지 않음" 경고가 뜰 수 있다. Self-Signed 인증서는 공인된 기관(CA)에서 발급된 것이 아니므로 브라우저가 신뢰하지 않기 때문이다.
  > - 마찬가지로 클라이언트 코드에서는 SSL 검증을 무시할 수 없다. 즉, 클라이언트 코드에서는 우리가 만든 서버를 사용할 수 없다.
  > - 그러나 서버 사이드 코드 (Next.js App Router의 Route Handlers 등) 에서는 이를 무시하도록 설정할 수 있다.
  > - 또한, 비록 자체 서명 인증서이지만 SSL/TLS를 사용하여 데이터를 암호화 하기때문에 데이터 보호는 확실하게 된다.

- `{"message":"Hello, FastAPI!"}`가 출력되는지 확인한다.
  - 잘 보인다면 배포 완료!

## 참고: 코드 수정 후 적용 방법

### (1) Git을 사용한 코드 업데이트

```bash
cd ~/snuclsai
git pull origin main
```

### (2) FastAPI 서비스 재시작

```bash
sudo systemctl restart snuclsai
```

### (3) Nginx 설정이 바뀌었다면 적용

```bash
sudo systemctl restart nginx
```

### (4) 전체 로그 확인

```bash
journalctl -u snuclsai --no-pager --since "5 minutes ago"
```

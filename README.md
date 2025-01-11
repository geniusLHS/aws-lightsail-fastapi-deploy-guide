# AWS Lightsail을 이용한 FastAPI 배포 로드맵

## 0. 작업 일정

| 날짜  | 해당 단계 | 작업 내용                                          | 예상 소요 시간 |
| ----- | --------- | -------------------------------------------------- | -------------- |
| 1일차 | 1, 2      | AWS Lightsail 생성 후 SSH 연결                     | 3시간          |
| 2일차 | 3, 4, 5   | 프로젝트 환경 설정 및 FastAPI 코드 백그라운드 실행 | 4시간          |
| 3일차 | 6         | Nginx 설정 및 실행                                 | 1시간          |
| 4일차 | 7, 8      | HTTPS 적용 후 최종 테스트                          | 3시간          |

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
  - 그래도 안된다면 lightsail 인스턴스 재부팅 해볼 것
  - 잘 접속 되다가 갑자기 Timeout 에러가 뜰 때가 있다. 서버가 열 받은 것 같은데 일단 밥 먹고 오면 해결된다.

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
  python3 -m venv venv
  source venv/bin/activate
  ```

  > - 이 때 터미널에서 `(venv)` 라는 표시가 나타나면 가상 환경에 접속된 상태이다.
  > - VSCode에서 다른 디렉토리를 갔다가 다시 돌아오거나 하면 `(venv)` 표시가 사라질 수 있는데, 이 때는 다시 `source venv/bin/activate` 명령어를 실행해야 한다.

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
ExecStart=/home/ubuntu/snuclsai/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

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

  # POST 요청
  curl http://localhost:8000/test -X POST -H "Content-Type: application/json" -d '{"message": "YAHO~~"}'
  ```

- 이제 서버 컴퓨터 외부에서의 요청도 처리할 수 있도록 해보자.

## 6. Nginx 설치 및 설정

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
sudo systemctl restart nginx
```

- 이제 서버 컴퓨터 외부에서도 요청을 보낼 수 있다.

  다음 코드를 실행해서 잘 작동하는지 테스트 해보자. (**로컬 컴퓨터** 터미널에서 실행)

  ```bash
  # GET 요청
  curl http://<퍼블릭 IPv4 주소> -X GET

  # POST 요청
  curl http://<퍼블릭 IPv4 주소>/test -X POST -H "Content-Type: application/json" -d '{"message": "YAHO~~"}'
  ```

## 7. HTTPS 적용

- 아직 미정

## 8. 최종 테스트

```bash
# 1. GET 요청
curl https://<주소>/ -X GET

# 2. POST 요청
curl https://<주소>/test -X POST -H "Content-Type: application/json" -d '{"message": "YAHO~~"}'
```

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

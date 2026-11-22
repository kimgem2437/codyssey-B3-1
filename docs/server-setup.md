# 서버 설정 기록 (Nginx + /health)

> 각 회색 박스는 **그대로 복사해서 붙여넣으면** 됩니다. 박스 위에 **[노트북]** 은 노트북 PowerShell, **[서버]** 는 SSH 접속 후 서버에서 입력한다는 뜻입니다.

## 1. SSH 접속

**[노트북]**

```
ssh -i $HOME\.ssh\b3-1-key.pem ubuntu@43.201.85.164
```

- 접속되면 프롬프트가 `ubuntu@ip-10-0-1-176:~$` 로 바뀝니다.

## 2. Nginx 설치

**[서버]**

```
sudo apt update
```

```
sudo apt install -y nginx
```

- `apt update`: 설치 가능한 패키지 목록을 최신으로 갱신 (서버 → 인터넷 아웃바운드 필요)
- `apt install -y nginx`: Nginx 설치. `-y`는 "계속할까요?"에 자동으로 yes

## 3. 실행 상태 확인

**[서버]**

```
systemctl status nginx --no-pager
```

- `active (running)` 이 보이면 정상
- `--no-pager`: 결과를 스크롤 화면(less) 없이 바로 출력. 빼고 실행했다가 화면에 갇히면 `q` 를 눌러 빠져나옴

```
curl -I http://localhost
```

- 첫 줄에 `HTTP/1.1 200 OK` 가 나오면 성공 (과제 필수 항목)

## 4. /health 엔드포인트 추가

**[서버]** 설정 파일을 엽니다.

```
sudo nano /etc/nginx/sites-available/default
```

`server_name _;` 줄을 찾아서 **그 바로 아래**에 다음 4줄을 붙여넣습니다.

```
        location = /health {
                default_type text/plain;
                return 200 "OK\n";
        }
```

- 저장: `Ctrl + O` → `Enter`, 종료: `Ctrl + X`
- `location = /health`: 주소가 정확히 /health 일 때만 적용
- `return 200 "OK\n"`: 파일 없이 상태 코드 200과 OK 문구를 바로 응답

**[서버]** 설정 문법을 검사합니다.

```
sudo nginx -t
```

- `syntax is ok`, `test is successful` 이 나와야 다음으로 진행
- 오류가 나면 nano로 다시 열어 괄호 `{ }` 와 세미콜론 `;` 확인

**[서버]** 설정을 반영합니다.

```
sudo systemctl reload nginx
```

**[서버]** 서버 안에서 확인합니다.

```
curl -i http://localhost/health
```

- `HTTP/1.1 200 OK` 와 마지막 줄 `OK` 가 나오면 성공

## 5. 외부에서 확인

**[서버]** SSH를 빠져나옵니다.

```
exit
```

**[노트북]**

```
curl.exe -i http://43.201.85.164/health
```

- 브라우저 주소창에 `http://43.201.85.164/health` 를 입력해도 됩니다. (`https` 아님)
- 이 화면을 **스크린샷** → `docs/images/` 에 저장 (외부 접속 증빙)

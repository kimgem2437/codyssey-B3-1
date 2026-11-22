# 트러블슈팅 실습: 보안 그룹 80 포트 제거

> 각 회색 박스는 그대로 복사해서 붙여넣으면 됩니다. **[노트북]** = 노트북 PowerShell, **[서버]** = SSH 접속 후 서버.
> 단계마다 결과 화면을 스크린샷으로 남깁니다.

## 0. 오류 만들기 (완료)

- `b3-1-web-sg` 인바운드에서 **HTTP 80 규칙 삭제** → SSH 22만 남음

## 1. 증상 확인

**[노트북]** 외부에서 /health 요청 (10초 안에 응답 없으면 포기)

```
curl.exe -i --max-time 10 http://43.201.85.164/health
```

- 예상 결과: `Connection timed out` 또는 `Failed to connect` → 응답이 아예 오지 않음
- 📸 스크린샷: `12-symptom-timeout.png`

## 2. 원인 좁히기

### 가설 목록

| 가설 | 이게 원인이라면 |
|---|---|
| A. 서버(EC2)가 꺼짐 | SSH도 안 돼야 함 |
| B. 라우팅/IGW 문제 | SSH도 안 돼야 함 |
| C. Nginx가 죽었거나 80을 안 듣고 있음 | 서버 안에서 localhost 요청도 실패해야 함 |
| D. 보안 그룹이 80을 막음 | 서버 안에서는 되는데 밖에서만 안 됨, 요청이 서버 로그에 안 찍힘 |

### 2-1. 포트별 연결 테스트

**[노트북]** 80번 포트

```
Test-NetConnection 43.201.85.164 -Port 80
```

**[노트북]** 22번 포트

```
Test-NetConnection 43.201.85.164 -Port 22
```

- 예상 결과: 80은 `TcpTestSucceeded : False`, 22는 `True`
- 해석: 같은 서버인데 22는 되고 80만 안 됨 → **서버는 켜져 있고 라우팅도 정상** (가설 A, B 탈락)
- 📸 스크린샷: `13-port-test-80-vs-22.png`

### 2-2. 서버 내부 확인

**[노트북]** SSH 접속

```
ssh -i $HOME\.ssh\b3-1-key.pem ubuntu@43.201.85.164
```

**[서버]** Nginx 실행 상태

```
systemctl is-active nginx
```

- 예상 결과: `active`

**[서버]** 80번 포트를 누가 듣고 있는지

```
sudo ss -tlnp | grep ':80'
```

- 예상 결과: `0.0.0.0:80` 에 `nginx` 가 보임 → 모든 주소에서 80을 받을 준비가 되어 있음

**[서버]** 서버 안에서 /health

```
curl -i http://localhost/health
```

- 예상 결과: `HTTP/1.1 200 OK` / `OK` → **Nginx 정상** (가설 C 탈락)
- 📸 스크린샷: `14-server-internal-ok.png`

### 2-3. 로그로 확인 (요청이 서버에 도착했는가?)

**[서버]** Nginx 접속 로그 마지막 10줄

```
sudo tail -n 10 /var/log/nginx/access.log
```

- 예상 결과: 방금 한 `127.0.0.1` (localhost) 요청은 찍혀 있지만, **노트북에서 보낸 요청은 없음**
- 해석: 외부 요청이 **서버에 도착하기 전에** 막혔다 → 서버 앞단의 방화벽 = 보안 그룹
- 📸 스크린샷: `15-access-log-no-external.png`

**[서버]** SSH 빠져나오기

```
exit
```

### 2-4. 보안 그룹 확인 (콘솔)

- EC2 → 보안 그룹 → `b3-1-web-sg` → 인바운드 규칙
- SSH 22만 있고 **HTTP 80 규칙이 없음** → **가설 D 확정**
- 📸 스크린샷: 방금 찍은 80 삭제 화면 (`16-sg-no-http.png`)

## 3. 조치 (복구)

- `b3-1-web-sg` → **인바운드 규칙 편집** → **규칙 추가**
  - 유형: **HTTP** / 소스: **Anywhere-IPv4 (0.0.0.0/0)**
- **규칙 저장**

## 4. 결과 확인

**[노트북]**

```
Test-NetConnection 43.201.85.164 -Port 80
```

- 예상 결과: `TcpTestSucceeded : True`

```
curl.exe -i --max-time 10 http://43.201.85.164/health
```

- 예상 결과: `HTTP/1.1 200 OK` / `OK`
- 📸 스크린샷: `17-recovered.png`

**[노트북 → 서버]** (선택) 이번엔 로그에 외부 요청이 찍혔는지 확인

```
ssh -i $HOME\.ssh\b3-1-key.pem ubuntu@43.201.85.164 "sudo tail -n 3 /var/log/nginx/access.log"
```

- 예상 결과: 127.0.0.1 이 아닌 **노트북의 공인 IP**에서 온 `GET /health ... 200` 줄이 보임
- 📸 스크린샷: `18-access-log-external.png`

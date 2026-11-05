# 개발자 작업실 구축 미션

> 이 저장소는 Codyssey와 GitHub 연동에 사용되었으며, 이 문서는 입학연수 개발 워크스테이션 미션의 수행 기록이다.

## 1. 미션 개요
Windows 개발 환경에서 터미널, 권한, Docker, Git/GitHub를 설정하고, Dockerfile 기반 웹 서버를 빌드·실행해 포트 매핑, 바인드 마운트, 볼륨 영속성을 검증한다.

## 2. 개발 환경
- OS: Windows (PowerShell 사용)
- PowerShell: 5.1.26100.9818
- WSL 배포판: Ubuntu, 기본 WSL 버전 2
- Git for Windows: 2.56.0.windows.4
- Docker: 29.9.2 (build a82e528)
- Docker Server: 29.9.2

## 3. 수행 체크리스트
- [x] 터미널 위치·파일 목록·숨김 파일 확인
- [x] 파일/디렉터리 생성, 복사, 이동, 이름 변경, 삭제
- [x] 파일 내용 확인 및 빈 파일 생성
- [x] Linux 파일·디렉터리 권한 변경
- [x] Docker 설치 및 엔진 점검
- [x] `hello-world`, Ubuntu 컨테이너 실행 및 운영
- [x] Nginx 기반 커스텀 이미지 빌드
- [x] 포트 매핑 및 브라우저 접속 확인
- [x] 바인드 마운트 변경 반영 확인
- [x] 볼륨 데이터 영속성 확인
- [x] README 및 증거 캡처 최종 정리
- [x] GitHub 저장소 생성·업로드 및 VS Code 연동 증거
- [ ] Codyssey 평가 요청

## 4. 터미널 기본 조작
PowerShell에서 `Documents\codyssey-workshop\cli-practice` 폴더를 사용해 실습했다.

```powershell
Get-Location
Get-ChildItem -Force
New-Item -ItemType File -Name empty.txt
"sample text for Codyssey" | Set-Content -Encoding ASCII notes.txt
Get-Content .\notes.txt
New-Item -ItemType Directory -Name work
Copy-Item .\notes.txt .\work\notes-copy.txt
Move-Item .\notes.txt .\work\notes.txt
Rename-Item .\work\notes.txt notes-renamed.txt
Remove-Item .\work\notes-copy.txt
Remove-Item .\work\notes-renamed.txt
Remove-Item .\work
```

확인 결과: `empty.txt`는 0바이트였고, `Get-ChildItem -Force`로 숨김 파일을 확인했다. 파일 내용은 `sample text for Codyssey`였다. 복사·이동·이름 변경 후 임시 파일과 폴더를 삭제했다.

## 5. 권한 실습 (WSL Ubuntu)
```bash
chmod 600 sample.txt
chmod 700 sample-dir
ls -ld sample.txt sample-dir
chmod 644 sample.txt
chmod 755 sample-dir
ls -ld sample.txt sample-dir
```

변경 전:
```text
drwx------ ... sample-dir
-rw------- ... sample.txt
```

변경 후:
```text
drwxr-xr-x ... sample-dir
-rw-r--r-- ... sample.txt
```

- `644`: 소유자는 읽기/쓰기, 그룹과 기타 사용자는 읽기 가능 (`rw-r--r--`).
- `755`: 소유자는 읽기/쓰기/접근, 그룹과 기타 사용자는 읽기/접근 가능 (`rwxr-xr-x`).
- 디렉터리의 `x` 권한은 디렉터리 안으로 접근(traverse)하는 권한이다.

## 6. Docker 설치 및 기본 점검
```powershell
docker --version
# Docker version 29.9.2, build a82e528

docker info --format '{{.ServerVersion}}'
# 29.9.2

docker run --rm hello-world
# Hello from Docker! (설치 및 데몬 통신 성공)

docker images
# hello-world:latest, ubuntu:latest 및 이후 빌드한 codyssey-web:1.0 확인
```

### Ubuntu 컨테이너
```powershell
docker run --name codyssey-ubuntu -it ubuntu bash
```
컨테이너 안에서 `pwd`, `ls -la`, `echo`를 실행하고 `exit`했다. `docker ps -a`에서 컨테이너가 `Exited (0)`으로 종료된 것을 확인했다.

### `exec`와 `attach`, 로그·리소스·중지
```powershell
docker run -dit --name codyssey-ubuntu-bg ubuntu bash
docker exec -it codyssey-ubuntu-bg bash
# 컨테이너 안에서 echo 실행 후 exit. 메인 컨테이너는 계속 Up

docker attach codyssey-ubuntu-bg
# 메인 셸에 연결. Ctrl+P, Ctrl+Q로 분리

docker logs codyssey-ubuntu-bg
docker stats --no-stream codyssey-ubuntu-bg
docker stop codyssey-ubuntu-bg
docker ps -a --filter name=codyssey-ubuntu-bg
```
`exec` 셸을 종료해도 컨테이너 메인 프로세스는 실행 중인 것을 확인했다. `docker stop` 후 컨테이너는 `Exited` 상태였다.

## 7. 커스텀 웹 서버 이미지 및 포트 매핑
### Dockerfile
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

`index.html`에 실습 제목과 한국어 설명을 작성했다.

```powershell
docker build -t codyssey-web:1.0 .
docker run -d --name codyssey-web -p 8080:80 codyssey-web:1.0
```
- 베이스 이미지: `nginx:alpine`
- 호스트 포트: `8080`
- 컨테이너 포트: `80`
- 접속 주소: http://localhost:8080/
- 브라우저에서 HTML 제목과 내용을 확인했다.

**증거:** [port-8080.png](evidence/port-8080.png)

## 8. 바인드 마운트 검증
기존 웹 컨테이너를 호스트 파일 바인드 마운트로 다시 실행했다.

```powershell
docker run -d --name codyssey-web -p 8080:80 `
  --mount "type=bind,source=$($PWD.Path)\index.html,target=/usr/share/nginx/html/index.html,readonly" `
  codyssey-web:1.0

docker inspect codyssey-web --format '{{json .Mounts}}'
```
검증 결과의 주요 값:
```json
{"Type":"bind","Destination":"/usr/share/nginx/html/index.html","RW":false}
```
호스트 `index.html` 내용을 수정한 뒤 이미지를 다시 빌드하지 않고 브라우저를 새로고침해 변경 문구 `바인드 마운트 변경이 반영되었습니다.`가 보이는 것을 확인했다.

**증거:** [bind-mount.png](evidence/bind-mount.png)

## 9. Docker 볼륨 영속성 검증
```powershell
docker volume create codyssey-data
docker run --name codyssey-volume-write -v codyssey-data:/data ubuntu sh -c "echo 'volume persistence works' > /data/proof.txt && cat /data/proof.txt"
docker rm codyssey-volume-write
docker run --rm -v codyssey-data:/data ubuntu cat /data/proof.txt
docker volume inspect codyssey-data
```
첫 컨테이너에서 `volume persistence works`를 기록한 뒤 컨테이너를 삭제하고, 새 컨테이너에서 같은 내용을 읽었다. 볼륨 데이터가 컨테이너 제거 후에도 유지됨을 확인했다.

## 10. Git 설정 및 GitHub 연동
```powershell
git config --global user.name "gigahitz7"
git config --global user.email "<configured email; mask in public evidence>"
git config --global init.defaultBranch main
git config --list

# 기록 예시 (실제 이메일은 공개 README/캡처에서 마스킹)
# user.name=gigahitz7
# user.email=<redacted>
# init.defaultBranch=main
```
`git config --list` 증거를 기록할 때 이메일 등 개인 정보는 마스킹한다.

- GitHub 저장소 URL: https://github.com/gigahitz7/hoyoung_codyssey
- GitHub/VS Code 작업 공간 캡처: [github-vscode.png](evidence/github-vscode.png)
- VS Code Source Control 캡처: [github-vscode-repo.png](evidence/github-vscode-repo.png)

## 11. 트러블슈팅
### 1) PowerShell에서 `docker` 명령을 찾지 못함
- 문제: `docker`가 cmdlet/프로그램으로 인식되지 않았다.
- 원인 가설: Docker Desktop/CLI가 설치되지 않았거나 PowerShell 환경에서 아직 명령을 찾지 못하는 상태였다.
- 확인: `docker --version`과 `docker info`가 모두 명령을 찾지 못했다.
- 해결 및 검증: Docker Desktop 설치·실행 후 새 PowerShell에서 재확인했고, `Docker version 29.9.2`와 서버 버전 `29.9.2`를 확인했다.

### 2) PowerShell 응답에서 한글이 깨져 보임
- 문제: `Invoke-WebRequest` 결과에서 한글이 깨진 문자로 표시됐다.
- 확인: 같은 페이지를 브라우저에서 열어 확인한 결과, 제목과 한국어 문장이 정상 표시됐다.
- 대안: PowerShell 5.1 출력의 문자 디코딩 문제로 판단하고 브라우저 렌더링으로 웹페이지 표시를 검증했다.

## 12. 보안 및 개인정보
- 비밀번호, 토큰, 개인키, 인증 코드 등은 문서·로그·스크린샷에 포함하지 않는다.
- 공개 저장소 커밋 기록에 이메일이 노출되지 않도록 확인하고, 필요하면 GitHub `noreply` 이메일을 사용한다.
- 계정 정보가 찍힌 `git config --list` 캡처는 마스킹 후 사용한다.


## 13. 사전평가 보완: 핵심 개념과 진단

### 이미지와 컨테이너
- Docker 이미지는 컨테이너를 만들 때 사용하는 변경 불가능한 템플릿이다. 컨테이너는 그 이미지로부터 실행된 격리된 인스턴스이며, 실행 중 변경은 기본적으로 컨테이너의 쓰기 계층에 생긴다.
- Dockerfile을 수정하면 이미지를 다시 빌드해야 변경이 반영된다. 컨테이너를 재시작하는 것만으로 이미지 내용이 바뀌지는 않는다.

### 네트워크와 포트 포워딩
- 컨테이너는 별도의 네트워크 네임스페이스를 사용하므로, 컨테이너의 80번 포트가 호스트에서 자동으로 보이지 않는다.
- `-p 8080:80`은 호스트의 8080번 포트로 들어온 요청을 컨테이너의 80번 포트로 전달한다. 따라서 브라우저에서 `http://localhost:8080`으로 접속한다.

### 절대 경로와 상대 경로
- 바인드 마운트의 source는 호스트 파일 경로다. 이 실습은 `$PWD.Path`를 사용해 현재 프로젝트 폴더를 기준으로 실제 절대 경로를 만들었다.
- 상대 경로는 명령을 실행한 현재 디렉터리에 따라 달라질 수 있어 재현성이 떨어진다. README에는 개인정보가 포함된 사용자 경로 대신 프로젝트 루트 기준 예시를 사용한다.

### 포트 충돌 진단 (Windows PowerShell)
Docker가 8080 포트를 이미 사용 중이라고 보고하면 먼저 수신 프로세스를 확인한다.

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen -ErrorAction SilentlyContinue | Select-Object LocalAddress, LocalPort, OwningProcess
Get-Process -Id <OwningProcess PID>
```

프로세스가 실제로 불필요한지 확인한 뒤에만 종료한다. 종료할 수 없거나 다른 앱이 사용하는 포트라면 `-p 8081:80`처럼 다른 호스트 포트를 선택하고 `http://localhost:8081`로 검증한다. 이 미션에서는 포트 충돌이 발생하지 않아 진단 절차는 예방용으로 기록한다.

### Git 원격 저장소와 push 검증
```text
git remote -v
origin  https://github.com/gigahitz7/hoyoung_codyssey.git (fetch)
origin  https://github.com/gigahitz7/hoyoung_codyssey.git (push)

git push -u origin main
main -> main (성공)
```

### 저장소 디렉터리 구조
```text
.
├── Dockerfile
├── index.html
├── README.md
├── cli-practice/
└── evidence/
    ├── bind-mount.png
    ├── port-8080.png
    ├── github-vscode.png
    └── github-vscode-repo.png
```





































## 14. 볼륨 백업 및 복원 검증
백업 아카이브는 임시 폴더에 생성해 저장소에는 포함하지 않았다.

```powershell
$BackupDir = Join-Path $env:TEMP "codyssey-backup"
New-Item -ItemType Directory -Force -Path $BackupDir | Out-Null
docker run --rm -v codyssey-data:/data --mount "type=bind,source=$BackupDir,target=/backup" ubuntu sh -c "tar -czf /backup/codyssey-data.tar.gz -C /data ."
Get-Item (Join-Path $BackupDir "codyssey-data.tar.gz")
docker volume create codyssey-data-restore
docker run --rm -v codyssey-data-restore:/data --mount "type=bind,source=$BackupDir,target=/backup,readonly" ubuntu sh -c "tar -xzf /backup/codyssey-data.tar.gz -C /data"
docker run --rm -v codyssey-data-restore:/data ubuntu cat /data/proof.txt
```

검증 결과: 복원된 새 볼륨에서 `volume persistence works`가 출력되어 백업 데이터 복원을 확인했다.

## 🔐 보너스 과제: SSH 키 설정

### 작업 목적
매번 GitHub 접속 시 입력하던 아이디/토큰 인증을 SSH 키 방식으로 전환하여,
인증 과정을 자동화하고 보안을 강화함.

### 작업 단계
1. SSH 키 생성 (공개키/개인키 한 쌍 생성)
2. 공개키를 GitHub 계정에 등록
3. 원격 저장소 주소를 HTTPS → SSH 방식으로 변경
4. SSH 연결 테스트로 정상 인증 확인

### 사용한 주요 명령어
```bash
# 원격 주소를 SSH로 변경
git remote set-url origin git@github.com:계정명/저장소명.git

# 변경된 원격 주소 확인
git remote -v

# SSH 연결 테스트
ssh -T git@github.com
```

### 결과 (인증 성공)
```
Hi 계정명! You've successfully authenticated,
but GitHub does not provide shell access.
```
→ "successfully authenticated" 메시지로 SSH 인증 성공 확인

### 기대 효과
- git push/pull 시 아이디·토큰 입력 불필요 → 작업 효율 향상
- 키 기반 인증으로 보안 강화









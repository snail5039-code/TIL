2026.07.22 3일차(GitHub 특강)


1. VSCODE 설치

```markdown
## Git / GitHub

Git : 버전 관리 프로그램 

버전 : 컴퓨터 소프트웨어의 특정 상태 

관리 : 어떤 일의 사무, 시설이나 물건의 유지, 개량

프로그램 : 컴퓨터에서 실행될 때 특정 작업을 수행하는 일련의 명령어들의 모음
```

- 우리가 레포트를 수정하는 것처럼 Git을 통해 버전관리를 한다. 날짜, 변경사항 등등

```markdown
중앙 집중식
분산 
Git은 분산 버전 관리를 사용 
### 중앙 서버와 각 사용자 모두가 따로따로 모든 것을 가지고 있는 형태
```

- Git을 활용하면 포토폴리오로 활용할 수 있다.
- 웹페이지랑 연결해서 만들 수 있다.
- 1일 1 커밋을 해서 잔디밭을 채우자

```markdown
TIL Today I Learned 잔디밭 관리시 이런식으로 만들어서 올리면 좋다
```

- winget install --id Git.Git -e --source winget → 깃 설치 코드
- VSCODE 터미널에서 Git 실행할 때 설치할 것

Ghostty, Kitty → 사용할 수도 있다.

```markdown
Working Directory : 실제로 파일을 만들고 수정하는 프로젝트 폴더
staging Area : 다음 커밋에 포함할 변경 사항을 선택해 두는 공간, git add를 실행하면 변경 사항
               이 이곳에 등록
Commits : git commit으로 확정한 버전 기록이 저장되는 공간 

Untracked : Git에 아직 반영되지 않은 것을 뜻함
```

```markdown
## 각 커밋은 작업 내용을 알 수 있도록 구체적인 메시지를 작성하는 것이 좋다.

### 좋은 예 :
README 작성
설치 방법 추가
오타 수정

### 좋지 않은 예 :
수정
작업함
최종
```

```markdown
git log : 지금까지 커밋한 로그 이력
HEAD : 현재 작업 중인 위치
main,master : 현재 작업중인 브랜치 이름
```

```markdown
## 순서

1. git add .
2. git commit -m ""
3. git log
4. git log --oneline
5. git config --global user.name
6. git config --global user.email
7. git remote add origin 원격저장소 URL
8. git remote -v 
9. git push origin master 

푸쉬 간 -u를 쓴 후에는 push만 써도 된다
```
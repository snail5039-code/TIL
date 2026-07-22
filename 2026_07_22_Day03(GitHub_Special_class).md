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

```markdown
## 내려받기
git clone https://github.com/snail5039-code/TIL.git

## VS Code 열기
code .

## 폴더 이동
cd
cd ~ : 최상위 폴더
cd - : 뒤로가기
cd .. : 부모 폴더 이동

## 파일 생성
touch

## 폴더 생성
mkdir
ex)
$ mkdir folder
$ mkdir 'happy hacking'

## 파일 목록
ls
-a : 숨긴 파일까지 검색
-l : long 옵션. 용량, 수정 날짜 등 파일 정보를 자세히 보여줌

## 지우기
rm 
ex)
$ rm test.txt
$ rm -r folder

## start, open
폴더/파일을 여는 명령어
Windows에서는 start를, Mac에서는 open을 사용할 수 있습니다.
```

```markdown
## 마크다운 : 텍스트 기반의 가벼운 마크업 언어

- .md 확장자, 개발과 관련된 많은 문서는 마크다운 형식으로 작성
- Github 문서의 시작과 끝
- README.md 파일을 통해 오픈 소스의 공식 문서 작성
```

```markdown
## 마크 다운 문법 

# 제목 1
## 제목 2
### 제목 3
#### 제목 4
##### 제목 5
###### 제목 6

- 순서가 없는 목록	
- 서브 목록	
- 서브 목록

+ 순서가 없는 목록
	+ 서브 목록	
	+ 서브 목록
	
* 순서가 없는 목록	
 * 서브 목록
 * 서브 목록
 
1. 순서가 있는 목록	
 1. 서브 목록	
 2. 서브 목록
 
 1. 혼합 해보기 1	
  - 순서 없음	
  + 순서 없음	
  * 순서 없음
  
 2. 혼합 해보기 2	
  1. 순서 있음	
  - 순서 없음	
  2. 순서 있음
  
## 코드
인라인 코드 : ``으로 감싸준다
블록 코드 : ``` 백틱 3번으로 감싸준다

## 링크
[GOOGLE] (<https://google.com>)
표시글자 : 주소
## 이미지
![Git로고] (<https://git-scm.com/images/logo@2x.png>)
대체 텍스트 : 주소
* 대체 텍스트란 이미지를 정상적으로 불어오지 못했을 때 표시되는 문구

## 강조
1. 기울임 : *글자* 혹은 _글자_
2. 굵게 : **글자** 혹은 __글자__
3. 취소 : ~~글자~~

## 수평선
- * _ 을 3번 이상 연속으로 작성
---
***
___
```

- 실제로 실습을 하려면  code 박스 밖에서 해야 실습할 수 있다. 참고 **
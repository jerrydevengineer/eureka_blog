---
date: 2026-05-06
lastmod: 2026-05-06
---
### Step 1: 터미널에서 프로젝트 경로로 갑니다

### Step 2: git 초기화를 합니다

``` Bash
git init
```

### Step 3: 원격 레포지토리랑 연결합니다

``` Bash
git remote add origin https://github.com/your-username/your-repo-name.git
```

### Step 4: 원격 브랜치 fetch 합니다

fetch를 하면 원격에 있는 브랜치들의 정보를 가져오지만 아직 로컬 데이터는 건드리지 않습니다.

``` Bash
git fetch
```

### Step 5: 새로운 브랜치로 이동합니다.

원격에서 만든 새로운 브랜치로 이동합니다. 아직 로컬에서 커밋한 파일이 없기 때문에, 깃이 readme 파일을 다른 파일을 삭제하지 않고 다운로드합니다.

``` Bash
git switch <your-new-branch-name>
```

### Step 6: Stage하고 Commit합니다.

모든 프로젝트 파일을 stage하고 commit합니다:

``` Bash
git add .
git commit -m "Add existing Spring Boot application code"
```

### Step 7: 원격 레포에 push합니다

``` Bash
git push -u origin <your-new-branch-name>
```

이제 새로운 브랜치에 프로젝트 코드도 올라가고 readme도 잘 있을거에요!
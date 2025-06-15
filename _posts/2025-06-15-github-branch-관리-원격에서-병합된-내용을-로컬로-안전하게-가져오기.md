---
layout: post
title: 'GitHub Branch 관리: 원격에서 병합된 내용을 로컬로 안전하게 가져오기'
date: 2025-06-15 22:17 +0900
categories: [Git]
tags: [git-branch]
---

### Summary

GitHub 블로그나 프로젝트를 운영할 때, 기능별로 브랜치를 나누어 효율적으로 작업할 수 있습니다.<br>예를 들어, 포스트 작성은 post 브랜치에서, 사이트의 주요 설정 변경은 main 브랜치에서 진행할 수 있습니다.

post 브랜치의 작업을 GitHub에서 main으로 병합(merge)했을 때, 로컬의 main 브랜치는 이를 어떻게 알 수 있을까요?

이 글에서는 원격 저장소의 변경 사항을 로컬 저장소로 가져오는 방법을 단계별로 안내합니다.

---

아래의 흐름대로, 브랜치 관리 방법을 살펴보겠습니다.

1. `post` 브랜치에서 새로운 글을 작성하고 원격 저장소에 `push` 합니다.
2. GitHub 에서 `post` 브랜치를 `main` 브랜치로 **Pull Request(PR)** 를 통해 병합( `merge` )합니다.
3. 로컬로 돌아와서 `main` 브랜치를 원격 저장소의 최신 상태에 맞게 `pull` 로 동기화합니다.
4. (선택 사항) 역할을 다한 `post` 브랜치를 삭제합니다.

---

## **Step 1 : post 브랜치에서 작업 및 push**

먼저, 새로운 포스트를 작성하기 위한 브랜치를 생성한 후 작업을 시작합니다

`post` 브랜치로 이동합니다.<br>해당 브랜치가 없으면 `-b` 옵션으로 생성 후, 바로 이동할 수 있습니다.

```terminal
git checkout -b post
```

원하는 작업을 모두 마친 후, 변경 사항을 `commit` 하고 원격 저장소에 `push` 합니다.

```terminal
# 변경된 파일들을 작성합니다.
git add .

# 변경 사항을 작성합니다.
git commit -m "{원하는 메시지를 작성}"

# post 브랜치를 원격 저장소에 push 합니다.
git push origin post
```

---

## **Step 2 : GitHub 에서 Pull Request(PR) 로 병합**

GitHub 으로 이동하면, 방금 `push` 한 브랜치에 대해 "**Compare & pull request**" 버튼이 활성화된 것을 볼 수 있습니다. 이 버튼을 클릭합니다.

PR 제목과 내용을 작성하고, 변경 사항을 확인한 후 "**Create pull request**" 버튼을 클릭합니다.<br>생성된 PR 페이지에서 특별한 충돌(Conflict)이 없다면, "**Merge pull request**" 버튼이 나타나고, 이를 클릭하면 최종적으로 `main` 브랜치에 변경 사항을 병합( `merge` )합니다.

이제 원격 저장소의 `main` 브랜치에 새로운 작업 내용을 올렸습니다. 하지만 아직 로컬의 `main` 브랜치는 변경 사항을 모르고 있습니다.

---

## **Step 3 : 로컬의 main 브랜치를 업데이트**

다시 터미널로 돌아와서, 원격 저장소의 최신 상태를 로컬의 main 브랜치로 가져옵니다.

지금은 post 브랜치에 있으니, main 브랜치로 이동해야 합니다.

```terminal
git checkout main
```

이때 Git은 친절하게 안내 메시지를 보여줍니다.

```terminal
Switched to branch 'main'
Your branch is behind 'origin/main' by 2 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)
```

이 메시지는 "로컬의 main 브랜치가 원격 저장소보다 2 commits 뒤쳐저 있으니, git pull 을 사용하여 업데이트"를 하라는 의미입니다.

```terminal
git pull origin main
```

`pull` 이 완료되면, 원격에서의 `main` 브랜치에 병합되었던 관련 파일들이 로컬에서의 `main` 브랜치에 모두 들어온 것을 확인할 수 있습니다.

**💡 git pull 과 git pull origin main 의 차이점?**

일반적으로 로컬 브랜치는 이름이 같은 원격 브랜치를 자동으로 추적하므로 `git pull` 을 사용해도 괜찮습니다. `git pull origin main` 은 더 정확한 명령어로, 둘 다 기능적으로는 동일하게 동작합니다.

---

## **Step 4 : 브랜치 정리하기 (선택 사항)**

`post` 브랜치는 `main` 에 모든 내용을 전달하였고, 역할을 마친 브랜치는 삭제하는 것이 관리 측면에서 좋습니다. (계속 사용할 브랜치라면 삭제하지 않아도 됩니다.)

```terminal
git branch -d post
```
---
layout: post
title: GitHub Pages에 Post 작성하는 방법 (간단 버전)
date: 2025-06-01 20:55 +0900
categories: [GitHub Pages]
tags: [jekyll, chirpy, guide]
---

### **Summary**

GitHub Pages 설정을 마친 후, Post를 작성하는 방법을 정리하였습니다. Jekyll을 이용해 블로그를 만들었다면 아래의 단계를 따라 Post를 작성할 수 있습니다.

Post 작성 전 자세한 설정 과정은 [여기](https://geraldtui.com/posts/create-a-digital-garden/)에 잘 설명되어 있습니다. (현재 사용하고 있는 테마는 Chirpy Theme 입니다.)

### **Reference**

- [Create a Digital Garden using Jekyll for FREE!](https://geraldtui.com/posts/create-a-digital-garden/)

---

## **Step 0**
>Post를 자동으로 생성!

Jekyll의 명령어를 사용하면 정해진 형식에 맞춰, markdown 파일을 자동으로 생성할 수 있습니다.

```terminal
bundle exec jekyll post "{Post 제목 입력}"
```

명령어를 실행하면 `_posts` 폴더 안에 `YYYY-MM-DD-{Post 제목 입력}.md` 와 같은 파일이 생성됩니다. 생성된 파일에는 아래와 같이 YAML 형식으로 기본적인 정보(Front matter)가 자동으로 입력되어 있습니다.

```yaml
---
layout: post
title: {Post 제목 입력}
date: YYYY-MM-DD HH:MM +0900
---
```

## **Step 1**
>Front matter 추가!

이제 생성된 markdown 파일을 열어 추가 정보를 입력할 수 있습니다.

`categories` 와 `tags` 는 글을 분류하는 중요한 정보입니다. 아래와 같이 대괄호(`[]`)를 사용해 요소를 추가할 수 있습니다.

- `categories` : Post의 카테고리 항목을 지정
- `tags` : Post의 태그를 지정

```yaml
---
layout: post
title: {Post 제목 입력}
date: YYYY-MM-DD HH:MM +0900
categories: [{카테고리 입력}]
tags: [{tag 1 입력}, {tag 2 입력}]
---
```

#### **🤔 categories와 tags도 front matter에 자동으로 생성되도록 하려면?**

`_config.yml` 파일의 끝 부분에 다음의 내용을 추가해 주세요!
```yaml
jekyll_compose:
  default_front_matter:
    posts:
      layout: post
      categories: []
      tags: []
```
빈 리스트 형식으로 2가지 필드가 자동으로 생성되는데요. \
front matter에는 총 5가지(`layout`, `title`, `date`, `categories`, `tags`)가 포합됩니다.

Front matter의 아래부터는 markdown 문법으로 Post 내용을 작성하면 됩니다.

## **Step 2**
>로컬 서버에서 실시간으로 확인!

글을 작성하면서 실제 블로그에 어떻게 보이는지 확인하고 싶을 때, 자신의 로컬 서버에서 미리 확인할 수 있습니다.

아래 명령어를 터미널에 입력해 로컬 서버를 실행합니다.

```terminal
bundle exec jekyll s
```

서버가 성공적으로 실행되면 터미널에 아래와 같은 메시지가 나타납니다.

```terminal
...{생략}...
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```

이제 브라우저에서 `http://127.0.0.1:4000/` 로 접속하면, 실제로 게시된 것처럼 자신의 블로그를 미리 확인할 수 있습니다. Post의 내용을 수정하고 저장될 때마다 해당 브라우저는 자동으로 내용을 업데이트 합니다.

최종으로 Commit과 Push를 해주면, `{xxx}.github.io` 에 Post가 게시된 것을 확인할 수 있습니다.
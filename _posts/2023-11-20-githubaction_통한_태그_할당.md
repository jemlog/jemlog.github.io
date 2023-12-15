---
title: Github actions를 통한 PR Assignee & Reviewers 자동 할당
author: jemlog
date: 2022-5-15 00:20:00
categories: [Github Actions]
tags: [Github Actions, Automatic]
pin: false
img_path: '/assets/img'
---


오랜만에 글을 작성하는 것 같습니다. 오늘은 Pull Request 과정에서 번거로움을 느꼈던 작업들을 자동화하는 방법에 대해 살펴볼까 합니다.

![asdfdsf](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/3b8a815a-afe2-4f46-a4d2-ec638bd7824a)

PR 작업시 항상 마주하는 부분입니다.
우리는 PR을 진행할때 누구에게 리뷰를 받을지, PR의 담당자는 누구인지 그리고 어떤 라벨에 해당하는지를 설정합니다. 이 과정에서 별도의 설정이 없다면 수동으로 인원을 할당하고 라벨을 직접 선택해야합니다. 아직 서비스 규모가 작고 PR 주기가 길다면 그리 귀찮지 않을 수 있습니다.



하지만 PR의 단위를 작게 가져가는 방향으로 협업 방식이 개선되고, 하루에도 몇 십번씩 PR, Merge, 배포를 진행하는 팀이 된다면 이 반복적인 일은 꽤나 귀찮게 다가올 것입니다. 비효율을 개선하고 싶었고, PR을 올리기만 하면 자동으로 할당해주는 방법을 찾아봤습니다. 그러던 중 Github actions을 사용한 자동화 방법들을 발견했고 오늘 그 중 한가지를 소개하고자 합니다.

Reviewer와 Assignee 자동 할당
Github Marketplace에 auto-assign-action 이라는 템플릿이 있고 Reviewer와 Assignee를 손쉽게 설정할 수 있습니다.

먼저 .github/workflows/ 디렉토리 내에 yml 파일을 하나 생성합니다.

```yaml
name: 'PR Auto Assign'
on:
pull_request:
types: [opened, ready_for_review]

permissions:
contents: read
pull-requests: write

jobs:
add-reviewers-and-assignee:
runs-on: ubuntu-latest
steps:
- uses: kentaro-m/auto-assign-action@v1.2.5
with:
repo-token: '${{ secrets.GITHUB_TOKEN }}'
configuration-path: '.github/auto_assign_config.yml'
```

PR을 생성할때 workflow를 활성화할 것이므로 초기에 PR에 열리는 opened와 Draft 상태에서 PR로 넘어갈때의 ready_for_review를 트리거로 만들었습니다.
permissions을 넣지 않으면 workflow 실행 과정에서 권한이 없다는 에러가 발생할 수 있으므로 꼭 permissons를 추가해야 합니다!
다음으로는 설정파일인 auto_assign_config.yml을 살펴보겠습니다.

```java
addAssignees: author
addReviewers: true
```
- addAssignees : PR을 생성한 사람이 자동으로 할당된다
- addReviewers : 프로젝트에 참여하는 팀원들을 자동으로 리뷰어로 등록한다

해당 설정까지 완료 후 PR을 등록하면 내가 Assignees로 등록되고, 팀원들은 자동으로 Reviewers로 등록됩니다. 더 자세한 설정은 auto-assign-action에 들어가면 자세히 확인할 수 있으니 체크해봐주세요!




[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags

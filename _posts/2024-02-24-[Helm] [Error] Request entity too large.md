---
title: "[Helm] [Error] Request entity too large"
date: 2024-02-24 17:38:00 +0900
categories: [Helm]
tags: [helm, troubleshooting, error]
---

# 문제 상황
* `helm install` 시 size limit 에러 발생
```bash
Error: create: failed to create: Request entity too large: limit is 3145728
```

<br>

# 해결 방법

.helmignore 파일 생성하여 helm install시 필요하지 않은 파일들은 무시하도록 설정<br>
아래 예시는 gitlab 공식 레포지토리의 .helmignore <br> https://gitlab.com/gitlab-org/charts/gitlab/-/blob/master/.helmignore?ref_type=heads
```
# Patterns to ignore when building packages.
# This supports shell glob matching, relative path matching, and
# negation (prefixed with !). Only one pattern per line.
.DS_Store
# Common VCS dirs
.git/
.gitignore
.bzr/
.bzrignore
.hg/
.hgignore
.svn/
# Common backup files
*.swp
*.bak
*.tmp
*~
# Various IDEs
.project
.idea/
*.tmproj
# Project/CI/CD related items
.*
.gitlab
.gitlab-ci.yml
.markdownlint.json
.dockerignore
.helmignore
Dangerfile
Rakefile
Gemfile
Gemfile.lock
gems/
ci/
doc/
examples/
images/
certs/
scripts/
spec/
build/
changelogs/
# CHANGELOG.md
bin/
spec/
# dependencies.io
dependencies.yml
deps.yml
dependencies_io/
```

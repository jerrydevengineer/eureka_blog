---
title: cli_log_analyzer
description: ""
tags:
  - claude_code
  - python
  - open_source
date: 2026-06-26
---
# Claude랑 Python으로 로그 분석기 만들기

어제는 claude랑 간단한 로그 분석기를 만들어 봤습니다. 간단히 파싱하고 error, warning 등 개수를 쉽게 파악할 수 있고, 슬랙, 디스코드, 이메일 전송이 가능합니다. 그리고 계속 프로세스를 돌리면서 level에 몇 개 이상 되면 알람을 보낼 수 있고 웹뷰도 가능합니다. sqlite를 내장해서 로그를 저장 후 쿼리를 날릴 수도 있습니다.
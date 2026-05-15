---
title:
tags:
  - 멀티캠퍼스부트캠프
  - 부트캠프
  - 유레카4기
  - db
description:
date: Tuesday, April 7th 2026, 5:14:07 pm
lastmod: Wednesday, May 6th 2026, 1:25:58 pm
---
이번주에는 DB에 관해서 배웠습니다. 우선 기본적인 SQL과 설치 방법, 이론 등을 배웠는데 이에 관해 간단히 정리해보겠습니다.

## DDL

데이터베이스 구조를 만들 때 사용되고, 테이블이나 컬럼 생성, 수정, 삭제 등이 됩니다.

CREATE: 새로운 테이블 생성
ALTER: 테이블 구조 변경
DROP: 테이블 삭제
TRUNCATE: 테이블 삭제(구조는 남아있고 데이터만 삭제)
RENAME: 테이블 이름 변경
DROP이나 TRUNCATE는 삭제 후 되돌릴 수 없으니 주의해야합니다.(delete는 auto commit 아니면 rollback 가능)

## DML

데이터 조작에 사용되며 실제 데이터를 다루는 쿼리들입니다.

SELECT: 데이터 조회
INSERT: 데이터 삽입
UPDATE: 데이터 수정
DELETE: 데이터 삭제

## DCL

접근 권한 제어에 사용되며 유저에게 테이블이나 DB에 관한 권한을 부여, 회수합니다. 실무에서는 실수할 거 같은 사람들에게 주는 계정을 따로 줄 수 있습

## TCL
# 📌 Django 기반 LMS 커뮤니티 기능 백엔드 개발 프로젝트
> ⚠️ 본 레포지토리는 팀 프로젝트 중 제가 담당한 기능을 중심으로 정리한 개인 포트폴리오입니다.

Django 기반 LMS 커뮤니티 기능 백엔드 개발 프로젝트
→ **파일 처리(S3)와 데이터 정합성 유지 문제를 중심으로 설계 및 구현**

---

## 🚀 프로젝트 소개

학습자 간 소통을 위한 커뮤니티 기능을 제공하는 LMS 백엔드 시스템입니다.
게시글, 댓글, 좋아요 기능과 함께 **파일 업로드 및 관리(S3)**, **조회수 캐싱(Redis)** 등을 포함합니다.

---

## 🧑‍💻 담당 역할

* 게시글 CUD(Create / Update / Delete) API 설계 및 구현
* S3 기반 파일 업로드 및 Presigned URL 처리
* 마크다운 기반 파일 파싱 및 저장 구조 설계
* 게시글 수정 시 파일 동기화 로직 구현
* S3 파일 삭제 및 데이터 정합성 보장 구조 설계

---

## ⚙️ 기술 스택

* **Backend**: Django, DRF
* **Database**: PostgreSQL
* **Cache**: Redis
* **Storage**: AWS S3
* **Infra**: Docker

---
## 🔗 관련 링크
- [팀 프로젝트 레포](https://github.com/OZ-Coding-School/oz_externship_be_07/tree/develop/apps/community)

### 📌 게시글 CUD API
- [게시글 추가](https://github.com/OZ-Coding-School/oz_externship_be_07/blob/develop/apps/community/views/post_list_view.py#L146)
- [게시글 수정/삭제](https://github.com/OZ-Coding-School/oz_externship_be_07/blob/develop/apps/community/views/post_detail_view.py#L116)

### 📌 게시글 비즈니스 로직 (Service Layer)
게시글 생성/수정/삭제와 함께  
본문 기반 파일 파싱을 통해 S3와 DB 상태를 동기화하는 핵심 로직을 담당
- [게시글 Services](https://github.com/OZ-Coding-School/oz_externship_be_07/blob/develop/apps/community/services/post_service.py#L127)

---

## 💥 핵심 문제 해결

### 1. S3 파일과 DB 데이터 불일치 문제

#### ❗ 문제

게시글 수정 시 기존 파일이 S3에 남아 **불필요한 스토리지 누수 발생**

#### ✅ 해결

본문에서 파일 목록을 파싱 후 DB와 비교하여 동기화 수행

```python
body_files = parse_from_content(new_body)
db_files   = PostFile.objects.filter(post=post)

to_add     = body_files - db_files
to_delete  = db_files   - body_files
```

#### 🎯 결과

* S3와 DB 상태를 항상 일치하게 유지
* 스토리지 누수 방지 및 운영 비용 감소

---

### 2. Presigned URL 만료 문제

#### ❗ 문제

S3 파일 URL이 만료되면서 이미지/파일 접근 불가

#### ✅ 해결

게시글 수정 시점마다 Presigned URL 자동 재발급

#### 🎯 결과

* 항상 유효한 URL 제공
* 사용자 경험(UX) 안정성 확보

---

### 3. 파일 삭제 순서로 인한 데이터 정합성 문제

#### ❗ 문제

DB 삭제 후 S3 삭제 시 orphan 파일 발생 가능

#### ✅ 해결

**S3 삭제 → DB 삭제 순서 강제**

#### 🎯 결과

* 데이터 정합성 보장
* 파일 누수 완전 차단

---

## 🧠 기술적 포인트

### ✔ 파일 관리 구조 설계

* 본문 기준 파일 추출 → DB 비교 → 자동 동기화
* 별도 관리 없이도 상태 일관성 유지

### ✔ 정규식 기반 마크다운 파싱

* 링크 / 이미지 / 첨부파일 패턴 분리 설계
* 역할별 정규식 분리로 유지보수성 확보

### ✔ Presigned URL Lifecycle 관리

* 만료 문제를 구조적으로 해결
* 클라이언트 의존성 최소화

### ✔ 서비스 레이어 분리

* View는 요청/응답 처리
* 비즈니스 로직은 Service Layer로 분리

---

## 🔥 트러블슈팅

### 1. DRF Validation 우선순위 문제

#### ❗ 문제

custom validation보다 DRF 기본 validation이 먼저 실행됨

#### ✅ 해결

`extra_kwargs`와 `error_messages`를 활용하여 기본 validation 메시지 override

#### 🎯 결과

* 사용자 친화적인 에러 메시지 제공
* validation 흐름 제어 가능

---

### 2. Git rebase 충돌 및 히스토리 꼬임

#### ❗ 문제

rebase 과정에서 커밋 충돌 및 non-fast-forward 에러 발생

#### ✅ 해결

* `rebase -i`, `--onto` 활용하여 히스토리 재정리
* `--force-with-lease`로 안전하게 push

#### 🎯 결과

* 커밋 히스토리 정합성 확보
* 협업 시 코드 리뷰 품질 향상

---

## 😅 아쉬운 점 및 개선 방향

### ❗ 대량 파일 삭제 성능 고려 부족

* 게시글에 포함된 파일 수 증가 시 S3 삭제 비용 증가 가능

👉 개선 방향

* 비동기 처리(Celery) 도입으로 성능 개선 예정

---

### ❗ 정규식 기반 파싱의 한계

* 마크다운 형식이 변형될 경우 잘못된 데이터 추출 가능

👉 개선 방향

* Markdown Parser 라이브러리 도입 고려

---

### ❗ 초기 설계 부족

* View / Service / Serializer 역할 분리가 명확하지 않아 리팩토링 증가

👉 개선 방향

* 초기 설계 단계에서 계층 구조 명확화

---

## 📌 회고

단순히 동작하는 기능 구현을 넘어,
**데이터 정합성과 유지보수성을 고려한 설계의 중요성**을 경험한 프로젝트였습니다.

특히 파일 처리 과정에서

* S3와 DB 간 상태 일치
* URL 만료 문제
* 삭제 순서 보장

과 같은 실제 서비스에서 발생할 수 있는 문제를 해결하며
백엔드 시스템 설계에 대한 이해도를 높일 수 있었습니다.

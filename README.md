# 📌 Django 기반 LMS 커뮤니티 백엔드 — 게시글 CUD 담당

> ⚠️ 본 레포지토리는 팀 프로젝트 중 제가 담당한 기능을 중심으로 정리한 개인 포트폴리오입니다.

**S3 파일 처리와 데이터 정합성 문제를 중심으로 설계한 Django/DRF 백엔드 개발자 박진규입니다.**  
마크다운 본문 파싱 → S3 동기화 → DB 정합성 보장까지 파일 라이프사이클 전반을 설계하고 구현했습니다.

---

## 📊 프로젝트 규모

| 항목 | 수치 |
|------|------|
| 전체 프로덕션 파일 | 39 files |
| 전체 코드 라인 | 2,935 LOC |
| 데이터 모델 | 7 Models |
| 테스트 코드 | 47 Tests |
| 프로젝트 기간 | 2025.03.05 ~ 2025.03.27 (약 4주) |
| 팀 구성 | BE 4인 (게시글 조회 / 게시글 CUD / 댓글 / Admin) |

---

## 🧑‍💻 담당 역할

- 게시글 CUD(Create / Update / Delete) API 설계 및 구현
- 마크다운 본문에서 정규식 4종으로 이미지·첨부파일 URL 파싱 및 분기 저장
- 게시글 수정 시 본문 기준 양방향 파일 동기화 (S3 ↔ DB)
- S3 삭제 → DB 삭제 순서 강제로 고아 파일 발생 원천 차단
- 게시글 수정마다 Presigned URL 자동 재발급 처리
- DRF Serializer 단계 유효성 검증 및 에러 메시지 커스터마이징

---

## ⚙️ 기술 스택

| 분류 | 기술 |
|------|------|
| Backend | Python 3.13, Django 5.2.8, DRF |
| Database | PostgreSQL |
| Cache | Redis |
| Storage | AWS S3 (Presigned URL) |
| Infra | Docker |
| API 문서 | drf-spectacular |

---

## 🗂️ 시스템 아키텍처

```
Client (HTTP Request)
        ↓
Django DRF (Views)          ← 요청 수신 · 권한 · 응답 반환
        ↓
  Serializers               ← 데이터 변환 · 유효성 검증
        ↓
  Service Layer             ← 비즈니스 로직 (파일 파싱 · 동기화)
      ↙   ↘
 AWS S3   PostgreSQL        ← 파일 스토리지 / 영구 저장소
```

> View는 요청/응답 처리만 담당하고, 파일 파싱·동기화 등 비즈니스 로직은 Service Layer로 분리했습니다.

---

## 🔗 관련 링크

- [팀 프로젝트 레포](https://github.com/OZ-Coding-School/oz_externship_be_07/tree/develop/apps/community)

### 📌 게시글 CUD API
- [게시글 생성](https://github.com/OZ-Coding-School/oz_externship_be_07/blob/develop/apps/community/views/post_list_view.py#L146)
- [게시글 수정 / 삭제](https://github.com/OZ-Coding-School/oz_externship_be_07/blob/develop/apps/community/views/post_detail_view.py#L116)

### 📌 핵심 비즈니스 로직 (Service Layer)
- [게시글 Service](https://github.com/OZ-Coding-School/oz_externship_be_07/blob/develop/apps/community/services/post_service.py#L127)

---

## 💥 핵심 문제 해결

### 1. S3 파일과 DB 데이터 불일치 문제

#### ❗ 문제
게시글 수정 시 본문에서 이미지를 삭제해도 S3·DB에 파일이 그대로 남아  
**스토리지 누수 및 DB 불일치 발생**

#### 🤔 판단 근거
파일 상태를 추적하는 별도 로직 없이 수정할 경우, 삭제된 파일이 S3에 계속 쌓여  
장기적으로 스토리지 비용 증가 및 데이터 불일치 문제로 이어질 것으로 판단

#### ✅ 해결
수정 요청마다 본문 파싱 결과와 DB 파일 목록을 양방향 비교해 자동 동기화

```python
# 수정 시 본문 기준 양방향 동기화
body_files = parse_from_content(new_body)
db_files   = PostFile.objects.filter(post=post)

to_add     = body_files - db_files   # 본문에 있지만 DB에 없음 → 자동 등록
to_delete  = db_files   - body_files # DB에 있지만 본문에서 제거됨 → S3 삭제 → DB 삭제
```

#### 🎯 결과
- S3와 DB 상태를 항상 일치하게 유지
- 수동 정리 작업 없이 스토리지 자동 정합성 확보

---

### 2. Presigned URL 만료로 인한 파일 접근 불가 문제

#### ❗ 문제
S3 Presigned URL은 유효 기간이 있어, 시간이 지나면  
이미지·첨부파일 링크가 **만료되어 프론트에서 404 발생**

#### 🤔 판단 근거
URL을 최초 발급 시점에만 저장하면 시간이 지날수록 접근 불가 케이스가 증가.  
수정 요청이 들어오는 시점은 사용자가 파일에 접근하는 시점이기도 하므로,  
**수정 요청마다 재발급**하면 항상 유효한 URL을 보장할 수 있다고 판단

#### ✅ 해결
게시글 수정 요청 시마다 기존 Presigned URL을 폐기하고 새 URL을 자동 발급

```
수정 요청 → 기존 URL 만료 확인 → S3 재발급 → 새 URL을 응답에 포함
```

#### 🎯 결과
- 응답에 포함되는 이미지·첨부파일 URL 유효성 항상 보장
- 사용자 경험(UX) 안정성 확보

---

### 3. 파일 삭제 순서로 인한 고아 파일 발생 문제

#### ❗ 문제
DB 레코드를 먼저 삭제하면 S3 삭제 실패 시  
**참조 레코드가 없는 고아 파일(orphan file)이 S3에 영구 잔류**

#### 🤔 판단 근거
DB 레코드가 살아있으면 나중에 재시도하거나 추적할 수 있는 근거가 남지만,  
DB가 먼저 사라지면 S3 파일의 존재 자체를 알 방법이 없어짐.  
따라서 **S3 삭제를 먼저 수행**하고 성공 후 DB를 삭제하는 순서가 더 안전하다고 판단

#### ✅ 해결
S3 삭제 → DB 삭제 순서를 파일 동기화·게시글 삭제 양쪽에 일관되게 강제 적용

```
① S3 파일 삭제 (실제 스토리지 정리)
        ↓
② DB 레코드 삭제 (참조 무결성 보장)
        ↓
③ 리소스 누수 없는 정합성 확보
```

#### 🎯 결과
- 고아 파일 발생 원천 차단
- 파일 누수 완전 방지 및 데이터 정합성 보장

---

## 🧠 기술적 포인트

### ✔ 정규식 4종 역할별 분리 설계

마크다운 본문에서 파일을 처리하는 과정을 **추출 → 정제 → 분기** 3단계로 나누고,  
단계마다 별도 정규식을 할당해 각 패턴의 책임을 명확히 구분했습니다.  
하나의 범용 정규식으로 처리하면 파일 형식 추가 시 패턴 전체를 수정해야 하므로,  
역할을 분리해 **변경 시 영향 범위를 최소화**했습니다.

| 정규식 | 역할 |
|--------|------|
| `RE_MARKDOWN_LINK` | 마크다운 링크·이미지 전체 추출 (`![alt](url)`, `[text](url)`) |
| `RE_FILE_URL_STRIP_QS` | Presigned URL의 쿼리스트링 제거 → 순수 경로 추출 |
| `RE_IMAGE_URL` | 이미지 확장자 필터 (`.jpg / .png / .gif / .webp`) |
| `RE_ATTACHMENT_URL` | 첨부파일 확장자 필터 (`.pdf / .zip / .docx`) |

파싱 결과는 **PostImage 테이블**(이미지)과 **PostAttachment 테이블**(첨부파일)에 분리 저장됩니다.

---

### ✔ Presigned URL Lifecycle 관리
- 만료 문제를 수정 요청 트리거 방식으로 구조적으로 해결
- 클라이언트 측 URL 갱신 로직 불필요 → 프론트 의존성 최소화

### ✔ 서비스 레이어 분리
- View: 요청 수신 · 권한 확인 · 응답 반환만 담당
- Service: 파일 파싱 · S3 동기화 · 삭제 순서 보장 등 비즈니스 로직 전담

---

## 🔥 트러블슈팅

### 1. DRF Validation 우선순위 문제

#### ❗ 문제
`custom validate`를 작성했지만 DRF 기본 `blank / null` 검증이 먼저 실행되어  
의도한 커스텀 에러 메시지가 출력되지 않음

#### 🔍 원인
DRF는 **필드 레벨 validation이 `validate_<field>` 보다 먼저 실행**되는 구조.  
기본 `blank` 에러가 우선순위를 가져 커스텀 메시지가 무시됨

#### ✅ 해결
`extra_kwargs`와 `error_messages`로 필드 단위 메시지를 직접 override

```python
extra_kwargs = {
    "title": {
        "error_messages": {"blank": "제목은 필수 값입니다."}
    }
}
```

#### 🎯 결과
- 사용자 친화적인 에러 메시지 제공
- DRF validation 실행 흐름 이해 및 API 응답 품질 개선

---

### 2. Git rebase 충돌 및 히스토리 꼬임

#### ❗ 문제
PR 정리 중 `rebase -i` 기준 커밋 설정 오류로 충돌 발생.  
커밋 메시지와 실제 변경 내용 불일치, `push` 시 `non-fast-forward` 에러

#### ✅ 해결
```bash
git branch backup/temp           # 브랜치 백업 먼저
git rebase -i <base-commit>      # 기준 커밋 재설정
git add <file>
git rebase --continue
git push --force-with-lease origin <branch>  # 안전한 강제 push
```

#### 🎯 결과
- PR 범위 외 변경 제거 완료
- 커밋 메시지와 코드 내용 정합성 회복 → 코드 리뷰 품질 향상

---

### 3. 마크다운 형식에 따른 파일 인식 오류

#### ❗ 문제
정규식이 표준 마크다운 형식만 커버하도록 설계되어  
변형된 URL 표기 방식이나 줄바꿈이 포함된 케이스에서 파일 누락 발생

#### ✅ 해결
발견된 케이스에 한해 정규식 패턴 보완.  
근본 해결을 위해 `mistune` 등 AST 기반 파싱 라이브러리 도입 검토

#### 🎯 결과
- 주요 케이스 해결 완료
- 추가 엣지케이스 대응 방향 확보 (개선 방향 참고)

---

## 😅 아쉬운 점 및 개선 방향

### ❗ 대량 파일 삭제 시 성능 저하 가능성

게시글에 포함된 파일 수가 많을수록 S3 삭제 요청이 동기적으로 증가해  
응답 지연 발생 가능. 이 부분을 프로젝트 중 충분히 고려하지 못했습니다.

👉 **개선 방향**: Celery 비동기 태스크로 S3 삭제를 메인 요청과 분리해 처리

---

### ❗ 정규식 기반 파싱의 엣지케이스 미완

정규식은 표준 마크다운 외 변형 케이스에서 파일 누락 가능성이 존재합니다.

👉 **개선 방향**: `mistune`, `markdown-it` 등 파싱 라이브러리로 AST 기반 파싱 전환

---

### ❗ 초기 설계 충분히 고민하지 못함

기능 구현에 집중하다 보니 View · Service · Serializer 역할 분리가  
초반에 명확하지 않아 중간에 구조 수정이 반복됐습니다.

👉 **개선 방향**: 인터페이스와 계층 구조를 먼저 정의하는 설계 우선 접근 필요

---

## 📌 회고

단순히 동작하는 기능 구현을 넘어, **데이터 정합성과 유지보수성을 고려한 설계의 중요성**을 경험한 프로젝트였습니다.

특히 파일 처리 과정에서 S3와 DB 간 상태 일치, URL 만료 문제, 삭제 순서 보장이라는  
실제 서비스에서 발생할 수 있는 문제를 직접 설계하고 해결하면서,  
**"동작하는 코드"와 "좋은 코드"는 다르다**는 것을 체감했습니다.

멘토 코드 리뷰를 통해 서비스 레이어 분리, 네이밍 직관성, 계층별 책임 분리의  
중요성을 배웠고, 이 경험을 앞으로의 개발에 기준점으로 삼겠습니다.

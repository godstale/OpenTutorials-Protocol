# Open Tutorials Course Bundler Protocol Changelog

이 문서는 Open Tutorials 강좌 번들러 프로토콜(Course Bundler Protocol) 규격의 버전별 변경 이력을 기록합니다. 외부 강좌 제작 도구 및 에이전트 빌더 개발 시 이 변경 이력을 참조하여 빌드 정합성을 맞출 수 있습니다.

---

## [v1.3.1] - 2026-07-12

### Added
- **4.1.2절 "제3자 리소스 고지(Third-Party Resource Notices)" 신설**: 다른 원저작물(책, 영상, 이미지, 오픈소스 코드 등)을 각색·인용하여 강좌를 제작한 경우, 리소스별 출처·라이선스·각색 범위·주의사항을 `license_file`에 고지할 수 있도록 규정하였습니다. Open Tutorials 앱은 이 내용을 강좌 상세/정보 화면에서 학습자에게 노출합니다.

### Changed
- **`license_file` 필드 사용 조건 완화**: 기존에는 `license`가 `"custom"`일 때만 사용 가능했으나, `"custom"`이 아닌 사전 정의 라이선스를 쓰는 경우에도 제3자 리소스 고지 목적으로 선택적으로 첨부할 수 있도록 변경하였습니다 (여전히 `"custom"`일 때는 Mandatory).

---

## [v1.3.0] - 2026-07-12

### Added
- **`package-manifest.json` 내 `license` 속성 추가 (Optional)**:
  - 타입: `String` (기본값: `"all-rights-reserved"`)
  - 설명: 강좌 콘텐츠에 적용되는 라이선스를 명시하는 필드를 추가하였습니다. 사전 정의된 값(`CC-BY-4.0`, `CC-BY-SA-4.0`, `CC-BY-NC-4.0`, `CC-BY-NC-SA-4.0`, `CC-BY-ND-4.0`, `CC-BY-NC-ND-4.0`, `CC0-1.0`, `all-rights-reserved`) 중 하나를 사용하거나, 직접 작성한 라이선스를 사용하려면 `"custom"`으로 지정합니다.
- **`package-manifest.json` 내 `license_file` 속성 추가 (Conditional)**:
  - 타입: `String`
  - 설명: `license`가 `"custom"`일 때 필수로 지정해야 하는 필드로, ZIP 루트에 포함된 커스텀 라이선스 전문 파일명을 가리킵니다 (예: `"LICENSE.md"`).
- **디렉토리 구조에 `LICENSE.md` 선택 항목 추가**: `license`가 `"custom"`인 번들은 ZIP 루트에 라이선스 전문 파일을 포함해야 합니다.
- **제6조 검증 규칙에 라이선스 검증 규칙(5번 항목) 추가**: `license` 값의 허용 범위 및 `custom` 지정 시 `license_file` 필수 여부를 명시하였습니다.

---

## [v1.2.1] - 2026-07-12

### Added
- **`package-manifest.json` 내 `language` 속성 추가 (Optional)**:
  - 타입: `String` (기본값: `"ko"`, 지원값: `"ko"`, `"en"`)
  - 설명: 강좌 패키지의 주 표시 언어를 정의하는 메타데이터를 추가하였습니다. 추후 플랫폼 내에서 다국어 강좌 필터링 등에 활용될 예정입니다.

---

## [v1.2.0] - 2026-07-12

### Changed
- **`package-manifest.json` 내 `target_age` 속성 규격화**:
  - 기존 자유 형식 문자열("10대 이상", "초등학생" 등) 대신 기계/프로그램 처리가 수월하도록, `all` (전연령), `x+` (x세 이상, 예: `10+`), `min-max` (연령대 범위, 예: `8-13`) 형식으로만 기재하도록 프로토콜 스펙을 엄격화하였습니다.

---

## [v1.1.1] - 2026-07-11

### Added
- **`package-manifest.json` 메타데이터 내 `author` 필드 추가 (Mandatory)**:
  - 타입: `Object` (하위 필드: `nickname` 필수, `email` 선택, `website` 선택)
  - 설명: 강좌 번들이 OpenTutorials-Browser 등에 등록될 때 작성자를 식별하기 위해 작성자의 닉네임, 이메일, 홈페이지 URL 정보를 포함하도록 규정하였습니다.

---

## [v1.1.0] - 2026-07-06

### Added
- **`package-manifest.json` 메타데이터 내 `tags` 필드 도입**:
  - 타입: `Array of String` (예: `["Python", "AI", "Beginner"]`)
  - 필수 여부: **Optional** (선택)
  - 설명: 강좌 패키지의 특징을 구분하고 상세 화면 등에서 뱃지 형태로 노출하기 위한 태그 정보를 지원합니다.
  - 관련 가이드라인 및 JSON 스키마 예시 항목에 `tags` 속성을 보강하였습니다.

---

## [v1.0.1] - 2026-07-06

### Added
- **학습 카드 마크다운 가이드라인 표준화 (제7조)**:
  - 강좌 상세 마크다운 카드(`.md`/`.mdx`) 작성 시 프리미엄 UI 렌더링 호환을 위한 스타일 권장 가이드라인이 추가되었습니다.
  - **헤더 레벨 (H1, H2) 사용**: 챕터 및 주요 주제 구분을 위한 구조화 규칙 정의
  - **테이블 표 (GFM Tables)**: 비교나 매핑 정보를 표현할 때 파이프 기호(`|`)를 사용한 테이블 표 작성 규칙 추가
  - **코드 블록 및 구문 강조 (Code Blocks)**: 프로그래밍 코드 기입 시 코드 블록 기호와 언어 식별자 명시 규칙 추가

---

## [v1.0.0] - 2026-07-05

### Added
- **통합 강좌 및 패키지 메타데이터 프로토콜 최초 배포**:
  - `package-manifest.json` 기반의 단일 패키지 메타데이터 구조화
  - `config.json`을 통한 목차(TOC) 트리 구성 및 카드 목록 매핑
  - `wiki.md`를 이용한 강좌 지식베이스 파일 강제화
  - 동영상 카드(`type: "video"`, 자막 포함) JSON 스펙 정의

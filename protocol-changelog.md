# Open Tutorials Course Bundler Protocol Changelog

이 문서는 Open Tutorials 강좌 번들러 프로토콜(Course Bundler Protocol) 규격의 버전별 변경 이력을 기록합니다. 외부 강좌 제작 도구 및 에이전트 빌더 개발 시 이 변경 이력을 참조하여 빌드 정합성을 맞출 수 있습니다.

---

## [v1.4.0] - 2026-08-02

### Breaking Changes
- **마크다운(`.md`)/동영상(`.json`, `type:"video"`)/애니메이션·인터랙션(`.json`, `type:"animation"`) 3종 카드 구분을 폐지하고 단일 "강좌 카드"로 통합했습니다.** 모든 학습 카드는 `cards/*.md` 파일 하나이며, `cards/` 아래 `.json` 카드는 더 이상 허용되지 않습니다(제6조 2번 항목).
- **카드 파일에 YAML 유사 frontmatter(`title`/`theme`/`background`/`foreground`)를 추가했습니다** (제4.3.1절). `config.json`에도 강좌 전체 기본값인 `card_style` 객체가 신설되었습니다(제4.2.1절).
- **동영상·모션(옛 애니메이션)·Lottie는 독립 카드가 아니라 마크다운 본문 임베드 블록으로 재정의되었습니다**: 펜스 코드 블록의 언어 식별자를 `vivo:video`/`vivo:motion`/`vivo:lottie`로 지정하고 그 안에 JSON 페이로드를 작성합니다(제4.3.2~4.3.6절). `vivo:motion`은 기존 v1.3.0 이하 애니메이션 카드의 단일 슬라이드 구조(`canvas`/`elements`/`steps`/`interactions`)를 그대로 재사용하되, 멀티 슬라이드 대신 임베드를 여러 번 배치하는 모델로 전환했습니다.
- **Lottie 애니메이션을 신규 임베드 타입으로 추가했습니다**(`vivo:lottie`, 제4.3.5절): 인라인 데이터는 금지되고 `assets/lottie/`의 외부 JSON 파일만 참조 가능하며, expressions(임의 JS)가 포함된 파일은 검증 오류로 거부됩니다.
- **디렉토리 구조에 `assets/motion/`·`assets/lottie/` 폴더가 신설되었습니다**(제3장). `vivo:motion`/`vivo:lottie`가 외부 파일을 참조할 때 사용합니다.
- 기존 v1.3.0 이하 카드 스키마(동영상/애니메이션 `.json` 카드)는 삭제되지 않고 **부록 A "Legacy v1.3.0 이하 카드 포맷"으로 강등**되었습니다. `bundler_protocol_version`이 `1.3.0` 이하인 기존 배포 강좌는 계속 재생 가능하나, 신규 제작에는 사용할 수 없습니다.

### Migration Notes
- 기배포 v1.3.0 이하 강좌는 변환 없이 그대로 유지·재생됩니다(하위 호환). 신규 강좌 제작 시에만 v1.4.0 스키마를 사용합니다.
- 기존 v1.3.0 이하 `type:"video"` 카드는 "제목 + `vivo:video` 임베드 하나"짜리 `.md` 파일로, `type:"animation"` 카드는 "제목 + `vivo:motion` 임베드 하나"짜리 `.md` 파일로 기계적으로 변환 가능합니다(자동 변환기는 별도 과제).
- 제6조 검증 규칙에 frontmatter 검증(4번)과 임베드 블록 검증(5번)이 신설되었고, 기존 애니메이션 카드 검증 로직(`kind`/`effect`/`trigger`/`event` allowlist, id 참조 무결성, `images/` 실존 검사, 스크립트 삽입 금지)은 `vivo:motion` 임베드 검증(5번)과 레거시 애니메이션 카드 검증(7번, 부록 A)에서 공통으로 재사용됩니다.

### Changed
- 문서 최상단 프로토콜 버전을 `1.3.0` → `1.4.0`으로 갱신하였습니다.

---

## [v1.3.0] - 2026-08-01

### Added
- **애니메이션 카드 `kind: "text"` 요소에 자동맞춤(Auto-Fit) 스타일 필드 신설 (제4.4.3.2절)**: `style.wrap`(자동 줄바꿈, 기본 `true`), `style.min_font_size`(자동 축소 하한, 기본 `12`), `style.overflow`(`"shrink"`/`"clip"`/`"visible"`, 기본 `"shrink"`), `style.vertical_align`(`"top"`/`"middle"`/`"bottom"`, 기본 `"middle"`), `style.line_height`(행간 배수, 기본 `1.3`)를 추가하였습니다. 이 프로토콜을 구현하는 모든 재생 엔진은 텍스트가 지정된 박스(`width`/`height`)를 벗어나지 않도록 자동 줄바꿈·폰트 축소·수직 정렬을 수행할 정합성 의무를 갖습니다. (배경: AI 에이전트가 생성한 애니메이션 카드에서 박스 대비 텍스트가 넘치거나 잘리는 품질 문제가 Vivo Studio 실사용 테스트에서 다수 발견되어, 저작자가 픽셀 단위로 텍스트 크기를 역산해야 하는 취약한 전제를 없애고 렌더러가 자동으로 맞추도록 책임을 이전함)
- **제6조 검증 규칙에 auto-fit 필드 검증 추가**: `style.overflow`/`style.vertical_align` allowlist 위반 검증, `style.min_font_size > style.font_size`인 경우의 거부 규칙을 명시하였습니다.

### Changed
- 문서 최상단 프로토콜 버전을 `1.2.0` → `1.3.0`으로 갱신하였습니다.

---

## [v1.2.0] - 2026-07-30

### Added
- **신규 학습 카드 타입 `type: "animation"` 도입 (제4.4절 신설)**: 텍스트 카드(`.md`), 동영상 카드(`.json`, `type: "video"`)에 이어 세 번째 카드 타입으로 **2D 애니메이션·인터랙션 카드**를 추가하였습니다. 도형(`rect`/`circle`/`ellipse`/`line`/`path`/`polygon`)·이미지·텍스트 요소와, 슬라이드 진입 시 자동/클릭으로 순차 재생되는 애니메이션 시퀀스(`steps`), 클릭 시 즉시 반응하는 독립 인터랙션(`interactions`)을 선언적 JSON으로 표현하는 **자체 "Scene/Slide" DSL**을 정의하였습니다. (배경: `docs/2d-animation-vivo-studio-review.md` — Lottie/Rive는 AI 에이전트가 직접 저작하기 어려운 포맷이라 채택하지 않고, 이 프로젝트가 이미 채택 중인 "카드 = 검증 가능한 JSON 데이터" 패턴을 그대로 확장하는 방향으로 결정)
  - `scene_info.canvas`, `scene_info.slides[]`, `elements[]`(`kind` 허용 목록: `rect`/`circle`/`ellipse`/`line`/`path`/`polygon`/`text`/`image`/`group`), `steps[].actions[]`(`effect` 허용 목록: `fade_in`/`fade_out`/`move_to`/`scale_to`/`rotate_to`/`show`/`hide`/`highlight`/`draw_path`), `interactions[]` 스키마를 신설하였습니다.
  - `kind: "image"` 요소는 마크다운 카드와 동일하게 ZIP 루트의 `images/` 폴더를 참조합니다(제3장 디렉토리 구조 다이어그램에 `03_animation.json` 예시 추가).
- **제6조 검증 규칙에 애니메이션 카드 스키마 검증(6번 항목) 추가**: `scene_info` 필수 필드 존재 여부, 요소/액션 id 참조 무결성, `kind`/`effect`/`trigger`/`event` allowlist 위반 검증, `images/` 실존 검증에 더해 **보안 검증**(`<script`, 인라인 이벤트 핸들러, `javascript:` 패턴 등 실행 가능 코드 삽입 금지)을 명시하였습니다. 이는 "카드는 항상 데이터, 재생 코드는 항상 앱 소유"라는 기존 보안 원칙을 신규 카드 타입에도 동일하게 적용하기 위함입니다.

### Changed
- 문서 최상단 프로토콜 버전을 `1.1.5` → `1.2.0`으로 갱신하였습니다.

---

## [v1.1.5] - 2026-07-12

### Added
- **4.1.2절 "제3자 리소스 고지(Third-Party Resource Notices)" 신설**: 다른 원저작물(책, 영상, 이미지, 오픈소스 코드 등)을 각색·인용하여 강좌를 제작한 경우, 리소스별 출처·라이선스·각색 범위·주의사항을 `license_file`에 고지할 수 있도록 규정하였습니다. Open Tutorials 앱은 이 내용을 강좌 상세/정보 화면에서 학습자에게 노출합니다.

### Changed
- **`license_file` 필드 사용 조건 완화**: 기존에는 `license`가 `"custom"`일 때만 사용 가능했으나, `"custom"`이 아닌 사전 정의 라이선스를 쓰는 경우에도 제3자 리소스 고지 목적으로 선택적으로 첨부할 수 있도록 변경하였습니다 (여전히 `"custom"`일 때는 Mandatory).

---

## [v1.1.4] - 2026-07-12

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

## [v1.1.3] - 2026-07-12

### Added
- **`package-manifest.json` 내 `language` 속성 추가 (Optional)**:
  - 타입: `String` (기본값: `"ko"`, 지원값: `"ko"`, `"en"`)
  - 설명: 강좌 패키지의 주 표시 언어를 정의하는 메타데이터를 추가하였습니다. 추후 플랫폼 내에서 다국어 강좌 필터링 등에 활용될 예정입니다.

---

## [v1.1.2] - 2026-07-12

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

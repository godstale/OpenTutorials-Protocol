# Open Tutorials Course Bundler Protocol Specification

**Version:** 1.6.0
**Status:** Active
**Scope:** Open Tutorials Course Packaging & local execution

---

## 1. 개요 (Introduction)

본 문서는 Open Tutorials 플랫폼에 강좌(Course Package)를 등록하고 배포하기 위한 통합 번들 ZIP 파일 구조 및 각 항목의 역할, 정합성 검증 규칙을 규정합니다.

이 프로토콜은 온디바이스 로컬 실행 환경(`db.json` 및 `public/courses`)과의 완벽한 호환을 보장하며, AI Agent 기반의 자동 강좌 제작 프로젝트에서 참조하여 강좌 파일을 무결성 있게 제작하기 위한 공식 가이드라인입니다.

> **v1.6.0 변경 요지:** 하나의 강좌가 여러 언어 버전(한국어 기본 + 선택적 영어/베트남어/인도네시아어)을 갖고 Vivo Academy에서 언어를 전환하며 볼 수 있도록 다국어 지원(Multilingual Course Support)을 추가했습니다. 언어별 콘텐츠는 `locales/<lang>/` 오버레이 폴더에 "달라지는 파일만" 배치하는 방식이며, 기존 v1.5.0 이하 단일언어 번들과 완벽히 하위 호환됩니다.
>
> **v1.5.0 변경 요지:** 3D 모델 뷰어/플레이어 임베드(`vivo:3d`, GLB 모델 및 변환 애니메이션)와 코드 실습/실행 환경 임베드(`vivo:runtime`, WASM 4종 및 로컬 네이티브 툴체인)를 신규 임베드 타입으로 추가했습니다. 기존 v1.4.0 번들과 완벽히 하위 호환됩니다.
>
> **v1.4.0 변경 요지 (통합 강좌 카드):** v1.3.0 이하까지 존재했던 마크다운(`.md`)/동영상(`.json`, `type:"video"`)/애니메이션·인터랙션(`.json`, `type:"animation"`) 3종 카드 구분을 폐지하고, 모든 학습 카드를 **단일 "강좌 카드"**(`cards/*.md` + YAML frontmatter + 임베드 블록)로 통합했습니다. 동영상·모션(애니메이션)·Lottie는 더 이상 독립된 카드 파일이 아니라, 마크다운 본문 흐름 중 원하는 위치에 삽입하는 **임베드 컴포넌트**입니다. 자세한 사항은 4장을, v1.3.0 이하 레거시 스키마는 부록 A를 참조하십시오.

---

## 2. 강좌 등록 및 처리 프로세스 (Registration Workflow)

Open Tutorials 플랫폼에서 강좌 번들이 처리되고 데이터베이스에 등록되는 단계는 다음과 같습니다.

1. **메타데이터 및 콘텐츠 기획**:
   - 강좌 카테고리, 대상 연령대, 학습 순서 제어 방식 등을 결정합니다.
   - 메인 지식베이스가 될 `wiki.md`와 개별 학습 카드(`cards/*.md`)를 설계합니다. 각 카드는 마크다운 본문이며, 필요한 경우 동영상(`vivo:video`)·모션 인터랙션(`vivo:motion`)·Lottie 애니메이션(`vivo:lottie`)·3D 모델(`vivo:3d`)·코드 실습 런타임(`vivo:runtime`) 임베드를 본문 중간에 삽입할 수 있습니다.
2. **패키지 매니페스트(`package-manifest.json`) 작성**:
   - 필수 프로토콜 항목(`bundler_protocol_version`, `target_age`, `category`)을 설정하고 제목 및 기본 정보를 정의합니다. (더 이상 개별 하위 강좌 리스트 `courses`는 지정하지 않습니다.)
3. **패키지 구성 및 config.json 작성**:
   - 루트 경로에 `config.json`, `wiki.md`, `cards/` 디렉토리를 생성합니다.
   - `config.json` 내부에 학습 카드 파일 목록과 목차(TOC) 트리 구조, 그리고 선택적으로 강좌 전체 기본 카드 스타일(`card_style`)을 정의합니다.
4. **단일 강좌 패키지 ZIP 압축**:
   - 루트에 `package-manifest.json`, `config.json`, `wiki.md`, `thumbnail.png`(선택), `cards/`, `images/`(선택), `assets/`(선택, `motion`/`lottie`/`models` 임베드가 외부 파일을 참조하는 경우)를 두고 최종 하나의 ZIP 파일로 압축합니다.
5. **대시보드 업로드 및 정합성 검증**:
   - 사용자가 플랫폼 내 업로드 UI를 통해 ZIP 파일을 등록하면 브라우저단에서 JSZip을 이용하여 필수 파일 구성, JSON 문법 포맷, TOC와 파일명의 1:1 매칭 등을 사전 검증합니다.
6. **로컬 데이터베이스 반영**:
   - 검증이 완료된 데이터는 서버 API(`/api/admin/packages/upload`)로 전송되어 `db.json`에 `course_packages` 레코드로 영구 저장되며(TOC 및 cards 데이터도 메타데이터 컬럼으로 통합), 정적 리소스는 `public/courses/[slug]/` 하위에 직접 압축 해제되어 저장됩니다.

---

## 3. 강좌 패키지 ZIP 파일 구조 (Directory Structure)

강좌 패키지 ZIP 파일의 루트와 하위 경로는 반드시 아래와 같은 구조를 준수해야 합니다. 압축 해제 시 최상위 루트에 공백 폴더가 없어야 하며, 즉시 아래 항목들이 존재해야 합니다.

```text
[강좌 패키지 ZIP 파일]
├── package-manifest.json           # 통합 강좌 및 패키지 메타데이터 (필수)
├── config.json                     # 목차 및 카드 목록 스키마 설정 (필수)
├── wiki.md                         # 강좌 지식베이스 마크다운 파일 (필수)
├── thumbnail.png                   # 대표 썸네일 이미지 (선택, package-manifest.json에 매핑 가능)
├── LICENSE.md                      # 라이선스 전문 및/또는 제3자 리소스 고지 (선택, license가 "custom"일 때 필수)
├── cards/                          # 강의 카드 디렉토리 (필수, .md/.mdx 파일만 허용 — .json 카드는 v1.4.0부터 금지)
│   ├── 01_intro.md                 # 강좌 카드 (frontmatter + 본문, 동영상/모션/Lottie/3D/런타임 임베드 포함 가능)
│   └── 02_concept.md
├── images/                         # 강좌 카드 및 vivo:motion 임베드(image 요소)에서 참조하는 비트맵/SVG 이미지 폴더 (선택)
│   └── schema.png
├── assets/                         # vivo:motion·vivo:lottie·vivo:3d 임베드가 외부 파일로 참조하는 리소스 폴더 (선택)
│   ├── motion/                     # vivo:motion 임베드가 { "src": "assets/motion/xxx.json" }로 참조하는 Scene DSL JSON
│   │   └── 01_variables.json
│   ├── lottie/                     # vivo:lottie 임베드가 { "src": "assets/lottie/xxx.json" }로 참조하는 Lottie 애니메이션 JSON
│   │   └── intro.json
│   └── models/                     # vivo:3d 임베드가 { "src": "assets/models/xxx.glb" }로 참조하는 GLB 3D 모델
│       └── robot_arm.glb
└── locales/                        # ★ 신설 (선택, v1.6.0) — 언어별 오버레이 폴더
    ├── en/
    │   ├── translation-status.json # ★ 신설 (필수, locales/<lang>/ 존재 시) — 번역 상태 메타데이터
    │   ├── wiki.md                 # 선택: 있으면 en 위키로 사용, 없으면 루트 wiki.md 폴백
    │   ├── config.json             # 선택: 있으면 en TOC(제목/설명 번역판)로 사용, 없으면 루트 config.json 폴백
    │   ├── cards/                  # 선택 오버라이드 (번역이 끝난 카드만 존재하면 됨)
    │   │   ├── 01_intro.md
    │   │   └── 02_concept.md
    │   ├── images/
    │   │   └── schema.png          # 선택: 다이어그램에 원어(한국어) 라벨이 박혀있는 경우에만 교체
    │   └── assets/
    │       └── motion/01_variables.json  # 선택: 텍스트가 포함된 모션 소스만 교체
    ├── vi/                         # ... (동일 구조, 전부 선택)
    └── id/                         # ... (동일 구조, 전부 선택)
```

> **레거시(v1.3.0 이하) 구조:** `bundler_protocol_version`이 `1.3.0` 이하인 번들은 `cards/` 아래 `.json`(`type:"video"`/`type:"animation"`) 카드를 계속 포함할 수 있습니다. 부록 A를 참조하십시오. 이 폴더 구조는 v1.4.0 이상(신규 제작)에 적용됩니다.
>
> **다국어(v1.6.0) 오버레이 구조:** `locales/` 폴더는 선택 항목이며, `bundler_protocol_version`이 `1.6.0` 미만이거나 `locales/`가 없는 번들은 기존과 동일한 단일 언어 강좌로 취급됩니다. 자세한 스키마는 4.4절을 참조하십시오.

---

## 4. 메타데이터 스키마 및 속성 명세 (Schema Specification)

### 4.1 package-manifest.json (패키지 메타데이터)

| 필드명 | 타입 | 필수 여부 | 설명 | 예시 |
| :--- | :--- | :---: | :--- | :--- |
| `title` | String | **Mandatory** | 강좌 패키지의 공식 명칭 | `"마케팅 에이전트 마스터"` |
| `slug` | String | Optional | 플랫폼 URL 경로로 활용될 고유 키 (누락 시 Title 기반 자동변환) | `"marketing-integrated-course"` |
| `description` | String | Optional | 강좌 패키지에 대한 종합 설명 | `"초급부터 고급까지 다루는 종합 마케팅 과정"` |
| `author` | Object | **Mandatory** | 강좌 작성자 정보 (닉네임, 이메일, 홈페이지 등) | 아래 작성자 정보 스키마 참조 |
| `thumbnail` | String | Optional | 대표 이미지 경로 | `"./thumbnail.png"` |
| `published` | Boolean | Optional | 즉시 공개 여부 (기본값: `true`) | `true` |
| `sequential_play` | Boolean | Optional | 카드들의 순차 수강 강제 여부 (기본값: `false`) | `false` |
| `force_checkpoint` | Boolean | Optional | 특정 체크포인트를 지나야만 다음 단계 활성화 (기본값: `false`) | `false` |
| `version` | String | Optional | 강좌 패키지 자체의 배포 버전 (기본값: `"1.0.0"`) | `"1.2.0"` |
| `changelog` | String | Optional | 버전별 주요 변경 사항 (기본값: `"최초 릴리즈"`) | `"TOC 구조 최적화 및 3장 실습 추가"` |
| `bundler_protocol_version` | String | **Mandatory** | 이 번들이 준수한 번들러 프로토콜 명세 버전. 버전이 `1.4.0` 이상이면 4장의 단일 강좌 카드 스키마가, `1.3.0` 이하이면 부록 A의 레거시 스키마가 적용됩니다 | `"1.5.0"` |
| `target_age` | String | **Mandatory** | 강좌 수강에 권장되는 대상 연령대 (`all` (전연령), `x+` (x세 이상), `min-max` (연령대 범위)) | `"all"`, `"10+"`, `"8-13"` |
| `category` | String | **Mandatory** | 강좌의 대분류 카테고리 | `"Programming"`, `"Design"`, `"Marketing"`, `"Math"` |
| `language` | String | Optional | 강좌 패키지의 주 언어 (기본값: `"ko"`). (v1.6.0부터 `base_language`로 대체되었으나 하위 호환을 위해 계속 지원되는 별칭입니다. 신규 제작 시 `base_language` 사용을 권장합니다.) | `"ko"`, `"en"` |
| `base_language` | String | Optional | 강좌의 기본(원본) 언어 (기본값: `"ko"`). 기존 `language` 필드를 대체하며, 둘 다 지정 시 동일 값이어야 함(다르면 검증 경고) | `"ko"` |
| `supported_languages` | Array of String | Optional | 이 강좌가 제공하는 언어 코드 목록 (기본값: `[base_language]`). `"ko"`/`"en"`/`"vi"`/`"id"` 중에서 선택하며 `base_language`를 반드시 포함해야 함 | `["ko", "en", "vi"]` |
| `tags` | Array of String | Optional | 강좌의 성격을 나타내는 태그 목록 | `["아두이노", "IoT", "하드웨어"]` |
| `license` | String | Optional | 강좌 콘텐츠에 적용되는 라이선스. 4.1.1절의 사전 정의 값 중 하나를 사용하거나, 직접 작성한 라이선스는 `"custom"`으로 지정 (기본값: `"all-rights-reserved"`) | `"CC-BY-4.0"`, `"all-rights-reserved"`, `"custom"` |
| `license_file` | String | Conditional | `license`가 `"custom"`일 때 **Mandatory**. 그 외의 경우에도 4.1.2절의 제3자 리소스 고지가 필요하면 선택적으로 사용 가능. ZIP 루트에 포함된 라이선스/고지 파일명 | `"LICENSE.md"` |

#### 4.1.1 라이선스(license) 사전 정의 목록

강좌 제작자는 아래 사전 정의된 라이선스 중 하나를 `license` 필드에 지정할 수 있습니다. 크리에이티브 커먼즈(CC) 라이선스는 교육 콘텐츠 재사용 조건을 표준화된 방식으로 명시할 수 있어 강좌 콘텐츠에 권장되는 라이선스 체계입니다.

| 값 | 명칭 | 설명 |
| :--- | :--- | :--- |
| `CC-BY-4.0` | CC BY 4.0 | 저작자 표시만 하면 상업적 이용, 변경, 재배포 모두 허용 |
| `CC-BY-SA-4.0` | CC BY-SA 4.0 | 저작자 표시 + 동일조건변경허락(2차 저작물도 동일 라이선스로 배포) |
| `CC-BY-NC-4.0` | CC BY-NC 4.0 | 저작자 표시 + 비영리 목적만 허용 |
| `CC-BY-NC-SA-4.0` | CC BY-NC-SA 4.0 | 저작자 표시 + 비영리 + 동일조건변경허락 |
| `CC-BY-ND-4.0` | CC BY-ND 4.0 | 저작자 표시 + 변경 금지(2차 저작물 제작 불가), 원본 그대로만 재배포 허용 |
| `CC-BY-NC-ND-4.0` | CC BY-NC-ND 4.0 | 저작자 표시 + 비영리 + 변경 금지 (가장 제한적인 CC 라이선스) |
| `CC0-1.0` | CC0 1.0 (퍼블릭 도메인 헌정) | 저작권을 포기하여 누구나 제한 없이 자유롭게 이용 가능 |
| `all-rights-reserved` | 모든 권리 보유 (기본값) | 재배포·변형·공유 등 어떠한 재사용도 허용하지 않는 저작권 있는 콘텐츠. `license` 필드 생략 시 자동 적용되는 기본값 |
| `custom` | 커스텀 라이선스 | 위 사전 정의 라이선스로 표현할 수 없는 자체 작성 라이선스. `license_file`에 지정한 파일(예: `LICENSE.md`)을 ZIP 루트에 반드시 포함해야 함 |

> CC 라이선스 전문은 https://creativecommons.org/licenses/ 에서 확인할 수 있습니다.

#### 4.1.2 제3자 리소스 고지 (Third-Party Resource Notices)

다른 사람의 원저작물(책, 영상, 이미지, 오픈소스 코드 등)을 각색·인용·재구성하여 강좌를 제작한 경우, 그 리소스별 출처와 라이선스, 사용 조건, 주의사항을 학습자에게 별도로 고지해야 할 수 있습니다. 이 고지 내용은 `license_file`에 지정한 파일(권장 파일명: `LICENSE.md`)에 함께 포함시킬 수 있으며, `license` 값이 `"custom"`이 아니어도(예: `CC-BY-NC-4.0` 등 사전 정의 라이선스를 쓰는 경우에도) `license_file`을 선택적으로 첨부하여 아래 내용을 기재할 수 있습니다.

- **원저작물 출처**: 원저작물의 제목, 저작자/제작자, 링크(가능한 경우)
- **적용 라이선스 또는 이용 조건**: 원저작물이 자체적으로 명시한 라이선스 또는 이용 약관 (예: 특정 유튜브 채널 영상은 자체 저작권 하에 있으며 임베드만 허용되는 경우 등)
- **각색/인용 범위**: 강좌 제작 과정에서 원저작물을 어느 정도로 각색·요약·인용했는지 (예: "영상 설명을 재구성한 문어체 해설이며 영상 자체는 원저작자 소유", "책의 특정 챕터 사례를 재구성함")
- **주의사항**: 재배포, 상업적 활용, 2차 수정 등에 관해 학습자가 유의해야 할 제한 사항

Open Tutorials 앱은 강좌 상세/정보 화면에서 `license` 값과 함께 `license_file`이 존재하면 그 내용을 그대로 노출하여, 학습자가 강좌에 포함된 리소스의 출처와 이용 조건을 확인할 수 있도록 합니다. 따라서 `license_file`은 단순 라이선스 전문뿐 아니라 위와 같은 고지 섹션을 포함하는 문서로 작성하는 것을 권장합니다.

#### 작성자 정보(author) 스키마
`author` 객체는 다음과 같은 하위 필드를 포함해야 합니다.

| 필드명 | 타입 | 필수 여부 | 설명 | 예시 |
| :--- | :--- | :---: | :--- | :--- |
| `nickname` | String | **Mandatory** | 작성자의 닉네임 | `"Kailash"` |
| `email` | String | Optional | 작성자의 이메일 주소 | `"godstale@hotmail.com"` |
| `website` | String | Optional | 작성자의 홈페이지 또는 블로그 URL | `"https://hardcopyworld.com"` |

---

### 4.2 패키지 루트의 config.json (목차 및 카드 목록)

패키지 루트에 위치하는 `config.json`은 강좌의 상세 목차(TOC) 트리와 연결되는 카드 파일들의 목록을 기술합니다.

```json
{
  "cards": [
    "01_intro.md",
    "02_concept.md"
  ],
  "toc": [
    {
      "type": "chapter",
      "title": "1장. 마케팅 입문",
      "description": "마케팅의 기초 정의와 시장 구조를 파악합니다.",
      "children": [
        {
          "type": "section",
          "title": "마케팅의 정의",
          "description": "고객 가치를 창출하는 핵심 프로세스 이해",
          "filename": "01_intro.md"
        }
      ]
    }
  ],
  "card_style": {
    "theme": "light",
    "background": "#ffffff",
    "foreground": "#1f2937"
  }
}
```

#### TOC 노드 필드 명세

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `type` | String | **Mandatory** | 노드의 성격 (`chapter`, `section`, `subsection` 중 하나) |
| `title` | String | **Mandatory** | 목차에 렌더링될 실제 한글/외국어 제목 (파일명과 달라야 함). 카드 frontmatter의 `title`보다 이 값이 우선 적용되는 소스 오브 트루스입니다(4.3.1절 참조) |
| `description` | String | **Mandatory** | 해당 단원의 요약 설명 (기본 템플릿 값 방치 금지) |
| `filename` | String | Conditional | 말단 노드(`section` 또는 `subsection`)인 경우 연결될 강좌 카드 파일명. `bundler_protocol_version`이 `1.4.0` 이상이면 `.md`/`.mdx`만 허용됩니다(예: `"01_intro.md"`). `1.3.0` 이하인 레거시 번들은 부록 A의 `.json` 카드도 허용됩니다 |
| `children` | Array | Conditional | 상위 노드(`chapter` 등)인 경우 하위 TOC 노드 배열 |

#### 4.2.1 card_style (강좌 기본 카드 스타일, v1.4.0 신설)

`config.json` 최상위에 선택적으로 `card_style` 객체를 지정하여 강좌 전체 카드의 기본 테마/배색을 정의할 수 있습니다. 개별 카드 파일의 frontmatter(4.3.1절)가 지정한 값은 이 기본값을 오버라이드합니다.

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `theme` | String | Optional | `"light"` 또는 `"dark"` |
| `background` | String | Optional | 카드 배경색, `#rrggbb` 형식의 6자리 hex 색상 |
| `foreground` | String | Optional | 카드 전경(텍스트) 색, `#rrggbb` 형식의 6자리 hex 색상 |

`card_style`을 생략하면 플랫폼 앱의 기본 테마를 그대로 사용합니다.

---

### 4.3 강좌 카드 스펙 (단일 카드, `cards/*.md`) — v1.4.0

모든 학습 카드는 **마크다운 문서 하나**입니다. 과거 동영상 전용이던 카드도 "제목 + 동영상 임베드 한 개"로 구성된 `.md` 문서로 표현합니다. `cards/` 아래 `.json` 카드 파일은 v1.4.0에서 금지됩니다(부록 A 참조).

#### 4.3.1 Frontmatter

카드 파일 최상단에 `---`로 감싼 YAML 유사 flat key-value 블록을 선택적으로 둘 수 있습니다.

```markdown
---
title: "변수와 자료형"
theme: dark
background: "#0f172a"
foreground: "#e2e8f0"
---

# 변수와 자료형

변수는 데이터를 담는 상자입니다...
```

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `title` | String | Optional | 카드 제목 표기용. `config.json`의 `toc[].title`이 소스 오브 트루스이며, 이 필드는 카드 파일 자체를 단독으로 열람할 때를 위한 보조 표기입니다. 둘이 다르면 `toc[].title`이 우선합니다 |
| `theme` | String | Optional | `"light"` 또는 `"dark"`. 지정 시 카드 컨테이너에 해당 다크/라이트 스코프가 적용됩니다 |
| `background` | String | Optional | 카드 배경색, `#rrggbb` 형식 |
| `foreground` | String | Optional | 카드 전경(텍스트) 색, `#rrggbb` 형식 |

세 필드 모두 생략 시 `config.json`의 `card_style`(4.2.1절) 값을, 그것도 없으면 플랫폼 기본 테마를 사용합니다.

#### 4.3.2 임베드 블록 공통 규칙

카드 본문 중 동영상·모션 인터랙션·Lottie 애니메이션·3D 모델·코드 실습 런타임을 삽입하려면, 마크다운 펜스 코드 블록(백틱 3개)의 언어 식별자로 `vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`/`vivo:runtime`을 지정하고 그 안에 페이로드를 작성합니다.

```markdown
계속해서 자료형을 살펴봅시다...

\`\`\`vivo:motion
{ "src": "assets/motion/01_variables.json" }
\`\`\`

이제 실습해 봅시다.
```

- 이 문법을 모르는 일반 마크다운 뷰어(GitHub 등)에서는 그냥 코드 블록으로 표시되어 본문이 깨지지 않습니다. Open Tutorials 재생 앱(및 Vivo Studio 미리보기)만 `vivo:*` 언어 식별자를 감지해 실제 임베드 컴포넌트로 치환합니다. 7.3절(코드 블록 가이드라인)의 일반 구문 강조 코드 블록과는 별개의 처리 경로입니다.
- **선언적 데이터 원칙**: 이 스펙의 모든 임베드 페이로드는 **선언적 데이터**만 표현합니다. `<script>` 태그, 인라인 이벤트 핸들러, `javascript:` URI, 임의의 실행 가능 코드는 절대 포함될 수 없으며, 문자열 필드에서 발견 시 검증 오류로 거부됩니다(6장 참조). 단, `vivo:runtime` 임베드의 코드 본문은 학습자가 화면에서 직접 편집·실행하기 위한 실습 대상 코드로 예외적으로 허용됩니다. 그러나 이 경우에도 컴파일러/인터프리터 실행 명령줄·경로(`command`, `args`, `cwd`, `env`, `interpreter` 등)를 카드가 지정하는 것은 엄격히 금지되며, 실제 실행 명령과 보안 샌드박스는 항상 플랫폼 앱(재생 엔진)의 툴체인 레지스트리가 소유합니다 — "카드 = 데이터, 실행 코드/환경 = 앱 소유" 원칙은 모든 임베드 타입에 일관되게 적용됩니다.
- `vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`/`vivo:runtime` 외의 `vivo:` 접두사 언어 식별자는 정의되지 않은 임베드 타입으로 검증 오류 처리됩니다.

#### 4.3.3 `vivo:video` — 동영상 임베드

```json
{
  "provider": "youtube",
  "video_id": "dQw4w9WgXcQ",
  "start": 0,
  "duration_seconds": 212,
  "subtitles": [
    { "start": 0.0, "end": 5.5, "text": "안녕하세요! 이번 시간에는 리액트의 기본 원리에 대해 알아보겠습니다." },
    { "start": 5.6, "end": 12.0, "text": "먼저 컴포넌트란 무엇인지 정의부터 살펴볼까요?" }
  ]
}
```

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `provider` | String | **Mandatory** | 영상 제공 플랫폼 (현재 `"youtube"` 고정) |
| `video_id` | String | **Mandatory** | 유튜브 영상 고유 ID |
| `start` | Number | Optional | 재생 시작 위치(초 단위). 기본값 `0` |
| `duration_seconds` | Number | Optional | 동영상 전체 길이(초) |
| `subtitles` | Array | Optional | 시간별 자막 정보 배열. 각 원소는 `start`(Number), `end`(Number), `text`(String) |
| `extend` | Boolean | Optional | 카드 레이어 전체 사용 여부. 기본값 `false` |
| `extend_only` | Boolean | Optional | 카드 레이어 전체 사용 고정 여부 (축소 불가). 기본값 `false` |

#### 4.3.4 `vivo:motion` — 모션·인터랙션 임베드

기존 v1.3.0 이하 애니메이션 카드의 **단일 슬라이드 구조**(`canvas` + `elements[]` + `steps[]` + `interactions[]`)를 그대로 재사용합니다. 멀티 슬라이드는 지원하지 않습니다 — 여러 장면이 필요하면 임베드를 본문에 여러 개 배치하고, "슬라이드 넘기기"는 본문 스크롤이 대신합니다.

페이로드는 인라인 JSON 또는 외부 파일 참조 중 하나입니다.

```json
{ "src": "assets/motion/01_variables.json" }
```

또는 인라인:

```json
{
  "canvas": { "width": 960, "height": 540, "background": "#ffffff" },
  "elements": [
    {
      "id": "el-title",
      "kind": "text",
      "x": 60, "y": 40, "width": 600, "height": 80,
      "content": "반응 속도란?",
      "style": { "font_size": 36, "font_weight": "bold", "color": "#111827", "align": "left" },
      "initial": { "opacity": 0 }
    },
    {
      "id": "el-box-a",
      "kind": "rect",
      "x": 100, "y": 200, "width": 120, "height": 120,
      "style": { "fill": "#10b981", "stroke": "#065f46", "stroke_width": 2, "corner_radius": 8 },
      "initial": { "opacity": 0, "scale": 0.8 }
    }
  ],
  "steps": [
    { "trigger": "on_enter", "actions": [
      { "target": "el-title", "effect": "fade_in", "duration": 0.6, "delay": 0, "easing": "ease_out" } ] },
    { "trigger": "on_click", "actions": [
      { "target": "el-box-a", "effect": "fade_in", "duration": 0.5, "easing": "ease_out" },
      { "target": "el-box-a", "effect": "scale_to", "to": 1.0, "duration": 0.5, "easing": "ease_out" } ] }
  ],
  "interactions": [
    { "event": "click", "target": "el-box-a", "action": { "effect": "highlight", "duration": 0.4 } }
  ]
}
```

**최상위 구조**

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :--- | :--- |
| `src` | String | Conditional | 외부 참조 방식일 때 **Mandatory**. `assets/motion/` 아래 실제 존재하는 JSON 파일 상대경로. 지정 시 아래 다른 필드는 무시되고 참조 파일의 내용이 페이로드로 취급됩니다 |
| `canvas` | Object | Conditional | 인라인 방식일 때 **Mandatory**. `width`(Number), `height`(Number), `background`(String, CSS 색상값) |
| `elements` | Array | Conditional | 인라인 방식일 때 **Mandatory**. 도형/이미지/텍스트 요소 목록 |
| `steps` | Array | Optional | 진입 시 순서대로 재생되는 애니메이션 시퀀스(reveal.js의 fragment 리빌 패턴). 기본값: 빈 배열 |
| `interactions` | Array | Optional | `steps` 진행 여부와 무관하게 특정 요소 클릭 시 즉시 반응하는 독립 이벤트 바인딩. 기본값: 빈 배열 |
| `extend` | Boolean | Optional | 카드 레이어 전체 사용 여부. 기본값 `false` |
| `extend_only` | Boolean | Optional | 카드 레이어 전체 사용 고정 여부 (축소 불가). 기본값 `false` |

임베드는 본문 폭 100%를 차지하는 반응형 박스로 렌더링되며, 높이는 `canvas`의 가로세로 비율로 결정됩니다.

##### 4.3.4.1 요소(`elements[]`) 구조

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `id` | String | **Mandatory** | 같은 임베드 내에서 유일한 요소 식별자. `steps`/`interactions`가 이 id로 요소를 참조함 |
| `kind` | String | **Mandatory** | 요소의 종류. 아래 허용 목록 중 하나여야 함: `rect`, `circle`, `ellipse`, `line`, `path`, `polygon`, `text`, `image`, `group` |
| `x`, `y` | Number | **Mandatory** (`group` 제외) | 캔버스 좌표계 기준 위치(좌상단) |
| `width`, `height` | Number | Conditional | `rect`/`image`/`text`/`group`은 필수. `circle`(반지름은 `style.radius`)/`path`/`line`/`polygon`은 불필요 |
| `content` | String | Conditional | `kind: "text"`일 때 **Mandatory**. 표시할 순수 텍스트(마크다운·HTML 불가) |
| `src` | String | Conditional | `kind: "image"`일 때 **Mandatory**. 번들 루트의 `images/` 폴더 기준 상대경로 (예: `"images/diagram.png"`, SVG 포함) |
| `d` | String | Conditional | `kind: "path"`일 때 **Mandatory**. SVG `<path>`의 `d` 속성과 동일한 문법의 경로 데이터 |
| `points` | String | Conditional | `kind: "line"` 또는 `"polygon"`일 때 **Mandatory**. `"x1,y1 x2,y2 ..."` 형식의 좌표 목록 |
| `style` | Object | Optional | `fill`, `stroke`, `stroke_width`, `corner_radius`, `radius`(circle 전용), `font_size`, `font_weight`, `color`, `align` 등 kind별 시각 속성. 사전 정의된 CSS 색상값/숫자만 허용하며 임의 CSS 문자열(`url(...)`, `expression(...)` 등)은 금지. `kind: "text"`의 자동맞춤 관련 하위 필드는 4.3.4.4절 참조 |
| `initial` | Object | Optional | 진입 시점의 초기 상태(`opacity`(0~1), `scale`, `rotation`). 기본값 `{ opacity: 1, scale: 1, rotation: 0 }` |

##### 4.3.4.2 액션(`steps[].actions[]`, `interactions[].action`) 및 `effect` 허용 목록

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `target` | String | **Mandatory** | 동작을 적용할 요소의 `id`. 반드시 같은 임베드의 `elements` 배열에 존재하는 id여야 함 |
| `effect` | String | **Mandatory** | 아래 허용 목록 중 하나. 임의 문자열이나 커스텀 함수명은 금지 |
| `to` | Number \| Object | Conditional | `move_to`(좌표 객체 `{x, y}`), `scale_to`(배율 Number), `rotate_to`(각도 Number)일 때 **Mandatory**인 목적값 |
| `duration` | Number | Optional | 애니메이션 지속시간(초). 기본값 `0.5` |
| `delay` | Number | Optional | 시작 지연시간(초). 기본값 `0` |
| `easing` | String | Optional | `"linear"`, `"ease_in"`, `"ease_out"`, `"ease_in_out"` 중 하나. 기본값 `"ease_out"` |

`effect` 허용 목록: `fade_in`/`fade_out`(불투명도 전환), `move_to`(이동), `scale_to`(확대/축소), `rotate_to`(회전), `show`/`hide`(즉시 표시/숨김), `highlight`(강조 펄스), `draw_path`(`kind:"path"` 전용, 선이 그려지는 효과). `morph_path` 등은 향후 확장 예약이며 현재는 지원하지 않습니다.

##### 4.3.4.3 트리거(`steps[].trigger`, `interactions[].event`)

`steps[].trigger`는 `"on_enter"`(진입 즉시 자동 재생) 또는 `"on_click"`(클릭해야 다음 단계 진행, reveal.js fragment 모델)만 허용됩니다. `interactions[].event`는 현재 `"click"` 값만 지원합니다.

##### 4.3.4.4 텍스트 자동맞춤(Text Auto-Fit) 규칙

`kind: "text"` 요소는 `width`/`height`가 지정된 하나의 바운딩 박스로 취급되며, 이 프로토콜을 구현하는 모든 재생 엔진(Open Tutorials 재생 앱, Vivo Studio 프리뷰 등)은 아래 규칙에 따라 텍스트가 박스를 벗어나지 않도록 **반드시** 자동맞춤 렌더링을 수행해야 합니다.

| `style` 필드 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `wrap` | Boolean | Optional | 기본값 `true`. `width`가 지정된 경우 텍스트를 해당 폭 내에서 자동 줄바꿈. `false`면 줄바꿈 없이 한 줄 강제 렌더링(오버플로우는 저작자 책임) |
| `min_font_size` | Number | Optional | 기본값 `12`. 줄바꿈 후에도 `height`를 벗어나면 `font_size`부터 이 값까지 점진 축소. 같은 요소의 `font_size`보다 큰 값은 무효(검증 오류) |
| `overflow` | String | Optional | 기본값 `"shrink"`. `"shrink"`(줄바꿈+축소 후 남는 초과분은 자름), `"clip"`(자동맞춤 없이 박스 경계에서 즉시 자름), `"visible"`(박스 무시, 원래 크기로 표시) 중 하나 |
| `vertical_align` | String | Optional | 기본값 `"middle"`. `"top"`/`"middle"`/`"bottom"` 중 하나 |
| `line_height` | Number | Optional | 기본값 `1.3`. 줄바꿈된 각 줄 사이의 행간 배수 |

#### 4.3.5 `vivo:lottie` — Lottie 애니메이션 임베드

```json
{ "src": "assets/lottie/intro.json", "loop": true, "autoplay": true, "speed": 1 }
```

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `src` | String | **Mandatory** | `assets/lottie/` 아래 실제 존재하는 Lottie JSON 파일 상대경로. **인라인으로 Lottie 데이터를 직접 삽입하는 것은 금지**됩니다(파일이 크고 수기 작성 대상이 아니며, 검증 가능한 별도 파일로 관리하기 위함) |
| `loop` | Boolean | Optional | 반복 재생 여부. 기본값 `true` |
| `autoplay` | Boolean | Optional | 화면 진입 시 자동 재생 여부. 기본값 `true` |
| `speed` | Number | Optional | 재생 속도 배율. 기본값 `1` |
| `extend` | Boolean | Optional | 카드 레이어 전체 사용 여부. 기본값 `false` |
| `extend_only` | Boolean | Optional | 카드 레이어 전체 사용 고정 여부 (축소 불가). 기본값 `false` |

> **보안 제약**: 참조된 Lottie JSON 파일은 **expressions(임의 JavaScript 표현식)를 포함해서는 안 됩니다.** 재생 엔진은 expressions를 지원하지 않는 빌드(`lottie-web/build/player/lottie_light` 등)로만 재생하며, 검증기는 expressions가 포함된 Lottie 파일을 거부합니다(6장 참조).

#### 4.3.6 SVG 이미지 참조 규칙

정적 또는 SMIL/CSS로 자체 애니메이션되는 SVG는 임베드 블록이 아니라 **기존 마크다운 이미지 문법**으로만 참조합니다.

```markdown
![회로도](images/circuit.svg)
```

재생 엔진은 이를 항상 `<img src>`로만 렌더링합니다(SVG를 마크다운 본문에 `dangerouslySetInnerHTML` 등으로 인라인 삽입하는 것은 금지). `<img>` 태그 안에서 재생되는 SVG는 SMIL/CSS 애니메이션이 동작하더라도 내부 `<script>`가 브라우저에 의해 차단되므로 안전합니다.

#### 4.3.7 `vivo:3d` — 3D 모델 임베드 (v1.5.0 신설)

강좌 카드 본문에 GLB 3D 모델을 삽입해 학습자가 회전·확대하며 살펴보고, 모델에 내장된 애니메이션 클립이나 선언적 변환 애니메이션을 재생할 수 있게 합니다.

##### 4.3.7.1 페이로드 전체 예시

```json
{
  "src": "assets/models/robot_arm.glb",
  "alt": "6축 산업용 로봇 팔 3D 모델",
  "height": 420,
  "background": "transparent",
  "auto_rotate": false,
  "camera": { "position": [0, 1.5, 3], "target": [0, 0.8, 0], "fov": 45 },
  "controls": { "enable_zoom": true, "enable_pan": true, "min_distance": 0.5, "max_distance": 20 },
  "lighting": { "ambient_intensity": 0.9, "directional_intensity": 1.2 },
  "animation": { "autoplay": true, "loop": true, "clip": "working_cycle", "speed": 1 },
  "transforms": [
    { "target": "Wheel_L", "property": "rotation.y", "from": 0, "to": 6.283, "duration": 4, "loop": true, "easing": "linear" }
  ]
}
```

##### 4.3.7.2 필드 명세

| 필드 | 타입 | 필수 여부 | 기본값 | 설명 |
| :--- | :--- | :---: | :--- | :--- |
| `src` | String | **Mandatory** | — | `assets/models/` 아래 실제 존재하는 **`.glb`** 파일의 상대경로. `.gltf`(외부 `.bin`/텍스처 참조)는 v1.5.0에서 **미지원** |
| `alt` | String | Optional | `""` | 대체 텍스트. 로딩 실패 시 화면에 표시되며 접근성 용도 |
| `height` | Number | Optional | `400` | 뷰포트 높이(px). 허용 범위 `120`~`1200` |
| `background` | String | Optional | `"transparent"` | `"transparent"` 또는 `#rrggbb` |
| `auto_rotate` | Boolean | Optional | `false` | 진입 시 자동 회전 시작 여부 |
| `camera.position` | Number[3] | Optional | 모델 바운딩 박스 기준 자동 산출 | 카메라 좌표 |
| `camera.target` | Number[3] | Optional | 모델 중심 | 카메라가 바라보는 지점 |
| `camera.fov` | Number | Optional | `45` | 시야각(도). `0 < fov < 180` |
| `controls.enable_zoom` | Boolean | Optional | `true` | 휠 줌 허용 |
| `controls.enable_pan` | Boolean | Optional | `true` | 우클릭 팬 허용 |
| `controls.min_distance` | Number | Optional | 자동 | 최소 줌 거리(> 0) |
| `controls.max_distance` | Number | Optional | 자동 | 최대 줌 거리(`min_distance`보다 커야 함) |
| `lighting.ambient_intensity` | Number | Optional | `0.9` | 환경광 세기. `0`~`10` |
| `lighting.directional_intensity` | Number | Optional | `1.2` | 직사광 세기. `0`~`10` |
| `animation.autoplay` | Boolean | Optional | `true` | 화면 진입 시 자동 재생 |
| `animation.loop` | Boolean | Optional | `true` | 반복 재생 |
| `animation.clip` | String | Optional | 첫 번째 클립 | 재생할 GLB 내장 클립 이름. 존재하지 않으면 재생 엔진이 첫 클립으로 폴백하고 안내 배지를 표시 |
| `animation.speed` | Number | Optional | `1` | 재생 속도 배율. `0.1`~`4` |
| `transforms` | Array | Optional | `[]` | 선언적 변환 애니메이션(4.3.7.3절). 최대 **16개** |
| `extend` | Boolean | Optional | `false` | 카드 레이어 전체 사용 여부 |
| `extend_only` | Boolean | Optional | `false` | 카드 레이어 전체 사용 고정 여부 (축소 불가) |

##### 4.3.7.3 `transforms[]` — 선언적 변환 애니메이션

GLB에 애니메이션 클립이 없는 정적 모델에도 움직임을 줄 수 있는 최소 DSL입니다. `vivo:motion`의 `easing` 어휘를 그대로 재사용합니다.

| 필드 | 타입 | 필수 여부 | 기본값 | 설명 |
| :--- | :--- | :---: | :--- | :--- |
| `target` | String | **Mandatory** | — | GLB 내부 노드 이름. 모델 전체는 `"__root__"` |
| `property` | String | **Mandatory** | — | `rotation.x`/`rotation.y`/`rotation.z`/`position.x`/`position.y`/`position.z`/`scale.x`/`scale.y`/`scale.z`/`scale` 중 하나. `rotation.*`의 단위는 **라디안** |
| `from` | Number | **Mandatory** | — | 시작값(유한수) |
| `to` | Number | **Mandatory** | — | 종료값(유한수) |
| `duration` | Number | **Mandatory** | — | 재생 시간(초). `0 < duration <= 600` |
| `delay` | Number | Optional | `0` | 시작 지연(초). `>= 0` |
| `loop` | Boolean \| String | Optional | `true` | `true`(반복) / `false`(1회) / `"pingpong"`(왕복) |
| `easing` | String | Optional | `"linear"` | `linear`/`ease_in`/`ease_out`/`ease_in_out` |

- `transforms`와 `animation.clip`은 **동시 사용 가능**합니다. 같은 노드의 같은 속성을 둘 다 건드리면 `transforms`가 나중에 적용됩니다(클립 적용 후 덮어쓰기).
- 존재하지 않는 `target` 노드 이름은 검증기가 잡을 수 없습니다(GLB 파싱 필요). 재생 엔진이 무시하고 개발자 콘솔에 경고를 남깁니다.

##### 4.3.7.4 3D 모델 제약 사항

- 모델 파일 1개의 크기는 **30MB 이하**여야 합니다. 검증기가 번들 전체를 메모리에 올려 검사하기 때문입니다.
- GLB 파일은 매직 바이트가 `glTF`(`0x67 0x6C 0x54 0x46`)이고 컨테이너 버전이 `2`여야 합니다.
- Draco/Meshopt 압축 모델은 v1.5.0 재생 엔진이 디코더를 탑재하지 않으므로 **사용 금지**입니다(로딩 실패). 2차 확장 대상입니다.

#### 4.3.8 `vivo:runtime` — 코드 IDE/런타임 임베드 (v1.5.0 신설)

강좌 카드 본문에 편집·실행 가능한 코드 실습 블록을 삽입합니다.

##### 4.3.8.1 펜스 본문 구조 및 파싱 규칙

`vivo:runtime` 펜스 본문은 **순수 JSON이 아닙니다.** 아래 세 부분으로 구성됩니다.

```text
<JSON 헤더>
--- 또는 --- <파일명>       ← 구분선
<코드 본문>
```

정확한 파싱 규칙:

1. 본문을 줄 단위로 훑으며 **`^---(?:[ \t]+(\S.*?))?[ \t]*$`** 정규식에 매치되는 줄을 **구분선**으로 봅니다.
2. 첫 구분선 **앞부분** = JSON 헤더. `serde_json` / `JSON.parse`로 파싱합니다.
3. 첫 구분선 **뒤부터 다음 구분선 전까지** = **엔트리 파일**. 파일명은 헤더의 `entry`, 없으면 언어별 기본값(4.3.8.4절).
   - 첫 구분선에 파일명이 붙어 있으면 그것이 엔트리 파일명이 되고, `entry`와 다르면 검증 오류입니다.
4. 두 번째 이후 구분선은 **반드시 파일명을 동반**해야 합니다. 파일명이 없으면 검증 오류입니다.
5. 구분선이 하나도 없으면 검증 오류입니다(코드가 없는 실습 블록은 무의미).

> **설계 배경:** 코드를 JSON 문자열에 넣으면 `\n`·`\"` 이스케이프가 과도해져 가독성이 저하되고 AI 에이전트의 작성 실패율이 급등합니다. 헤더만 JSON으로 두면 검증기는 여전히 `serde_json`을 쓸 수 있습니다.

##### 4.3.8.2 다중 파일 예시

````markdown
```vivo:runtime
{ "lang": "web", "mode": "web", "title": "플렉스박스 정렬 실습", "entry": "index.html" }
---
<!doctype html>
<div class="row"><span>A</span><span>B</span></div>
<script src="app.js"></script>
--- style.css
.row { display: flex; gap: 8px; }
--- app.js
console.log("loaded");
```
````

##### 4.3.8.3 헤더 필드 명세

| 필드 | 타입 | 필수 여부 | 기본값 | 설명 |
| :--- | :--- | :---: | :--- | :--- |
| `lang` | String | **Mandatory** | — | 4.3.8.4절 표의 값 중 하나 |
| `mode` | String | **Mandatory** | — | `"wasm"` / `"web"` / `"native"`. `lang`과의 조합은 4.3.8.4절 표로 제한 |
| `view` | String | Optional | `"split"` | `"split"` (에디터+결과), `"preview"` (에디터 숨기고 실행 결과만 노출), `"editor"` (에디터만 노출) |
| `extend` | Boolean | Optional | `false` | 카드 레이어 전체 사용 여부 (토글 가능) |
| `extend_only` | Boolean | Optional | `false` | 카드 레이어 전체 사용 고정 여부 (축소 불가) |
| `extend_visual` | Boolean | Optional | `false` | **vivo:runtime 전용**. 에디터/툴바 없이 **실행 결과(Visual)만 카드 레이어 전체에 고정** |
| `title` | String | Optional | `""` | 실습 블록 헤더에 표시할 제목 |
| `entry` | String | Optional | 언어별 기본값(4.3.8.4절) | 엔트리 파일명 |
| `read_only_lines` | Number[] | Optional | `[]` | **엔트리 파일 기준** 1-base 행 번호. 해당 줄은 편집 불가로 잠깁니다 |
| `height` | Number | Optional | `380` | 에디터/프리뷰 영역 높이(px). `160`~`1200` |
| `autorun` | Boolean | Optional | `lang === "web"`이면 `true`, 그 외 `false` | 화면 진입 시 자동 실행. **`mode: "native"`에서는 `true` 지정 자체가 검증 오류** |
| `show_console` | Boolean | Optional | `true` | Console 탭 노출 여부 |
| `timeout_ms` | Number | Optional | `10000` | 실행 타임아웃(ms). `1000`~`60000` |
| `packages` | String[] | Optional | `[]` | `lang: "python"`, `mode: "wasm"` 전용. 4.3.8.5절 허용 목록만 사용 가능 |

**헤더에 넣을 수 없는 것(검증 오류):** `command`, `args`, `compile_args`, `run_args`, `cwd`, `env`, `interpreter`, `path`, `shell`, `exec` 등 실행 명령/경로를 지정하는 모든 키. 명령은 앱이 소유합니다.

##### 4.3.8.4 `lang` × `mode` 허용 조합

| `lang` | 허용 `mode` | 기본 `entry` | 실행 수단 |
| :--- | :--- | :--- | :--- |
| `python` | `wasm`, `native` | `main.py` | Pyodide / 로컬 `python` |
| `javascript` | `wasm`, `native` | `main.js` | Web Worker / 로컬 `node` |
| `typescript` | `wasm` | `main.ts` | esbuild-wasm 트랜스파일 → Web Worker |
| `sql` | `wasm` | `main.sql` | sql.js (인메모리 SQLite) |
| `web` | `web` | `index.html` | 샌드박스 iframe |
| `c` | `native` | `main.c` | 로컬 C 컴파일러 |
| `cpp` | `native` | `main.cpp` | 로컬 C++ 컴파일러 |
| `rust` | `native` | `main.rs` | 로컬 `rustc` |
| `java` | `native` | `Main.java` | 로컬 `javac` + `java` |
| `go` | `native` | `main.go` | 로컬 `go run` |

표에 없는 조합(예: `lang: "c"` + `mode: "wasm"`)은 검증 오류입니다. **셸 스크립트(`bash`/`powershell`)는 의도적으로 제외**합니다 — 임의 시스템 명령 실행과 구분이 불가능하기 때문입니다.

##### 4.3.8.5 `packages` 허용 목록 (Python WASM 전용)

앱에 함께 번들되는 휠만 사용할 수 있습니다. 목록 밖의 값은 검증 오류입니다.

```text
numpy, matplotlib
(참고: pandas, sympy, scipy는 스펙상 예약이며 환경에 미포함 시 안내 표시)
```

##### 4.3.8.6 파일명 규칙

모든 파일명(엔트리 포함)은 아래를 만족해야 합니다. 네이티브 모드에서 임시 디렉터리에 실제 파일로 기록되므로 경로 탈출 방지가 목적입니다.

- 정규식 `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`
- `/`, `\`, `..` 포함 금지
- 한 임베드 안에서 중복 금지
- 파일 개수 최대 **10개**, 파일 하나당 최대 **64KB**

---

### 4.4 다국어 강좌 지원 (Multilingual Course Support, v1.6.0 신설)

하나의 강좌 패키지가 여러 언어 버전을 갖고, Vivo Academy에서 **같은 강좌 안에서 언어를 전환**하여 볼 수 있도록 하는 다국어 지원 구조입니다. 가격/공개 여부 등 강좌 메타데이터는 언어와 무관하게 강좌 단위로 공유됩니다. 지원 언어는 기본 `ko`, 선택적으로 `en`/`vi`/`id`입니다. 선택된 언어의 콘텐츠는 AI 자동 번역 초안으로 생성되고 강사가 Vivo Studio에서 검수/수정하며, 언어별 번역 상태(`draft`/`reviewed`/`published` 등)를 추적할 수 있습니다.

#### 4.4.1 설계 원칙

기존 프로토콜이 이미 채택한 두 가지 원칙을 다국어 지원에도 그대로 확장 적용합니다.

- **오버레이(Overlay) 방식**: 새 언어를 위해 강좌 전체를 복제하지 않습니다. 기본 언어(base language) 콘텐츠가 이미 루트에 존재하므로, 각 언어는 "달라지는 파일만" 추가로 두는 **오버레이 폴더**로 표현합니다. 오버레이 폴더에 파일이 없으면 항상 루트(base) 파일로 **폴백**합니다.
- **선언적 데이터 우선**: 카드 본문(`cards/*.md`, `wiki.md`) 안의 `vivo:video`/`vivo:motion` 등 임베드 JSON은 카드 파일의 일부이므로, 카드 파일 하나를 번역하면 그 안의 자막(`subtitles[].text`)이나 다이어그램 텍스트(`elements[].content`)도 함께 번역됩니다. 별도의 "임베드 텍스트 전용 번역 포맷"을 새로 만들 필요가 없습니다.

이 두 원칙 덕분에 하위 호환은 자동으로 따라옵니다: 오버레이 폴더(`locales/`)가 없는 기존 ZIP은 오늘과 완전히 동일하게 동작합니다(4.4.7절).

#### 4.4.2 오버레이 디렉토리 구조 (`locales/`)

```text
[강좌 패키지 ZIP 파일]
├── package-manifest.json           # 강좌 단위 공유 메타데이터 (가격/공개여부/카테고리 등) — 언어 무관
├── config.json                     # 기본 언어(base language) TOC + 카드 목록
├── wiki.md                         # 기본 언어 위키
├── thumbnail.png
├── LICENSE.md
├── cards/                          # 기본 언어 카드 (기존과 100% 동일)
│   ├── 01_intro.md
│   └── 02_concept.md
├── images/                         # 공통 기본 이미지
│   └── schema.png
├── assets/                         # 공통 기본 motion/lottie/models
│   └── motion/01_variables.json
└── locales/                        # ★ 신설 (선택) — 언어별 오버레이
    ├── en/
    │   ├── translation-status.json # ★ 신설 (필수, locales/<lang>/ 존재 시) — 번역 상태 메타데이터
    │   ├── wiki.md                 # 선택: 있으면 en 위키로 사용, 없으면 루트 wiki.md 폴백
    │   ├── config.json             # 선택: 있으면 en TOC(제목/설명 번역판)로 사용, 없으면 루트 config.json 폴백
    │   ├── cards/
    │   │   ├── 01_intro.md         # 선택 오버라이드 (번역이 끝난 카드만 존재하면 됨)
    │   │   └── 02_concept.md
    │   ├── images/
    │   │   └── schema.png          # 선택: 다이어그램에 한국어 라벨이 박혀있는 경우에만 교체
    │   └── assets/
    │       └── motion/01_variables.json  # 선택: 텍스트가 포함된 모션 소스만 교체
    ├── vi/
    │   └── ... (동일 구조, 전부 선택)
    └── id/
        └── ... (동일 구조, 전부 선택)
```

- `locales/` 폴더는 **완전히 선택 항목**입니다. 없으면 3장의 기존 구조가 그대로 적용됩니다.
- `cards/`, `images/`, `assets/`의 기존 하위 규칙(허용 확장자, `assets/motion|lottie|models` 서브폴더 규칙 등)은 `locales/<lang>/` 하위에서도 동일하게 적용됩니다 — 구조를 그대로 복제하되 파일 존재 여부만 선택적입니다.

#### 4.4.3 경로 해석(Resolution) 규칙 — 모든 언어별 자원에 공통 적용

강좌를 언어 `L`로 렌더링할 때, 임의의 상대 경로 `p` (예: `wiki.md`, `config.json`, `cards/01_intro.md`, `images/schema.png`, `assets/motion/01_variables.json`)를 조회하는 절차:

1. `L`이 base language(기본값 `ko`)이면 → 항상 루트의 `p`를 사용.
2. `L`이 base language가 아니면 →
   a. `locales/<L>/<p>` 파일이 ZIP/저장소 내에 실존하면 그것을 사용.
   b. 존재하지 않으면 루트의 `p`로 **폴백**.

이 규칙 하나로 "텍스트 오버라이드"와 "에셋 오버라이드"를 동일하게 설명할 수 있습니다.

#### 4.4.4 언어별 텍스트 콘텐츠 오버라이드

| 대상 | 오버라이드 단위 | 방식 |
| :--- | :--- | :--- |
| 위키 본문 | 파일 전체 | `locales/<lang>/wiki.md` 존재 시 전체 교체 |
| 카드 본문(및 그 안의 임베드 JSON 텍스트) | 파일 전체 | `locales/<lang>/cards/<파일명>.md` 존재 시 전체 교체. `vivo:video`의 `subtitles[].text`, `vivo:motion`의 `elements[].content` 등 카드 안에 인라인된 텍스트는 카드 파일을 번역하는 것만으로 함께 번역됨 (별도 스키마 불필요) |
| 목차(TOC) 제목/설명 | 파일 전체 | `locales/<lang>/config.json` 존재 시 TOC의 `title`/`description` 등 텍스트가 번역된 버전으로 전체 교체 |
| 강좌 패키지 자체의 `title`/`description`/`tags`/`changelog` (package-manifest.json) | 개별 필드 | 4.4.6.4절 참조 — course-level 메타데이터는 언어 무관 공유가 원칙이나, 표시용 제목/설명만 예외적으로 언어별 오버라이드 허용 (`locales/<lang>/manifest-overrides.json`, 4.4.6.4절) |

##### 4.4.4.1 `config.json` 번역판 검증 규칙

TOC 구조 자체(트리 모양, `filename` 매핑)는 언어에 따라 달라지면 안 되므로(같은 강좌를 다른 언어로 보는 것이지 다른 강좌가 아님), `locales/<lang>/config.json`은 다음을 만족해야 합니다.

- `cards` 배열의 파일명 집합이 루트 `config.json`의 `cards` 배열과 **동일**해야 함 (번역 파일이 있든 없든, 참조하는 base 카드 파일명 기준).
- `toc` 트리의 노드 개수·순서·`type`·`filename`이 루트 `config.json`과 **완전히 동일**해야 함. 오직 `title`/`description`(및 노드별 번역 상태 참조용 필드)만 달라질 수 있음.
- `card_style`은 언어별로 달라질 이유가 없으므로, 언어별 `config.json`에 지정되어 있어도 **무시**하고 항상 루트 값을 사용 (일관성 유지, 혼란 방지).

이 규칙은 "번역자가 실수로 구조를 바꿔서 두 언어 버전이 다른 강좌처럼 어긋나는" 사고를 검증 단계에서 차단하기 위함입니다.

#### 4.4.5 언어별 에셋(이미지/다이어그램) 오버라이드

**공유 vs 교체 판단 규칙:** 기본 원칙은 **"에셋은 기본적으로 공유하고, 언어 의존적 콘텐츠가 박혀 있을 때만 교체"**입니다. 검증기가 자동으로 강제할 수는 없는 저작 가이드라인이므로, Vivo Studio UI/가이드 문서 수준에서 아래 판단 기준을 안내합니다.

| 상황 | 권장 |
| :--- | :--- |
| 이미지/GLB 모델이 순수 시각 자료이며 텍스트 라벨이 없음 (사진, 아이콘, 범용 3D 모델 등) | **공유** — 언어별 오버라이드 금지(또는 불필요). `locales/<lang>/` 아래 동일 경로 파일을 만들지 않음 |
| 이미지/다이어그램 안에 한국어(또는 원어) 텍스트 라벨이 렌더링되어 있음 (`images/schema.png` 안에 "입력값" 같은 한글 라벨) | **교체** — 해당 언어 라벨로 다시 그린 이미지를 `locales/<lang>/images/schema.png`로 배치 |
| `vivo:motion` 외부 참조 JSON(`assets/motion/*.json`)의 `elements[].content`(텍스트 요소)에 언어 의존 텍스트가 있음 | **교체** — `locales/<lang>/assets/motion/<동일 파일명>.json`으로 텍스트만 번역된 버전을 배치. `canvas`/도형 좌표 등 비텍스트 요소는 동일하게 유지 권장 |
| `vivo:lottie`(`assets/lottie/*.json`) 안에 텍스트 레이어가 있음 | **교체** — 동일 방식으로 `locales/<lang>/assets/lottie/<동일 파일명>.json` |
| `vivo:3d`(`assets/models/*.glb`) | 원칙적으로 **공유** (3D 모델에 텍스트 라벨이 각인된 특수 케이스가 아니면 언어별 교체 불필요). 30MB 제한이 있는 큰 바이너리이므로 불필요한 언어별 중복 배치는 지양 |

**메커니즘:** 이미지/에셋 오버라이드는 4.4.3절의 "경로 해석 규칙"을 그대로 재사용합니다. 즉 새로운 스키마가 필요 없고, `locales/<lang>/images/<파일명>`, `locales/<lang>/assets/<motion|lottie|models>/<파일명>`에 **동일 상대경로**로 파일을 두기만 하면 검증기/재생 엔진이 자동으로 우선 사용합니다.

**검증 규칙:**
- `locales/<lang>/images/`, `locales/<lang>/assets/**`에 존재하는 모든 파일은, 그 파일명이 카드(base 또는 해당 언어 오버라이드 카드) 어딘가에서 참조되는 상대경로와 **일치**해야 함 (고아 파일 금지 — 기존 6장의 "미사용 리소스 정합성" 원칙 확장).
- `vivo:3d`(GLB), `vivo:runtime` 파일 실습 코드는 언어별 오버라이드 대상에서 **제외**를 권장(가이드라인 수준). GLB는 위 표 참조. `vivo:runtime` 코드 실습은 학습자가 직접 코드를 읽고 편집하는 대상이라 코드 자체(변수명/문자열 리터럴 등)의 언어별 이원화는 실효성이 낮음 — 필요 시 `title`/`entry` 주석 등 카드 본문 번역으로 흡수.

#### 4.4.6 번역 상태 메타데이터 (`locales/<lang>/translation-status.json`)

`locales/<lang>/` 폴더가 존재하면 **필수**로 두어야 하는 파일입니다. 해당 언어 콘텐츠의 생성 경위와 파일 단위 검수 상태를 추적합니다.

```json
{
  "language": "en",
  "source_language": "ko",
  "status": "draft",
  "generated_by": "ai_auto_translation",
  "generated_at": "2026-08-20T09:00:00Z",
  "updated_at": "2026-08-24T03:12:00Z",
  "items": [
    { "path": "wiki.md",              "status": "reviewed",  "reviewed_at": "2026-08-24T03:00:00Z" },
    { "path": "config.json",          "status": "reviewed",  "reviewed_at": "2026-08-24T03:00:00Z" },
    { "path": "cards/01_intro.md",    "status": "draft" },
    { "path": "cards/02_concept.md",  "status": "not_translated" },
    { "path": "images/schema.png",    "status": "draft" }
  ]
}
```

##### 4.4.6.1 필드 명세

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `language` | String | **Mandatory** | 이 오버레이 폴더가 속한 언어 코드. 폴더명(`en`/`vi`/`id`)과 일치해야 함 |
| `source_language` | String | **Mandatory** | 번역 원본 언어 (통상 `package-manifest.json`의 `base_language`와 동일) |
| `status` | String | **Mandatory** | 해당 언어 전체의 **롤업(rollup) 상태**. `not_translated` / `draft` / `reviewed` / `published` 중 하나. Vivo Academy 언어 선택 UI는 이 값이 `published`인 언어만 학습자에게 노출 (Studio 미리보기에서는 draft/reviewed도 저작자 본인에게는 노출) |
| `generated_by` | String | Optional | 콘텐츠 생성 주체. `ai_auto_translation`(AI 자동 번역) / `human`(수동 작성) / `hybrid` |
| `generated_at` | String (ISO 8601) | Optional | 최초 생성(AI 번역 초안 생성) 시각 |
| `updated_at` | String (ISO 8601) | Optional | 마지막 수정/검수 시각 |
| `items` | Array | Optional | 파일 단위 세부 상태. 생략 시 `status`(롤업 값)가 모든 파일에 동일 적용된 것으로 간주 |

##### 4.4.6.2 `items[]` 원소 명세

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `path` | String | **Mandatory** | base 루트 기준 상대경로 (예: `"cards/01_intro.md"`). 4.4.3절 경로 해석 규칙의 `p`와 동일한 값 |
| `status` | String | **Mandatory** | `not_translated`(오버라이드 파일 없음, 루트로 폴백 중) / `draft`(AI 생성, 미검수) / `reviewed`(강사 검수 완료) / `published`(검수 완료 + 학습자 공개) |
| `reviewed_at` | String (ISO 8601) | Optional | 검수 완료 시각 |
| `reviewed_by` | String | Optional | 검수자 식별자(닉네임/이메일). PII 노출 최소화를 위해 앱단에서 마스킹 여부는 구현체 재량 |

> **구현체 확장 필드 허용 (Gate B 결정):** `items[]` 원소에는 위 명세 필드(`path`/`status`/`reviewed_at`/`reviewed_by`) 외에 구현체별 추가 필드를 포함할 수 있습니다 (예: 재번역 감지용 `source_hash`). 검증기는 알려지지 않은 추가 필드를 거부하지 말고 **무시**해야 합니다 — 미지 필드의 존재만으로 검증 오류가 되어서는 안 됩니다.

##### 4.4.6.3 상태(`status`) 값 정의

| 값 | 의미 |
| :--- | :--- |
| `not_translated` | 해당 언어의 오버라이드 파일이 아직 없음(루트 base 콘텐츠로 자동 폴백 중) |
| `draft` | AI 자동 번역으로 생성된 초안. 강사 미검수 상태 — Vivo Studio에서만 미리보기 가능, 학습자에게는 비공개 |
| `reviewed` | 강사가 Vivo Studio에서 검수·수정 완료. 아직 공개 전(내부 QA 단계) |
| `published` | 검수 완료 + 학습자에게 실제로 노출 중. Vivo Academy 언어 전환 스위처에 이 언어가 나타남 |

> 강좌 자체의 `published`(공개 여부, `package-manifest.json`)와 언어별 `published`(번역 상태)는 **서로 다른 축**입니다. 강좌가 비공개면 어떤 언어든 노출되지 않고, 강좌가 공개여도 특정 언어가 아직 `draft`/`reviewed`면 그 언어만 학습자에게 숨겨집니다.

##### 4.4.6.4 `package-manifest.json` 다국어 필드 (요약)

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `base_language` | String | Optional (기본값 `"ko"`) | 강좌의 기본(원본) 언어. 기존 `language` 필드를 대체하되, 하위 호환을 위해 `language`도 계속 허용(별칭) — 상세는 4.1절 표 참조 |
| `supported_languages` | Array of String | Optional (기본값 `[base_language]`) | 이 강좌가 제공하는 언어 코드 목록. `ko`/`en`/`vi`/`id` 중에서 선택. `base_language`는 항상 포함되어야 함 — 상세는 4.1절 표 참조 |

가격(`price` 등 향후 필드)·`published`·`sequential_play`·`category` 등 기존 course-level 필드는 **언어 무관**으로 유지됩니다.

#### 4.4.7 기존 단일언어 강좌 패키지 하위 호환성

- **구조적 호환**: `locales/` 폴더는 완전 선택 항목이므로, 기존에 배포된 모든 ZIP/DB 레코드는 스키마 변경 없이 그대로 유효합니다.
- **필드 호환**: 신설 필드(`base_language`, `supported_languages`)는 모두 Optional이며 기본값이 기존 동작과 동일합니다(`base_language` 기본값 `"ko"` = 기존 `language` 기본값과 동일, `supported_languages` 기본값 `[base_language]` = "언어 하나짜리 강좌"라는 기존 의미와 동일).
- **`language` 필드 유지**: 기존 `language` 필드는 폐기하지 않고 `base_language`의 별칭(alias)으로 계속 허용합니다. 검증기는 둘 다 있을 때 값이 다르면 경고(오류 아님)만 발생시킵니다. 즉, 기존 제작 도구/에이전트가 `language` 필드만 채워도 계속 정상 동작합니다.
- **마이그레이션 불필요**: `db.json`/`public/courses/[slug]/` 등 기존 저장소 레코드에 대해 별도 배치 마이그레이션 스크립트가 필요 없습니다. 로컬 실행 환경/서버는 압축 해제 시 `locales/` 폴더 존재 여부만 확인하고, 없으면 오늘과 동일한 코드 경로를 그대로 탑니다.
- **재생 엔진 호환**: 구버전(다국어 미지원) 재생 엔진이 `locales/` 필드가 포함된 신규 번들을 만나면, 미지의 최상위 폴더로 안전하게 무시하고 기존처럼 루트 콘텐츠만 재생합니다 (알려지지 않은 임베드를 무시하는 기존 v1.4/v1.5 하위 호환 패턴과 동일한 접근).

---

## 5. 제작 예시 (Implementation Example)

### 5.1 package-manifest.json 예시

```json
{
  "title": "AI 튜터 기반 파이썬 프로그래밍",
  "slug": "ai-tutor-python-course",
  "description": "AI 튜터와 대화하며 배우는 파이썬 기초부터 활용까지",
  "author": {
    "nickname": "Kailash",
    "email": "godstale@hotmail.com",
    "website": "https://hardcopyworld.com"
  },
  "thumbnail": "./thumbnail.png",
  "published": true,
  "sequential_play": true,
  "force_checkpoint": false,
  "version": "1.0.0",
  "changelog": "최초 릴리즈",
  "bundler_protocol_version": "1.5.0",
  "target_age": "10+",
  "category": "Programming",
  "language": "ko",
  "tags": ["Python", "AI", "Beginner"],
  "license": "CC-BY-4.0"
}
```

### 5.2 강좌 카드(`cards/01_intro.md`) 예시

```markdown
---
title: "변수와 자료형"
theme: light
---

# 변수와 자료형

변수는 데이터를 담는 상자입니다. 아래 영상으로 먼저 개념을 확인해 봅시다.

\`\`\`vivo:video
{ "provider": "youtube", "video_id": "dQw4w9WgXcQ", "start": 0 }
\`\`\`

이제 다이어그램으로 자료형의 종류를 살펴보겠습니다. 각 상자를 클릭해 보세요.

\`\`\`vivo:motion
{ "src": "assets/motion/01_variables.json" }
\`\`\`

정리하면, 파이썬의 기본 자료형은 다음과 같습니다.

| 자료형 | 설명 |
| :--- | :--- |
| `int` | 정수 |
| `str` | 문자열 |
```

---

## 6. 제작 시 주의사항 및 검증 규칙 (Strict Validation Rules)

강좌 번들 작성 시 다음 항목 중 하나라도 위배되면 플랫폼 내 업로드 단계에서 검증 오류가 발생하여 등록이 거부됩니다.

1. **상위 디렉토리 미포함 (Flat ZIP)**:
   - ZIP 파일 압축 시, 최상위 경로에 단일 폴더가 있고 그 안에 `package-manifest.json`이 들어있는 이중 레이어 구조는 금지됩니다. ZIP 파일을 열었을 때 바로 `package-manifest.json`, `config.json`, `wiki.md`, `cards/`가 최상위 루트에 존재해야 합니다.
2. **TOC-Card 1:1 매칭 및 파일 정합성**:
   - `config.json`의 `cards` 배열 내 모든 파일명은 실제 ZIP 안의 `cards/` 폴더 내에 정확히 존재해야 하며, 대소문자까지 일치해야 합니다.
   - `bundler_protocol_version`이 `1.4.0` 이상이면 `cards/` 아래 파일은 `.md`/`.mdx` 확장자만 허용되며, `.json` 카드 파일이 하나라도 존재하면 검증이 거부됩니다. `1.3.0` 이하인 레거시 번들은 부록 A 규칙(마크다운 `.md`/`.mdx` 및 동영상·애니메이션 카드 `.json` 확장자 허용)이 그대로 적용됩니다.
   - `toc` 트리에서 `filename`이 명시된 노드는 반드시 `cards` 배열에 있는 파일명 중 하나여야 하며, `cards` 배열에 나열된 모든 파일이 `toc` 트리 어딘가에 한 번씩만 매핑되어야 합니다.
3. **무성의한 기본 텍스트 방치 금지**:
   - `toc` 노드의 `title`이 파일명과 완전히 동일(예: title이 `01_intro`인 경우)하면 안 됩니다. 사용자가 읽기 좋은 언어(예: `1강. 오리엔테이션 및 입문 가이드`)로 설정되어야 합니다.
   - `toc` 노드의 `description`이 `"강좌 상세 카드를 확인하세요."` 와 같은 기본값으로 설정되어 있다면 검증이 반려됩니다. 각 노드에 알맞은 요약 정보가 구체적으로 기술되어야 합니다.
4. **강좌 카드 Frontmatter 검증 (v1.4.0 신설)**:
   - 카드 파일 최상단에 frontmatter 블록(`---`로 시작·종료)이 있는 경우, flat key-value 파싱이 성공해야 합니다. 파싱 실패 시 검증 오류입니다.
   - `theme` 필드가 지정된 경우 `"light"` 또는 `"dark"` 중 하나가 아니면 거부됩니다.
   - `background`/`foreground` 필드가 지정된 경우 `^#[0-9a-fA-F]{6}$` 형식(6자리 hex 색상)이 아니면 거부됩니다.
   - 정의되지 않은 frontmatter 키는 오류가 아닌 경고로 처리됩니다.
   - `config.json`의 `card_style`(4.2.1절) 객체에도 동일한 스키마와 규칙이 적용됩니다.
5. **임베드 블록(`vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`/`vivo:runtime`) 검증**:
   - `vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`/`vivo:runtime` 외의 `vivo:` 접두사 언어 식별자는 정의되지 않은 임베드 타입으로 거부됩니다.
   - `vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`: 펜스 블록 내용은 유효한 JSON이어야 하며, 파싱 실패 시 거부됩니다.
   - `vivo:video`: `provider`(현재 `"youtube"`만 허용)와 `video_id`(문자열) 필수. `subtitles` 지정 시 배열 타입이어야 하며 각 원소는 `start`/`end`(Number)와 `text`(String)를 가져야 합니다.
   - `vivo:motion`: `src` 참조 방식이면 `assets/motion/` 폴더 내에 해당 파일이 실제로 존재해야 합니다. 인라인/참조 페이로드 모두 4.3.4절 스키마(`canvas`/`elements` 필수, `elements[].kind` 허용 목록 준수, `steps[].actions[].target`/`interactions[].target`의 id 참조 무결성, `steps[].actions[].effect`/`interactions[].action.effect` 허용 목록 준수, `steps[].trigger`/`interactions[].event` 허용 목록 준수, kind별 Mandatory/Conditional 필드 누락 검사, `kind:"image"`의 `src`가 가리키는 파일이 `images/` 폴더 내에 실제 존재하는지 검사, 4.3.4.4절 텍스트 자동맞춤 필드 검증)를 만족해야 합니다.
   - `vivo:lottie`: 반드시 `src` 필드로 `assets/lottie/` 내 파일을 참조해야 하며, 인라인 Lottie 데이터 직접 삽입은 거부됩니다. `src`가 가리키는 파일이 실제로 존재하지 않으면 거부됩니다. 참조된 Lottie JSON 내부에 expressions(문자열 타입 값을 갖는 `"x"` 프로퍼티)가 발견되면 거부됩니다.
   - `vivo:3d` (v1.5.0 신설):
     - `src` 필드가 필수이며 비어 있지 않아야 하고, 반드시 `assets/models/` 경로 아래의 `.glb` 파일이어야 합니다(`.gltf`는 거부).
     - `src`가 가리키는 파일이 번들에 실제 존재해야 합니다.
     - 해당 파일의 첫 4바이트가 `glTF`(`0x67 0x6C 0x54 0x46`)이고 바이트 4~7의 LE u32가 `2`인 유효한 GLB(glTF 2.0 binary) 파일이어야 합니다.
     - 파일 크기가 30MB 이하여야 합니다.
     - `camera.position`/`camera.target`은 길이 3의 유한수 배열이어야 하고, `camera.fov`는 `0 < fov < 180`이어야 합니다.
     - `background`는 `"transparent"` 또는 `#rrggbb`, `height`는 120~1200, `lighting.*`는 0~10, `animation.speed`는 0.1~4 범위 내여야 합니다.
     - `transforms`는 배열이고 최대 16개여야 하며, 각 원소의 `target`(비어있지 않은 문자열), `property`(허용 목록), `from`/`to`(유한수), `duration`(0 < duration <= 600), `delay`(>= 0), `loop`(bool 또는 `"pingpong"`), `easing`(허용 목록) 규칙을 준수해야 합니다.
   - `vivo:runtime` (v1.5.0 신설):
     - 펜스 본문에 구분선(`^---(?:[ \t]+(\S.*?))?[ \t]*$`)이 최소 1개 이상 존재해야 합니다.
     - 첫 구분선 이전의 헤더는 유효한 JSON이어야 합니다.
     - `lang`은 허용 언어 목록(`python`, `javascript`, `typescript`, `sql`, `web`, `c`, `cpp`, `rust`, `java`, `go`)에 속해야 하며, `mode`는 `"wasm"`, `"web"`, `"native"` 중 하나이고 4.3.8.4절의 `(lang, mode)` 허용 조합표를 준수해야 합니다.
     - 헤더에 실행 명령/경로를 지정하는 금지 키(`command`, `args`, `compile_args`, `run_args`, `cwd`, `env`, `interpreter`, `path`, `shell`, `exec`)가 포함되어 있어서는 안 됩니다.
     - 모든 파일명(엔트리 포함)은 정규식 `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`를 만족해야 하며 경로 탈출 문자(`/`, `\`, `..`)가 없고 중복되지 않아야 합니다.
     - 두 번째 이후의 구분선은 반드시 파일명을 동반해야 합니다.
     - 헤더의 `entry`가 지정된 경우, 첫 구분선에 파일명이 명시되어 있다면 두 값이 일치해야 합니다.
     - 파일 개수는 최대 10개, 파일당 크기는 64KB 이하여야 합니다.
     - `read_only_lines`는 양의 정수 배열이어야 하며 각 행 번호는 엔트리 파일의 행 수 이하여야 합니다.
     - `mode: "native"`일 때 `autorun: true` 지정은 거부됩니다.
     - `packages` 필드는 `lang == "python" && mode == "wasm"` 조합에서만 허용되며, 허용 목록(`numpy`, `matplotlib`)에 속해야 합니다.
     - `height`(160~1200), `timeout_ms`(1000~60000) 범위를 만족해야 합니다.
   - **보안 검증(카드=데이터 원칙)**: frontmatter 값, `vivo:runtime`의 헤더 JSON 값, 그리고 `vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`의 모든 문자열 필드에 `<script`, `on\w+=`(인라인 이벤트 핸들러), `javascript:` 패턴이 포함되어서는 안 되며, 발견 시 즉시 업로드가 거부됩니다. (단, `vivo:runtime`의 코드 본문은 실습 대상 코드이므로 문자열 스캔을 적용하지 않습니다.)
6. **라이선스(`license`) 필드 검증**:
   - `license` 필드를 명시하는 경우, 4.1.1절에 정의된 사전 정의 값(`CC-BY-4.0`, `CC-BY-SA-4.0`, `CC-BY-NC-4.0`, `CC-BY-NC-SA-4.0`, `CC-BY-ND-4.0`, `CC-BY-NC-ND-4.0`, `CC0-1.0`, `all-rights-reserved`) 또는 `"custom"` 중 하나여야 합니다. 그 외 임의 문자열은 검증 오류로 처리됩니다.
   - `license` 필드를 생략하면 `"all-rights-reserved"`가 기본값으로 적용됩니다.
   - `license`가 `"custom"`인 경우 `license_file` 필드가 반드시 존재해야 하며, 그 값이 가리키는 파일이 ZIP 루트에 실제로 포함되어 있어야 합니다. 누락 시 업로드가 거부됩니다.
   - `license`가 `"custom"`이 아닌 경우 `license_file`은 필수는 아니지만, 4.1.2절의 제3자 리소스 고지가 필요하면 선택적으로 지정할 수 있습니다. 지정한 경우 그 값이 가리키는 파일이 ZIP 루트에 실제로 포함되어 있어야 합니다.
7. **레거시(v1.3.0 이하) 카드 검증**:
   - `bundler_protocol_version`이 `1.3.0` 이하인 번들에 한해, 부록 A에 기술된 동영상 카드(`type:"video"`)·애니메이션 카드(`type:"animation"`) 스키마 검증 규칙이 그대로 적용됩니다(kind/effect/trigger/event 허용 목록, id 참조 무결성, `images/` 실존 검사, 스크립트 삽입 금지 포함). `1.4.0` 이상인 번들에는 `cards/*.json` 자체가 허용되지 않으므로(2번 항목) 이 규칙은 적용되지 않습니다.

8. **다국어 오버레이(`locales/`) 검증 (v1.6.0 신설)**:
   - `locales/` 폴더가 존재하면, 그 하위 각 언어 폴더명은 `en`/`vi`/`id` 중 하나여야 하며 (base language와 동일한 폴더는 금지 — 예: `base_language`가 `ko`인데 `locales/ko/`를 두는 것은 중복이므로 거부), `package-manifest.json`의 `supported_languages`에 해당 언어 코드가 포함되어 있어야 합니다.
   - `locales/<lang>/` 폴더가 존재하면 `locales/<lang>/translation-status.json`이 **반드시** 존재해야 하며, 유효한 JSON이어야 하고 4.4.6절 스키마(특히 `language`, `source_language`, `status` Mandatory 필드 및 허용값)를 만족해야 합니다.
   - `locales/<lang>/config.json`이 존재하는 경우, 4.4.4.1절 규칙(파일명 집합·트리 모양·`filename` 완전 동일, `card_style` 무시)을 만족해야 합니다.
   - `locales/<lang>/cards/`, `locales/<lang>/images/`, `locales/<lang>/assets/**`에 존재하는 파일은 그 파일 자체에 대해 4.3절(카드) 및 4.4.5절(에셋 오버라이드) 및 기존 이미지/에셋 검증 규칙이 **동일하게** 적용됩니다(예: 카드 확장자 제한, GLB 매직바이트 검사 등 — 언어 오버라이드라고 검증이 느슨해지지 않음).
   - `translation-status.json`의 `items[].path`에 나열된 경로는 실제로 `locales/<lang>/` 아래 존재하는 파일과 일치해야 합니다(`status`가 `not_translated`인 항목 제외). 반대로 `locales/<lang>/` 아래 실존하지만 `items[]`에 나열되지 않은 파일은 경고 처리(정보 누락). **`items[]` 원소에 명세에 없는 추가 필드(구현체별 확장, 예: 재번역 감지용 `source_hash`)가 포함되어 있어도 거부하지 않으며 무시합니다.**
   - `supported_languages`에 나열되었으나 `locales/<lang>/` 폴더 자체가 없는 언어는 거부(선언만 하고 콘텐츠가 전혀 없는 상태 금지). 단, 폴더는 있고 안이 비어 있어(=모든 항목이 base로 폴백) `translation-status.json`만 있는 경우는 `not_translated` 상태로 허용(번역 착수 전 자리표시 용도).

---

## 7. 학습 카드 마크다운 가이드라인 (Markdown Card Guidelines)

가독성 높고 일관성 있는 학습 화면 UI를 위해 학습 카드 마크다운 작성 시 아래 가이드라인을 준수할 것을 권장합니다.

1. **헤더 레벨 (H1, H2) 사용**:
   - 학습 카드의 최상단 타이틀은 `# 챕터 및 레슨 제목` (H1) 형태로 시작해야 합니다.
   - 본문 내 주요 주제 구분은 `## 주제` (H2), 하위 항목은 `### 세부 주제` (H3)를 사용하여 구조화합니다. H1 및 H2 요소는 학습 화면에서 하단 경계선과 굵은 볼드 폰트 스타일로 자동 강조됩니다.
2. **테이블 표 (Tables)**:
   - 강좌 제작 시 데이터 비교나 매핑 정보를 표현할 때는 GFM(GitHub Flavored Markdown) 규격인 파이프 기호(`|`)를 사용한 테이블 표를 사용해야 합니다.
   - 예시:
     ```markdown
     | 통신 방식 | 주요 특징 | 추천 활용 상황 |
     | :--- | :--- | :--- |
     | USB | 유선 연결, 디버깅 용이 | PC/스마트폰과 직접 연동 시 |
     | Wi-Fi | 무선 연결, 인터넷 웹 통신 | 클라우드 데이터 전송 시 |
     ```
   - 플랫폼 학습 화면은 이 마크다운 테이블을 반응형 테두리와 정돈된 패딩 스타일을 지닌 프리미엄 테이블 UI로 자동 파싱하여 렌더링합니다.
3. **코드 블록 및 구문 강조 (Code Blocks & Syntax Highlighting)**:
   - 아두이노 스케치 소스 코드 및 프로그래밍 코드를 포함할 때는 반드시 코드 블록 기호(백틱 3개, ` ``` `)를 사용해야 합니다.
   - 이때 반드시 코드 블록 바로 옆에 언어 식별자(예: `cpp`, `arduino`, `javascript`, `json` 등)를 명시하여 소스 코드가 가독성 있게 강조되도록 합니다. **단, `vivo:video`/`vivo:motion`/`vivo:lottie`/`vivo:3d`/`vivo:runtime` 언어 식별자는 일반 구문 강조가 아니라 4.3.2절의 임베드 컴포넌트로 특별 처리됩니다.**
   - 예시:
     ```arduino
     void setup() {
       Serial.begin(9600);
     }
     ```
   - 플랫폼 학습 화면은 이 코드 블록을 다크 모드 기반의 프리미엄 코드 박스(상단 헤더에 언어 표시 및 1-클릭 클립보드 복사 버튼 내장)로 변환하여 제공합니다.

---

## 부록 A. Legacy v1.3.0 이하 카드 포맷 (읽기 전용, 신규 제작 금지)

> 아래 스펙은 `bundler_protocol_version`이 `1.3.0` 이하인 기존 배포 번들의 재생 호환을 위해 보존됩니다. **신규 강좌 제작 시 이 절의 카드 타입을 사용하지 마십시오** — 4.3절의 단일 강좌 카드(`vivo:video`/`vivo:motion` 임베드)로 대체되었습니다.

### A.1 동영상 카드 JSON 스펙 (`cards/*.json`, `type: "video"`)

```json
{
  "title": "개념 설명 영상",
  "type": "video",
  "video_info": {
    "provider": "youtube",
    "video_id": "dQw4w9WgXcQ",
    "duration_seconds": 212,
    "subtitles": [
      { "start": 0.0, "end": 5.5, "text": "안녕하세요! 이번 시간에는 리액트의 기본 원리에 대해 알아보겠습니다." },
      { "start": 5.6, "end": 12.0, "text": "먼저 컴포넌트란 무엇인지 정의부터 살펴볼까요?" }
    ]
  }
}
```

- `title`: 카드의 제목입니다.
- `type`: 카드의 물리 타입으로 반드시 `"video"` 여야 합니다.
- `video_info`: 동영상 상세 정보 객체입니다. `provider`(현재 `"youtube"` 고정), `video_id`(유튜브 영상 고유 ID), `duration_seconds`(선택), `subtitles`(선택, `start`/`end`/`text` 배열)

### A.2 애니메이션·인터랙션 카드 JSON 스펙 (`cards/*.json`, `type: "animation"`)

학습 카드가 도형·이미지·텍스트로 구성된 2D 애니메이션 또는 클릭 인터랙션(PPT식 슬라이드 진행, 다이어그램 단계별 조립, 클릭 시 반응 등)일 경우, 마크다운이나 동영상 카드 대신 이 절의 **"Scene/Slide JSON DSL"** 스펙을 따르는 구조화 JSON을 사용합니다.

> 이 스펙은 **선언적 데이터**만 표현합니다. `<script>` 태그, 인라인 이벤트 핸들러, `javascript:` URI, 임의의 HTML/CSS 등 실행 가능한 코드는 이 카드 타입에 절대 포함될 수 없으며, 문자열 필드에 이러한 패턴이 발견되면 검증 오류로 거부됩니다. 실제로 애니메이션을 재생하는 코드는 카드 파일이 아니라 항상 플랫폼 앱(재생 엔진)에 고정 탑재된 신뢰 코드가 담당합니다.

```json
{
  "title": "반응 속도 개념 다이어그램",
  "type": "animation",
  "scene_info": {
    "version": "1.0",
    "canvas": { "width": 960, "height": 540, "background": "#ffffff" },
    "slides": [
      {
        "id": "slide-1",
        "elements": [
          { "id": "el-title", "kind": "text", "x": 60, "y": 40, "width": 600, "height": 80,
            "content": "반응 속도란?",
            "style": { "font_size": 36, "font_weight": "bold", "color": "#111827", "align": "left" },
            "initial": { "opacity": 0 } },
          { "id": "el-box-a", "kind": "rect", "x": 100, "y": 200, "width": 120, "height": 120,
            "style": { "fill": "#10b981", "stroke": "#065f46", "stroke_width": 2, "corner_radius": 8 },
            "initial": { "opacity": 0, "scale": 0.8 } },
          { "id": "el-arrow", "kind": "path", "d": "M240 260 L360 260",
            "style": { "stroke": "#374151", "stroke_width": 3 },
            "initial": { "opacity": 0 } },
          { "id": "el-diagram", "kind": "image", "x": 400, "y": 150, "width": 300, "height": 220,
            "src": "images/reaction-diagram.png",
            "initial": { "opacity": 0 } }
        ],
        "steps": [
          { "trigger": "on_enter", "actions": [
            { "target": "el-title", "effect": "fade_in", "duration": 0.6, "delay": 0, "easing": "ease_out" } ] },
          { "trigger": "on_click", "actions": [
            { "target": "el-box-a", "effect": "fade_in", "duration": 0.5, "easing": "ease_out" },
            { "target": "el-box-a", "effect": "scale_to", "to": 1.0, "duration": 0.5, "easing": "ease_out" } ] },
          { "trigger": "on_click", "actions": [
            { "target": "el-arrow", "effect": "draw_path", "duration": 0.8, "easing": "linear" } ] },
          { "trigger": "on_click", "actions": [
            { "target": "el-diagram", "effect": "fade_in", "duration": 0.6, "easing": "ease_out" } ] }
        ],
        "interactions": [
          { "event": "click", "target": "el-box-a", "action": { "effect": "highlight", "duration": 0.4 } }
        ]
      }
    ]
  }
}
```

**최상위 구조**: `title`(String, Mandatory), `type`(String, Mandatory, `"animation"`), `scene_info`(Object, Mandatory: `version` Optional 기본 `"1.0"`, `canvas` Mandatory, `slides` Mandatory 배열 최소 1개).

**슬라이드(`slides[]`) 구조**: `id`(Mandatory, 카드 내 유일), `elements`(Mandatory), `steps`(Optional, 기본 빈 배열), `interactions`(Optional, 기본 빈 배열).

**요소(`elements[]`) 구조**: `id`(Mandatory, 슬라이드 내 유일), `kind`(Mandatory, 허용 목록: `rect`/`circle`/`ellipse`/`line`/`path`/`polygon`/`text`/`image`/`group`), `x`/`y`(Mandatory, `group` 제외), `width`/`height`(Conditional), `content`(Conditional, `text`), `src`(Conditional, `image`, `images/` 기준 상대경로), `d`(Conditional, `path`), `points`(Conditional, `line`/`polygon`), `style`(Optional, 텍스트 자동맞춤 필드는 4.3.4.4절과 동일 규칙 — `wrap`/`min_font_size`/`overflow`/`vertical_align`/`line_height`), `initial`(Optional).

**액션(`steps[].actions[]`, `interactions[].action`)**: `target`(Mandatory, 같은 슬라이드 id 참조), `effect`(Mandatory, 허용 목록: `fade_in`/`fade_out`/`move_to`/`scale_to`/`rotate_to`/`show`/`hide`/`highlight`/`draw_path`), `to`(Conditional), `duration`(Optional 기본 `0.5`), `delay`(Optional 기본 `0`), `easing`(Optional 기본 `"ease_out"`).

**트리거**: `steps[].trigger`는 `"on_enter"`/`"on_click"`, `interactions[].event`는 `"click"`만 허용.

### A.3 레거시 검증 규칙

- 동영상 카드: `title` 누락 시 거부, `type`은 반드시 `"video"`, `video_info` 객체 및 `video_info.provider`(`"youtube"`만 허용)·`video_info.video_id`(문자열) 필수, `video_info.subtitles` 지정 시 배열 타입.
- 애니메이션 카드: `title` 누락 시 거부, `type`은 반드시 `"animation"`, `scene_info`/`scene_info.canvas`/`scene_info.slides`(최소 1개) 필수, 슬라이드 `id` 카드 내 유일, 요소 `id` 슬라이드 내 유일, `elements[].kind` 허용 목록 준수, `kind:"image"`의 `src`가 `images/` 폴더 내 실존, `steps[].actions[].target`/`interactions[].target`의 id 참조 무결성, `steps[].actions[].effect`/`interactions[].action.effect` 허용 목록 준수(`morph_path` 등 미지원 값 거부), `steps[].trigger`/`interactions[].event` 허용 목록 준수, kind/effect별 Mandatory/Conditional 필드 누락 검사, 문자열 필드의 `<script`/`on\w+=`/`javascript:` 스캔, 텍스트 자동맞춤 필드(`overflow`/`vertical_align`/`min_font_size`) 검증.

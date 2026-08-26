# 강좌 다국어(Multilingual) 지원 스키마 변경 계획서

**문서 상태:** Draft (계획 단계 — protocol.md/protocol-changelog.md 실파일 반영 전)
**작성 근거:** VPM 사전 확인 — 현재 `protocol.md` 90행 `language` 필드는 강좌 패키지 최상위에 문자열 하나로만 정의되어 있으며(기본값 `"ko"`), 강좌 하나 = 언어 하나 구조입니다.
**대상 반영 버전(제안):** `v1.6.0` (본 문서는 계획서이며, 실제 `protocol.md`/`protocol-changelog.md` 수정은 별도 Gate 통과 후 후속 지시로 진행)

---

## 0. 요구사항 요약

1. 하나의 강좌(course)가 여러 언어 버전을 갖고, Vivo Academy에서 **같은 강좌 안에서 언어를 전환**해서 볼 수 있어야 한다. 가격/공개 여부 등 강좌 메타데이터는 언어와 무관하게 강좌 단위로 공유된다.
2. 선택된 언어(en/vi/id)의 콘텐츠는 **AI 자동 번역 초안**으로 생성되고, 강사가 Vivo Studio에서 검수/수정한다. 언어별 **번역 상태**(draft/reviewed/published 등)를 추적할 수 있어야 한다.
3. 텍스트뿐 아니라 이미지/다이어그램 등 **에셋도 언어별로 오버라이드** 가능해야 한다 (공통 에셋 공유 + 필요 시 언어별 교체).
4. 지원 언어: 기본 `ko`, 선택적으로 `en`/`vi`/`id`.
5. 기존 단일언어 강좌 패키지는 **마이그레이션 없이** 그대로 동작해야 한다(하위 호환).

---

## 1. 설계 원칙

기존 프로토콜이 이미 채택한 두 가지 원칙을 다국어 지원에도 그대로 확장 적용합니다.

- **오버레이(Overlay) 방식**: 새 언어를 위해 강좌 전체를 복제하지 않습니다. 기본 언어(base language) 콘텐츠가 이미 루트에 존재하므로, 각 언어는 "달라지는 파일만" 추가로 두는 **오버레이 폴더**로 표현합니다. 오버레이 폴더에 파일이 없으면 항상 루트(base) 파일로 **폴백**합니다.
- **선언적 데이터 우선**: 카드 본문(`cards/*.md`, `wiki.md`) 안의 `vivo:video`/`vivo:motion` 등 임베드 JSON은 카드 파일의 일부이므로, 카드 파일 하나를 번역하면 그 안의 자막(`subtitles[].text`)이나 다이어그램 텍스트(`elements[].content`)도 함께 번역됩니다. 별도의 "임베드 텍스트 전용 번역 포맷"을 새로 만들 필요가 없습니다.

이 두 원칙 덕분에 하위 호환은 자동으로 따라옵니다: 오버레이 폴더(`locales/`)가 없는 기존 ZIP은 오늘과 완전히 동일하게 동작합니다.

---

## 2. a. 디렉토리/파일 구조안

### 2.1 신규 최상위 폴더: `locales/`

```text
[강좌 패키지 ZIP 파일]
├── package-manifest.json           # 강좌 단위 공유 메타데이터 (가격/공개여부/카테고리 등) — 언어 무관, 변경 없음
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

### 2.2 경로 해석(Resolution) 규칙 — 모든 언어별 자원에 공통 적용

강좌를 언어 `L`로 렌더링할 때, 임의의 상대 경로 `p` (예: `wiki.md`, `config.json`, `cards/01_intro.md`, `images/schema.png`, `assets/motion/01_variables.json`)를 조회하는 절차:

1. `L`이 base language(기본값 `ko`)이면 → 항상 루트의 `p`를 사용.
2. `L`이 base language가 아니면 →
   a. `locales/<L>/<p>` 파일이 ZIP/저장소 내에 실존하면 그것을 사용.
   b. 존재하지 않으면 루트의 `p`로 **폴백**.

이 규칙 하나로 "텍스트 오버라이드"와 "에셋 오버라이드"를 동일하게 설명할 수 있습니다 (3장/4장 참조).

### 2.3 기존 구조와의 호환

- `locales/` 폴더는 **완전히 선택 항목**입니다. 없으면 3장(현행 구조)이 그대로 적용됩니다.
- `cards/`, `images/`, `assets/`의 기존 하위 규칙(허용 확장자, `assets/motion|lottie|models` 서브폴더 규칙 등)은 `locales/<lang>/` 하위에서도 동일하게 적용됩니다 — 구조를 그대로 복제하되 파일 존재 여부만 선택적입니다.

---

## 3. b. 언어별 텍스트 콘텐츠 오버라이드 방식

| 대상 | 오버라이드 단위 | 방식 |
| :--- | :--- | :--- |
| 위키 본문 | 파일 전체 | `locales/<lang>/wiki.md` 존재 시 전체 교체 |
| 카드 본문(및 그 안의 임베드 JSON 텍스트) | 파일 전체 | `locales/<lang>/cards/<파일명>.md` 존재 시 전체 교체. `vivo:video`의 `subtitles[].text`, `vivo:motion`의 `elements[].content` 등 카드 안에 인라인된 텍스트는 카드 파일을 번역하는 것만으로 함께 번역됨 (별도 스키마 불필요) |
| 목차(TOC) 제목/설명 | 파일 전체 | `locales/<lang>/config.json` 존재 시 TOC의 `title`/`description` 등 텍스트가 번역된 버전으로 전체 교체 |
| 강좌 패키지 자체의 `title`/`description`/`tags`/`changelog` (package-manifest.json) | 개별 필드 | 5장 참조 — course-level 메타데이터는 언어 무관 공유가 원칙이나, 표시용 제목/설명만 예외적으로 언어별 오버라이드 허용 (`locales/<lang>/manifest-overrides.json`, 5.3절) |

### 3.1 `config.json` 번역판 검증 규칙(제안)

TOC 구조 자체(트리 모양, `filename` 매핑)는 언어에 따라 달라지면 안 되므로(같은 강좌를 다른 언어로 보는 것이지 다른 강좌가 아님), `locales/<lang>/config.json`은 다음을 만족해야 합니다.

- `cards` 배열의 파일명 집합이 루트 `config.json`의 `cards` 배열과 **동일**해야 함 (번역 파일이 있든 없든, 참조하는 base 카드 파일명 기준).
- `toc` 트리의 노드 개수·순서·`type`·`filename`이 루트 `config.json`과 **완전히 동일**해야 함. 오직 `title`/`description`(및 4.1절에서 정의할 노드별 번역 상태 참조용 필드)만 달라질 수 있음.
- `card_style`은 언어별로 달라질 이유가 없으므로, 언어별 `config.json`에 지정되어 있어도 **무시**하고 항상 루트 값을 사용 (일관성 유지, 혼란 방지).

이 규칙은 "번역자가 실수로 구조를 바꿔서 두 언어 버전이 다른 강좌처럼 어긋나는" 사고를 검증 단계에서 차단하기 위함입니다.

---

## 4. c. 언어별 에셋(이미지/다이어그램) 오버라이드 방식

### 4.1 공유 vs 교체 판단 규칙

기본 원칙은 **"에셋은 기본적으로 공유하고, 언어 의존적 콘텐츠가 박혀 있을 때만 교체"**입니다. 검증기가 자동으로 강제할 수는 없는 저작 가이드라인이므로, Vivo Studio UI/가이드 문서 수준에서 아래 판단 기준을 안내합니다.

| 상황 | 권장 |
| :--- | :--- |
| 이미지/GLB 모델이 순수 시각 자료이며 텍스트 라벨이 없음 (사진, 아이콘, 범용 3D 모델 등) | **공유** — 언어별 오버라이드 금지(또는 불필요). `locales/<lang>/` 아래 동일 경로 파일을 만들지 않음 |
| 이미지/다이어그램 안에 한국어(또는 원어) 텍스트 라벨이 렌더링되어 있음 (`images/schema.png` 안에 "입력값" 같은 한글 라벨) | **교체** — 해당 언어 라벨로 다시 그린 이미지를 `locales/<lang>/images/schema.png`로 배치 |
| `vivo:motion` 외부 참조 JSON(`assets/motion/*.json`)의 `elements[].content`(텍스트 요소)에 언어 의존 텍스트가 있음 | **교체** — `locales/<lang>/assets/motion/<동일 파일명>.json`으로 텍스트만 번역된 버전을 배치. `canvas`/도형 좌표 등 비텍스트 요소는 동일하게 유지 권장 |
| `vivo:lottie`(`assets/lottie/*.json`) 안에 텍스트 레이어가 있음 | **교체** — 동일 방식으로 `locales/<lang>/assets/lottie/<동일 파일명>.json` |
| `vivo:3d`(`assets/models/*.glb`) | 원칙적으로 **공유** (3D 모델에 텍스트 라벨이 각인된 특수 케이스가 아니면 언어별 교체 불필요). 30MB 제한이 있는 큰 바이너리이므로 불필요한 언어별 중복 배치는 지양 |

### 4.2 메커니즘

이미지/에셋 오버라이드는 3장의 "경로 해석 규칙"을 그대로 재사용합니다. 즉 새로운 스키마가 필요 없고, `locales/<lang>/images/<파일명>`, `locales/<lang>/assets/<motion|lottie|models>/<파일명>`에 **동일 상대경로**로 파일을 두기만 하면 검증기/재생 엔진이 자동으로 우선 사용합니다.

### 4.3 검증 규칙(제안)

- `locales/<lang>/images/`, `locales/<lang>/assets/**`에 존재하는 모든 파일은, 그 파일명이 카드(base 또는 해당 언어 오버라이드 카드) 어딘가에서 참조되는 상대경로와 **일치**해야 함 (고아 파일 금지 — 기존 3장/6장의 "미사용 리소스 정합성" 원칙 확장).
- `vivo:3d`(GLB), `vivo:runtime` 파일 실습 코드는 언어별 오버라이드 대상에서 **제외**를 권장(가이드라인 수준). GLB는 4.1절 표 참조. `vivo:runtime` 코드 실습은 학습자가 직접 코드를 읽고 편집하는 대상이라 코드 자체(변수명/문자열 리터럴 등)의 언어별 이원화는 실효성이 낮음 — 필요 시 `title`/`entry` 주석 등 카드 본문 번역으로 흡수.

---

## 5. d. 번역 상태 메타데이터 필드 설계

### 5.1 `locales/<lang>/translation-status.json` (신설)

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

#### 필드 명세

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `language` | String | **Mandatory** | 이 오버레이 폴더가 속한 언어 코드. 폴더명(`en`/`vi`/`id`)과 일치해야 함 |
| `source_language` | String | **Mandatory** | 번역 원본 언어 (통상 `package-manifest.json`의 `base_language`와 동일) |
| `status` | String | **Mandatory** | 해당 언어 전체의 **롤업(rollup) 상태**. `not_translated` / `draft` / `reviewed` / `published` 중 하나. Vivo Academy 언어 선택 UI는 이 값이 `published`인 언어만 학습자에게 노출 (Studio 미리보기에서는 draft/reviewed도 저작자 본인에게는 노출) |
| `generated_by` | String | Optional | 콘텐츠 생성 주체. `ai_auto_translation`(AI 자동 번역) / `human`(수동 작성) / `hybrid` |
| `generated_at` | String (ISO 8601) | Optional | 최초 생성(AI 번역 초안 생성) 시각 |
| `updated_at` | String (ISO 8601) | Optional | 마지막 수정/검수 시각 |
| `items` | Array | Optional | 파일 단위 세부 상태. 생략 시 `status`(롤업 값)가 모든 파일에 동일 적용된 것으로 간주 |

#### `items[]` 원소 명세

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `path` | String | **Mandatory** | base 루트 기준 상대경로 (예: `"cards/01_intro.md"`). 2.2절 경로 해석 규칙의 `p`와 동일한 값 |
| `status` | String | **Mandatory** | `not_translated`(오버라이드 파일 없음, 루트로 폴백 중) / `draft`(AI 생성, 미검수) / `reviewed`(강사 검수 완료) / `published`(검수 완료 + 학습자 공개) |
| `reviewed_at` | String (ISO 8601) | Optional | 검수 완료 시각 |
| `reviewed_by` | String | Optional | 검수자 식별자(닉네임/이메일). PII 노출 최소화를 위해 앱단에서 마스킹 여부는 구현체 재량 |

### 5.2 상태(`status`) 값 정의

| 값 | 의미 |
| :--- | :--- |
| `not_translated` | 해당 언어의 오버라이드 파일이 아직 없음(루트 base 콘텐츠로 자동 폴백 중) |
| `draft` | AI 자동 번역으로 생성된 초안. 강사 미검수 상태 — Vivo Studio에서만 미리보기 가능, 학습자에게는 비공개 |
| `reviewed` | 강사가 Vivo Studio에서 검수·수정 완료. 아직 공개 전(내부 QA 단계) |
| `published` | 검수 완료 + 학습자에게 실제로 노출 중. Vivo Academy 언어 전환 스위처에 이 언어가 나타남 |

> 강좌 자체의 `published`(공개 여부, `package-manifest.json`)와 언어별 `published`(번역 상태)는 **서로 다른 축**입니다. 강좌가 비공개면 어떤 언어든 노출되지 않고, 강좌가 공개여도 특정 언어가 아직 `draft`/`reviewed`면 그 언어만 학습자에게 숨겨집니다.

### 5.3 `package-manifest.json`에 추가할 필드 (요약 — 상세는 8장 초안 참조)

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `base_language` | String | Optional (기본값 `"ko"`) | 강좌의 기본(원본) 언어. 기존 `language` 필드를 대체하되, 하위 호환을 위해 `language`도 계속 허용(별칭) |
| `supported_languages` | Array of String | Optional (기본값 `[base_language]`) | 이 강좌가 제공하는 언어 코드 목록. `ko`/`en`/`vi`/`id` 중에서 선택. `base_language`는 항상 포함되어야 함 |

가격(`price` 등 향후 필드)·`published`·`sequential_play`·`category` 등 기존 course-level 필드는 **언어 무관**으로 유지 — 요구사항 1과 동일.

---

## 6. e. 기존 단일언어 강좌 패키지 하위 호환성

- **구조적 호환**: `locales/` 폴더는 완전 선택 항목이므로, 기존에 배포된 모든 ZIP/DB 레코드는 스키마 변경 없이 그대로 유효합니다.
- **필드 호환**: 신설 필드(`base_language`, `supported_languages`)는 모두 Optional이며 기본값이 기존 동작과 동일합니다(`base_language` 기본값 `"ko"` = 기존 `language` 기본값과 동일, `supported_languages` 기본값 `[base_language]` = "언어 하나짜리 강좌"라는 기존 의미와 동일).
- **`language` 필드 유지**: 기존 `language` 필드는 폐기하지 않고 `base_language`의 별칭(alias)으로 계속 허용합니다. 검증기는 둘 다 있을 때 값이 다르면 경고(오류 아님)만 발생시킵니다. 즉, 기존 제작 도구/에이전트가 `language` 필드만 채워도 계속 정상 동작합니다.
- **마이그레이션 불필요**: `db.json`/`public/courses/[slug]/` 등 기존 저장소 레코드에 대해 별도 배치 마이그레이션 스크립트가 필요 없습니다. 로컬 실행 환경/서버는 압축 해제 시 `locales/` 폴더 존재 여부만 확인하고, 없으면 오늘과 동일한 코드 경로를 그대로 탑니다.
- **재생 엔진 호환**: 구버전(다국어 미지원) 재생 엔진이 `locales/` 필드가 포함된 신규 번들을 만나면, 미지의 최상위 폴더로 안전하게 무시하고 기존처럼 루트 콘텐츠만 재생합니다 (알려지지 않은 임베드를 무시하는 기존 v1.4/v1.5 하위 호환 패턴과 동일한 접근).

---

## 7. 검증(Validation) 규칙 초안 (제6장에 추가될 항목)

1. `locales/` 폴더가 존재하면, 그 하위 각 언어 폴더명은 `en`/`vi`/`id` 중 하나여야 하며 (base language와 동일한 폴더는 금지 — 예: `base_language`가 `ko`인데 `locales/ko/`를 두는 것은 중복이므로 거부), `package-manifest.json`의 `supported_languages`에 해당 언어 코드가 포함되어 있어야 합니다.
2. `locales/<lang>/` 폴더가 존재하면 `locales/<lang>/translation-status.json`이 **반드시** 존재해야 하며, 유효한 JSON이어야 하고 5.1절 스키마(특히 `language`, `source_language`, `status` Mandatory 필드 및 허용값)를 만족해야 합니다.
3. `locales/<lang>/config.json`이 존재하는 경우, 3.1절 규칙(파일명 집합·트리 모양·`filename` 완전 동일, `card_style` 무시)을 만족해야 합니다.
4. `locales/<lang>/cards/`, `locales/<lang>/images/`, `locales/<lang>/assets/**`에 존재하는 파일은 그 파일 자체에 대해 4.3~4.3.x절(카드) 및 기존 이미지/에셋 검증 규칙이 **동일하게** 적용됩니다(예: 카드 확장자 제한, GLB 매직바이트 검사 등 — 언어 오버라이드라고 검증이 느슨해지지 않음).
5. `translation-status.json`의 `items[].path`에 나열된 경로는 실제로 `locales/<lang>/` 아래 존재하는 파일과 일치해야 합니다(`status`가 `not_translated`인 항목 제외). 반대로 `locales/<lang>/` 아래 실존하지만 `items[]`에 나열되지 않은 파일은 경고 처리(정보 누락).
6. `supported_languages`에 나열되었으나 `locales/<lang>/` 폴더 자체가 없는 언어는 거부(선언만 하고 콘텐츠가 전혀 없는 상태 금지). 단, 폴더는 있고 안이 비어 있어(=모든 항목이 base로 폴백) `translation-status.json`만 있는 경우는 `not_translated` 상태로 허용(번역 착수 전 자리표시 용도).

---

## 8. f. protocol.md / protocol-changelog.md 반영 섹션 초안

> 아래는 문구 초안이며, 이번 태스크에서는 실제 파일에 반영하지 않습니다. Gate 통과 후 별도 지시에 따라 적용합니다.

### 8.1 `protocol.md` 반영 초안

**(1) 문서 상단 버전/변경 요지 (1장) 추가 문구:**

> **v1.6.0 변경 요지:** 하나의 강좌가 여러 언어 버전(한국어 기본 + 선택적 영어/베트남어/인도네시아어)을 갖고 Vivo Academy에서 언어를 전환하며 볼 수 있도록 다국어 지원(Multilingual Course Support)을 추가했습니다. 언어별 콘텐츠는 `locales/<lang>/` 오버레이 폴더에 "달라지는 파일만" 배치하는 방식이며, 기존 v1.5.0 이하 단일언어 번들과 완벽히 하위 호환됩니다.

**(2) 제3장 디렉토리 구조 다이어그램에 `locales/` 블록 추가** (본 문서 2.1절 트리를 그대로 삽입).

**(3) 제3장에 하위 호환 안내문 추가:**

> **다국어(v1.6.0) 오버레이 구조:** `locales/` 폴더는 선택 항목이며, `bundler_protocol_version`이 `1.6.0` 미만이거나 `locales/`가 없는 번들은 기존과 동일한 단일 언어 강좌로 취급됩니다. 자세한 스키마는 4.4절을 참조하십시오.

**(4) 제4.1절 `package-manifest.json` 표에 필드 추가:**

| 필드명 | 타입 | 필수 여부 | 설명 | 예시 |
| :--- | :--- | :---: | :--- | :--- |
| `base_language` | String | Optional | 강좌의 기본(원본) 언어 (기본값: `"ko"`). 기존 `language` 필드를 대체하며, 둘 다 지정 시 동일 값이어야 함(다르면 검증 경고) | `"ko"` |
| `supported_languages` | Array of String | Optional | 이 강좌가 제공하는 언어 코드 목록 (기본값: `[base_language]`). `"ko"`/`"en"`/`"vi"`/`"id"` 중에서 선택하며 `base_language`를 반드시 포함해야 함 | `["ko", "en", "vi"]` |

기존 `language` 필드 설명에 다음 문구 추가:

> (v1.6.0부터 `base_language`로 대체되었으나 하위 호환을 위해 계속 지원되는 별칭입니다. 신규 제작 시 `base_language` 사용을 권장합니다.)

**(5) 신규 절 "4.4 다국어 강좌 지원 (Multilingual Course Support, v1.6.0 신설)" 추가:**

본 문서 2~5장 내용(오버레이 구조, 경로 해석 규칙, 텍스트/에셋 오버라이드 방식, `translation-status.json` 스키마)을 그대로 정식 절로 옮겨 작성.

**(6) 제6장 검증 규칙에 항목 추가:** 본 문서 7장 내용을 그대로 8번 항목으로 추가.

### 8.2 `protocol-changelog.md` 반영 초안

```markdown
## [v1.6.0] - <반영일 TBD>

### Added
- **다국어 강좌 지원 (Multilingual Course Support, 제4.4절)**:
  - 하나의 강좌 패키지가 여러 언어 버전(기본 `ko` + 선택적 `en`/`vi`/`id`)을 갖고, Vivo Academy에서 같은 강좌 안에서 언어를 전환하여 볼 수 있습니다. 가격/공개 여부 등 강좌 메타데이터는 언어와 무관하게 강좌 단위로 공유됩니다.
  - 신설 `locales/<lang>/` 오버레이 폴더 구조: 언어별로 "달라지는 파일만" 두고, 없는 파일은 base 언어(루트) 콘텐츠로 자동 폴백합니다. `wiki.md`/`config.json`/`cards/*`/`images/*`/`assets/**` 모두 동일한 경로 해석 규칙을 따릅니다.
  - `package-manifest.json`에 `base_language`(기존 `language`의 후속, 별칭으로 하위호환 유지)와 `supported_languages` 필드를 추가했습니다.
  - 언어별 번역 상태 추적을 위한 `locales/<lang>/translation-status.json` 스키마를 신설했습니다: 언어 전체 롤업 상태(`not_translated`/`draft`/`reviewed`/`published`) 및 파일 단위 세부 상태(`items[]`)를 기록하며, AI 자동 번역 초안(`generated_by: "ai_auto_translation"`) 이후 강사가 Vivo Studio에서 검수하는 워크플로우를 지원합니다. `published` 상태의 언어만 Vivo Academy 학습자 화면의 언어 스위처에 노출됩니다.
  - `config.json` 언어별 오버라이드 시 TOC 트리 구조(`filename`, `type`, 노드 개수/순서)는 base와 동일해야 하며 `title`/`description` 텍스트만 달라질 수 있도록 제한했습니다(구조 드리프트 방지).
  - 제6조 검증 규칙에 다국어 오버레이 검증 항목(8번)을 추가했습니다.

### Non-Breaking / Compatibility
- 기존 v1.5.0 이하 단일언어 강좌 번들과 완벽히 하위 호환됩니다. `locales/` 폴더가 없으면 기존과 완전히 동일하게 동작하며, 별도 마이그레이션이 필요하지 않습니다. `language` 필드는 계속 지원됩니다.
```

---

## 9. 열린 이슈 / 후속 논의 필요 사항 (Gate에서 확인 필요)

- **TOC 챕터(비-leaf) 노드의 안정적 식별자**: 현재 TOC 챕터/섹션 노드에는 `filename`(leaf 전용) 외에 안정적인 `id`가 없습니다. 본 계획은 `config.json` 전체 파일 오버라이드 + "구조 동일성 검증"으로 이 문제를 우회했지만, 향후 노드 단위 부분 오버라이드가 필요해지면 TOC 노드에 선택적 `id` 필드 도입을 재검토해야 합니다.
- **`supported_languages`에서 언어를 제거(철회)하는 경우**: 이미 `published`였던 언어를 다시 비공개로 내리는 시나리오의 상태 전이 규칙(예: `published` → `draft`로 되돌리기 허용 여부)은 Vivo Studio UX 정책에 달려 있어 본 문서 범위 밖으로 남겨둡니다.
- **AI 자동 번역 파이프라인 자체(번역 API 연동, 재번역 트리거 조건 등)**: 본 계획은 산출물 스키마만 다루며, 번역 생성 파이프라인의 구현 방식은 Vivo Studio 쪽 별도 설계 문서 대상입니다.

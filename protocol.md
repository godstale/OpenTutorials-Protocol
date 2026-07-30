# Open Tutorials Course Bundler Protocol Specification

**Version:** 1.2.0  
**Status:** Active  
**Scope:** Open Tutorials Course Packaging & local execution

---

## 1. 개요 (Introduction)

본 문서는 Open Tutorials 플랫폼에 강좌(Course Package)를 등록하고 배포하기 위한 통합 번들 ZIP 파일 구조 및 각 항목의 역할, 정합성 검증 규칙을 규정합니다. 

이 프로토콜은 온디바이스 로컬 실행 환경(`db.json` 및 `public/courses`)과의 완벽한 호환을 보장하며, AI Agent 기반의 자동 강좌 제작 프로젝트에서 참조하여 강좌 파일을 무결성 있게 제작하기 위한 공식 가이드라인입니다.

---

## 2. 강좌 등록 및 처리 프로세스 (Registration Workflow)

Open Tutorials 플랫폼에서 강좌 번들이 처리되고 데이터베이스에 등록되는 단계는 다음과 같습니다.

1. **메타데이터 및 콘텐츠 기획**:
   - 강좌 카테고리, 대상 연령대, 학습 순서 제어 방식 등을 결정합니다.
   - 메인 지식베이스가 될 `wiki.md`와 개별 학습 카드(`cards/*.md`)의 마크다운, 동영상, 애니메이션·인터랙션 콘텐츠를 설계합니다.
2. **패키지 매니페스트(`package-manifest.json`) 작성**:
   - 필수 프로토콜 항목(`bundler_protocol_version`, `target_age`, `category`)을 설정하고 제목 및 기본 정보를 정의합니다. (더 이상 개별 하위 강좌 리스트 `courses`는 지정하지 않습니다.)
3. **패키지 구성 및 config.json 작성**:
   - 루트 경로에 `config.json`, `wiki.md`, `cards/` 디렉토리를 생성합니다.
   - `config.json` 내부에 학습 카드 파일 목록과 목차(TOC) 트리 구조를 정의합니다.
4. **단일 강좌 패키지 ZIP 압축**:
   - 루트에 `package-manifest.json`, `config.json`, `wiki.md`, `thumbnail.png`(선택), `cards/` 및 `images/` (마크다운에서 참조하는 이미지 리소스 폴더, 선택)를 두고 최종 하나의 ZIP 파일로 압축합니다.
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
├── cards/                          # 강의 카드 디렉토리 (필수)
│   ├── 01_intro.md                 # 마크다운 강의 카드
│   ├── 02_video.json               # 동영상 강의 카드 (JSON)
│   └── 03_animation.json           # 애니메이션·인터랙션 카드 (JSON)
└── images/                         # 마크다운 카드 및 애니메이션 카드(image 요소)에서 참조하는 이미지 파일 폴더 (선택)
    └── schema.png
```

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
| `bundler_protocol_version` | String | **Mandatory** | 이 번들이 준수한 번들러 프로토콜 명세 버전 | `"1.2.0"` |
| `target_age` | String | **Mandatory** | 강좌 수강에 권장되는 대상 연령대 (`all` (전연령), `x+` (x세 이상), `min-max` (연령대 범위)) | `"all"`, `"10+"`, `"8-13"` |
| `category` | String | **Mandatory** | 강좌의 대분류 카테고리 | `"Programming"`, `"Design"`, `"Marketing"`, `"Math"` |
| `language` | String | Optional | 강좌 패키지의 주 언어 (기본값: `"ko"`) | `"ko"`, `"en"` |
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

패키지 루트에 위치하는 `config.json`은 강좌의 상세 목차(TOC) 트리와 연결되는 마크다운 및 동영상 파일들의 목록을 기술합니다.

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
  ]
}
```

#### TOC 노드 필드 명세

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `type` | String | **Mandatory** | 노드의 성격 (`chapter`, `section`, `subsection` 중 하나) |
| `title` | String | **Mandatory** | 목차에 렌더링될 실제 한글/외국어 제목 (파일명과 달라야 함) |
| `description` | String | **Mandatory** | 해당 단원의 요약 설명 (기본 템플릿 값 방치 금지) |
| `filename` | String | Conditional | 말단 노드(`section` 또는 `subsection`)인 경우 연결될 학습 카드 파일명. 마크다운(`.md`/`.mdx`) 또는 동영상 카드(`.json`) 형식 지원. (예: `"01_intro.md"`, `"02_video.json"`) |
| `children` | Array | Conditional | 상위 노드(`chapter` 등)인 경우 하위 TOC 노드 배열 |

---

### 4.3 동영상 카드 JSON 스펙 (`cards/*.json`)
학습 카드가 동영상일 경우, 기존 마크다운 파일 대신 구조화된 JSON 형식을 사용합니다. 파일의 필수 스키마는 다음과 같습니다.

```json
{
  "title": "개념 설명 영상",
  "type": "video",
  "video_info": {
    "provider": "youtube",
    "video_id": "dQw4w9WgXcQ",
    "duration_seconds": 212,
    "subtitles": [
      {
        "start": 0.0,
        "end": 5.5,
        "text": "안녕하세요! 이번 시간에는 리액트의 기본 원리에 대해 알아보겠습니다."
      },
      {
        "start": 5.6,
        "end": 12.0,
        "text": "먼저 컴포넌트란 무엇인지 정의부터 살펴볼까요?"
      }
    ]
  }
}
```

##### 필드 상세 정보
- `title`: 카드의 제목입니다.
- `type`: 카드의 물리 타입으로 반드시 `"video"` 여야 합니다.
- `video_info`: 동영상 상세 정보 객체입니다.
  - `provider`: 영상 제공 플랫폼 (현재 `"youtube"` 고정)
  - `video_id`: 유튜브 영상 고유 ID (예: `dQw4w9WgXcQ`)
  - `duration_seconds`: (선택) 동영상 전체 길이(초)
  - `subtitles`: (선택) 시간별 자막 정보 배열
    - `start`: 자막 노출 시작 시점 (초 단위 실수)
    - `end`: 자막 노출 종료 시점 (초 단위 실수)
    - `text`: 해당 구간의 자막 텍스트

---

### 4.4 애니메이션·인터랙션 카드 JSON 스펙 (`cards/*.json`, `type: "animation"`)

학습 카드가 도형·이미지·텍스트로 구성된 2D 애니메이션 또는 클릭 인터랙션(PPT식 슬라이드 진행, 다이어그램 단계별 조립, 클릭 시 반응 등)일 경우, 마크다운이나 동영상 카드 대신 이 절의 **"Scene/Slide JSON DSL"** 스펙을 따르는 구조화 JSON을 사용합니다.

> 이 스펙은 **선언적 데이터**만 표현합니다. `<script>` 태그, 인라인 이벤트 핸들러, `javascript:` URI, 임의의 HTML/CSS 등 실행 가능한 코드는 이 카드 타입에 절대 포함될 수 없으며, 문자열 필드에 이러한 패턴이 발견되면 검증 오류로 거부됩니다(6장 참조). 실제로 애니메이션을 재생하는 코드는 카드 파일이 아니라 항상 플랫폼 앱(재생 엔진)에 고정 탑재된 신뢰 코드가 담당합니다 — "카드 = 데이터, 재생 코드 = 앱 소유" 원칙은 이 카드 타입에도 100% 동일하게 적용됩니다.

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
          },
          {
            "id": "el-arrow",
            "kind": "path",
            "d": "M240 260 L360 260",
            "style": { "stroke": "#374151", "stroke_width": 3 },
            "initial": { "opacity": 0 }
          },
          {
            "id": "el-diagram",
            "kind": "image",
            "x": 400, "y": 150, "width": 300, "height": 220,
            "src": "images/reaction-diagram.png",
            "initial": { "opacity": 0 }
          }
        ],
        "steps": [
          {
            "trigger": "on_enter",
            "actions": [
              { "target": "el-title", "effect": "fade_in", "duration": 0.6, "delay": 0, "easing": "ease_out" }
            ]
          },
          {
            "trigger": "on_click",
            "actions": [
              { "target": "el-box-a", "effect": "fade_in", "duration": 0.5, "easing": "ease_out" },
              { "target": "el-box-a", "effect": "scale_to", "to": 1.0, "duration": 0.5, "easing": "ease_out" }
            ]
          },
          {
            "trigger": "on_click",
            "actions": [
              { "target": "el-arrow", "effect": "draw_path", "duration": 0.8, "easing": "linear" }
            ]
          },
          {
            "trigger": "on_click",
            "actions": [
              { "target": "el-diagram", "effect": "fade_in", "duration": 0.6, "easing": "ease_out" }
            ]
          }
        ],
        "interactions": [
          {
            "event": "click",
            "target": "el-box-a",
            "action": { "effect": "highlight", "duration": 0.4 }
          }
        ]
      }
    ]
  }
}
```

#### 4.4.1 최상위 구조

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `title` | String | **Mandatory** | 카드의 제목 |
| `type` | String | **Mandatory** | 반드시 `"animation"` 이어야 함 |
| `scene_info` | Object | **Mandatory** | 씬 전체를 담는 컨테이너 객체 |
| `scene_info.version` | String | Optional | DSL 자체의 스펙 버전 (기본값 `"1.0"`, 현재 유일한 값) |
| `scene_info.canvas` | Object | **Mandatory** | 좌표계 기준이 되는 캔버스 크기/배경. `width`(Number), `height`(Number), `background`(String, CSS 색상값) 하위 필드를 가짐 |
| `scene_info.slides` | Array | **Mandatory** | 카드 내부에서 순차 진행되는 슬라이드(페이지) 배열, 최소 1개 이상. 슬라이드 간 이동(이전/다음)은 학습 화면이 카드 자체의 컨트롤로 제공하며, TOC상의 챕터/카드 이동과는 별개의 카드 내부 단계임 |

#### 4.4.2 슬라이드(`slides[]`) 구조

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `id` | String | **Mandatory** | 카드 내에서 유일한 슬라이드 식별자 |
| `elements` | Array | **Mandatory** | 해당 슬라이드에 배치되는 도형/이미지/텍스트 요소 목록 |
| `steps` | Array | Optional | 슬라이드 진입 시 순서대로 재생되는 애니메이션 시퀀스(reveal.js의 fragment 리빌 패턴). 기본값: 빈 배열(정적 슬라이드) |
| `interactions` | Array | Optional | `steps` 진행 여부와 무관하게, 특정 요소를 클릭했을 때 즉시 반응하는 독립적인 이벤트 바인딩 목록. 기본값: 빈 배열 |

#### 4.4.3 요소(`elements[]`) 구조

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `id` | String | **Mandatory** | 같은 슬라이드 내에서 유일한 요소 식별자. `steps`/`interactions`가 이 id로 요소를 참조함 |
| `kind` | String | **Mandatory** | 요소의 종류. 아래 4.4.3.1의 허용 목록 중 하나여야 함 |
| `x`, `y` | Number | **Mandatory** (`group` 제외) | 캔버스 좌표계 기준 위치(좌상단) |
| `width`, `height` | Number | Conditional | `rect`/`image`/`text`/`group`은 필수. `circle`(반지름은 `style.radius`)/`path`/`line`/`polygon`은 불필요 |
| `content` | String | Conditional | `kind: "text"`일 때 **Mandatory**. 표시할 순수 텍스트(마크다운·HTML 불가) |
| `src` | String | Conditional | `kind: "image"`일 때 **Mandatory**. ZIP 루트의 `images/` 폴더 기준 상대경로 (예: `"images/diagram.png"`) |
| `d` | String | Conditional | `kind: "path"`일 때 **Mandatory**. SVG `<path>`의 `d` 속성과 동일한 문법의 경로 데이터 |
| `points` | String | Conditional | `kind: "line"` 또는 `"polygon"`일 때 **Mandatory**. `"x1,y1 x2,y2 ..."` 형식의 좌표 목록 |
| `style` | Object | Optional | `fill`, `stroke`, `stroke_width`, `corner_radius`, `radius`(circle 전용), `font_size`, `font_weight`, `color`, `align` 등 kind별 시각 속성. 사전 정의된 CSS 색상값/숫자만 허용하며 임의 CSS 문자열(`url(...)`, `expression(...)` 등)은 금지 |
| `initial` | Object | Optional | 슬라이드 진입 시점의 초기 상태(`opacity`(0~1), `scale`, `rotation`). 기본값은 `{ opacity: 1, scale: 1, rotation: 0 }`이며, `steps`로 이후 등장시킬 요소는 보통 `opacity: 0` 등으로 숨긴 상태에서 시작 |

##### 4.4.3.1 `kind` 허용 목록

`rect`, `circle`, `ellipse`, `line`, `path`, `polygon`, `text`, `image`, `group`

#### 4.4.4 애니메이션 액션 (`steps[].actions[]`, `interactions[].action`)

| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `target` | String | **Mandatory** | 동작을 적용할 요소의 `id`. 반드시 같은 슬라이드의 `elements` 배열에 존재하는 id여야 함 |
| `effect` | String | **Mandatory** | 아래 4.4.4.1 허용 목록 중 하나. 임의 문자열이나 커스텀 함수명은 금지 |
| `to` | Number \| Object | Conditional | `move_to`(좌표 객체 `{x, y}`), `scale_to`(배율 Number), `rotate_to`(각도 Number)일 때 **Mandatory**인 목적값 |
| `duration` | Number | Optional | 애니메이션 지속시간(초). 기본값 `0.5` |
| `delay` | Number | Optional | 시작 지연시간(초). 기본값 `0` |
| `easing` | String | Optional | `"linear"`, `"ease_in"`, `"ease_out"`, `"ease_in_out"` 중 하나. 기본값 `"ease_out"` |

##### 4.4.4.1 `effect` 허용 목록

| 값 | 설명 |
| :--- | :--- |
| `fade_in` / `fade_out` | 불투명도를 0↔1로 전환 |
| `move_to` | 지정 좌표로 이동 |
| `scale_to` | 지정 배율로 확대/축소 |
| `rotate_to` | 지정 각도로 회전 |
| `show` / `hide` | 트랜지션 없이 즉시 표시/숨김 |
| `highlight` | 강조 펄스(외곽선/색상 플래시) 효과 |
| `draw_path` | `kind: "path"` 요소 전용. 선이 그려지는 듯한 효과로 순차 노출 |

> `morph_path`(경로 변형) 등 추가 효과는 향후 버전에서 확장 예약된 항목이며, 현재 v1.2.0 기준으로는 지원 목록에 없으므로 사용할 수 없습니다.

#### 4.4.5 트리거(`steps[].trigger`)

| 값 | 설명 |
| :--- | :--- |
| `on_enter` | 슬라이드가 화면에 표시되는 즉시 자동 재생 |
| `on_click` | 사용자가 화면(또는 카드가 제공하는 "다음" 컨트롤)을 클릭해야 다음 단계로 진행 (reveal.js의 fragment 클릭 리빌과 동일한 모델) |

`interactions[].event`는 현재 `"click"` 값만 지원합니다.

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
  "bundler_protocol_version": "1.2.0",
  "target_age": "10+",
  "category": "Programming",
  "language": "ko",
  "tags": ["Python", "AI", "Beginner"],
  "license": "CC-BY-4.0"
}
```

---

## 6. 제작 시 주의사항 및 검증 규칙 (Strict Validation Rules)

강좌 번들 작성 시 다음 항목 중 하나라도 위배되면 플랫폼 내 업로드 단계에서 검증 오류가 발생하여 등록이 거부됩니다.

1. **상위 디렉토리 미포함 (Flat ZIP)**:
   - ZIP 파일 압축 시, 최상위 경로에 단일 폴더가 있고 그 안에 `package-manifest.json`이 들어있는 이중 레이어 구조는 금지됩니다. ZIP 파일을 열었을 때 바로 `package-manifest.json`, `config.json`, `wiki.md`, `cards/`가 최상위 루트에 존재해야 합니다.
2. **TOC-Card 1:1 매칭 및 파일 정합성**:
   - `config.json`의 `cards` 배열 내 모든 파일명은 실제 ZIP 안의 `cards/` 폴더 내에 정확히 존재해야 하며, 대소문자까지 일치해야 합니다. (마크다운 `.md`, `.mdx` 및 동영상 카드 `.json` 확장자 포함)
   - `toc` 트리에서 `filename`이 명시된 노드는 반드시 `cards` 배열에 있는 파일명 중 하나여야 하며, `cards` 배열에 나열된 모든 파일이 `toc` 트리 어딘가에 한 번씩만 매핑되어야 합니다.
3. **무성의한 기본 텍스트 방치 금지**:
   - `toc` 노드의 `title`이 파일명과 완전히 동일(예: title이 `01_intro`인 경우)하면 안 됩니다. 사용자가 읽기 좋은 언어(예: `1강. 오리엔테이션 및 입문 가이드`)로 설정되어야 합니다.
   - `toc` 노드의 `description`이 `"강좌 상세 카드를 확인하세요."` 와 같은 기본값으로 설정되어 있다면 검증이 반려됩니다. 각 노드에 알맞은 요약 정보가 구체적으로 기술되어야 합니다.
4. **동영상 카드(`cards/*.json`) 스키마 검증**:
   - `title`(String)이 누락되면 업로드 API(`/api/admin/packages/upload`)가 즉시 오류를 반환합니다.
   - `type`은 반드시 문자열 `"video"`여야 합니다.
   - `video_info` 객체가 반드시 존재해야 하며, 그 하위의 `provider`는 현재 `"youtube"` 값만 허용됩니다.
   - `video_info.video_id`는 반드시 존재하는 문자열여야 합니다.
   - `video_info.subtitles`가 포함될 경우 반드시 배열(Array) 타입이어야 합니다.
   - **(권장, 서버 미검증)** `subtitles` 각 원소는 `start`(초 단위 시작 시각) < `end`(초 단위 종료 시각) 관계를 만족하고 시간 순으로 정렬된 상태로 제작해야 합니다. 서버는 배열 타입 여부만 검증하므로, 순서가 어긋나거나 구간이 역전된 자막을 넣어도 업로드는 성공하지만 학습 화면의 "자막 탐색" 패널 하이라이트와 클릭 탐색(seek) 기능이 오작동합니다.
5. **라이선스(`license`) 필드 검증**:
   - `license` 필드를 명시하는 경우, 4.1.1절에 정의된 사전 정의 값(`CC-BY-4.0`, `CC-BY-SA-4.0`, `CC-BY-NC-4.0`, `CC-BY-NC-SA-4.0`, `CC-BY-ND-4.0`, `CC-BY-NC-ND-4.0`, `CC0-1.0`, `all-rights-reserved`) 또는 `"custom"` 중 하나여야 합니다. 그 외 임의 문자열은 검증 오류로 처리됩니다.
   - `license` 필드를 생략하면 `"all-rights-reserved"`가 기본값으로 적용됩니다.
   - `license`가 `"custom"`인 경우 `license_file` 필드가 반드시 존재해야 하며, 그 값이 가리키는 파일이 ZIP 루트에 실제로 포함되어 있어야 합니다. 누락 시 업로드가 거부됩니다.
   - `license`가 `"custom"`이 아닌 경우 `license_file`은 필수는 아니지만, 4.1.2절의 제3자 리소스 고지가 필요하면 선택적으로 지정할 수 있습니다. 지정한 경우 그 값이 가리키는 파일이 ZIP 루트에 실제로 포함되어 있어야 합니다.
6. **애니메이션 카드(`cards/*.json`, `type: "animation"`) 스키마 검증**:
   - `title`(String)이 누락되면 업로드가 거부됩니다. `type`은 반드시 문자열 `"animation"`이어야 합니다.
   - `scene_info` 객체와 그 하위 `scene_info.canvas`(`width`/`height`), `scene_info.slides`(최소 1개 이상의 배열)가 반드시 존재해야 합니다.
   - 각 슬라이드의 `id`는 카드 내에서 유일해야 하며, 각 슬라이드 `elements[].id`는 같은 슬라이드 내에서 유일해야 합니다.
   - `elements[].kind`는 4.4.3.1절의 허용 목록(`rect`, `circle`, `ellipse`, `line`, `path`, `polygon`, `text`, `image`, `group`) 중 하나가 아니면 거부됩니다.
   - `kind: "image"` 요소의 `src`가 가리키는 파일은 ZIP의 `images/` 폴더 내에 실제로 존재해야 합니다(동영상 카드의 `video_id` 실존 검증과 동일한 원칙).
   - `steps[].actions[].target`과 `interactions[].target`은 반드시 같은 슬라이드의 `elements[].id` 중 하나를 참조해야 하며, 정의되지 않은 id를 참조하면 거부됩니다.
   - `steps[].actions[].effect`와 `interactions[].action.effect`는 4.4.4.1절의 허용 목록(`fade_in`, `fade_out`, `move_to`, `scale_to`, `rotate_to`, `show`, `hide`, `highlight`, `draw_path`) 중 하나가 아니면 거부됩니다. `morph_path` 등 미지원 값도 거부 대상입니다.
   - `steps[].trigger`는 `"on_enter"` 또는 `"on_click"`, `interactions[].event`는 `"click"`만 허용됩니다.
   - 4.4.3절과 4.4.4절에서 kind/effect별 **Mandatory**·Conditional로 선언된 필드가 누락되면 거부됩니다: `kind: "text"`의 `content`, `kind: "image"`의 `src`, `kind: "path"`의 `d`, `kind: "line"`/`"polygon"`의 `points`, `group` 외 요소의 `x`/`y`, `effect: "move_to"`/`"scale_to"`/`"rotate_to"`의 `to`.
   - **보안 검증(카드=데이터 원칙)**: `content`, `style`의 모든 문자열 값, `src`, `d`, `points` 등 텍스트 필드 어디에도 `<script`, `on\w+=`(인라인 이벤트 핸들러), `javascript:` 패턴이 포함되어서는 안 되며, 발견 시 즉시 업로드가 거부됩니다.

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
   - 이때 반드시 코드 블록 바로 옆에 언어 식별자(예: `cpp`, `arduino`, `javascript`, `json` 등)를 명시하여 소스 코드가 가독성 있게 강조되도록 합니다.
   - 예시:
     ```arduino
     void setup() {
       Serial.begin(9600);
     }
     ```
   - 플랫폼 학습 화면은 이 코드 블록을 다크 모드 기반의 프리미엄 코드 박스(상단 헤더에 언어 표시 및 1-클릭 클립보드 복사 버튼 내장)로 변환하여 제공합니다.

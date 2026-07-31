# PDF 렌더링 결과를 HTML로 재현하기 위한 구현 규칙

## 목적

`C:\dev\연산작업\패키지\make_pdf7.2_260625.py`는 JSON 데이터를 LaTeX로 변환한 뒤 PDF로 컴파일한다.

HTML 뷰어에서 필요한 것은 PDF 파일 생성이 아니라, 이 스크립트가 PDF에 보여 주던 화면 결과를 HTML에서 최대한 동일하게 재현하는 것이다. 따라서 이 문서는 PDF 컴파일, TeX 빌드 폴더, 보조 파일 정리 같은 출력 절차를 제외하고, HTML 구현에 필요한 렌더링 규칙만 정리한다.

## 성공 기준

- 같은 JSON을 열었을 때 PDF와 HTML의 정보 구조가 같아야 한다.
- 문제 페이지에는 상단 헤더, 섹션 제목, 문제 번호, 문제 본문, 이미지, blocks, choices가 같은 순서로 보여야 한다.
- 정답 및 해설 영역에는 문제 번호, 정답, 해설이 같은 순서로 보여야 한다.
- `[img]`, `[img:파일명]`, 파이프 표, 보기 박스, 선지 열 배치가 PDF와 대응되어야 한다.
- HTML은 LaTeX를 그대로 출력하지 않고, MathJax와 DOM/CSS로 화면을 재현해야 한다.

## 제외할 내용

HTML 재현에는 다음 내용이 필요하지 않다.

- `COMPILE_PDF`, `LATEX_ENGINE`, `PDF_RUN_COUNT`
- `_latex_build`, `LATEX_JOBNAME`
- `.aux`, `.log` 등 LaTeX 보조 파일 정리
- `compile_tex_to_pdf()`
- `process_one_json()` 안의 TeX 파일 저장 및 PDF 복사 절차

단, PDF의 화면과 관련된 값은 유지한다. 예를 들어 여백, 단 개수, 선지 배치 기준, 이미지 크기 기준은 HTML/CSS로 옮겨야 한다.

## 입력 JSON 구조

스크립트가 워크시트로 인정하는 최소 구조는 다음과 같다.

```json
{
  "unit_large": "1. 도형의 성질",
  "unit_medium": "2. 사각형의 성질",
  "worksheet_title": "워크시트",
  "sections": [
    {
      "section_title": "평행사변형의 성질",
      "items": [
        {
          "q_number": 1,
          "question": {
            "stem": "문제 본문",
            "blocks": []
          },
          "choices": ["① 내용", "② 내용"],
          "answer": "②",
          "explanation": "해설",
          "images": []
        }
      ]
    }
  ]
}
```

HTML 렌더러에서 핵심으로 써야 하는 필드는 다음과 같다.

| JSON 필드 | HTML 역할 |
|---|---|
| `unit_large` | 문제 페이지 상단 헤더 첫 줄 일부 |
| `unit_medium` | 문제 페이지 상단 헤더 첫 줄 일부 |
| `worksheet_title` | 문제 페이지 상단 제목 |
| `sections[].section_title` | 문제 목록 첫 부분의 섹션 제목 박스 |
| `sections[].items[].q_number` | 문제 번호 및 정답 번호 |
| `question.stem` | 문제 본문 |
| `question.blocks` | 보기 박스, 표, 보조 문장 |
| `choices` | 객관식 선지 |
| `answer` | 정답 |
| `explanation` | 해설 |
| `images` | 순차형 `[img]` 태그에 대응되는 이미지 이름 목록 |

## 전체 화면 구조

PDF의 전체 구성은 다음 순서다.

1. 섹션별 문제 페이지
2. 정답 및 해설 페이지

`ONE_SECTION_PER_PAGE = True`이므로 PDF에서는 섹션마다 새 페이지를 시작한다. HTML에서는 이를 다음 중 하나로 구현할 수 있다.

- 섹션별 페이지형 패널
- 섹션 사이에 강한 구분선과 페이지 여백 느낌을 주는 블록
- 인쇄 모드에서 섹션마다 page break 적용

문제 영역과 정답 영역은 모두 2단 흐름이다.

| 설정 | 값 | HTML 대응 |
|---|---:|---|
| `PROBLEM_COLUMN_COUNT` | 2 | 문제 목록 2단 |
| `ANSWER_COLUMN_COUNT` | 2 | 정답 및 해설 2단 |
| `COLUMN_SEP_MM` | 8.0 | 단 사이 간격 |
| `COLUMN_RULE_PT` | 0.4 | 단 사이 세로선 |

CSS 예시:

```css
.pdf-problem-columns,
.pdf-answer-columns {
  column-count: 2;
  column-gap: 8mm;
  column-rule: 0.4pt solid #d7e2ec;
}

.pdf-item {
  break-inside: avoid;
}
```

## 문제 페이지 헤더

문제 페이지 상단은 `worksheetheader`에 해당한다.

구성:

1. `unit_large`와 `unit_medium`을 한 줄에 표시
2. `worksheet_title`을 더 큰 파란 제목으로 표시
3. 아래에 옅은 구분선 표시

`unit_large`가 `1. 도형의 성질`처럼 숫자 점 형식이면 로마 숫자로 바뀐다.

예:

```text
1. 도형의 성질
```

HTML 표시:

```text
I. 도형의 성질
```

`unit_large`와 `unit_medium` 사이에는 PDF에서 `1.8em` 정도의 간격이 들어간다.

색상:

| 용도 | 값 |
|---|---|
| 제목 파랑 `TitleBlue` | `#245C86` |
| 구분선 `RuleGray` | `#D7E2EC` |

## 섹션 제목 박스

`section_title`이 있으면 문제 목록 맨 앞에 박스로 표시한다.

PDF 규칙:

- 배경색: `#F3F7FB`
- 테두리색: `#D8E3ED`
- 글자색: `#4E6679`
- 글자 크기: 약 11pt
- 굵게 표시
- 내부 여백: 6pt
- 아래 간격: 3.8mm

HTML 대응:

```html
<div class="pdf-section-title">섹션 제목</div>
```

```css
.pdf-section-title {
  padding: 6pt;
  border: 0.5pt solid #d8e3ed;
  background: #f3f7fb;
  color: #4e6679;
  font-size: 11pt;
  font-weight: 700;
  margin-bottom: 3.8mm;
}
```

## 문제 아이템 구조

문제 한 개는 다음 순서로 렌더링된다.

1. 문제 번호
2. `question.stem`
3. `question.blocks`
4. `choices`
5. 문제 하단 간격

PDF의 `questionline`은 문제 번호를 왼쪽 고정 폭에 두고, 본문을 오른쪽에 둔다.

HTML 구조 예시:

```html
<article class="pdf-item">
  <div class="pdf-question-line">
    <span class="pdf-question-no">1.</span>
    <div class="pdf-question-body">문제 본문</div>
  </div>
  <div class="pdf-blocks">...</div>
  <ol class="pdf-choices">...</ol>
</article>
```

CSS 예시:

```css
.pdf-question-line {
  display: grid;
  grid-template-columns: 2.2em minmax(0, 1fr);
  align-items: start;
}

.pdf-question-no {
  color: #245c86;
  font-weight: 700;
}

.pdf-item {
  margin-bottom: 20mm;
}
```

## 텍스트와 수식

PDF에서는 `$...$` 구간을 LaTeX 수식으로 보존한다. HTML에서는 MathJax가 같은 구간을 렌더링하게 두면 된다.

필요한 처리:

- 일반 텍스트는 HTML escape
- `$...$`, `\(...\)`, `\[...\]`는 MathJax 입력으로 보존
- JSON 안의 실제 개행과 문자열 `\n`은 줄바꿈으로 정리
- 단, `\neq`, `\nabla` 같은 LaTeX 명령은 `\n` 처리 때문에 깨지면 안 된다.

## 이미지 태그

스크립트는 두 가지 이미지 태그를 처리한다.

| 태그 | 의미 |
|---|---|
| `[img:파일명]` | 명시된 파일명을 찾아 표시 |
| `[img]` | `item.images` 배열의 현재 순번 이미지를 표시 |

이미지 검색 확장자:

```text
.png, .jpg, .jpeg, .pdf
```

HTML 뷰어에서는 브라우저 표시를 고려해 최소한 다음 확장자를 우선 지원한다.

```text
.png, .jpg, .jpeg, .webp, .gif
```

PDF 원본 규칙:

- 이미지 위 간격: 1.0mm
- 이미지 아래 간격: 1.0mm
- PNG/JPG는 DPI 기준 실제 너비를 계산한다.
- 계산된 너비는 현재 단 폭의 95%를 넘지 않는다.
- 기본 fallback 너비는 `0.43\linewidth`다.
- 투명 래스터 이미지는 흰 배경으로 합성한다.

HTML 대응:

- 이미지 컨테이너는 가운데 정렬
- 기본 `max-width`는 단 폭의 95%
- DPI 기반 실제 크기 재현은 브라우저에서 어려우므로, 우선 CSS 변수로 이미지 표시 너비를 제어한다.
- 투명 PNG가 PDF와 다르게 보이면 흰 배경 컨테이너를 적용한다.

```css
.pdf-image-wrap {
  margin: 1mm 0;
  text-align: center;
}

.pdf-image-wrap img {
  max-width: 95%;
  height: auto;
  background: #fff;
}
```

이미지가 없을 때 PDF는 null 이미지 박스를 표시한다.

HTML에서도 단순 누락 텍스트보다 박스 형태가 PDF와 더 가깝다.

```html
<div class="pdf-missing-image">
  <strong>×</strong>
  <span>이미지 없음</span>
  <small>파일명</small>
</div>
```

## 파이프 표

스크립트는 텍스트 줄에서 파이프 표를 감지해 LaTeX 표로 바꾼다.

표로 인정되는 조건:

- 빈 줄 없이 연속된 줄이어야 한다.
- 각 줄은 `|`로 나뉜 셀이 2개 이상이어야 한다.
- 모든 행의 셀 개수가 같아야 한다.
- `---`, `:---`, `---:` 형태의 Markdown 구분 행은 제거한다.
- 최종 표시 행이 2행 이상이어야 한다.
- 수식 `$...$` 안의 `|`는 셀 구분자로 보지 않는다.

HTML 대응:

```html
<table class="pdf-table">
  <tbody>
    <tr><td>...</td><td>...</td></tr>
  </tbody>
</table>
```

```css
.pdf-table {
  width: 98%;
  margin: 1mm auto;
  border-collapse: collapse;
  table-layout: fixed;
}

.pdf-table td {
  border: 1px solid #222;
  padding: 4pt;
  text-align: center;
  vertical-align: middle;
}
```

## question.blocks

`question.blocks`는 PDF에서 문제 본문 아래에 들어가는 보조 블록이다.

스크립트가 기대하는 기본 형태:

```json
{
  "type": "box",
  "lines": ["내용 1", "내용 2"]
}
```

처리 규칙:

| `type` | PDF 표시 | HTML 대응 |
|---|---|---|
| `box` | 가운데 정렬 박스 | `.pdf-box-block` |
| `statement_set` | `| 보기 |` 라벨과 전체 폭 박스 | `.pdf-statement-set` |
| 기타 | 파이프 표이면 표, 아니면 가운데 줄 목록 | 표 또는 `.pdf-centered-lines` |

### box

PDF 규칙:

- 폭: 현재 줄 폭의 72%
- 테두리: 0.5pt
- 내부 여백: 6pt
- 가운데 정렬

HTML 대응:

```css
.pdf-box-block {
  width: 72%;
  margin: 0 auto;
  padding: 6pt;
  border: 0.5pt solid #222;
  text-align: center;
}
```

### statement_set

PDF 규칙:

- 위에 `| 보기 |` 라벨 표시
- 라벨은 왼쪽에서 1.8em 들여쓰기
- 내용 박스는 전체 폭
- 내부는 왼쪽 정렬

HTML 대응:

```html
<div class="pdf-statement-set">
  <div class="pdf-statement-label">| 보기 |</div>
  <div class="pdf-statement-box">...</div>
</div>
```

## choices

선지는 길이에 따라 3열, 2열, 1열로 배치된다.

LaTeX 제거 후 길이를 계산한다. `\frac`, `\sqrt`, `\times` 같은 명령은 길이 계산에서 단순화된다.

판정 규칙:

| 배치 | 조건 |
|---|---|
| 3열 | 가장 긴 선지 길이 6 이하이고, 전체 길이 24 이하 |
| 2열 | 가장 긴 선지 길이 10 이하이고, 전체 길이 42 이하 |
| 1열 | 그 외 |

폭:

| 배치 | 선지 폭 |
|---|---:|
| 3열 | 31% |
| 2열 | 48% |
| 1열 | 100% |

라벨:

```text
① ② ③ ④ ⑤
```

3열 배치:

```text
① ② ③
④ ⑤ 빈칸
```

2열 배치:

```text
① ②
③ ④
⑤ 빈칸
```

1열 배치:

```text
①
②
③
④
⑤
```

HTML 대응:

```html
<ol class="pdf-choices layout-2">
  <li><span>①</span><div>...</div></li>
  <li><span>②</span><div>...</div></li>
</ol>
```

```css
.pdf-choices {
  display: grid;
  gap: 0.6mm 4%;
  list-style: none;
  padding: 0;
  margin: 0.3mm 0 0;
}

.pdf-choices.layout-3 {
  grid-template-columns: repeat(3, 31%);
  justify-content: space-between;
}

.pdf-choices.layout-2 {
  grid-template-columns: repeat(2, 48%);
  justify-content: space-between;
}

.pdf-choices.layout-1 {
  grid-template-columns: 1fr;
}

.pdf-choice {
  display: grid;
  grid-template-columns: 1.35em minmax(0, 1fr);
  align-items: start;
}
```

## 정답 및 해설

`APPEND_ANSWER_EXPLANATION = True`이므로 PDF에는 정답 및 해설 페이지가 붙는다.

구성:

1. 새 페이지 시작
2. `정답 및 해설` 헤더
3. 구분선
4. 정답 아이템 2단 흐름

정답 아이템 구조:

```text
1. 정답: ②
해설 내용
```

HTML 대응:

```html
<section class="pdf-answer-page">
  <header class="pdf-answer-header">정답 및 해설</header>
  <div class="pdf-answer-columns">
    <article class="pdf-answer-item">
      <div><strong>1.</strong> 정답: <strong>②</strong></div>
      <div class="pdf-explanation">해설</div>
    </article>
  </div>
</section>
```

주의:

- 해설의 `[img]`는 문제 stem에서 사용한 순차 이미지 개수만큼 건너뛴 뒤 `images` 배열에서 매핑한다.
- `[img:파일명]`은 순번과 무관하게 명시 파일명을 사용한다.

## 색상과 타이포그래피

PDF 설정:

| 항목 | 값 |
|---|---|
| 기본 글자 크기 | 10pt |
| 줄 간격 | 1.60 |
| 기본 글꼴 | Malgun Gothic |
| 용지 | A4 |
| 상단 여백 | 14mm |
| 하단 여백 | 15mm |
| 좌우 여백 | 14mm |

HTML 대응:

```css
.pdf-sheet {
  font-family: "Malgun Gothic", "맑은 고딕", sans-serif;
  font-size: 10pt;
  line-height: 1.6;
  color: #111;
  background: #fff;
}

.pdf-page {
  max-width: 210mm;
  margin: 0 auto;
  padding: 14mm 14mm 15mm;
}
```

## 구현 순서 제안

1. 공통 텍스트 렌더러 작성
   - 입력: 문자열
   - 처리: 이미지 태그 분리, 파이프 표 감지, MathJax 보존
   - 검증: stem, explanation, choice 안의 이미지가 표시되는지 확인

2. 문제 아이템 렌더러 작성
   - 입력: item
   - 처리: q_number, stem, blocks, choices 순서 유지
   - 검증: PDF와 문제 구성 순서가 같은지 확인

3. choices 배치 함수 작성
   - 입력: choices 배열
   - 처리: 3열, 2열, 1열 판정
   - 검증: 짧은 선지는 3열, 중간 선지는 2열, 긴 선지는 1열이 되는지 확인

4. blocks 렌더러 작성
   - 입력: question.blocks
   - 처리: box, statement_set, table, fallback
   - 검증: 박스 폭과 정렬이 PDF와 대응되는지 확인

5. 섹션 및 정답 페이지 렌더러 작성
   - 입력: worksheet JSON
   - 처리: 섹션별 문제 페이지, 정답 및 해설 페이지
   - 검증: 2단 흐름과 섹션 구분이 PDF와 대응되는지 확인

## 기존 HTML 뷰어와의 차이

현재 HTML 뷰어는 이미 다음 기능을 갖고 있다.

- MathJax 로딩
- `[img:파일명]` 이미지 치환
- stem, blocks, choices, answer, explanation 표시
- 이미지 누락 표시

추가로 맞춰야 할 가능성이 큰 부분:

- `[img]` 순차 이미지 매핑
- PDF와 같은 파이프 표 파싱
- `question.blocks`의 `box`, `statement_set` 전용 렌더링
- PDF와 같은 choices 3열, 2열, 1열 판정
- 문제와 정답의 2단 흐름
- 섹션 제목 박스와 문제 페이지 헤더 스타일
- 정답 및 해설 페이지를 PDF 구조와 같은 별도 영역으로 표시

## 원본 코드 기준 위치

| 내용 | 함수 또는 설정 |
|---|---|
| 문제 페이지 설정 | `ONE_SECTION_PER_PAGE`, `PROBLEM_COLUMN_COUNT`, `QUESTION_GAP_MM` |
| 선지 배치 기준 | `choice_layout()`, `render_choices()` |
| 이미지 태그 처리 | `IMG_TAG_PATTERN`, `render_text_with_images()` |
| 파이프 표 처리 | `split_pipe_cells()`, `parse_pipe_table_lines()`, `render_pipe_table_from_lines()` |
| blocks 처리 | `render_question_blocks()`, `render_box_block()`, `render_statement_set_block()` |
| 문제 렌더링 | `render_item()` |
| 정답 렌더링 | `render_answer_item()` |
| 2단 흐름 | `render_flow_columns()` |
| 섹션 페이지 | `render_section_page()` |
| 정답 페이지 | `build_answer_pages()` |


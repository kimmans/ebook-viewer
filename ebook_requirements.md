# 단일 웹페이지 PDF 전자책 뷰어 요구사항

## 1. 프로젝트 개요

### 1.1 프로젝트명
단일 HTML 파일 PDF 전자책 뷰어

### 1.2 목적
설치나 빌드 과정 없이 **하나의 HTML 파일**로 완성되는 PDF 전자책 뷰어 구현
- 모든 라이브러리를 CDN에서 로드
- HTML + CSS + JavaScript가 모두 하나의 파일에 포함
- 서버에 업로드만 하면 바로 동작

### 1.3 기술 스택
- **HTML5**: 단일 파일 구조
- **CSS3**: `<style>` 태그 내 모든 스타일
- **JavaScript**: `<script>` 태그 내 모든 로직
- **외부 라이브러리**: CDN 링크로만 로드
  - PDF.js: Mozilla CDN
  - Turn.js: CDN
  - jQuery: CDN (Turn.js 의존성)

## 2. 파일 구조

### 2.1 단일 파일 구조
```
📁 웹사이트 폴더/
├── 📄 ebook-viewer.html     ← 이 파일 하나만!
├── 📄 sample.pdf           ← 보여줄 PDF 파일들
├── 📄 brochure.pdf
└── 📄 catalog.pdf
```

### 2.2 HTML 파일 구조
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <!-- 기본 설정 -->
    <meta charset="UTF-8">
    <title>PDF 전자책 뷰어</title>
    
    <!-- 외부 라이브러리 CDN -->
    <script src="https://cdnjs.cloudflare.com/...jquery.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/...pdf.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/...turn.min.js"></script>
    
    <!-- 모든 CSS 스타일 -->
    <style>
        /* 모든 스타일이 여기에 */
    </style>
</head>
<body>
    <!-- 전자책 뷰어 HTML 구조 -->
    <div id="viewer-container">
        <!-- UI 요소들 -->
    </div>
    
    <!-- 모든 JavaScript 로직 -->
    <script>
        // 모든 기능이 여기에
    </script>
</body>
</html>
```

## 3. 기능 요구사항

### 3.1 핵심 기능

#### 3.1.1 PDF 선택 및 로딩
```html
<!-- 파일 선택 드롭다운 -->
<select id="pdf-selector">
    <option value="">PDF 선택...</option>
    <option value="./sample.pdf">샘플 문서</option>
    <option value="./brochure.pdf">회사 소개서</option>
    <option value="./catalog.pdf">제품 카탈로그</option>
</select>

<!-- 또는 파일 업로드 -->
<input type="file" id="pdf-upload" accept=".pdf">
```

#### 3.1.2 책 스타일 뷰어
- 양면 페이지 표시 (데스크톱)
- 페이지 넘기기 애니메이션
- 실제 책처럼 보이는 그림자/그라데이션 효과

#### 3.1.3 컨트롤
- 이전/다음 페이지 버튼
- 페이지 점프 (번호 입력)
- 줌 인/아웃
- 전체화면 모드

### 3.2 사용자 인터페이스

#### 3.2.1 레이아웃
```
┌─────────────────────────────────────┐
│ [PDF 선택 ▼] [파일업로드] [전체화면] │ ← 상단 툴바
├─────────────────────────────────────┤
│                                     │
│     📖 📖                          │ ← 책 영역
│   [페이지1][페이지2]                │
│                                     │
├─────────────────────────────────────┤
│ [◀] [3/20] [▶] [🔍-][100%][🔍+]    │ ← 하단 컨트롤
└─────────────────────────────────────┘
```

#### 3.2.2 반응형 동작
- **데스크톱**: 양면 보기
- **태블릿**: 단면 보기 (세로 모드)
- **모바일**: 한 페이지씩, 터치 스와이프

## 4. 기술적 구현

### 4.1 외부 라이브러리 로딩
```html
<!-- jQuery (Turn.js 의존성) -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>

<!-- PDF.js -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js"></script>

<!-- Turn.js -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/turn.js/3.0.2/turn.min.js"></script>
```

### 4.2 PDF 처리 로직
```javascript
// 전역 변수
let currentPDF = null;
let currentPage = 1;
let totalPages = 0;
let currentZoom = 1;

// PDF 로드 함수
async function loadPDF(url) {
    const pdf = await pdfjsLib.getDocument(url).promise;
    currentPDF = pdf;
    totalPages = pdf.numPages;
    
    // Turn.js 초기화
    initializeTurnJS();
    
    // 첫 페이지들 렌더링
    renderPage(1);
    if (totalPages > 1) renderPage(2);
}

// 페이지 렌더링
async function renderPage(pageNum) {
    const page = await currentPDF.getPage(pageNum);
    const canvas = document.getElementById(`page-${pageNum}`);
    const context = canvas.getContext('2d');
    
    const viewport = page.getViewport({ scale: currentZoom });
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    
    await page.render({ canvasContext: context, viewport }).promise;
}
```

### 4.3 Turn.js 통합
```javascript
function initializeTurnJS() {
    $('#book').turn({
        width: 800,
        height: 600,
        autoCenter: true,
        gradients: true,
        elevation: 50
    });
    
    // 페이지 변경 이벤트
    $('#book').bind('turned', function(event, page, view) {
        currentPage = page;
        updateUI();
        
        // 필요한 페이지 미리 렌더링
        preloadPages(page);
    });
}
```

## 5. 구체적인 구현 계획

### 5.1 HTML 구조
```html
<div id="viewer-container">
    <!-- 상단 툴바 -->
    <div id="toolbar">
        <select id="pdf-selector">
            <option value="">PDF 선택...</option>
        </select>
        <input type="file" id="pdf-upload" accept=".pdf">
        <button id="fullscreen-btn">전체화면</button>
    </div>
    
    <!-- 책 영역 -->
    <div id="book-container">
        <div id="book">
            <!-- 페이지들이 동적으로 생성됨 -->
        </div>
    </div>
    
    <!-- 하단 컨트롤 -->
    <div id="controls">
        <button id="prev-btn">◀</button>
        <span id="page-info">1 / 1</span>
        <button id="next-btn">▶</button>
        <button id="zoom-out">🔍-</button>
        <span id="zoom-info">100%</span>
        <button id="zoom-in">🔍+</button>
    </div>
    
    <!-- 로딩 표시 -->
    <div id="loading" style="display: none;">
        PDF 로딩 중...
    </div>
</div>
```

### 5.2 CSS 스타일
```css
#viewer-container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

#book-container {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 600px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    border-radius: 10px;
    padding: 20px;
}

#book {
    box-shadow: 0 15px 35px rgba(0,0,0,0.1);
}

/* 반응형 */
@media (max-width: 768px) {
    #book {
        width: 100% !important;
        height: auto !important;
    }
}
```

### 5.3 JavaScript 이벤트 처리
```javascript
// DOM 로드 완료 후 초기화
$(document).ready(function() {
    setupEventListeners();
    initializeViewer();
});

function setupEventListeners() {
    // PDF 선택
    $('#pdf-selector').change(function() {
        const url = $(this).val();
        if (url) loadPDF(url);
    });
    
    // 파일 업로드
    $('#pdf-upload').change(function(e) {
        const file = e.target.files[0];
        if (file && file.type === 'application/pdf') {
            const url = URL.createObjectURL(file);
            loadPDF(url);
        }
    });
    
    // 컨트롤 버튼들
    $('#prev-btn').click(() => $('#book').turn('previous'));
    $('#next-btn').click(() => $('#book').turn('next'));
    $('#zoom-in').click(() => changeZoom(0.1));
    $('#zoom-out').click(() => changeZoom(-0.1));
}
```

## 6. 배포 및 사용

### 6.1 배포 방법
1. `ebook-viewer.html` 파일을 웹서버에 업로드
2. 같은 폴더에 PDF 파일들 업로드
3. 브라우저에서 접속: `https://yoursite.com/ebook-viewer.html`

### 6.2 사용 방법
1. 브라우저에서 페이지 열기
2. 드롭다운에서 PDF 선택 또는 파일 업로드
3. 마우스/터치로 페이지 넘기기
4. 하단 컨트롤로 줌, 페이지 이동

### 6.3 커스터마이징
- HTML 파일 내 CSS 부분 수정으로 디자인 변경
- JavaScript 부분에서 기능 추가/수정
- 드롭다운 옵션 수정으로 기본 PDF 목록 변경

---

## 결론

**한 개의 HTML 파일**만으로 완전한 PDF 전자책 뷰어를 구현할 수 있습니다. 복잡한 설치나 빌드 과정 없이, 웹서버에 파일만 올리면 바로 동작하는 완전한 솔루션입니다.

필요한 것:
- ✅ 하나의 HTML 파일
- ✅ 인터넷 연결 (CDN 라이브러리 로드용)
- ✅ 웹서버 (로컬 파일 시스템에서도 일부 기능 가능)

설치 불필요:
- ❌ Node.js
- ❌ npm
- ❌ Webpack
- ❌ 복잡한 빌드 과정
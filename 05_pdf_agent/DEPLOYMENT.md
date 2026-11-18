# PDFChatbot 배포 가이드

macOS 메뉴바 앱 PDFChatbot을 DMG로 패키징하고 배포하는 전 과정을 기술한다.

---

## 목차

1. [배포 전 체크리스트](#1-배포-전-체크리스트)
2. [의존성 설치](#2-의존성-설치)
3. [py2app 빌드 설정](#3-py2app-빌드-설정)
4. [아이콘 생성](#4-아이콘-생성)
5. [앱 빌드](#5-앱-빌드)
6. [Apple 서명 및 공증](#6-apple-서명-및-공증)
7. [DMG 생성](#7-dmg-생성)
8. [업데이트 배포](#8-업데이트-배포)
9. [사용자 설치 가이드](#9-사용자-설치-가이드)
10. [트러블슈팅](#10-트러블슈팅)

---

## 1. 배포 전 체크리스트

### 필수 확인 사항

- [ ] `menubar_app.py` 실행 정상 동작 확인
- [ ] 감시 폴더 PDF 자동 로드 테스트
- [ ] Ollama / OpenAI / Anthropic / Google 프로바이더 전환 테스트
- [ ] 채팅, 분석, 설정 탭 전체 UI 동작 확인
- [ ] `pyproject.toml` 버전 업데이트
- [ ] Apple Developer ID 인증서 준비 (공증 시 필요)

### 버전 관리

`menubar_app.py` 상단에 버전 상수를 정의한다:

```python
__version__ = "1.0.0"
```

---

## 2. 의존성 설치

### Poetry 환경 업데이트

```bash
# 프로젝트 루트에서
poetry install

# langchain-google-genai 설치 확인
poetry run python -c "from langchain_google_genai import ChatGoogleGenerativeAI; print('OK')"
```

### py2app 설치

py2app은 Poetry 환경 외부에서 관리한다:

```bash
pip install py2app create-dmg
```

---

## 3. py2app 빌드 설정

`05_pdf_agent/setup.py`를 생성한다:

```python
"""
setup.py — py2app 빌드 설정
실행: python setup.py py2app
"""
from setuptools import setup

APP = ['menubar_app.py']

DATA_FILES = [
    ('ui',      ['ui/index.html', 'ui/chat.js', 'ui/style.css',
                 'ui/settings.js', 'ui/analysis.js']),
    ('agent',   ['agent/__init__.py', 'agent/llm_config.py',
                 'agent/tools.py']),
    ('api',     ['api/__init__.py', 'api/server.py', 'api/engine_state.py']),
    ('api/routes', ['api/routes/__init__.py', 'api/routes/chat.py',
                    'api/routes/document.py', 'api/routes/settings.py']),
    ('session', ['session/__init__.py', 'session/session_manager.py']),
]

OPTIONS = {
    'argv_emulation': False,          # macOS 13+ 에서 True 시 크래시 가능
    'iconfile': 'assets/icon.icns',
    'packages': [
        'rumps', 'uvicorn', 'fastapi', 'starlette', 'pydantic',
        'requests', 'langchain_core', 'langchain_ollama',
        'langchain_openai', 'langchain_anthropic', 'langchain_google_genai',
        'langchain_classic', 'langchain_community', 'langchain_huggingface',
        'faiss', 'rank_bm25', 'pypdf', 'fitz',
        'sentence_transformers', 'torch', 'transformers',
    ],
    'excludes': ['streamlit', 'ragas', 'playwright', 'chromadb'],
    'plist': {
        'CFBundleName':             'PDFChatbot',
        'CFBundleDisplayName':      'PDFChatbot',
        'CFBundleIdentifier':       'com.yourname.pdfchatbot',
        'CFBundleVersion':          '1.0.0',
        'CFBundleShortVersionString': '1.0.0',
        'LSUIElement':              True,   # Dock 아이콘 숨김 (메뉴바 전용)
        'NSHighResolutionCapable':  True,
        # 필요한 권한 설명 (macOS Privacy 팝업용)
        'NSDownloadsFolderUsageDescription': 'PDF 파일에 접근합니다.',
        'NSDocumentsFolderUsageDescription': 'PDF 파일에 접근합니다.',
    },
}

setup(
    app=APP,
    data_files=DATA_FILES,
    options={'py2app': OPTIONS},
    setup_requires=['py2app'],
)
```

> **주의:** `sentence_transformers`와 `torch`는 크기가 크다(~1GB).
> 배포 대상이 Gemini/OpenAI/Anthropic 전용이라면 `excludes`에 추가해 빌드 크기를 줄일 수 있다.

---

## 4. 아이콘 생성

### 1024×1024 PNG 준비

`05_pdf_agent/assets/icon.png`를 1024×1024 픽셀로 준비한다.

### ICNS 변환 (macOS 내장 도구)

```bash
mkdir -p assets/icon.iconset

# 필요한 해상도 생성
sips -z 16 16     assets/icon.png --out assets/icon.iconset/icon_16x16.png
sips -z 32 32     assets/icon.png --out assets/icon.iconset/icon_16x16@2x.png
sips -z 32 32     assets/icon.png --out assets/icon.iconset/icon_32x32.png
sips -z 64 64     assets/icon.png --out assets/icon.iconset/icon_32x32@2x.png
sips -z 128 128   assets/icon.png --out assets/icon.iconset/icon_128x128.png
sips -z 256 256   assets/icon.png --out assets/icon.iconset/icon_128x128@2x.png
sips -z 256 256   assets/icon.png --out assets/icon.iconset/icon_256x256.png
sips -z 512 512   assets/icon.png --out assets/icon.iconset/icon_256x256@2x.png
sips -z 512 512   assets/icon.png --out assets/icon.iconset/icon_512x512.png
sips -z 1024 1024 assets/icon.png --out assets/icon.iconset/icon_512x512@2x.png

# ICNS 변환
iconutil -c icns assets/icon.iconset -o assets/icon.icns
```

---

## 5. 앱 빌드

### 빌드 스크립트 (`05_pdf_agent/build_app.sh`)

```bash
#!/usr/bin/env bash
set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$SCRIPT_DIR"

echo "=== 1. 기존 빌드 정리 ==="
rm -rf build dist

echo "=== 2. py2app 빌드 ==="
# Poetry 가상환경의 Python 사용
PYTHON="$(poetry env info --executable)"
$PYTHON setup.py py2app

echo "=== 3. 빌드 결과 ==="
du -sh dist/PDFChatbot.app
echo "빌드 완료: dist/PDFChatbot.app"
```

```bash
chmod +x build_app.sh
./build_app.sh
```

### 빌드 결과물 구조

```
05_pdf_agent/dist/
  PDFChatbot.app/
    Contents/
      MacOS/
        PDFChatbot          ← 실행 바이너리
      Resources/
        ui/                 ← HTML/JS/CSS
        agent/
        api/
        session/
        lib/                ← Python 프레임워크 + 패키지
      Info.plist
```

---

## 6. Apple 서명 및 공증

> Apple Developer Program 가입($99/년) 및 Developer ID Application 인증서 필요.
> 서명·공증 없이도 배포 가능하나, 사용자에게 "확인되지 않은 개발자" 경고가 표시된다.

### 6-1. 인증서 확인

```bash
security find-identity -v -p codesigning | grep "Developer ID"
# 출력 예: "Developer ID Application: Your Name (XXXXXXXXXX)"
```

### 6-2. entitlements.plist 생성

`05_pdf_agent/entitlements.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- localhost TCP 소켓 (Ollama, uvicorn) -->
  <key>com.apple.security.network.client</key>
  <true/>
  <key>com.apple.security.network.server</key>
  <true/>
  <!-- PDF 파일 읽기 -->
  <key>com.apple.security.files.user-selected.read-only</key>
  <true/>
  <!-- lsof subprocess 실행 -->
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <true/>
</dict>
</plist>
```

### 6-3. 코드 서명

```bash
TEAM_ID="XXXXXXXXXX"   # Apple Developer Team ID
CERT="Developer ID Application: Your Name ($TEAM_ID)"

codesign --deep --force --options runtime \
  --sign "$CERT" \
  --entitlements entitlements.plist \
  dist/PDFChatbot.app

# 서명 확인
codesign --verify --deep --strict dist/PDFChatbot.app
spctl --assess --verbose dist/PDFChatbot.app
```

### 6-4. 공증 제출

```bash
# 공증용 ZIP 생성
ditto -c -k --keepParent dist/PDFChatbot.app PDFChatbot.zip

# 공증 제출 (Apple 서버, ~5~10분)
xcrun notarytool submit PDFChatbot.zip \
  --apple-id "your@email.com" \
  --team-id "$TEAM_ID" \
  --password "앱_전용_비밀번호" \   # Apple ID → 앱 전용 암호 생성
  --wait

# 공증 결과를 앱에 부착 (스테이플)
xcrun stapler staple dist/PDFChatbot.app

# 최종 확인
spctl --assess --verbose --type exec dist/PDFChatbot.app
# → accepted
```

> **앱 전용 암호:** [appleid.apple.com](https://appleid.apple.com) → 보안 → 앱 전용 암호 생성

---

## 7. DMG 생성

### create-dmg 설치

```bash
brew install create-dmg
```

### DMG 배경 이미지 준비 (선택)

600×400 PNG를 `assets/dmg_background.png`로 준비한다.
화살표나 "Applications 폴더로 드래그" 안내 텍스트를 포함하면 좋다.

### DMG 빌드 스크립트

```bash
VERSION="1.0.0"

create-dmg \
  --volname "PDFChatbot" \
  --volicon "assets/icon.icns" \
  --background "assets/dmg_background.png" \
  --window-pos 200 120 \
  --window-size 600 400 \
  --icon-size 100 \
  --icon "PDFChatbot.app" 175 190 \
  --hide-extension "PDFChatbot.app" \
  --app-drop-link 425 190 \
  "PDFChatbot-${VERSION}.dmg" \
  "dist/"
```

### 결과물

```
PDFChatbot-1.0.0.dmg   ← 사용자에게 배포하는 파일
```

---

## 8. 업데이트 배포

### GitHub Releases 활용 (권장)

1. GitHub 레포지터리에 새 Release 생성
2. `PDFChatbot-x.x.x.dmg` 파일을 Release Assets에 업로드
3. 앱 내 버전 체크 코드가 자동 알림

### 앱 내 버전 체크 (menubar_app.py에 추가)

```python
GITHUB_REPO = "username/PDFChatbot"   # 실제 레포로 변경
__version__ = "1.0.0"

@rumps.timer(3600)   # 1시간마다 확인
def _check_update(self, _) -> None:
    try:
        resp = requests.get(
            f"https://api.github.com/repos/{GITHUB_REPO}/releases/latest",
            timeout=5,
        )
        latest = resp.json().get("tag_name", "").lstrip("v")
        if latest and latest > __version__:
            html_url = resp.json().get("html_url", "")
            rumps.notification(
                "PDFChatbot 업데이트",
                f"v{latest} 버전이 출시되었습니다.",
                "클릭하면 다운로드 페이지로 이동합니다.",
            )
            # rumps는 알림 클릭 콜백을 지원하지 않으므로 메뉴 항목으로 안내
            self._update_item.title = f"업데이트 다운로드 (v{latest})"
    except Exception:
        pass
```

### 모델 목록 원격 관리 (선택)

앱 재빌드 없이 지원 모델을 추가하려면 GitHub에 JSON 파일을 올린다:

```
https://raw.githubusercontent.com/username/PDFChatbot/main/models.json
```

```json
{
  "ollama":    ["qwen2.5:7b", "llama3.2:3b", "gemma3:4b", "phi4-mini"],
  "openai":    ["gpt-4o-mini", "gpt-4o", "gpt-4-turbo", "gpt-4.1"],
  "anthropic": ["claude-3-5-haiku-20241022", "claude-3-5-sonnet-20241022"],
  "google":    ["gemini-2.0-flash", "gemini-2.0-flash-lite", "gemini-1.5-pro"]
}
```

`api/routes/settings.py`에서 앱 시작 시 이 파일을 fetch해 `_MODEL_OPTIONS`를 갱신하면 모델 추가 시 앱을 재배포하지 않아도 된다.

---

## 9. 사용자 설치 가이드

배포 시 README 또는 GitHub Releases 설명에 포함할 내용:

### 요구 사항

| 항목 | 버전 |
|---|---|
| macOS | 13 Ventura 이상 |
| Ollama (로컬 모드 사용 시) | 최신 버전 |
| API 키 (클라우드 모드 사용 시) | OpenAI / Anthropic / Google 중 택일 |

### 설치 절차

**1단계: Ollama 설치 (로컬 모드 사용 시)**

```bash
# https://ollama.ai/download 에서 다운로드하거나
brew install ollama

# 모델 다운로드 (약 4–5 GB)
ollama pull qwen2.5:7b
```

**2단계: PDFChatbot 설치**

1. `PDFChatbot-x.x.x.dmg` 파일을 다운로드한다.
2. DMG를 마운트하고 `PDFChatbot.app`을 `/Applications` 폴더로 드래그한다.
3. 최초 실행 시 Control+클릭 → 열기 (처음 1회만, 이후 정상 실행)

**3단계: 실행 및 설정**

1. Spotlight(`⌘ Space`) → PDFChatbot 검색 → 실행
2. 메뉴바 🤖 아이콘 클릭
3. **클라우드 모델 사용 시:** 설정 탭 → 프로바이더 선택 → API 키 입력 → 저장
4. **감시 폴더 설정 (선택):** 감시 폴더 설정... 메뉴 → 논문 PDF 폴더 선택
5. PDF를 미리보기 또는 PDF Expert 등으로 열면 자동 로드

---

## 10. 트러블슈팅

### py2app 빌드 실패 — ModuleNotFoundError

`setup.py`의 `packages` 목록에 누락된 패키지를 추가한다.

```bash
# 빌드 로그에서 누락 패키지 확인
python setup.py py2app 2>&1 | grep "No module"
```

### 앱 실행 시 "확인되지 않은 개발자" 경고

```bash
# 터미널에서 격리 속성 제거 (개발·테스트용)
xattr -rd com.apple.quarantine /Applications/PDFChatbot.app
```

또는 시스템 설정 → 개인정보 보호 및 보안 → "그래도 열기" 클릭.

### langchain-google-genai ImportError

```bash
poetry add langchain-google-genai
poetry run python -c "from langchain_google_genai import ChatGoogleGenerativeAI"
```

### Ollama 연결 오류 (⚠️ 아이콘)

```bash
# Ollama 서비스 실행 확인
ollama list
curl http://localhost:11434/api/tags
```

### 감시 폴더 PDF 자동 로드 안 됨

- 앱 시작 후 15초 이내에는 동작하지 않는다 (서버 준비 대기).
- `lsof` 권한 문제: 시스템 설정 → 개인정보 보호 → 전체 디스크 접근 권한 → PDFChatbot 추가.

---

## 배포 버전 히스토리

| 버전 | 날짜 | 변경 사항 |
|---|---|---|
| 1.0.0 | 2026-04-01 | 최초 릴리스: Ollama / OpenAI / Anthropic / Google Gemini 지원 |

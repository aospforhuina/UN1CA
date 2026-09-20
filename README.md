<h1 align="center">
  <img loading="lazy" src="readme-res/banner.png"/>
</h1>
<p align="center">
  <a href="https://github.com/salvogiangri/UN1CA/blob/sixteen/LICENSE"><img loading="lazy" src="https://img.shields.io/github/license/salvogiangri/UN1CA?style=for-the-badge&logo=github"/></a>
  <a href="https://github.com/salvogiangri/UN1CA/commits/sixteen"><img loading="lazy" src="https://img.shields.io/github/last-commit/salvogiangri/UN1CA/sixteen?style=for-the-badge"/></a>
  <a href="https://github.com/salvogiangri/UN1CA/stargazers"><img loading="lazy" src="https://img.shields.io/github/stars/salvogiangri/UN1CA?style=for-the-badge"/></a>
  <a href="https://github.com/salvogiangri/UN1CA/actions/workflows/ci.yml"><img loading="lazy" src="https://img.shields.io/github/actions/workflow/status/salvogiangri/UN1CA/ci.yml?style=for-the-badge"/></a>
  <a href="https://crowdin.com/project/UN1CA"><img loading="lazy" src="https://img.shields.io/badge/Crowdin-263238?style=for-the-badge&logo=crowdin"/></a>
</p>

<p align="center">UN1CA <i>(/ˈu.ni.ka/)</i> — 삼성 Galaxy 기기용 커스텀 펌웨어(custom ROM) 프로젝트</p>

<p align="center">
  <a href="https://github.com/salvogiangri/UN1CA/discussions">🚀 Discussions</a>
  •
  <a href="https://t.me/unicarom">💬 Telegram</a>
</p>

---

# UN1CA (정리본)

> 이 문서는 원본 README를 **알기 쉽게 정리한 요약본**입니다.
> 원저작자의 원문은 하단 **[원본 README 보기](#6-원본-readme-original-un1ca-readme)** 를 펼치면 수정 없이 그대로 확인할 수 있습니다.

## 목차
1. [프로젝트 소개](#1-프로젝트-소개)
2. [주요 기능](#2-주요-기능)
3. [저장소 구조](#3-저장소-구조)
4. [빌드하기](#4-빌드하기)
5. [라이선스 & 기여](#5-라이선스--기여)
6. [원본 README](#6-원본-readme-original-un1ca-readme)

---

## 1. 프로젝트 소개
- **UN1CA** 는 Samsung Galaxy 기기용 **작업 중(WIP) 커스텀 펌웨어**입니다.
- 최신 **Galaxy S22 펌웨어**를 베이스로, **One UI 8** 계열 UX 위에 다양한 최적화·개선·독점 기능을 통합합니다.
- 빌드 시스템이 **툴 자동 빌드 → 펌웨어 다운로드/추출 → 패치 적용 → 플래셔블 zip 생성**까지 전 과정을 자동으로 처리합니다.
- 목표는 빠르고 부드러운 모던 UX + 파워 유저를 위한 시스템 레벨 개선입니다.
- 버그 리포트·기능 제안·기여는 언제나 환영입니다.

## 2. 주요 기능

### 핵심 기능
- 최신 안정 **Galaxy S22 펌웨어** 기반 · **EROFS** 파일시스템
- **Galaxy AI** 지원 (오디오 지우개, 브라우징/통화/그리기/노트/필기/사진/성적표/쓰기 어시스트, 인터프리터, Now Brief, 시맨틱 검색)
- 고급 애니메이션 · 네이티브/라이브 블러 · AOD 클럭 트랜지션 · 적응형 색감/주사율 · 추가 밝기
- 픽처 리마스터 · 오브젝트/그림자/반사 지우개 · 이미지 클리퍼
- 다중 사용자(Multi user) · Samsung DeX* · 카메라 프라이버시 토글
- 불필요한 시스템 서비스/앱 **Debloat** · 전체 앱에서 Dual Messenger 사용
- 커스텀 FlipFont · 야외 모드 · 4자리 자동 PIN 확인
- **BluetoothLibraryPatcher** · **KnoxPatch** 내장
- 추가 CSC 기능 (통화 녹음, Hiya, 상태바 네트워크 속도, AltZLife)

> \* USB-C DP 미지원 기기는 HDMI DeX 불가

### UN1CA 독점 기능
- 내장 **OTA 업데이트 앱**
- 네이티브/라이브 블러 토글 · One UI Home 애니메이션 옵션 · Vulkan 렌더러 토글
- 키 증명 스푸프([TrickyStore](https://github.com/5ec1cff/TrickyStore)) 옵션* · **Play Integrity Fix** 내장
- 설치 앱 숨기기([Hide My Applist](https://github.com/Dr-TSNG/Hide-My-Applist)) · 개발자 옵션 숨기기
- 앱 다운그레이드 허용 · 구(old targetSdk) 앱 설치 허용 · 안전 스크린샷 토글
- 스크린샷/녹화 감지 토글 · Google 포토 무제한 백업 · 게임 FPS 언락 토글

> \* 유효한 키박스(keybox) 필요

> 전체 기능 목록은 [원본 README](#6-원본-readme-original-un1ca-readme) 참조.

---

## 3. 저장소 구조

| 경로 | 역할 |
|---|---|
| `external/` | 외부 툴 소스 (git submodule: android-tools, apktool, erofs-utils, img2sdat, samloader, signapk) |
| `platform/` | SoC 플랫폼 정의 (exynos2100, sm7125, sm7325, sm8450) |
| `target/` | 기기별 정의 (o1s, a53x, b0q, p3s, r9s, t2s 등 15개) |
| `unica/` | ROM 공통 코어 (configs/, patches/, mods/) |
| `prebuilts/` | 커밋된 사전 빌드 바이너리 |
| `scripts/` | 빌드 시스템 본체 |
| `security/` | 서명 키/인증서 |
| `buildenv.sh` | 빌드 환경 셋업 |

각 경로/파일의 상세 역할은 **[UN1CA_BUILD_GUIDE.md](UN1CA_BUILD_GUIDE.md)** 참조.

---

## 4. 빌드하기

```bash
# 갤럭시 S21 5G (Exynos / o1s) 예시
git submodule update --init --recursive   # 외부 툴 서브모듈 초기화
source buildenv.sh o1s                    # 빌드 환경 로드
./scripts/build_dependencies.sh            # 필수 툴 빌드
./scripts/make_rom.sh -z                  # 메인 빌드 + 플래셔블 zip 생성
```

> 설치 패키지, JDK 17, 커널 모듈 등 환경 설정부터 단계별 빌드 방법·산출물 위치까지:
> **[UN1CA_BUILD_GUIDE.md](UN1CA_BUILD_GUIDE.md)** 를 읽어주세요.

---

## 5. 라이선스 & 기여
- 이 프로젝트는 [GNU GPL v3.0](LICENSE) 하에 배포됩니다. 외부 의존성은 각자의 라이선스(Apache-2.0, MIT, GPL-2.0 등)를 따릅니다.
- 버그 제보·기능 요청·기여는 **[Discussions](https://github.com/salvogiangri/UN1CA/discussions)** 와 **[Telegram](https://t.me/unicarom)** 을 이용하세요.

---

## 6. 원본 README (Original UN1CA README)

> 아래는 원저작자의 README **원문**입니다 (수정 없이 보존).

<details>
<summary><b>클릭해서 펼치기 / 접기</b></summary>

<br/>

<h1 align="center">
  <img loading="lazy" src="readme-res/banner.png"/>
</h1>
<p align="center">
  <a href="https://github.com/salvogiangri/UN1CA/blob/sixteen/LICENSE"><img loading="lazy" src="https://img.shields.io/github/license/salvogiangri/UN1CA?style=for-the-badge&logo=github"/></a>
  <a href="https://github.com/salvogiangri/UN1CA/commits/sixteen"><img loading="lazy" src="https://img.shields.io/github/last-commit/salvogiangri/UN1CA/sixteen?style=for-the-badge"/></a>
  <a href="https://github.com/salvogiangri/UN1CA/stargazers"><img loading="lazy" src="https://img.shields.io/github/stars/salvogiangri/UN1CA?style=for-the-badge"/></a>
  <a href="https://github.com/salvogiangri/UN1CA/actions/workflows/ci.yml"><img loading="lazy" src="https://img.shields.io/github/actions/workflow/status/salvogiangri/UN1CA/ci.yml?style=for-the-badge"/></a>
  <a href="https://crowdin.com/project/UN1CA"><img loading="lazy" src="https://img.shields.io/badge/Crowdin-263238?style=for-the-badge&logo=crowdin"/></a>
</p>
<p align="center">UN1CA <i>(/ˈu.ni.ka/)</i> is a work-in-progress custom firmware for Samsung Galaxy devices.</p>

<p align="center">
  <a href="https://github.com/salvogiangri/UN1CA/discussions">🚀 Discussions</a>
  •
  <a href="https://t.me/unicarom">💬 Telegram</a>
</p>

# What is UN1CA?
UN1CA is a work-in-progress custom firmware for Samsung Galaxy devices, designed to provide a refined, optimized and more rich One UI experience.
It is based on the latest and greatest iteration of Samsung's UX and it integrates numerous improvements, optimizations and exclusive features.

The UN1CA build system automatically builds the required tools, downloads and extracts firmware components, applies the required patches and generates a flashable zip for the target device.

The goal is to deliver a fast, smooth and modern UX while offering additional tools, modifications and system‑level enhancements tailored for power users.

Any form of contribution, suggestions, bug report or feature request for the project will be welcome.

# Features
### Core features:
- Based on the latest stable Galaxy S22 firmware
- EROFS powered
- Galaxy S25 wallpapers/sounds included
- Galaxy AI support
  - Audio eraser
  - Browsing assist
  - Call assist
  - Drawing assist
  - Interpreter
  - Note assist
  - Now brief
  - Photo assist
  - Semantic search
  - Transcript assist
  - Writing assist
- High end animations
- Native/live blur support
- AOD clock transition support
- Adaptive color tone support
- Adaptive refresh rate support
- Extra brightness support
- Picture remaster support
- Object, shadow and reflection eraser support
- Image clipper support
- Multi user support
- Samsung DeX support*
- Camera privacy toggle support
- Debloated from useless system services/additional apps
- Dual Messenger available for all apps
- Custom FlipFont fonts support
- Outdoor mode support
- Auto PIN confirm with 4 digits
- [BluetoothLibraryPatcher](https://github.com/3arthur6/BluetoothLibraryPatcher) integrated
- [KnoxPatch](https://github.com/salvogiangri/KnoxPatch) integrated
- Extra CSC features enabled (Call recording, Hiya, Network speed in status bar, AltZLife)

\* DeX via HDMI not available for devices without USB-C DP support

### UN1CA-exclusive features:
- Integrated OTA updates app
- Native/live blur toggle
- One UI Home animations option
- Vulkan renderer toggle
- Key attestation spoof ([TrickyStore](https://github.com/5ec1cff/TrickyStore)) options*
- Play Integrity Fix integrated
- Ability to hide installed apps ([Hide My Applist](https://github.com/Dr-TSNG/Hide-My-Applist))
- Ability to hide developer options
- Allow app downgrade toggle
- Allow installing apps with old targetSdk toggle
- Allow secure screenshot toggle
- Screenshot/screen recording detection toggle
- Unlimited backup storage on Google Photos
- Games FPS unlock toggle

\* Requires a valid keybox

# Licensing
This project is licensed under the terms of the [GNU General Public License v3.0](LICENSE). External dependencies might be distributed under a different license, such as:
- [android-tools](https://github.com/nmeum/android-tools), licensed under the [Apache License 2.0](https://github.com/nmeum/android-tools/blob/master/LICENSE)
- [apktool](https://github.com/iBotPeaches/Apktool), licensed under the [Apache License 2.0](https://github.com/iBotPeaches/Apktool/blob/master/LICENSE.md)
- [erofs-utils](https://github.com/sekaiacg/erofs-utils/), dual license ([GPL-2.0](https://github.com/sekaiacg/erofs-utils/blob/dev/LICENSES/GPL-2.0), [Apache-2.0](https://github.com/sekaiacg/erofs-utils/blob/dev/LICENSES/Apache-2.0))
- [img2sdat](https://github.com/xpirt/img2sdat), licensed under the [MIT License](https://github.com/xpirt/img2sdat/blob/master/LICENSE)
- [platform_build](https://android.googlesource.com/platform/build/) (ext4_utils, f2fs_utils, signapk), licensed under the [Apache License 2.0](https://source.android.com/docs/setup/about/licenses)

# Contributors
<a href="https://github.com/salvogiangri/UN1CA/graphs/contributors"><img loading="lazy" src="https://contrib.rocks/image?repo=salvogiangri/UN1CA"/></a>

# Credits
A special thanks goes to the following for their invaluable contributions in no particular order:
- **[ShaDisNX255](https://github.com/ShaDisNX255)** for his help, time and for his [NcX ROM](https://github.com/ShaDisNX255/NcX_Stock) which inspired this project
- **[DavidArsene](https://github.com/DavidArsene)** for his help and time
- **[paulowesll](https://github.com/paulowesll)** for his help and support
- **[Simon1511](https://github.com/Simon1511)** for his support and some of the device-specific patches
- **[ananjaser1211](https://github.com/ananjaser1211)** for troubleshooting and his time
- **[Fede2782](https://github.com/Fede2782)** for his contributions and help with Exynos/MTK support
- **[iDrinkCoffee](https://github.com/iDrinkCoffee-TG)** and **[RisenID](https://github.com/RisenID)** for their support
- **[LineageOS Team](https://www.lineageos.org/)** for their original [OTA updater implementation](https://github.com/LineageOS/android_packages_apps_Updater)
- *All the UN1CA project forks, contributors, testers and users ❤️*

<br/>

</details>

---

*이 문서 상단은 프로젝트를 한눈에 보기 위한 요약 정리본이며, 하단 펼침 메뉴 안에는 프로젝트 원저작자의 README 원문이 보존되어 있습니다.*

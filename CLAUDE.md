# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

아래는 이 저장소에 원래 있던 일반 지침이며, 그대로 보존한다.

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

## 프로젝트 개요

UN1CA는 삼성 Galaxy 기기용 커스텀 펌웨어(롬) 빌드 시스템이다. 셸 스크립트만으로 구성되어 있으며
**툴 컴파일 → 순정 펌웨어 다운로드/추출 → 패치·모드 적용 → APK 재빌드 → 플래셔블 zip 생성**을
전부 자동화한다. 컴파일 언어 코드나 테스트 스위트는 없다.

- 설치 패키지·JDK·커널 모듈·단계별 빌드 절차: **[UN1CA_BUILD_GUIDE.md](UN1CA_BUILD_GUIDE.md)**
- 기능 목록·프로젝트 소개: **[README.md](README.md)**

## 명령어

### 빌드 파이프라인

```bash
source buildenv.sh t2s              # 필수. 반드시 source 로 (새 터미널마다)
source buildenv.sh --debug t2s      # 무서명(디버그) 빌드

./scripts/build_dependencies.sh     # 외부 툴 빌드 → out/tools/bin (external/make.sh 심볼릭 링크)
./scripts/download_fw.sh [-f]       # 삼성 FUS 다운로드 → out/odin (-f 로 재시도)
./scripts/extract_fw.sh             # 이미지 추출 → out/fw
./scripts/make_rom.sh [-f] [-x] [-z]  # 메인 빌드
```

`make_rom.sh` 옵션: `-f/--force` 강제 재빌드, `-x/--no-target-files` work_dir까지만,
`-z/--build-rom-zip` 플래셔블 zip까지. 펌웨어가 없으면 `download_fw.sh`/`extract_fw.sh`를
내부적으로 자동 호출하므로, `build_dependencies.sh` 후 바로 `./scripts/make_rom.sh -z` 하나로 끝난다.

### 그 외

```bash
./scripts/build_flashable_zip.sh [-i <source.zip>] [-o <out.zip>] out/<target>-target_files.zip
./scripts/cleanup.sh all|odin|fw|work_dir|logs|tools
./scripts/print_modules_info.sh      # 적용될 모듈 목록 출력
./scripts/generate_ota_manifest.sh
unica <cmd>                          # scripts/<cmd>.sh 실행기 (buildenv.sh 가 정의하는 셸 함수)
```

### 린트

```bash
shellcheck -e SC1090,SC1091 -s bash <file>
```

CI(`.github/workflows/ci.yml`)는 `platform scripts target unica` 의 `*.sh` 전체를 검사하며
`*ssi.sh`, `config.sh`, `sff.sh` 는 제외한다. 그다음 `target/` 디렉터리에서 매트릭스를 생성해
모든 기기를 병렬 빌드한다(Ubuntu 24.04). **테스트 스위트는 없고, 검증 수단은 shellcheck + 실제 빌드뿐이다.**

### 빌드 캐시

`make_rom.sh`는 `unica/`, `platform/<platform>/`, `target/<codename>/` 전체의 SHA1을
`$WORK_DIR/.completed` 에 기록한다. 해시가 같으면 "No changes detected" 로 빌드를 건너뛴다.
모듈을 수정했는데 반영이 안 되면 `-f` 를 쓴다.

## 아키텍처

### 2-펌웨어 모델 (가장 중요한 개념)

빌드는 서로 다른 두 순정 펌웨어를 합친다.

| | 역할 | 정의 위치 |
|---|---|---|
| **SOURCE** | 베이스 롬 (예: Galaxy S22 One UI 8.0) | `unica/configs/<ssi>.sh` |
| **TARGET** | 대상 기기의 순정 펌웨어 | `target/<codename>/config.sh` |

대상 기기에 없는 최신 UX/기능을 SOURCE 에서 가져오고, 기기 고유 blobs·`fs_config`·`file_context`
는 TARGET 펌웨어(또는 `prebuilts/samsung/<device>/`)에서 가져온다.

**SSI(Single System Image)** 는 SoC 계열 추상화다. 현재 `essi`(Exynos)와 `qssi`(Snapdragon) 두 가지뿐이며
`TARGET_OS_SINGLE_SYSTEM_IMAGE` 로 선택된다 → `unica/configs/essi.sh` 가 로드된다.
`TARGET_OS_FILE_SYSTEM_TYPE`(기본 `erofs`), `TARGET_OS_BUILD_SYSTEM_EXT_PARTITION` 등
`TARGET_OS_*` 변수들이 여기서 결정된다.

### config.sh 병합 순서

`source buildenv.sh <target>` → `scripts/internal/gen_config_file.sh` 가 아래 순서로 source 하여
`out/config.sh` 를 생성한다 (뒤가 앞을 덮어씀):

```
unica/configs/version.sh  →  target/<c>/config.sh  →  platform/<p>/config.sh
  →  target/<c>/config.sh (다시, 의도된 HACK — 기기 설정이 플랫폼 설정을 이기도록)
  →  unica/configs/<ssi>.sh  →  out/config.sh
```

이후 모든 스크립트는 `set -o allexport; source "$OUT_DIR/config.sh"` 한 상태로 실행된다.
`ROM_VERSION` 은 `version.sh` 에서 `VERSION_MAJOR.MINOR.PATCH-<git short hash>[-dirty]` 로 자동 생성된다.
`ROM_IS_OFFICIAL` 은 `security/unica_platform.pk8`/`unica_ota.pk8` 의 SHA1을 알려진 값과 비교해 결정된다.

### 빌드 파이프라인 (make_rom.sh)

```
create_work_dir.sh   →  SOURCE 펌웨어를 work_dir 에 전개한 뒤 TARGET 펌웨어를 덮어씀
apply_modules.sh ×4  →  platform/patches → target/patches → unica/patches → unica/mods
BUILD_APKS           →  apktool 로 smali 패치가 적용된 APK/JAR 재빌드 (최대 8 병렬)
create_target_files_zip.sh  →  out/<code>_<ver>-target_files.zip
build_flashable_zip.sh      →  out/UN1CA_<ver>_<날짜>_<code>[-sign].zip
```

`apply_modules.sh` 는 대상 폴더의 하위 디렉터리를 `find | LC_ALL=C sort` 로 정렬해 순서대로 적용한다.
모듈명 접두사 관례: `__` (예: `__props`, `__floating_feature`) 가 가장 먼저, `_` (예: `_debloat`) 가 그다음.

### 모듈 시스템

`platform/<p>/patches/`, `target/<c>/patches/`, `unica/patches/`, `unica/mods/` 의 **하위 디렉터리 하나 = 모듈 하나**.

```
모듈명/
├── module.prop          # id=, name=, author=, description=  (없으면 에러)
├── customize.sh         # 빌드 중 source 되는 셸 스크립트 (핵심 로직)
├── disable              # 이 파일이 있으면 모듈 전체 스킵
├── system/ vendor/ product/ odm/ system_ext/   # work_dir 에 그대로 덮어쓰는 파일 트리
├── <partition>.prop     # 예: system.prop → 해당 파티션에 SET_PROP
└── smali/<partition>/<apk or jar>/NNNN-*.patch  # unified diff, apktool 디코드 트리에 git apply
```

- `customize.sh` 는 **별도 프로세스가 아니라 현재 셸에서 source** 된다. 따라서 `scripts/utils/module_utils.sh`·
  `common_utils.sh`·`smali_utils.sh` 의 모든 함수와 `$WORK_DIR`, `$TARGET_*`, `$SOURCE_*`, `$DEBUG` 에
  그대로 접근할 수 있다. `return 1` 로 빌드를 중단시킨다.
- `customize.sh` 에 `SKIPUNZIP=1` 이 있으면 파일 트리 복사를 건너뛴다(스크립트로만 처리하는 모듈).
- 모듈 루트에 `TARGET_OS_SINGLE_SYSTEM_IMAGE` 와 같은 이름의 하위 디렉터리가 있으면 그쪽이 루트로 승격된다
  (예: `mods/deknox/essi/` → Exynos 빌드에서만 그 내용 사용).
- `smali/` 패치는 파티션별 디렉터리(`system/priv-app/.../Foo.apk/0001-*.patch`)에 두고 `git apply` 로 적용된다.

**patches vs mods**: `patches/` 는 기기를 SOURCE 롬 베이스라인까지 끌어올리는 이식·기능 수정(NFC, 카메라,
SELinux, vintf, blobs 이식 등), `mods/` 는 사용자 대상 기능 토글(테마, CSC, DeX, OTA 설정 등)이다.

### customize.sh 에서 쓸 수 있는 주요 함수

`scripts/utils/` 참조. 전부 `SRC_DIR` 환경변수를 요구한다.

| 함수 | 용도 |
|---|---|
| `SET_PROP <part> <prop> <value>` / `SET_PROP_IF_DIFF` / `GET_PROP` | build.prop 조작 (`-d` 로 삭제) |
| `ADD_TO_WORK_DIR <src> <part> <file> <uid> <gid> <mode> <selabel>` | work_dir 에 파일 주입. `<src>` 가 `A/B` 형식이면 `out/fw/A_B`, 단일 이름이면 `prebuilts/samsung/<name>` 로 해석 |
| `DELETE_FROM_WORK_DIR <part> <file>` | 제거 |
| `SET_METADATA <part> <entry> <uid> <gid> <mode> <selabel>` | `fs_config-*` / `file_context-*` 갱신 |
| `SMALI_PATCH <part> <apk> <smali> <op> [method] [value] [replacement]` | op: `null` / `remove` / `replace` / `replaceall` / `return` / `strip` |
| `APPLY_PATCH` / `DECODE_APK` | apktool 디코드 및 diff 적용 |
| `HEX_PATCH <file> <from> <to>` | 바이너리 HEX 패치 (길이 동일해야 함) |
| `SET_FLOATING_FEATURE_CONFIG` / `GET_FLOATING_FEATURE_CONFIG` | `floating_feature.xml` 조작 |
| `LOG` / `LOGW` / `LOGE` / `LOG_STEP_IN` / `LOG_STEP_OUT` / `EVAL` / `ABORT` | 로깅·실행 |
| `DOWNLOAD_FILE` / `GET_GALAXY_STORE_DOWNLOAD_URL` | 다운로드 |

유효한 파티션명은 `system`, `vendor`, `product`, `system_ext`, `odm`, `vendor_dlkm`, `odm_dlkm`,
`system_dlkm` 뿐이다 (`IS_VALID_PARTITION_NAME`).

### 서명

`security/unica_platform.*`, `security/unica_ota.*` 는 `.gitignore` 에 등록되어 **저장소에 없다**.
키가 없으면 테스트/AOSP 키로 대체되거나 `--debug` 무서명 빌드가 되며, 이는 정상 동작이다.
CI 는 GitHub Secrets 로 실제 키를 주입한다.

### installer/

플래셔블 zip 의 edify 인스톨러 스크립트. `target/<c>/installer/` 에 위치한다.
- `assertions.edify` — 설치 전 부트로더 버전 검사, 복구 차단
- `customize.sh` — GitHub releases 에서 기기별 프로프라이어터리 펌웨어 tar 4종 다운로드·검증·복호화·패치
- `install-end.edify` — `/dev/block/by-name/*` 에 오프셋 단위 write

## 코드 컨벤션

- 모든 `.sh` 는 GPL-3.0-or-later 헤더(`# Copyright (c) <year> Salvo Giangreco`)로 시작한다.
- 함수 정의 블록은 `# [` … `# ]` 로 감싼다. `source` 될 때 함수 본문이 실행되지 않도록 하는 관례이며,
  실제 실행 코드는 `# ]` 뒤에 온다. 이 마커를 지우면 `buildenv.sh` source 시 스크립트가 그대로 실행된다.
- 함수는 실패 시 `return 1` + `LOGE`, 호출부는 `|| exit 1` 로 처리한다. `_CHECK_NON_EMPTY_PARAM` 으로
  필수 인자를 검증하는 패턴을 따른다.
- user-facing 문자열은 큰따옴표, `[[ ]]` 와 `local` 을 사용한다. shellcheck 지시자를 인라인으로 단다.

## 알려진 함정

- **`source buildenv.sh <target>` 없이는 어떤 스크립트도 실행되지 않는다.** 모든 스크립트가 `$SRC_DIR`
  부재 시 즉시 실패한다. 터미널을 새로 열 때마다 다시 source 해야 한다.
- **`external/*` 서브모듈은 커밋 `43d6c3bb` 에서 복원되었다.** 이 저장소는 재구성된 클론이라 원래
  index 에 gitlink 엔트리(`160000`)가 없었고, 그래서 `git submodule update --init --recursive` 가
  **아무 메시지 없이 no-op** 으로 끝나 `external/*` 가 빈 디렉터리로 남았다.
  upstream(`salvogiangri/UN1CA`, `sixteen`)의 gitlink 6개를 커밋해 넣었으므로 이제 정상 동작한다.
  확인: `git submodule status` (앞에 `-` 면 미클론, SHA 만 나오면 정상).
  만약 이 커밋이 없는 옛 클론을 쓰고 있다면 gitlink 가 다시 없을 수 있으니 위 명령으로 먼저 확인할 것.
- JDK **17** 이 필요하다 (apktool / signapk 의 Gradle 빌드). 최신 JDK 에서는 깨진다.
- `erofs`, `f2fs` 커널 모듈을 로드해야 한다: `sudo modprobe erofs f2fs`.
- 디스크 여유 80 GB 이상 권장 (슈퍼 파티션 10.6 GiB + 펌웨어 2종 + 중간 산출물).
- `unica/*/patches|mods` 하위 폴더를 새로 만들면 `module.prop` 이 반드시 있어야 한다. 없으면 `apply_modules.sh` 가 에러로 중단한다.
- `GET_GALAXY_STORE_DOWNLOAD_URL` 등 일부 모듈은 빌드 중 삼성/GitHub 서버에 접속한다. 네트워크가 필요하다.
- 다운로드한 순정 펌웨어는 재배포 금지 라이선스다.

---


- `CLAUDE.md` 파일의 최상단에 있는 카파시 원칙(Karpathy's Rules)은 절대 수정하거나 아래로 내리지 마세요. 
- 자동으로 빌드 명령어를 추가해야 할 때만 최하단에 덧붙이세요.

---

## 현재 작업 상태 (Current Status)

> **기준일: 2026-09-26** · 아래 내용은 `UN1CA_BUILD_GUIDE.md`(2026-09-21 작성)와 **당일 실제 환경 점검 결과**를
> 근거로 정리했다. 대화 기록이 아니라 **저장소 산출물 + 명령으로 검증한 시스템 상태**가 출처이므로,
> 의심되는 항목은 반드시 명령으로 재확인할 것. 상태가 바뀌면 이 섹션을 갱신한다.

### 목표

- **대상 기기: Galaxy S21+ 5G (Exynos) = `t2s`** — 실기기는 **SM-G996N (한국 내수용)**
  빌드 산출물은 SM-G996B(유럽) / SM-G996N(한국) **통합 패키지**
- 목적: UN1CA 커스텀 롬을 t2s 용으로 빌드하여 **SM-G996N 실기기**에 설치
- 베이스(SOURCE): `SM-S901B/EUX/...` Galaxy S22 Exynos One UI 8.0 — SSI `essi` (`unica/configs/essi.sh`)
- 목표 순정(TARGET): `SM-G996B/AUT/...` (`target/t2s/config.sh`)
  - ⚠️ `TARGET_FIRMWARE` 는 **유럽(SM-G996B/AUT)** 기준이다. 한국 G996N 대응은
    `target/t2s/patches/variants/` 가 blobs(firmware, `tee_SM-G996N`, nvram, NFC `libsn100u_fw.so`)를
    이식하는 방식으로 이미 처리되어 있어 **별도 수정 없이 G996N 설치 가능**하다.
  - 다만 실기기 CSC/통신 환경이 한국인데 TARGET 펌웨어가 유럽인 점은 추후 검토 대상 (롬 파일은 아직 미변경)

### 환경 점검 결과 (2026-09-26 실측)

| 항목 | 상태 |
|---|---|
| OS / 커널 | Arch Linux / `6.18.49-2-lts` ✅ |
| CPU / RAM / 디스크 | 12코어 / 19 GB / `/home/arch/disk2` **203 GB 여유** ✅ |
| `build_utils.sh` 의 `DEPENDENCIES` 55종 | **누락 0개** ✅ |
| gtest 헤더 (`/usr/include/gtest`) | 존재 ✅ |
| `lld` | **없음** ❌ (저장소 `extra` 에 22.1.8-1) |
| JDK | **26만 설치됨. JDK 17 없음** ❌ (sdkman 미설치, `extra` 에 jdk17-openjdk 17.0.20.1.u1-1) |
| erofs · f2fs 커널 모듈 | **미로드** ❌ (모듈 파일은 `/usr/lib/modules/6.18.49-2-lts/.../erofs.ko.zst` 에 존재) |
| sudo | 암호 필요 (무암호 아님) |

### 저장소 상태 (2026-09-26 실측)

| 항목 | 상태 |
|---|---|
| 브랜치 / 원격 | `main` / `origin = https://github.com/aospforhuina/UN1CA.git` |
| 커밋 | `43d6c3bb external: register submodule gitlinks`, `4d1060cf added some file`, `ad5fe00 cleaned readme.md`, `fa24aee Initial commit: Reconstructed project` |
| `external/*` 6종 | **정상 클론 완료** — gitlink 6개 복원 + upstream 고정 커밋으로 체크아웃 ✅ |
| `out/` | **없음 — 빌드 이력 전무** |
| `security/` | `aosp_*` 키만 존재. `unica_platform.*` / `unica_ota.*` 없음 (정상, `.gitignore`) |
| 문서 | `UN1CA_BUILD_GUIDE.md`(t2s 기준 개정), `README.md`(한국어 정리본), `CLAUDE.md` |

> ℹ️ `UN1CA_BUILD_GUIDE.md` 는 2026-09-26 자로 **`t2s` 기준 개정**되었다. 기기 식별값·부트로더·산출물
> 이름은 모두 t2s 로 교체되었고, 서브모듈 절차(5장)도 이 저장소 현실에 맞게 수정되었다.
> 패키지 **설치 여부**만은 이 문서의 "환경 점검 결과" 표를 기준으로 판단할 것.

### 블로커 — 이것 없이는 빌드 불가

1. ~~`external/` 6종 소스 부재~~ → **해결됨** (커밋 `43d6c3bb`, gitlink 복원 + 클론 완료)
2. **JDK 17 부재** — apktool / signapk 의 Gradle 빌드가 JDK 26 에서 깨진다.
3. **`lld` 부재** — erofs-utils / android-tools 를 clang + `ENABLE_FULL_LTO=ON` 으로 빌드할 때 필요.
4. **erofs · f2fs 미로드** — 이미지 생성/마운트 단계에서 필요.

### TODO — 기기 수령 전 (빌드 준비)

- [ ] `sudo pacman -S --needed lld jdk17-openjdk` (또는 sdkman 으로 Temurin 17)
- [ ] `JAVA_HOME` / `PATH` 를 JDK 17 로 고정 → `java -version` 이 17 인지 확인
- [ ] `sudo modprobe erofs f2fs`
- [x] `external/` 6종 서브모듈 초기화 (`git submodule update --init --recursive`) — 완료
- [ ] `source buildenv.sh t2s` → `out/config.sh` 생성 및 `TARGET_NAME` / `ROM_VERSION` 출력 확인
      (`TARGET_NAME="Galaxy S21+ 5G (Exynos)"`, `TARGET_CODENAME="t2s"` 인지 확인)
- [ ] `./scripts/print_modules_info.sh` 로 적용될 모듈 목록 확인
- [ ] `./scripts/build_dependencies.sh` → `out/tools/bin` 에 툴 생성 확인 (30분~1시간 이상 소요)

### TODO — 기기 수령 후 (빌드 → 설치 → 검증)

- [ ] `./scripts/download_fw.sh` — FUS 에서 SOURCE·TARGET 펌웨어 2종 다운로드 (실패 시 `-f` 재시도)
- [ ] `./scripts/extract_fw.sh` — `out/fw/` 에 이미지 추출 확인
- [ ] `./scripts/make_rom.sh -z` — work_dir + 패치/모드 적용 + APK 재빌드 + 플래셔블 zip
- [ ] 산출물 확인: `out/UN1CA_<ver>_<날짜>_t2s.zip` (무서명 빌드면 `-sign` 없음)
- [ ] 실기기(SM-G996N)를 **순정 펌웨어 상태**로 준비 — `target/t2s/installer/assertions.edify` 요구사항:
      G996B 는 부트로더 `G996BXXSJHZC2`, **G996N 은 `G996NKSSCHZA9`** (그보다 최신이면 설치 거부)
- [ ] afaneh92 복구 미사용 확인 (사용 중이면 설치 거부됨)
- [ ] 복구를 통한 zip 설치 → 초기 부팅 확인
- [ ] 검증: 부팅 안정성, Galaxy AI, DeX, KnoxPatch, Play Integrity Fix,
      카메라(RAW 패치), CSC 기능(통화 녹음 등), 내장 OTA 앱 업데이트 조회
- [ ] 서명: 실제 UN1CA 서명 zip 을 만들려면 `security/unica_platform.*`, `security/unica_ota.*` 필요
      (없으면 테스트/AOSP 키 또는 `--debug` 무서명으로 진행됨)

### 다음 세션에서 바로 할 일

1. 위 **"기기 수령 전"** 체크리스트의 미완료 항목부터. 특히 `external/` 클론과 JDK 17.
2. 실제로 빌드를 돌려보고 실패 지점을 이 섹션에 갱신한다.

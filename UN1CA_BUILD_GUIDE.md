# UN1CA 빌드 & 환경 설정 가이드 (t2s — Galaxy S21+ 5G Exynos)

> 작성일: 2026-09-21 · 개정: 2026-09-26 (작업 대상 `o1s` → `t2s` 로 변경)
> 기준 브랜치: `main`
> 대상: 이 저장소(`UN1CA`) 를 받아서 **Galaxy S21+ 5G (Exynos) = `t2s`** 용 커스텀 롬을 빌드하는 경우
> 실기기: **SM-G996N (한국 내수용)** — 산출물은 SM-G996B(유럽)/SM-G996N(한국) 통합 패키지

---

## 1. 프로젝트 개요

**UN1CA** (`/ˈu.ni.ka/`) 는 삼성 Galaxy 기기용 커스텀 펌웨어(롬) 프로젝트입니다.
최신 Galaxy S22 펌웨어를 베이스로 삼고, One UI 8 계열 UX에 각종 최적화·기능(EROFS, Galaxy AI,
KnoxPatch, Play Integrity Fix, 다중 사용자, Debloat 등)을 얹은 **플래셔블 zip**을 만들어 냅니다.

빌드 시스템이 다음 일을 전부 자동으로 처리합니다.

1. 필요한 툴(tools) 자동 컴파일 (`scripts/build_dependencies.sh`)
2. 삼성 FUS 서버에서 순정 펌웨어 다운로드 (`scripts/download_fw.sh`, samloader 사용)
3. 펌웨어 이미지(`system.img`, `vendor.img`, `product.img` 등) 추출 (`scripts/extract_fw.sh`)
4. 플랫폼/기기/ROM 패치와 모드 적용 (`scripts/make_rom.sh`)
5. 패치된 APK/JAR 재빌드 (apktool)
6. 플래셔블(!) OTA 방식 zip 생성 (에디파이 edify 인스톨러 포함)

원본 저장소: https://github.com/salvogiangri/UN1CA
(본 저장소의 원격은 `https://github.com/aospforhuina/UN1CA` 로, 재구성된 클론입니다)

---

## 2. 현재 저장소 상태 (2026-09-26 실측)

| 항목 | 상태 |
|---|---|
| Git 브랜치 | `main` |
| Git 원격 | `https://github.com/aospforhuina/UN1CA` |
| HEAD 커밋 | `ad5fe00 cleaned readme.md` (그 아래 `fa24aee Initial commit: Reconstructed project`) |
| external 서브모듈 | **초기화 불가 — gitlink 미등록** (아래 ⚠️ 참조) |
| `out/` 디렉터리 | 없음 (아직 빌드 안 함) |
| 작업 대상 | **`t2s`** = Galaxy S21+ 5G (Exynos), 실기기 SM-G996N |

> ⚠️ **`git submodule update --init --recursive` 가 이 저장소에서는 동작하지 않습니다.**
> 재구성된 클론이라 index 에 gitlink 엔트리(`160000`)가 없어서 위 명령이 **아무 출력 없이 no-op** 으로
> 끝나고, `external/{android-tools,apktool,erofs-utils,img2sdat,samloader,signapk}` 는 **빈 디렉터리**로
> 남습니다. 확인: `git ls-files -s | awk '$1=="160000"'` (아무것도 안 나오면 gitlink 없음).
> → 5장의 "서브모듈 초기화" 대신 **`.gitmodules` 의 URL 로 직접 클론**해야 합니다.

---

## 3. 시스템 요구사항

### 3.1 하드웨어 권장 사양
- CPU: 8코어 이상 (apktool 병렬 빌드는 최대 8잡으로 제한됨, `nproc` 기준)
- RAM: 16 GB 이상 권장 (12~19 GB면 충분, CI는 8 GB 러너로도 빌드함)
- 디스크: **여유 80 GB 이상 권장**
  - `out/`: 펌웨어 2종(BAS, 순정 S22 + 목표 기기) + tools + work_dir + zip 들이 쌓임
  - t2s 슈퍼 파티션 자체가 ~10.6 GiB (11,429,478,400 B) → work_dir/이미지/zip이 각각 수 GB~10 GB+
- 인터넷: 필수 (FUS, GitHub releases, Gradle 의존성 다운로드)

### 3.2 OS
- **Arch Linux** (본 환경) 또는 **Ubuntu 24.04** (CI 기준)
- WSL2 에서도 동작하도록 erofs-utils에 `RUN_ON_WSL` 플래그가 있음

### 3.3 커널 요구사항
- `erofs`, `f2fs` 커널 모듈 필요 (마운트 용도)
  - 확인/로드: `sudo modprobe erofs f2fs`
  - Arch 기본 `linux` 커널 패키지에 포함되어 있음 (본 환경에 이미 존재 확인됨: `/usr/lib/modules/.../fs/erofs`, `f2fs`)

---

## 4. 필수 패키지 설치

빌드 스크립트가 실제로 검사하는 의존성(`scripts/utils/build_utils.sh`의 `DEPENDENCIES`) 기준으로 정리했습니다.

### 4.1 Arch Linux (pacman) — 권장 명령

```bash
sudo pacman -S --needed \
  attr bc brotli bzip2 ccache clang cmake cpio curl ffmpeg file gawk \
  git grep gtest jq jdk17-openjdk libunwind libwebp lld lz4 make p7zip \
  pcre2 perl protobuf python python-pip rsync sed sudo tar unzip \
  util-linux vim zip zstd coreutils findutils
```

설명:
- `clang` `lld` : erofs-utils / android-tools 컴파일 (LTO 포함)
- `gtest` : android-tools 빌드용 (CI의 libgtest-dev)
- `protobuf` : `protoc` (android-tools 빌드 의존)
- `libunwind` `pcre2` : android-tools 빌드 의존 (CI의 libunwind-dev, libpcre2-dev)
- `brotli` `bzip2` `zstd` `lz4` : erofs-utils/이미지 처리 (CI의 libbrotli-dev, libbz2-dev, libzstd-dev)
- `libwebp` : `cwebp` (CI의 webp)
- `p7zip` : `7z` / `vim` : `xxd` / `rsync` : rsync
- `ffmpeg` : 미디어(사운드/이미지) 처리
- `attr` : `getfattr` (파일 xattr/셀레늄 컨텍스트 처리)
- JDK: **Temurin 17 권장** (CI와 동일). `jdk17-openjdk` 대신 sdkman/AUR로 Temurin 17 설치 가능:

  ```bash
  # (선택) sdkman으로 CI와 동일한 Temurin 17 설치 예
  curl -s "https://get.sdkman.io" | bash
  source ~/.sdkman/bin/sdkman-init.sh
  sdk install java 17.0.14-tem
  sdk use java 17.0.14-tem
  ```

> 참고: 본 시스템의 기본 java는 OpenJDK 26입니다. apktool/signapk의 Gradle 빌드가 최신 JDK와
> 불호환일 수 있으므로(CI는 17 고정) JDK 17 설치 후 6장의 JAVA_HOME 설정을 권장합니다.

### 4.2 Ubuntu / Debian (apt) — CI(`.github/workflows/ci.yml`) 기준

```bash
sudo apt update
sudo apt install -y \
  attr ccache clang ffmpeg libbrotli-dev libbz2-dev libgtest-dev \
  libprotobuf-dev libunwind-dev libpcre2-dev libzstd-dev lld \
  protobuf-compiler webp \
  bc cpio curl git jq lz4 p7zip-full perl python3 python3-pip \
  python3-venv rsync unzip vim zip zstd
sudo modprobe erofs f2fs
# JDK 17 (Temurin) 별도 설치 필요
```

---

## 5. 저장소 준비 (환경 설정)

```bash
cd /home/arch/disk2/UN1CA

# 1) 외부 툴 소스 6종 확보 — ★ 수동 클론 (서브모듈 명령이 동작하지 않음)
#    git submodule update --init --recursive 는 이 저장소에서 no-op 이다 (2장 ⚠️ 참조)
git clone https://github.com/UN1CA/external_android-tools.git      external/android-tools
git clone https://github.com/iBotPeaches/Apktool.git               external/apktool
git clone https://github.com/sekaiacg/erofs-utils.git              external/erofs-utils
git clone https://github.com/UN1CA/external_img2sdat.git           external/img2sdat
git clone https://github.com/ananjaser1211/samloader.git           external/samloader
git clone https://github.com/UN1CA/external_signapk.git            external/signapk

# 2) (권장) git 사용자 정보 — 일부 툴 빌드에서 필요할 수 있음
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"

# 3) 커널 모듈 로드
sudo modprobe erofs f2fs

# 4) JDK 17 준비 (apktool/signapk 의 Gradle 빌드는 JDK 17 필요, 최신 JDK 에서 깨짐)
sudo pacman -S --needed jdk17-openjdk lld
#    또는 sdkman 으로 Temurin 17:
#    source ~/.sdkman/bin/sdkman-init.sh && sdk use java 17.0.14-tem
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export PATH="$JAVA_HOME/bin:$PATH"
java -version   # openjdk 17 확인
```

> ⚠️ **gitlink 가 없으므로 각 서브모듈의 원래 고정 커밋을 알 수 없습니다.** 위처럼 기본 브랜치 HEAD 를
> 클론하면 upstream 이 의도한 버전과 다를 수 있습니다. 특히 `external/apktool` 은
> `external/patches/apktool/0001-feat-support-aapt-optimization.patch` 를 `git apply` 하므로
> 패치가 실패하면 해당 시점에 맞는 태그/커밋으로 체크아웃해야 합니다.

## 6. 빌드 환경 불러오기 — `buildenv.sh`

> ⚠️ `source` 로 실행해야 합니다 (서브셸에서 실행하면 환경변수가 안 남습니다).

```bash
cd /home/arch/disk2/UN1CA
source buildenv.sh t2s           # 일반 빌드
# source buildenv.sh --debug t2s # 디버그(무서명) 빌드
```

실행 결과 예시 (`out/config.sh` 생성 후 출력):
```
==============================
TARGET_NAME="Galaxy S21+ 5G (Exynos)"
TARGET_CODENAME="t2s"
...
ROM_VERSION="3.2.0-<hash>"
==============================
```

`buildenv.sh`가 하는 일:
- `SRC_DIR`, `OUT_DIR`, `ODIN_DIR`(다운로드 펌웨어), `FW_DIR`(추출 펌웨어),
  `TOOLS_DIR`, `WORK_DIR`, `APKTOOL_DIR`, `TMP_DIR` 내보내기
- `$TOOLS_DIR/bin` 을 PATH에 추가
- `out/config.sh` 자동 생성:
  `unica/configs/version.sh` + `target/t2s/config.sh` + `platform/exynos2100/config.sh` + `unica/configs/essi.sh` 병합
- `unica` 별칭 → `scripts/<명령>.sh` 실행기
- `croot` → 저장소 루트로 cd

```bash
unica --help        # 사용 가능한 명령 목록 보기
unica make_rom -z   # 예: ROM 빌드 + 플래셔블 zip
```

---

## 7. 디렉토리/파일 구조와 역할

```
UN1CA/
├── buildenv.sh              # 빌드 환경 셋업 (source 로 사용)
├── README.md                # 프로젝트 소개/기능 목록
├── LICENSE                  # GPL-3.0
├── .gitmodules              # external 서브모듈 6종 정의
├── .gitattributes           # 바이너리/텍스트 속성 (LF 강제, blobs 바이너리 지정)
├── .gitignore               # out/, UN1CA 서명 키 제외
├── .github/
│   ├── workflows/ci.yml     # CI: shellcheck + 모든 타겟 빌드 (Ubuntu 24.04)
│   └── workflows/blobs.yml  # 매일 prebuilts 갱신 PR 생성 (서명 키 시크릿 사용)
├── external/                # ★ 외부 툴 소스 — 수동 클론 필요 (gitlink 미등록, 5장 참조)
│   ├── android-tools/       #   구글 안드로이드 도구 포크 (adb/fastboot/e2fsdroid/lpmake/avbtool 등)
│   ├── apktool/             #   APK 추출/리패키징 도구 (smali 패치 후 재빌드에 사용)
│   ├── erofs-utils/         #   EROFS 파일시스템 도구 (mkfs/fsck/dump/extract.erofs)
│   ├── img2sdat/            #   이미지 → SDAT 변환 (OTA 증분용)
│   ├── samloader/           #   삼성 FUS 펌웨어 다운로더 (IMEI 사용, python venv로 설치)
│   ├── signapk/             #   APK 서명 도구 (Gradle 빌드)
│   ├── make.sh              #   위 툴들을 단독 빌드하는 스크립트
│   └── patches/apktool/     #   apktool에 적용되는 패치 (aapt 최적화)
├── platform/                # ★ SoC 플랫폼 정의 (exynos2100 / sm7125 / sm7325 / sm8450)
│   └── exynos2100/          #   t2s의 플랫폼
│       ├── config.sh        #   파티션 크기/BOARD API 레벨/Wi-Fi·BT 기능 플래그
│       ├── debloat.sh       #   플랫폼 공통 제거 앱 목록
│       ├── sff.sh           #   SEC Floating Feature 목록
│       ├── installer/       #   recovery.fstab 등
│       └── patches/         #   플랫폼 공통 패치 모듈 (blobs/chiclet/singlesim/stock_props...)
├── prebuilts/               # ★ 커밋된 사전 빌드 바이너리 (2.4 GB)
│   ├── samsung/<기기>/      #   순정 기기별 props/fs_config/file_context (prebuilt blobs)
│   └── bootable/deprecated-ota/updater   # 레거시 OTA 업데이터 바이너리
├── scripts/                 # ★ 빌드 시스템 본체
│   ├── build_dependencies.sh    # 툴 전부 컴파일/설치 → out/tools
│   ├── download_fw.sh           # FUS에서 펌웨어 다운로드 (samloader venv 사용)
│   ├── extract_fw.sh            # 펌웨어 → system/vendor/product ... 이미지 추출
│   ├── make_rom.sh              # ★ 메인 빌드 (패치 적용 + APK 재빌드 + zip 생성)
│   ├── build_fs_image.sh        # EROFS/ext4 파티션 이미지 생성
│   ├── build_flashable_zip.sh   # target-files zip → 플래셔블 zip (증분 지원)
│   ├── generate_ota_manifest.sh # OTA 매니페스트 생성
│   ├── apktool.sh               # apktool 래퍼 (추출/재빌드)
│   ├── unsign_bin.sh            # 바이너리 서명 제거
│   ├── cleanup.sh               # out/ 정리
│   ├── print_modules_info.sh    # 적용될 패치/모드 목록 출력
│   ├── internal/                # (내부 로직)
│   │   ├── gen_config_file.sh       # out/config.sh 생성
│   │   ├── create_work_dir.sh       # 추출 이미지 → work_dir 복사
│   │   ├── apply_modules.sh         # ★ 모듈(패치/모드) 적용 엔진
│   │   ├── create_target_files_zip.sh
│   │   ├── build_full_ota_zip.sh    # 플래셔블 zip (edify 인스톨러 포함)
│   │   ├── build_incremental_ota_zip.sh
│   │   └── update_prebuilt_blobs.sh # prebuilts 갱신 (CI용)
│   └── utils/                     # 공통 함수 (common/build/firmware/install/log/module/smali)
├── security/                # ★ 서명 키/인증서
│   ├── aosp_platform.pk8/.pem, aosp_testkey.*   # 저장소 커밋된 공개용 키(AOSP 계열)
│   ├── avb/testkey_rsa4096.pem
│   └── unica_platform.pk8/.pem, unica_ota.*     # ★ 실제 UN1CA 서명 키 — gitignore로 저장소에 없음
│                                                #   (테스트 서명으로 대체되어 빌드는 진행됨)
├── target/                  # ★ 기기 정의 15개 (a52q, a53x, a71, b0q, o1s, p3s, r9s, t2s ...)
│   └── t2s/                 #   대상 기기: Galaxy S21+ 5G (Exynos) — 상세는 8장
├── unica/                   # ★ ROM 공통 코어
│   ├── configs/
│   │   ├── version.sh       #   롬 버전 (3.2.0 + git 해시/-dirty 자동 부착)
│   │   ├── essi.sh          #   Exynos용 SOURCE 펌웨어 정의 (Galaxy S22 Exynos, One UI 8.0)
│   │   └── qssi.sh          #   Snapdragon용 SOURCE 펌웨어 정의
│   ├── patches/             #   공통 패치 모듈 (adb/bluetooth/camera/_debloat/dvfs/ese/fmradio/fs/
│   │                        #   legacy/miscs/nfc/proca/product_feature/_props/rro/saiv/selinux/
│   │                        #   signature/spen/stock_blobs/uwb/vintf/vndk)
│   ├── mods/                #   기능 모드 (blocklist, bootanim, bootsound, callsounds, choidujour,
│   │                        #   csc, daagent, deknox, gamesprops, hma, keybox, knoxpatch, monotype,
│   │                        #   multiuser, outdoor, paradigm, pif, pinconfirm, preload, prophide,
│   │                        #   settings, setupwizard, tweaks, wallpaper)
│   └── debloat.sh           #   전역 제거 앱 목록
└── readme-res/              # README 이미지
```

### 모듈(패치) 구조 — `module.prop` + `customize.sh`
`unica/patches/*` · `unica/mods/*` · `target/<기기>/patches/*` — 각 폴더는 **모듈** 단위입니다.

```
어떤_모듈/
├── module.prop      # id=, name=, author=, description=
├── customize.sh     # 빌드 중 실행되는 셸 로직 (LOG / SET_METADATA / HEX_PATCH / SMALI_PATCH 등)
├── system/ vendor/ product/ odm/ system_ext/   # → work_dir에 덮어쓰는 파일 트리
├── smali/           # *.apk/*.jar/클래스 당 *N.patch (역어셈블 smali 텍스트 패치)
├── vendor.prop / fs_config-* / file_context-*  # 메타데이터/파일 컨텍스트
└── disable          # 이 파일이 있으면 해당 모듈 비활성화
```

적용 순서 (`make_rom.sh`): **플랫폼 patches → 기기 patches → unica patches → unica mods**

---

## 8. t2s 타겟 상세

`target/t2s/` — **Galaxy S21+ 5G (Exynos)** 전용 정의

| 항목 | 값 |
|---|---|
| TARGET_NAME | `Galaxy S21+ 5G (Exynos)` |
| TARGET_CODENAME | `t2s` |
| TARGET_PLATFORM | `exynos2100` (Exynos 2100, 한때 “부다페스트” SoC) |
| TARGET_FIRMWARE | `SM-G996B/AUT/355399273736683` (목표 순정 펌웨어: SM-G996B, 유럽 AUT CSC, IMEI) |
| SOURCE_FIRMWARE (essi) | `SM-S901B/EUX/350330051234562` (베이스: Galaxy S22 Exynos One UI 8.0) |
| 단일 시스템 이미지 | `essi` (Exynos Single System Image) |
| 슈퍼 파티션 크기 | 11,429,478,400 B (~10.6 GiB) |
| SDK 버전 | 기기 35 / 소스 36 |
| 주요 하드웨어 특성 | 120 Hz HFR, MDNIE HW, eSE(NXP), 지문(초음파), 듀얼 스피커, eSIM |

### 주요 파일 역할 (`target/t2s/`)

| 파일 | 역할 |
|---|---|
| `config.sh` | 기기 식별값, 다이나믹 파티션 크기, SEC 제품 기능 플래그(카메라/LCD/DVFS/WLAN 등) |
| `sff.sh` | SEC Floating Feature (AOD 라이브 클럭, 추가 밝기) |
| `debloat.sh` | 이 기기에서 제거할 앱 (`WifiRROverlayAppH2E` 등) |
| `installer/customize.sh` | 플래셔블 zip 생성 시 UN1CA GitHub releases에서 `modem/vbmeta` 등 **프로프라이어터리 펌웨어 tar** 4종 다운로드·검증·복호화·패치 |
| `installer/assertions.edify` | 설치 전 기기 판정: 부트로더 버전 강제 (`G996BXXSJHZC2` / `G996NKSSCHZA9`), afaneh92 복구 차단 |
| `installer/install-end.edify` | 설치 중 `/dev/block/by-name/bota`(및 radio/vbmeta)에 dpm/harx/keystorage/ldfw/sboot/ssp/tzar/tzsw/uh/up_param/vddcal_fw **오프셋 단위 write** |
| `overlay/values/*.xml` | 기기 프로퍼티 오버라이드 (예: 노치/디스플레이 컷아웃 좌표) |
| `camera/camera-feature.xml` | 카메라 피처 XML |
| `dvfs/siop_model.xml` | 전원(성능) 관리 모델 |
| `vintf/` | 기기 인터페이스(센서 등) 정의 |
| `patches/raw_camera/` | `libexynoscamera3.so` 바이너리 HEX 패치 — 모든 카메라에서 RAW 강제 지원 |
| `patches/variants/` | **변종 통합 모듈**: SM-G996B(유럽)와 **SM-G996N(한국)** 을 하나의 vendor로 통합. G996N 펌웨어 blobs(`vendor/firmware/SM-G996N/` 의 NFC `libsn100u_fw.so`, `vts.bin`, NPU/mfc/audio bin 등 + `vendor/tee_SM-G996N/` TEE 파일 일체)를 G996B 경로로 이식 + `init.t2s.unify.rc` / SELinux 규칙 추가 |

> ℹ️ `installer/assertions.edify` 를 보면 **SM-G996B(유럽)와 SM-G996N(한국) 2개 변종**이 정의되어 있고,
> `patches/variants/` 에 G996N 전용 blobs 가 이미 들어 있습니다.
> 즉 **빌드 결과 zip 은 G996N 국내 내수용 기기에 그대로 설치 가능한 통합 패키지**입니다.
> `TARGET_FIRMWARE` 가 유럽(`SM-G996B/AUT`) 기준인 것은 변종 통합 방식상 의도된 구조입니다.

---

## 9. t2s 빌드 절차 (실행 순서)

```bash
cd /home/arch/disk2/UN1CA

# 0) (위 5~6장 참고) external 6종 수동 클론 + JDK 17 / lld 설치
#    ※ git submodule update --init --recursive 는 이 저장소에서 no-op

# 1) 빌드 환경 로드 (매 새 터미널에서 실행)
source buildenv.sh t2s

# 2) 필수 도구 자동 빌드 → out/tools (android-tools, apktool, erofs-utils,
#    img2sdat, samloader(venv), signapk) ※ 30분~1시간 이상 소요
./scripts/build_dependencies.sh

# 3) 삼성 FUS에서 순정 펌웨어 2종 다운로드 → out/odin
#    (SOURCE: SM-S901B/EUX..., TARGET: SM-G996B/AUT...)
#    ※ samloader가 필요하므로 2)를 먼저 실행할 것 / 재시도: -f
./scripts/download_fw.sh

# 4) 펌웨어 추출 → out/fw (system.img, vendor.img, product.img, system_ext.img, odm.img 등)
./scripts/extract_fw.sh

# 5) 메인 빌드
./scripts/make_rom.sh          # work_dir 구성 + 모든 패치/모드 적용 + APK 재빌드 + target-files zip
#    옵션:
#    -f, --force            : 변경사항이 없어도 강제 재빌드
#    -z, --build-rom-zip    : 플래셔블 zip 까지 생성 (이게 최종 산출물!)
#    -x, --no-target-files  : work_dir까지만, zip 만들지 않음

# 6) (5에서 -z 안 했으면) 최종 플래셔블 zip 생성
./scripts/build_flashable_zip.sh out/t2s_*-target_files.zip
```

> 5번의 `make_rom.sh`는 펌웨어가 없으면 **3~4단계를 자동으로 실행**합니다.
> 즉 2번 후 곧바로 `./scripts/make_rom.sh -z` 만 실행해도 전체 빌드가 됩니다.
> (Ci 워크플로우도 `build_dependencies → download_fw → extract_fw → make_rom` 순서를 사용)

### 빌드가 내부적으로 하는 일 (make_rom.sh)
1. `platform/exynos2100/patches` → `target/t2s/patches` → `unica/patches` → `unica/mods` 순으로 `apply_modules.sh` 실행
2. `smali/` 패치 수만큼 apktool로 APK/JAR 재빌드
3. `build_info.txt`와 함께 `out/t2s_<버전>-target_files.zip` 생성
4. (`-z`) `build_flashable_zip.sh` → OTA용 edify 인스톨러 + 기기별 프로프라이어터리 펌웨어 포함한 최종 zip 생성

### 산출물
| 생성물 | 위치 |
|---|---|
| 도구들 | `out/tools/bin/*` |
| 다운로드 펌웨어 | `out/odin/SM-G996B_AUT/.downloaded` 등 |
| 추출 펌웨어 | `out/fw/SM-G996B_AUT/system.img` 등 |
| 패치 작업 트리 | `out/target/t2s/work_dir/` |
| target-files zip | `out/t2s_3.2.0-<hash>-target_files.zip` |
| **플래셔블 zip** | `out/UN1CA_3.2.0-<hash>_<날짜>_t2s-sign.zip` (무서명 빌드는 `-sign` 없음) |
| 실행 로그 | `out/target/t2s/<명령>-<타임스탬프>.log` (ANSI 컬러 제거) |

### 설치 방법 (개요)
- 생성된 `UN1CA_*.zip` 은 기기 설치용 패키지입니다. 설치는 삼성 복구/OTA 방식(에디파이 인스톨러)으로 진행되며,
  자세한 설치는 UN1CA 공식 채널(README의 Discussions/Telegram)과 원저작자의 지침을 따르세요.
- 순정 펌웨어 설치 상태가 요구됩니다: `assertions.edify` 가 부트로더 버전을 검사합니다 —
  **SM-G996N(한국 내수용)은 `G996NKSSCHZA9`**, SM-G996B(유럽)는 `G996BXXSJHZC2`.
  그보다 최신 부트로더면 설치가 거부됩니다.
- 설치 후 업데이트는 롬에 내장된 **UN1CA OTA 업데이트 앱**으로 진행할 수 있습니다.

---

## 10. 문제 해결 (FAQ)

| 증상 | 원인/해결 |
|---|---|
| `external/*` 이 비어 있다 | **이 저장소에서는 `git submodule update --init --recursive` 가 no-op 이다** (gitlink 미등록). 5장대로 `.gitmodules` URL 로 직접 클론할 것 |
| `The following dependencies are missing` | 4장의 패키지 누락 → 해당 패키지 설치 후 재시도 |
| erofs/f2fs mount 실패 | `sudo modprobe erofs f2fs`, 커널은 리눅스(스톡) 권장 |
| FUS 다운로드 실패/타임아웃 | 네트워크 문제가 흔함 → `./scripts/download_fw.sh -f` 로 재시도 |
| `samloader` 오류 | venv 미설치 → `build_dependencies.sh` 다시 실행 |
| Gradle(apktool/signapk) 빌드 실패 | JDK 버전 문제 → JDK 17(Temurin) 고정 후 `JAVA_HOME`/PATH 설정 |
| 빌드 중단 후 재개 | `make_rom.sh`는 해시로 변경 감지 → 변경 없으면 스킵, 강제는 `-f` |
| `security/unica_*.pk8` 없음 경고 | 정상 — 실제 키는 저장소에 없음. 무서명(`--debug`) 또는 테스트 키로 빌드됨 |
| GitHub 액세스 불가 | `installer/customize.sh`가 releases에서 프로프라이어터리 펌웨어를 받으므로 GitHub 접근 필요 |
| 설치 중 `E3003` | 부트로더 버전 불일치 — SM-G996N 은 `G996NKSSCHZA9` 필요 (더 최신이면 거부) |
| 설치 중 `E3000` | afaneh92 복구 사용 중 — 다른 복구로 교체 필요 |
| `lld: command not found` / LTO 링크 실패 | `sudo pacman -S lld` |

---

## 11. 참고

- CI 구성 (Ubuntu 24.04): `shellcheck` → 매트릭스 빌드(모든 target) → `build_dependencies.sh` → `download_fw.sh` → `extract_fw.sh` → `make_rom.sh`
- 빌드 순정 펌웨어는 삼성 FUS에서 내려받으므로 **재배포 금지 라이선스**에 유의하세요.
- 라이선스: GPL-3.0 (외부 의존성은 각각의 라이선스: Apache-2.0, MIT, GPL-2.0 등 — README "라이선스 & 기여" 절)
- 외부 툴: android-tools, apktool, erofs-utils, img2sdat, samloader, signapk
- 디버그 빌드: `source buildenv.sh --debug t2s` 후 빌드 → 무서명 zip이 생성됨
- 서명 키가 없어도 빌드는 정상 진행됩니다. (실제 UN1CA 서명 zip을 만들려면 프로젝트 소유자의 키가 필요)
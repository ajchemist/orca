# 퍼스널 나이틀리 파이프라인 — 포크에서 패치 적용 빌드를 GitHub Actions로 뽑아 쓰기

> 업스트림에 PR을 올려두고 머지를 기다리는 동안, **내 패치(+ 큐레이션한 남의 PR)가
> 적용된 앱 번들**을 GitHub Actions 컴퓨트로 계속 뽑아서 운영 도구로 쓰는 패턴.
> SuperCmd(`mac-studio-m4m:~/Downloads/scm/SuperCmdLabs/SuperCmd`,
> `JOURNAL-gha-self-signed-build.md`)에서 검증한 구조를 일반화한 문서다.
> 이 저장소(Orca fork)의 인스턴스: `nightly` 브랜치 + `.github/workflows/nightly-macos.yml`.

작성일: 2026-08-12

---

## 0. 원칙

1. **PR 브랜치는 절대 오염시키지 않는다.** 워크플로 파일·manifest·이 문서는
   포크 전용 브랜치(`nightly`)에만 존재한다. PR diff에는 기능 코드만 남는다.
2. **빌드 대상은 브랜치가 아니라 조립 결과다.** nightly 브랜치 자체는 파이프라인
   정의만 담고, 실제 빌드 트리는 CI가 매번 `upstream/main + manifest의 패치들`을
   머지해 만든다. → 업스트림 트래킹에 리베이스 유지보수가 필요 없다.
3. **퍼블릭 포크면 macOS 러너 포함 Actions가 무료다.** 디버깅 반복에 부담이 없다.

## 1. 구조

```
fork(ajchemist/orca)
├── main                    ← 업스트림 미러 (수동 sync: git push fork origin/main:main)
├── feat/beads-task-source  ← PR 브랜치 (워크플로 없음, 깨끗함)
└── nightly                 ← 기본 브랜치(cron이 여기서 돈다). 내용물:
    ├── .github/workflows/nightly-macos.yml   ← 파이프라인
    ├── .github/nightly-refs.txt              ← 패치 manifest
    └── NIGHTLY-PIPELINE.md                   ← 이 문서
```

- **manifest**(`.github/nightly-refs.txt`): 한 줄에 ref 하나.
  - `fork:<branch>` — 내 패치 브랜치
  - `pr:<number>` — 업스트림 PR 헤드 (**큐레이션**: 머지 전인 남의 PR을 미리 포함)
  - `upstream:<branch>` — 업스트림 브랜치
- **트리거 2모드**:
  - `schedule`(03:30 KST) / 입력 없는 `workflow_dispatch` → **조립 모드**:
    upstream/main 체크아웃 → manifest 순서대로 `git merge --no-ff` → 빌드.
    머지 충돌은 그 자리에서 실패(로그에 어느 패치인지 표시) — 해당 브랜치를
    리베이스해 다시 푸시하면 끝.
  - `workflow_dispatch` + `ref` 입력 → **그대로 빌드 모드**: 특정 브랜치 검증용.
    (GitHub이 cron을 **기본 브랜치에서만** 돌리므로 포크 기본 브랜치를 nightly로
    바꿔둔다. 워크플로 파일이 없는 브랜치도 `ref` 입력으로 빌드 가능 — 이게
    PR 브랜치를 깨끗하게 유지하는 핵심.)

## 2. 사용법

```bash
# 수동 실행 (조립 모드)
gh workflow run nightly-macos.yml --repo ajchemist/orca --ref nightly

# 특정 브랜치를 그대로 빌드
gh workflow run nightly-macos.yml --repo ajchemist/orca --ref nightly -f ref=feat/beads-task-source

# 결과 확인 / 다운로드
gh run list --repo ajchemist/orca --workflow nightly-macos.yml -L 3
gh run download <run-id> --repo ajchemist/orca -n orca-nightly-macos-arm64 -D ~/Downloads
```

- **패치 추가/큐레이션**: `nightly` 브랜치의 `.github/nightly-refs.txt`에 줄 추가
  (`pr:13990` 처럼). 로컬 체크아웃 없이 GitHub 웹/Contents API로 편집해도 된다.
- **PR이 업스트림에 머지되면**: manifest에서 그 줄만 지운다.
- **fork main 동기화**(선택): `git fetch origin && git push fork origin/main:main`.

## 3. 다른 오픈소스 프로젝트에 적용하는 체크리스트

1. 포크 생성(퍼블릭 유지 — Actions 무료), PR 브랜치는 평소처럼.
2. `nightly` 브랜치 = 업스트림 main 스냅샷 + 워크플로 + manifest + 문서.
   (로컬 트리를 건드리기 싫으면 SuperCmd 저널처럼 `gh api PUT .../contents/...`로
   파일만 직접 커밋.)
3. 포크 **기본 브랜치를 nightly로** 변경: cron이 기본 브랜치에서만 돈다.
   `gh api -X PATCH repos/<me>/<repo> -f default_branch=nightly`
4. 워크플로 작성 시 프로젝트별로 반드시 확인할 것:
   - **패키지 매니저/노드 핀**: `packageManager` 필드(pnpm/action-setup가 읽음),
     `engines.node`(setup-node `node-version-file: package.json`).
   - **electron-builder publish 가드**: 설정에 `publish:` 블록이 있으면 CI에서
     `GH_TOKEN`을 절대 노출하지 말 것(태그 없는 브랜치 빌드는 기본 정책상 publish
     안 하지만, 토큰이 있으면 사고 경로가 생긴다). 필요하면 `--publish never`.
   - **서명**: 기본은 `CSC_IDENTITY_AUTO_DISCOVERY=false`(ad-hoc). macOS TCC
     권한(Accessibility 등)을 재빌드 사이에 유지하려면 SuperCmd 저널의
     self-signed 인증서 방식 도입 — p12를 base64로 secret에 넣고, CI 키체인
     import 후 `security add-trusted-cert -d -r trustRoot -p codeSign`으로
     trust시켜야 electron-builder가 valid identity로 인식한다(원문 저널 §2–3 참조).
   - **네이티브 툴체인**: Swift/Xcode 버전(러너 macos-15 권장), python, cmake 등.
   - **lockfile 드리프트**: 패치 머지 후 `--frozen-lockfile` 실패 대비 폴백.
5. 아티팩트 retention(기본 14일)과 이름에 아키텍처 명시.

## 4. 이 저장소(Orca) 특이사항

- 빌드: `pnpm run build:mac` — typecheck + 데스크톱 빌드 + macOS 네이티브 헬퍼
  (computer-use, notification-status) + electron-builder(x64/arm64 dual). CI 러너
  macos-15(arm64), 1회 30분 안팎.
- 산출물: `dist/orca-macos-arm64.dmg`(+zip), `dist/orca-macos-x64.dmg`.
- electron-builder 설정에 `publish: github(stablyai/orca)` 블록이 있으므로 위의
  publish 가드가 특히 중요하다.
- 로컬 빌드와 동일하게 버전이 `x.y.z-rc.N.local.<ts>.<sha>`로 스탬프되어 실행 중
  인스턴스와 구분된다.
- 미서명 번들이므로 다운로드 후 격리 해제가 필요할 수 있다:
  `xattr -dr com.apple.quarantine /Applications/Orca.app`

## 5. 참고

- 원형: SuperCmd의 `JOURNAL-gha-self-signed-build.md` (self-signed 서명까지 포함한
  풀버전; 인증서 secret 3종, electron-builder의 valid-identity 요구사항, Xcode/Swift
  버전 이슈 등 실패 사례별 해결책 기록).
- 이 파이프라인 도입 커밋: fork `nightly` 브랜치 최초 3커밋.

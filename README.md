# 내부 Zephyr 저장소 (doyoonk/zephyr)

upstream `zephyrproject-rtos/zephyr` 의 **릴리즈 태그만** 가져와서 내부 수정을 얹어
관리합니다. upstream 커밋 히스토리는 담지 않습니다 (태그당 depth 1).

- 원격: <https://github.com/doyoonk/zephyr> (public)
- 이 문서와 `tools/zvendor` 는 원격의 `meta` orphan 브랜치에 있습니다

## 워크스페이스 규약

작업 디렉터리 위치와 이름은 **각자 정합니다.** 문서에서는 그 최상위를 `$ZEPHYR_WS`
로 표기합니다. 아래 세 개의 **상대 배치만** 지키면 됩니다.

```
$ZEPHYR_WS/
├── zephyr.git      bare 저장소 (upstream 임포트 전용)
├── zephyr/         소스 작업용 clone — main, b_vX.Y
└── zephyr-meta/    meta 브랜치 clone — 이 문서와 tools/zvendor
```

셸에 한 번 정의해 두면 이후 명령을 그대로 복사해 쓸 수 있습니다. 예시일 뿐이니
원하는 경로로 바꾸십시오.

```sh
export ZEPHYR_WS=~/zephyrproject-rtos     # 예: ~/work/zephyr, /srv/zephyr, ...
export ZVENDOR_REPO=$ZEPHYR_WS/zephyr.git
```

`~/.bashrc` 에 넣어두면 편합니다. `$ZVENDOR_REPO` 는 임포트 담당자만 필요합니다.

## 빠른 시작

### 소스만 작업하는 경우 (대부분의 개발자)

`zephyr/` clone 하나면 충분합니다. bare 저장소도 `zephyr-meta/` 도 필요 없습니다.

```sh
export ZEPHYR_WS=~/zephyrproject-rtos
mkdir -p "$ZEPHYR_WS" && cd "$ZEPHYR_WS"
git clone https://github.com/doyoonk/zephyr.git
cd zephyr && git checkout b_v4.4
```

### upstream 태그를 임포트하는 경우 (릴리즈 담당자)

bare 저장소와 도구가 추가로 필요합니다.

```sh
export ZEPHYR_WS=~/zephyrproject-rtos
export ZVENDOR_REPO=$ZEPHYR_WS/zephyr.git
mkdir -p "$ZEPHYR_WS" && cd "$ZEPHYR_WS"

git clone -b meta --single-branch https://github.com/doyoonk/zephyr.git zephyr-meta

git init --bare --initial-branch=main zephyr.git
git --git-dir="$ZVENDOR_REPO" remote add origin \
    https://github.com/doyoonk/zephyr.git
git --git-dir="$ZVENDOR_REPO" remote add upstream \
    https://github.com/zephyrproject-rtos/zephyr.git
git --git-dir="$ZVENDOR_REPO" config remote.upstream.tagOpt --no-tags
git --git-dir="$ZVENDOR_REPO" config gc.auto 0
git --git-dir="$ZVENDOR_REPO" fetch origin \
    '+refs/heads/main:refs/heads/main' \
    '+refs/heads/b_*:refs/heads/b_*' \
    '+refs/vendor/*:refs/vendor/*'
git --git-dir="$ZVENDOR_REPO" symbolic-ref HEAD refs/heads/main

cd zephyr-meta && ./tools/zvendor status
```

`refs/vendor/*` 는 일반 clone 에 따라오지 않으므로 refspec 으로 명시해서 받습니다.
`meta` 는 일부러 제외합니다 — bare 는 임포트 전용입니다.

`git clone --mirror` 로도 되지만, `remote.origin.mirror=true` 가 설정되어 무심코
`git push origin` 을 하면 원격 ref 를 통째로 덮어쓰거나 지울 수 있으니 위 방식을
권장합니다.

`zvendor init` 은 **저장소를 맨 처음 만들 때만** 쓰십시오. 이미 원격이 존재하는데
`init` 부터 다시 하면 별개의 히스토리가 생깁니다.

같은 태그를 임포트하면 누가 어디서 하든 같은 해시가 나오므로, 위 절차로 만든
저장소는 기존 저장소와 완전히 동일합니다.

## 브랜치 규칙

```
main (v4.3.0 시작)
  │
  ├── b_v4.3 ── (v4.3.1) ── (v4.3.2) ── ...
  │
(v4.4.0)
  │
  ├── b_v4.4 ── (v4.4.1) ── (v4.4.2) ── ...
  │
(v4.5.0)
  │
  ├── b_v4.5 ── ...
  │
 ...
```

- `main` — upstream 마이너 베이스라인(`vX.Y.0`)만 순서대로 올라가는 트렁크
- `b_vX.Y` — `main` 의 `vX.Y.0` 지점에서 분기한 유지보수 브랜치.
  해당 라인의 패치 릴리즈(`vX.Y.1`, `vX.Y.2` …)를 순서대로 머지
- 개발/내부 수정은 `b_vX.Y` 에서 수행

원격 브랜치는 `main`, `b_vX.Y`, 그리고 예외인 `meta` 뿐이어야 합니다.
실험/검증 브랜치는 clone 안에 로컬로만 두고 push 하지 마십시오.

## 현재 상태

```
* 1a0e98f65 2026-08-06 (b_v4.4)       zephyr v4.4.2
* f8037beb4 2026-06-10                zephyr v4.4.1
* 00bb962e3 2026-04-14 (HEAD -> main) zephyr v4.4.0
| * c89767d46 2026-06-23 (b_v4.3)     zephyr v4.3.1
|/
* 687134f37 2025-11-13                zephyr v4.3.0
```

커밋 5개, 122MB. 내부 수정이 아직 없어 모두 fast-forward 로 붙어 일직선입니다.
내부 커밋이 생긴 뒤부터 `Merge zephyr vX.Y.Z into b_vX.Y` 머지 커밋이 생깁니다.

## 내부 수정이 버전업에도 남는 원리

upstream 태그를 그대로 덮어쓰면 내부 수정이 사라집니다. 그래서 태그 스냅샷을
`refs/vendor/*` 아래 별도 커밋 체인으로 쌓고, 그 체인을 브랜치에 **머지**합니다.

```
refs/vendor/lines/main    ●(v4.3.0) ───────────● (v4.4.0) ─────● (v4.5.0)
                           │                    │
refs/vendor/lines/v4.3     └─● (v4.3.1) ─● (v4.3.2)
                                                │
refs/vendor/lines/v4.4                          └─● (v4.4.1) ─● (v4.4.2)
```

각 `●` 는 upstream 트리 그대로에 **우리가 지정한 부모**를 붙인 단일 커밋입니다
(`git commit-tree`). 부모가 있으므로 merge-base 가 생기고, 머지는 3-way 로 동작해
"upstream 이 실제로 바꾼 부분"만 적용됩니다. 내부 수정은 충돌이 나지 않는 한 그대로
보존되고, 같은 줄을 upstream 도 고친 경우에만 충돌로 보고됩니다.

vendor 커밋은 **upstream 태그의 날짜 + 고정 identity(`Zephyr Vendor Import
<vendor@internal>`)** 로 만들어지므로, 같은 태그를 같은 부모 위에 임포트하면 항상
같은 해시가 나옵니다. 누가 어느 머신에서 임포트해도 결과가 동일합니다.

`refs/vendor/*` 는 브랜치가 아니라서 `git branch` 와 GitHub 브랜치 목록에는
`main`, `b_vX.Y`, `meta` 만 보입니다. 일반 clone 에는 따라오지 않으니 필요하면:

```sh
git fetch origin 'refs/vendor/*:refs/vendor/*'
```

## 사용법 (임포트)

```sh
cd "$ZEPHYR_WS/zephyr-meta"

./tools/zvendor status      # 브랜치 / 임포트된 태그 / vendor 라인
./tools/zvendor available   # 아직 안 가져온 upstream 릴리즈 태그
./tools/zvendor add v4.5.0  # 새 마이너: main 에 머지 + b_v4.5 자동 생성
./tools/zvendor add v4.4.3  # 패치 릴리즈: b_v4.4 에 머지
./tools/zvendor push        # main, b_*, refs/vendor/* 를 origin 으로 push
./tools/zvendor gc          # 재패킹
```

대상 bare 저장소는 **인자로 지정**합니다. 우선순위는
`--repo` > `ZVENDOR_REPO` 환경변수 > 기본값(`<스크립트>/../zephyr.git`) 입니다.
`$ZVENDOR_REPO` 를 export 해두지 않았다면 매번 붙이십시오.

```sh
./tools/zvendor --repo "$ZEPHYR_WS/zephyr.git" add v4.4.3
./tools/zvendor --repo ../zephyr.git status          # 상대경로도 가능
```

상대경로는 절대경로로 정규화되어 출력됩니다. 전체 옵션:

```
usage: zvendor [options] <command>

options:
  -r, --repo <path>     bare repo to operate on
  -u, --upstream <url>  upstream repository URL
  -h, --help            this message
```

`add` 는 태그 형식(`vX.Y.Z`)으로 대상을 자동 판별합니다.

- `Z == 0` → `refs/vendor/lines/main` 에 쌓고 `main` 에 머지, 없으면 `b_vX.Y` 생성
- `Z > 0` → `refs/vendor/lines/vX.Y` 에 쌓고 `b_vX.Y` 에 머지

태그는 반드시 순서대로 임포트해야 합니다 (`v4.4.1` 전에 `v4.4.0`).

### 충돌이 났을 때

`zvendor add` 가 중단되면서 안내를 출력합니다. **bare 에 worktree 를 만들지 말고**
`zephyr/` clone 에서 해결한 뒤 push 하십시오.

```sh
cd "$ZEPHYR_WS/zephyr"
git fetch origin 'refs/vendor/*:refs/vendor/*'
git checkout b_v4.4
git merge refs/vendor/tags/v4.4.3     # 충돌 해결 → git add → git commit
git push origin b_v4.4
```

그 다음 bare 를 동기화합니다.

```sh
git --git-dir="$ZEPHYR_WS/zephyr.git" fetch origin '+refs/heads/*:refs/heads/*'
```

## 개발 워크플로우

```sh
cd "$ZEPHYR_WS/zephyr"
git checkout b_v4.4
# 작업 후
git push origin b_v4.4
```

실험/검증은 clone 안에서 로컬 브랜치로만 하고 push 하지 않습니다.
실수 방지가 필요하면:

```sh
git remote set-url --push origin no_push
```

작업 디렉터리에 도구가 만드는 상태 폴더(`.omc/` 등)가 생긴다면 소스 트리에
`.gitignore` 를 추가하지 말고 로컬 exclude 를 쓰십시오. `.gitignore` 를 커밋하면
upstream 태그를 머지할 때마다 diff 에 끼어듭니다.

```sh
echo '.omc/' >> "$ZEPHYR_WS/zephyr/.git/info/exclude"
```

## meta 브랜치 (문서/도구, 유일한 예외)

원격 브랜치는 원칙적으로 `main` 과 `b_vX.Y` 뿐이지만, **`meta` 는 의도적인 예외**입니다.
이 문서(`README.md`)와 임포트 도구(`tools/zvendor`)를 담습니다.

`meta` 는 **orphan 브랜치**로, 부모 커밋이 없는 독립된 루트에서 시작합니다.
zephyr 소스와 히스토리도 파일도 전혀 공유하지 않으므로, upstream 태그를 머지할 때
이 파일들이 diff 에 끼어들지 않습니다.

```
* (b_v4.4)  zephyr v4.4.2
* (main)    zephyr v4.4.0
* ...       zephyr v4.3.0     ← 루트 1

* (meta)    docs + tools      ← 루트 2, 연결선 없음
```

히스토리가 무관하므로 실수로 머지되지 않습니다 (Git 2.9+ 는
`--allow-unrelated-histories` 를 명시해야만 허용). 절대 그렇게 하지 마십시오.

### 수정

`zephyr-meta/` clone 에서 직접 고치고 push 합니다.

```sh
cd "$ZEPHYR_WS/zephyr-meta"
# README.md / tools/zvendor 수정 후
git commit -am "meta: ..." && git push origin meta
```

`zephyr/` clone 에서 `git checkout meta` 는 하지 마십시오. 6만 개 가까운 파일이
통째로 사라졌다 돌아오면서 빌드 캐시가 무효화됩니다. 굳이 한 워킹트리에서
다루려면 `git worktree add ../zephyr-meta meta` 를 쓰십시오.

`zvendor push` 는 `main`, `b_*`, `refs/vendor/*` 만 push 하므로 `meta` 는 건드리지
않습니다. 문서 변경은 위처럼 직접 push 하십시오.

## 내부 수정 포워드 포트

`b_v4.3` 의 내부 수정은 자동으로 `b_v4.4` 로 넘어가지 않습니다.

- 전체 반영: `git checkout b_v4.4 && git merge b_v4.3`
- 특정 커밋만: `git cherry-pick <sha>`

`main` 에 공통 내부 수정을 두면 새 `b_vX.Y` 가 `main` 에서 분기하므로 자동 상속됩니다.
라인 전용 핫픽스만 `b_vX.Y` 에 두는 것을 권장합니다.

## 제약

- upstream 커밋 히스토리가 없으므로 upstream 이력에 대한 `git blame` / `git bisect`
  는 불가능합니다. 태그 단위 스냅샷만 있습니다.
- 이 저장소는 zephyr 소스 트리만 담습니다. west 워크스페이스(modules, HAL 등)는
  별도 manifest 저장소에서 관리합니다.
- upstream 태그는 원격에 push 하지 않습니다 (`refs/vendor/tags/*` 로만 추적).
- 저장소 첫 화면에는 기본 브랜치 `main` 의 내용(= zephyr 자체 `README.rst`)이 뜹니다.
  이 문서를 보려면 `meta` 브랜치를 선택해야 합니다.

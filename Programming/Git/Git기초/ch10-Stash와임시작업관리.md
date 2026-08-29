# 10장. Stash와 임시 작업 관리

📖 [◀ 목차](00-목차.md) | [◀ 이전: 9장. Rebase와 Cherry-pick](ch09-Rebase와CherryPick.md) | [다음: 11장. Pull Request와 코드 리뷰 워크플로우 ▶](ch11-PullRequest와코드리뷰워크플로우.md)

---

## 들어가며

한창 코드를 고치는 중인데 갑자기 "지금 당장 급한 버그를 고쳐야 한다"는 요청이 들어온 적이 있을 것입니다. 지금 작업 디렉터리에는 아직 커밋할 준비가 안 된 어중간한 변경 사항이 널려 있고, 그렇다고 이 상태로 브랜치를 옮기려니 Git이 막아설 때도 있습니다. 커밋하기엔 이르고, 그냥 버리기엔 아까운 이런 변경 사항을 잠시 서랍 속에 넣어뒀다가 나중에 다시 꺼내 쓸 수 있게 해주는 것이 이번 장의 주제인 `git stash`입니다.

이번 장에서는 `git stash`로 작업 중인 변경 사항을 임시 보관하는 방법, 보관해둔 목록을 확인하고 다시 꺼내 쓰는 방법, 이름을 붙여 여러 개를 구분하는 방법을 다룹니다. 이어서 브랜치를 깜빡하고 엉뚱한 곳에서 작업을 시작했을 때 stash로 상황을 정리하는 실무 시나리오를 살펴보고, 마지막으로 추적되지 않는 파일을 한 번에 정리하는 `git clean`까지 다룹니다.

## 10.1 stash란 무엇인가

`git stash`는 작업 디렉터리와 스테이징 영역의 변경 사항을 커밋하지 않은 채로 따로 떼어내 보관하고, 작업 디렉터리를 마지막 커밋 상태로 깨끗하게 되돌리는 명령입니다. 보관된 내용은 스택(stack) 형태로 쌓이며, 필요할 때 다시 꺼내 작업 디렉터리에 적용할 수 있습니다.

실습을 위해 저장소를 하나 만들고, 커밋 없이 파일을 고쳐보겠습니다.

```bash
mkdir stash-lab && cd stash-lab
git init
echo "function renderHeader() { return '<header></header>'; }" > ui.js
git add ui.js
git commit -m "feat: 기본 헤더 컴포넌트 추가"

echo "function renderFooter() { return '<footer></footer>'; }" >> ui.js
```

`ui.js`를 수정만 하고 아직 커밋하지 않은 상태입니다. `git status`로 확인해봅니다.

```bash
git status
```

```text
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   ui.js

no changes added to commit (use "git add" and/or "git commit -a")
```

이 상태에서 다른 브랜치로 급히 옮겨야 한다고 해봅시다. 이 변경 사항을 stash에 넣습니다.

```bash
git stash
```

```text
Saved working directory and index state WIP on main: 8f2a3c1 feat: 기본 헤더 컴포넌트 추가
```

다시 `git status`를 확인하면 작업 디렉터리가 깨끗해진 것을 볼 수 있습니다.

```bash
git status
```

```text
On branch main
nothing to commit, working tree clean
```

`ui.js`에 있던 변경 내용은 사라진 것이 아니라 stash라는 별도의 스택에 안전하게 보관되어 있습니다. `git stash`는 사실 `git stash push`의 축약형이며, 옵션 없이 실행하면 스테이징 여부와 관계없이 추적되고 있는 파일의 모든 변경 사항을 보관합니다. 다만 아직 Git이 추적하지 않는 새 파일(untracked file)은 기본적으로 stash 대상에 포함되지 않는다는 점에 주의해야 합니다. 새 파일까지 함께 보관하려면 `-u`(`--include-untracked`) 옵션을 붙입니다.

```bash
git stash -u
git stash --include-untracked
```

`.gitignore`로 무시되고 있는 파일까지 포함하고 싶다면 `-a`(`--all`) 옵션을 씁니다. 이 옵션은 빌드 산출물처럼 의도적으로 무시해둔 파일까지 건드릴 수 있으므로 평소에는 잘 쓰지 않고, 정말 필요한 경우에만 사용합니다.

## 10.2 stash 목록 확인하기: git stash list

stash는 여러 개를 동시에 쌓아둘 수 있습니다. 지금까지 보관해둔 목록은 `git stash list`로 확인합니다.

```bash
git stash list
```

```text
stash@{0}: WIP on main: 8f2a3c1 feat: 기본 헤더 컴포넌트 추가
```

`stash@{0}`이 가장 최근에 저장한 stash를 가리키는 이름입니다. stash를 하나 더 만들면 번호가 어떻게 밀리는지 확인해봅니다.

```bash
echo "function renderSidebar() { return '<aside></aside>'; }" >> ui.js
git stash
git stash list
```

```text
stash@{0}: WIP on main: 8f2a3c1 feat: 기본 헤더 컴포넌트 추가
stash@{1}: WIP on main: 8f2a3c1 feat: 기본 헤더 컴포넌트 추가
```

가장 최근 것이 항상 `stash@{0}`이고, 오래된 것일수록 번호가 커집니다. 이 번호는 고정된 이름이 아니라 스택에서의 상대적인 위치이므로, 앞의 stash를 꺼내거나 지우면 뒤에 있던 stash들의 번호도 하나씩 당겨진다는 점을 기억해 둡니다.

## 10.3 다시 꺼내 쓰기: git stash pop과 git stash apply

보관해둔 변경 사항을 다시 작업 디렉터리로 가져오는 방법은 두 가지입니다.

```bash
git stash pop
```

```text
On branch main
Changes not staged for commit:
        modified:   ui.js

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a)
```

`git stash pop`은 가장 최근 stash(`stash@{0}`)를 작업 디렉터리에 적용하고, 성공적으로 적용됐다면 그 stash를 스택에서 제거합니다. 목록을 확인해보면 하나가 줄어 있습니다.

```bash
git stash list
```

```text
stash@{0}: WIP on main: 8f2a3c1 feat: 기본 헤더 컴포넌트 추가
```

반면 `git stash apply`는 stash를 작업 디렉터리에 적용만 하고, 스택에서 제거하지 않습니다.

```bash
git stash apply
git stash list
```

```text
stash@{0}: WIP on main: 8f2a3c1 feat: 기본 헤더 컴포넌트 추가
```

목록에 여전히 남아 있는 것을 볼 수 있습니다. `apply`는 같은 stash를 다른 브랜치에도 적용해보고 싶거나, 적용한 결과가 마음에 들지 않을 때를 대비해 원본을 남겨두고 싶은 경우에 유용합니다. 반면 `pop`은 "이제 이 변경 사항은 다시 작업 디렉터리로 완전히 돌아왔으니 stash에는 더 이상 남겨둘 필요가 없다"는 것이 확실할 때 더 편리합니다.

두 명령 모두 특정 stash를 지정해서 적용할 수 있습니다. 기본값은 항상 `stash@{0}`입니다.

```bash
git stash apply stash@{1}
git stash pop stash@{1}
```

pop이든 apply든, 적용하려는 내용이 지금 작업 디렉터리의 변경 사항과 충돌하면 병합 충돌과 똑같은 방식으로 충돌 마커가 파일에 남습니다. 이 경우 5장에서 배운 절차(마커를 지우고 `git add`로 해결 표시)대로 정리하면 되고, `pop`은 충돌이 나면 안전을 위해 해당 stash를 스택에서 지우지 않고 그대로 남겨둡니다.

## 10.4 이름 붙여 저장하기: git stash push -m

stash가 여러 개 쌓이면 `stash@{0}`, `stash@{1}`이라는 번호만으로는 어떤 게 어떤 작업인지 구분하기 어렵습니다. `git stash push -m "메시지"`로 각 stash에 설명을 붙여두면 나중에 훨씬 알아보기 쉽습니다.

```bash
echo "function renderModal() { return '<div class=\"modal\"></div>'; }" >> ui.js
git stash push -m "모달 컴포넌트 작업 중"

echo "// 다크 모드 스타일 실험" > theme.css
git add theme.css
git stash push -m "다크 모드 실험"
```

```bash
git stash list
```

```text
stash@{0}: On main: 다크 모드 실험
stash@{1}: On main: 모달 컴포넌트 작업 중
```

메시지 없이 그냥 `git stash`로 저장했을 때 자동으로 붙던 `WIP on main: ...` 형태 대신, `On main: 다크 모드 실험`처럼 직접 붙인 메시지가 표시됩니다. `git stash push`는 특정 파일만 골라서 보관하는 것도 가능합니다.

```bash
git stash push -m "ui.js만 보관" ui.js
```

이렇게 경로를 지정하면 나머지 파일의 변경 사항은 작업 디렉터리에 그대로 남고, 지정한 파일만 stash로 옮겨집니다.

각 stash에 실제로 어떤 내용이 담겨 있는지는 `git stash show`로 확인합니다.

```bash
git stash show
```

```text
 theme.css | 1 +
 1 file changed, 1 insertion(+)
```

기본값은 가장 최근 stash(`stash@{0}`)에 대한 변경 통계(diffstat)만 보여줍니다. 실제 변경된 내용까지 자세히 보고 싶다면 `-p`(`--patch`) 옵션을 붙입니다.

```bash
git stash show -p stash@{1}
```

```text
diff --git a/ui.js b/ui.js
index 8f2a3c1..3f4a5b6 100644
--- a/ui.js
+++ b/ui.js
@@ -1,2 +1,3 @@
 function renderHeader() { return '<header></header>'; }
 function renderFooter() { return '<footer></footer>'; }
+function renderModal() { return '<div class="modal"></div>'; }
```

## 10.5 stash 지우기: git stash drop과 stash@{n}

더 이상 필요 없어진 stash는 `git stash drop`으로 지웁니다.

```bash
git stash drop stash@{1}
```

```text
Dropped stash@{1} (7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e)
```

인자를 생략하면 `pop`이나 `apply`와 마찬가지로 가장 최근 stash(`stash@{0}`)를 지웁니다. 특정 stash를 지정하고 싶을 때는 반드시 `stash@{n}` 형태로 번호를 명시합니다. 앞서 언급했듯 이 번호는 스택에서의 상대적인 위치이므로, 하나를 지우고 나면 그 뒤에 있던 stash들의 번호가 한 칸씩 당겨집니다. 여러 stash를 순서대로 지울 때는 매번 `git stash list`로 번호를 다시 확인하는 습관을 들이는 것이 안전합니다.

쌓아둔 stash를 한꺼번에 모두 지우고 싶다면 `git stash clear`를 씁니다.

```bash
git stash clear
```

이 명령은 되돌릴 수 없습니다. 스택에 있던 모든 stash가 즉시 사라지므로, 정말로 더 이상 필요 없는지 `git stash list`로 확인한 뒤에 실행하는 것이 좋습니다.

## 10.6 실무 시나리오: 브랜치 없이 시작해버린 작업 구하기

새 기능을 만들려면 먼저 `feature/...` 같은 브랜치를 새로 만들고 그 위에서 작업해야 한다는 것을 알고 있으면서도, 급한 마음에 브랜치를 만드는 것을 깜빡하고 `main`에서 바로 코드를 고치기 시작하는 실수는 생각보다 자주 일어납니다. 한참 작업하다가 뒤늦게 이 사실을 깨달았을 때, stash를 쓰면 지금까지의 작업을 잃지 않고 올바른 브랜치로 옮길 수 있습니다.

```bash
git switch main
# main에서 바로 작업을 시작해버린 상황
echo "function validateEmail(v) { return v.includes('@'); }" >> ui.js
```

```bash
git status
```

```text
On branch main
Changes not staged for commit:
        modified:   ui.js
```

아직 커밋 전이므로, 변경 사항을 stash에 담아두고 새 브랜치를 만들어 옮긴 뒤 다시 꺼내면 됩니다.

```bash
git stash push -m "이메일 검증 함수 작업분"
git switch -c feature/email-validation
git stash pop
```

```text
On branch feature/email-validation
Changes not staged for commit:
        modified:   ui.js

Dropped refs/stash@{0} (...)
```

이제 `main`은 깨끗한 상태로 남아 있고, 작업 내용은 새로 만든 `feature/email-validation` 브랜치로 안전하게 옮겨졌습니다.

이 "stash에 담아두고 → 새 브랜치를 만들고 → 다시 꺼낸다"는 세 단계는 자주 반복되는 흐름이라, Git은 이 과정을 한 번에 처리하는 명령을 따로 제공합니다.

```bash
git stash branch feature/email-validation
```

`git stash branch <브랜치이름>`은 stash를 만들었던 시점의 커밋을 기준으로 새 브랜치를 만들고 그 브랜치로 옮긴 뒤, 가장 최근 stash를 적용하고 성공하면 스택에서 지우는 일까지 한 번에 처리합니다. 특히 stash를 저장해둔 뒤로 `main`이 계속 앞으로 나아가서 지금의 `main` 위에 stash를 그대로 적용하면 충돌이 날 만한 상황이라면, `git stash branch`가 stash 시점의 커밋을 기준으로 브랜치를 만들어주므로 훨씬 안전합니다. 특정 stash를 지정하고 싶다면 이름 뒤에 이어서 씁니다.

```bash
git stash branch feature/email-validation stash@{2}
```

## 10.7 추적되지 않는 파일 정리하기: git clean

작업하다 보면 실험 삼아 만든 파일, 빌드 도구가 생성한 임시 파일처럼 Git이 추적하지 않는(untracked) 파일이 작업 디렉터리에 쌓이기도 합니다. `git status`는 이런 파일들을 "Untracked files"로 알려주지만, 하나씩 손으로 지우기는 번거롭습니다. `git clean`은 이런 추적되지 않는 파일을 한 번에 정리해주는 명령입니다.

`git clean`은 되돌릴 수 없는 삭제를 실행하는 명령이라서, 아무 옵션 없이 실행하면 Git이 기본적으로 실행을 거부합니다.

```bash
git clean
```

```text
fatal: clean.requireForce defaults to true and neither -i, -n, nor -f given; refusing to clean
```

### 미리보기: git clean -n

실제로 지우기 전에 무엇이 지워질지 미리 확인하려면 `-n`(`--dry-run`) 옵션을 씁니다.

```bash
echo "temp" > scratch.tmp
mkdir build && echo "out" > build/output.txt

git clean -n
```

```text
Would remove scratch.tmp
```

기본적으로 `git clean`은 파일만 대상으로 하고, 디렉터리(`build/`)는 지우지 않습니다. `-n`은 실제로 아무것도 지우지 않고 "지운다면 무엇이 지워질지"만 미리 보여주므로, `git clean`을 실행하기 전에는 항상 먼저 `-n`으로 확인해보는 습관을 들이는 것이 좋습니다.

### 실제로 삭제하기: -f와 -d

미리보기 결과가 예상과 같다면 `-f`(`--force`)를 붙여 실제로 삭제합니다.

```bash
git clean -f
```

```text
Removing scratch.tmp
```

디렉터리까지 함께 정리하고 싶다면 `-d` 옵션을 추가합니다.

```bash
git clean -n -d
```

```text
Would remove build/
```

```bash
git clean -f -d
# 또는 붙여서
git clean -fd
```

```text
Removing build/
```

`.gitignore`로 무시되고 있는 파일(예: `node_modules/`, 빌드 산출물)까지 포함해서 정리하고 싶다면 `-x` 옵션을 더합니다.

```bash
git clean -fdx
```

`-x`는 정말로 필요한 상황이 아니면 쓰지 않는 것이 좋습니다. `.gitignore`에 등록해둔 파일 중에는 로컬 설정 파일이나 IDE 캐시처럼 지워지면 다시 만들기 번거로운 것들도 섞여 있을 수 있기 때문입니다.

### git clean의 위험성

`git clean -f`로 지워진 파일은 Git이 애초에 추적한 적이 없는 파일이므로, `git checkout`이나 `git reset`처럼 되돌릴 수 있는 안전망이 없습니다. 휴지통으로 가는 것도 아니고, 어떤 stash나 커밋에도 기록되지 않은 채 즉시 완전히 사라집니다. 따라서 `git clean`을 실행하기 전에는 다음을 지키는 것이 안전합니다.

- 반드시 `-n`으로 먼저 무엇이 지워질지 확인한다.
- `-d`, 특히 `-x`를 붙일 때는 지워질 목록을 더 꼼꼼히 살펴본다.
- 하나씩 확인하며 지우고 싶다면 대화형 모드인 `-i`(`--interactive`)를 쓴다.

```bash
git clean -i
```

```text
Would remove the following items:
  scratch.tmp  build/
*** Commands ***
    1: clean                2: filter by pattern    3: select by numbers
    4: ask each             5: quit                  6: help
What now>
```

`-i`는 지울 후보 목록을 보여주고, 패턴으로 걸러내거나 번호로 골라서 선택적으로 지울 수 있게 해줍니다. 무엇이 지워질지 100% 확신이 서지 않는 상황에서는 `-f`로 한 번에 지우기보다 `-i`로 하나씩 확인하는 편이 사고를 줄이는 길입니다.

참고로 `git clean`은 추적되지 않는 파일을 대상으로 하는 명령이고, 이미 커밋된 파일이나 스테이징된 변경 사항은 건드리지 않습니다. 커밋된 파일을 이전 상태로 되돌리고 싶다면 `git clean`이 아니라 `git restore`나 `git reset` 같은 명령을 써야 합니다.

## 요약

- `git stash`(`git stash push`의 축약형)는 작업 디렉터리와 스테이징 영역의 변경 사항을 스택에 임시 보관하고 작업 디렉터리를 마지막 커밋 상태로 되돌린다. 기본적으로 추적되지 않는 새 파일은 대상에서 빠지며, 포함하려면 `-u`(또는 무시된 파일까지 포함하려면 `-a`)를 쓴다.
- `git stash list`로 보관된 stash 목록을 확인한다. `git stash pop`은 적용 후 스택에서 제거하고, `git stash apply`는 적용만 하고 스택에 남긴다.
- `git stash push -m "메시지"`로 이름을 붙여 저장하면 여러 stash를 구분하기 쉬워지고, `git stash show`(`-p`를 붙이면 상세 diff)로 내용을 확인할 수 있다.
- `git stash drop stash@{n}`으로 특정 stash를 지우고, `git stash clear`로 전부 지운다. `stash@{n}`의 번호는 스택 안에서의 상대적 위치이므로 항목을 지우거나 꺼낼 때마다 바뀔 수 있다.
- 브랜치를 만들지 않고 실수로 작업을 시작했을 때는 `git stash push`로 변경 사항을 옮겨두고 새 브랜치를 만들어 `git stash pop`으로 되찾거나, 이 과정을 한 번에 처리하는 `git stash branch <브랜치이름>`을 쓸 수 있다.
- `git clean`은 추적되지 않는 파일을 정리한다. `-n`으로 항상 먼저 미리보기하고, `-f`로 파일을, `-fd`로 디렉터리까지, `-fdx`로 무시된 파일까지 지운다. 삭제된 내용은 복구할 방법이 없으므로 신중하게 다루거나 `-i`로 하나씩 확인하며 지운다.

## 연습문제

1. `git stash pop`과 `git stash apply`의 차이를 설명하고, 각각 어떤 상황에서 선택해야 하는지 예를 들어 서술하시오.
2. stash를 세 개 만든 뒤 `git stash drop stash@{0}`을 실행했다. 남은 두 stash의 번호가 어떻게 바뀌는지 설명하시오.
3. `git stash push -m "메시지"`로 stash를 저장하는 방법과, 특정 stash의 상세 변경 내용을 `-p` 옵션으로 확인하는 명령을 각각 작성하시오.
4. `main`에서 실수로 브랜치를 만들지 않고 작업을 시작했다는 것을 뒤늦게 깨달았다. `git stash`를 이용해 이 작업을 새 브랜치 `feature/urgent-fix`로 옮기는 절차를 두 가지 방법(단계별 명령, `git stash branch` 한 번에 처리)으로 각각 작성하시오.
5. `git clean -fdx`가 `git clean -fd`와 어떻게 다른지, 그리고 `git clean`을 실행하기 전에 왜 항상 `-n` 옵션으로 먼저 확인해야 하는지 설명하시오.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 9장. Rebase와 Cherry-pick](ch09-Rebase와CherryPick.md) | [다음: 11장. Pull Request와 코드 리뷰 워크플로우 ▶](ch11-PullRequest와코드리뷰워크플로우.md)

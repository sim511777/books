# 18장. 실전 프로젝트: 협업 시나리오로 배우는 Git 워크플로우

📖 [◀ 목차](00-목차.md) | [◀ 이전: 17장. Git Hooks와 자동화](ch17-GitHooks와자동화.md) | [다음: 부록 ▶](appendix-부록.md)

---

## 들어가며

이 책은 저장소 만들기부터 시작해 커밋, 브랜치, 원격 저장소, rebase, Pull Request, 브랜치 전략, 태그, 그리고 앞 장의 Git Hooks까지 Git이 제공하는 기능들을 하나씩 살펴봤습니다. 하지만 실무에서는 이 기능들이 따로따로 쓰이지 않습니다. "브랜치를 만들고, 커밋하고, 푸시하고, 리뷰받고, 병합하고, 정리하는" 흐름 전체가 하나로 이어져서 매일 반복됩니다.

이번 장에서는 지금까지 배운 내용을 하나의 시나리오로 엮어 처음부터 끝까지 따라가 봅니다. 두 명의 협업자 **A**와 **B**가 아주 작은 프로젝트를 함께 만든다고 가정하고, 저장소 초기화부터 기능 개발, 코드 리뷰, 충돌 해결, 릴리스 태깅까지 실제로 실행 가능한 Git 명령어를 순서대로 따라가겠습니다. 실습을 진행할 때는 A와 B의 작업을 한 컴퓨터에서 번갈아 가며 실행해도 되고, 필요하다면 별도의 디렉터리 두 개를 각각의 로컬 저장소로 사용해 실제 협업과 더 비슷하게 시험해 볼 수도 있습니다.

## 예제 프로젝트 소개: 할 일 목록 스크립트

이번 장에서 다룰 프로젝트는 `todo.sh`라는 셸 스크립트 하나로 이루어진 아주 작은 할 일 목록 관리 도구입니다. 특정 프로그래밍 언어의 문법을 깊이 이해할 필요는 없으며, 오직 Git 워크플로우를 연습하기 위한 소재로만 사용합니다. 최종적으로는 다음과 같은 파일들로 구성됩니다.

- `README.md` - 프로젝트 소개
- `todo.sh` - 할 일을 추가/조회하는 셸 스크립트
- `CHANGELOG.md` - 버전별 변경 이력 (13장에서 다룬 태그와 연결됩니다)

## 1단계: 저장소 초기화와 원격 연결

먼저 A가 프로젝트를 처음 시작하는 사람이라고 가정합니다. A는 로컬에 저장소를 만들고 첫 커밋을 남긴 뒤, 원격 저장소를 연결합니다. 원격 저장소는 GitHub 등에 미리 빈 저장소(`todo-app`)를 만들어 두었다고 가정합니다.

```bash
mkdir todo-app
cd todo-app
git init
```

```bash
cat > README.md << 'EOF'
# todo-app

셸 스크립트로 만든 아주 작은 할 일 목록 관리 도구입니다.
EOF
```

```bash
cat > todo.sh << 'EOF'
#!/bin/sh
# todo.sh - 아주 간단한 할 일 목록 관리 스크립트

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

echo "todo-app: 아직 명령이 구현되지 않았습니다."
EOF
chmod +x todo.sh
```

```bash
git add README.md todo.sh
git commit -m "chore: 프로젝트 초기 구조 생성"
```

이제 원격 저장소를 연결하고 첫 푸시를 진행합니다. 6장에서 다룬 것처럼 기본 브랜치 이름은 `main`을 사용합니다.

```bash
git branch -M main
git remote add origin https://github.com/example/todo-app.git
git push -u origin main
```

`-u`(`--set-upstream`) 옵션 덕분에 이후로는 `git push`, `git pull`만 입력해도 `origin`의 `main` 브랜치와 자동으로 연결됩니다.

## 2단계: 각자 기능 브랜치 만들기

이제 협업자 B가 이 저장소를 clone해서 합류합니다.

```bash
git clone https://github.com/example/todo-app.git
cd todo-app
```

A는 "할 일 추가" 기능을, B는 "할 일 목록 조회" 기능을 각자 독립적으로 개발하기로 합니다. 두 사람 모두 4장에서 배운 대로 `main`에서 새 기능 브랜치를 만들어 작업을 시작합니다.

A의 컴퓨터에서:

```bash
git switch main
git pull origin main
git switch -c feature/add-task
```

B의 컴퓨터에서:

```bash
git switch main
git pull origin main
git switch -c feature/list-task
```

두 브랜치 모두 같은 지점(`main`의 최신 커밋)에서 갈라져 나왔지만, 이후로는 서로 독립적으로 커밋이 쌓입니다.

## 3단계: 각자 작업하고 커밋, 푸시하기

**A의 작업.** `todo.sh`에 할 일을 추가하는 기능을 작성합니다.

```bash
cat > todo.sh << 'EOF'
#!/bin/sh
# todo.sh - 아주 간단한 할 일 목록 관리 스크립트

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

add_task() {
    echo "$1" >> "$TASK_FILE"
    echo "추가됨: $1"
}

case "$1" in
    add)
        add_task "$2"
        ;;
    *)
        echo "사용법: todo.sh add \"할 일 내용\""
        ;;
esac
EOF
```

```bash
git add todo.sh
git commit -m "feat: 할 일 추가(add) 기능 구현"
git push -u origin feature/add-task
```

**B의 작업.** 같은 `todo.sh`에 목록을 조회하는 기능을 작성합니다. B는 A의 변경 사항을 아직 모르는 상태로, `main`을 기준으로 작업합니다.

```bash
cat > todo.sh << 'EOF'
#!/bin/sh
# todo.sh - 아주 간단한 할 일 목록 관리 스크립트

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

list_tasks() {
    if [ ! -s "$TASK_FILE" ]; then
        echo "등록된 할 일이 없습니다."
        return
    fi
    cat -n "$TASK_FILE"
}

case "$1" in
    list)
        list_tasks
        ;;
    *)
        echo "사용법: todo.sh list"
        ;;
esac
EOF
```

```bash
git add todo.sh
git commit -m "feat: 할 일 목록 조회(list) 기능 구현"
git push -u origin feature/list-task
```

이 시점에서 원격 저장소에는 `main`, `feature/add-task`, `feature/list-task` 세 브랜치가 모두 존재하며, 두 기능 브랜치는 아직 서로의 변경 사항을 모르는 상태입니다.

## 4단계: Pull Request 열기와 코드 리뷰

A가 먼저 작업을 마쳤으므로, GitHub 저장소 페이지에서 `feature/add-task` → `main`으로 향하는 Pull Request를 엽니다. 11장에서 다룬 것처럼 PR은 "이 브랜치의 변경 사항을 `main`에 병합해도 되는지" 검토를 요청하는 절차입니다.

PR을 연 뒤 B가 리뷰어로 지정되어 변경된 `todo.sh`를 살펴보고 다음과 같은 리뷰 코멘트를 남겼다고 가정해 봅시다.

> **B의 리뷰 코멘트**: `add_task` 함수에서 `$1`이 비어 있는 경우(할 일 내용을 안 넘긴 경우)에 대한 처리가 없어요. 빈 문자열이 그대로 추가되지 않도록 검사를 하나 추가해 주시겠어요?

A는 이 코멘트를 반영해 코드를 수정합니다.

```bash
git switch feature/add-task
```

```bash
cat > todo.sh << 'EOF'
#!/bin/sh
# todo.sh - 아주 간단한 할 일 목록 관리 스크립트

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

add_task() {
    if [ -z "$1" ]; then
        echo "할 일 내용을 입력해 주세요. 사용법: todo.sh add \"할 일 내용\""
        return 1
    fi
    echo "$1" >> "$TASK_FILE"
    echo "추가됨: $1"
}

case "$1" in
    add)
        add_task "$2"
        ;;
    *)
        echo "사용법: todo.sh add \"할 일 내용\""
        ;;
esac
EOF
```

```bash
git add todo.sh
git commit -m "fix: add 명령에 빈 값 검증 추가"
git push origin feature/add-task
```

같은 브랜치로 다시 푸시하면, 이미 열려 있던 PR에는 새 커밋이 자동으로 반영됩니다. 별도로 PR을 새로 만들 필요는 없습니다. A는 리뷰 코멘트에 "반영했습니다. 확인 부탁드립니다."라고 답을 남깁니다.

## 5단계: 승인과 병합

B가 수정된 코드를 다시 확인하고 문제가 없다고 판단해 PR을 **승인(Approve)**합니다. 12장에서 다룬 GitHub Flow에서는 승인을 받은 브랜치를 곧바로 `main`에 병합하는 것이 기본 흐름입니다. GitHub의 PR 화면에서는 보통 다음과 같은 병합 방식을 선택할 수 있습니다.

- **Merge commit**: 두 브랜치의 이력을 그대로 유지하며 병합 커밋을 하나 추가합니다.
- **Squash and merge**: 기능 브랜치의 모든 커밋을 하나로 합쳐 `main`에 커밋 하나로 반영합니다.
- **Rebase and merge**: 기능 브랜치의 커밋들을 `main` 끝에 재배치해 병합 커밋 없이 이어 붙입니다.

이번 시나리오에서는 커밋 이력을 깔끔하게 유지하기 위해 **Squash and merge**를 선택했다고 가정합니다. 병합 버튼을 누르면 원격 저장소의 `main` 브랜치에 변경 사항이 반영됩니다. 이 과정을 로컬 명령으로 표현하면 다음과 같습니다.

```bash
git switch main
git pull origin main
```

`git pull`을 실행하면 방금 병합된 A의 변경 사항이 A와 B 모두의 로컬 `main`에 반영됩니다.

## 6단계: 병합이 끝난 브랜치 정리

병합이 끝난 기능 브랜치는 더 이상 필요하지 않으므로 정리합니다. 로컬 브랜치와 원격 브랜치를 각각 삭제해야 합니다.

```bash
git branch -d feature/add-task
```

`-d`(`--delete`)는 해당 브랜치가 현재 `main`에 완전히 병합된 경우에만 삭제를 허용하는 안전한 옵션입니다. 아직 병합되지 않은 커밋이 남아 있다면 Git이 삭제를 거부하고 경고를 보여줍니다.

원격 저장소에 남아 있는 브랜치도 삭제합니다.

```bash
git push origin --delete feature/add-task
```

GitHub의 PR 화면에서도 병합 후 "Delete branch" 버튼으로 같은 작업을 할 수 있습니다. 어느 쪽을 쓰든 결과는 같습니다.

## 7단계: 충돌 상황과 해결

A의 PR이 병합된 뒤, B의 `feature/list-task` 브랜치는 여전히 병합 전의 옛 `main`을 기준으로 하고 있습니다. B가 이어서 작업을 마무리하고 병합을 준비하는 과정에서 충돌이 발생하는 상황을 만들어 보겠습니다.

B는 `case` 문 부분을 A와 비슷한 위치에서 수정했다고 가정합니다.

```bash
git switch feature/list-task
```

```bash
cat > todo.sh << 'EOF'
#!/bin/sh
# todo.sh - 아주 간단한 할 일 목록 관리 스크립트

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

list_tasks() {
    if [ ! -s "$TASK_FILE" ]; then
        echo "등록된 할 일이 없습니다."
        return
    fi
    cat -n "$TASK_FILE"
}

case "$1" in
    list)
        list_tasks
        ;;
    ls)
        list_tasks
        ;;
    *)
        echo "사용법: todo.sh list"
        ;;
esac
EOF
git add todo.sh
git commit -m "feat: list 명령에 ls 별칭 추가"
```

이제 B는 자신의 브랜치를 최신 `main`(A의 변경 사항이 이미 반영된) 위에 올려놓기 위해 9장에서 배운 rebase를 사용합니다.

```bash
git fetch origin
git rebase origin/main
```

두 브랜치 모두 `todo.sh`의 같은 부분(파일 앞부분의 함수 정의와 `case` 문)을 서로 다르게 바꿨기 때문에, Git은 자동으로 병합하지 못하고 다음과 같은 메시지를 출력하며 멈춥니다.

```
CONFLICT (content): Merge conflict in todo.sh
error: could not apply <커밋 해시>... feat: list 명령에 ls 별칭 추가
```

`todo.sh`를 열어 보면 5장에서 배운 충돌 표시가 보입니다.

```
<<<<<<< HEAD
add_task() {
    if [ -z "$1" ]; then
        echo "할 일 내용을 입력해 주세요. 사용법: todo.sh add \"할 일 내용\""
        return 1
    fi
    echo "$1" >> "$TASK_FILE"
    echo "추가됨: $1"
}

case "$1" in
    add)
        add_task "$2"
        ;;
=======
list_tasks() {
    if [ ! -s "$TASK_FILE" ]; then
        echo "등록된 할 일이 없습니다."
        return
    fi
    cat -n "$TASK_FILE"
}

case "$1" in
    list)
        list_tasks
        ;;
    ls)
        list_tasks
        ;;
>>>>>>> feat: list 명령에 ls 별칭 추가
```

두 기능(`add`와 `list`)이 같은 스크립트, 같은 `case` 문 안에 함께 있어야 하므로, 충돌을 해결하며 두 기능을 하나로 합쳐 줍니다.

```bash
cat > todo.sh << 'EOF'
#!/bin/sh
# todo.sh - 아주 간단한 할 일 목록 관리 스크립트

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

add_task() {
    if [ -z "$1" ]; then
        echo "할 일 내용을 입력해 주세요. 사용법: todo.sh add \"할 일 내용\""
        return 1
    fi
    echo "$1" >> "$TASK_FILE"
    echo "추가됨: $1"
}

list_tasks() {
    if [ ! -s "$TASK_FILE" ]; then
        echo "등록된 할 일이 없습니다."
        return
    fi
    cat -n "$TASK_FILE"
}

case "$1" in
    add)
        add_task "$2"
        ;;
    list|ls)
        list_tasks
        ;;
    *)
        echo "사용법: todo.sh add \"내용\" | todo.sh list"
        ;;
esac
EOF
```

충돌을 해결한 파일을 스테이징하고 rebase를 계속 진행합니다.

```bash
git add todo.sh
git rebase --continue
```

rebase는 커밋 이력을 다시 쓰는 작업이므로, 이미 원격에 `feature/list-task`를 한 번 푸시한 적이 있다면 일반적인 `git push`로는 거부됩니다. 이런 경우 9장에서 다룬 것처럼 `--force-with-lease` 옵션을 사용해 강제로 덮어씁니다.

```bash
git push --force-with-lease origin feature/list-task
```

`--force-with-lease`는 `--force`와 달리, 원격 브랜치가 내가 마지막으로 확인한 상태 그대로일 때만 강제 푸시를 허용합니다. 다른 사람이 그 사이에 같은 브랜치에 추가로 푸시했다면 오히려 실패하도록 막아 주므로, 협업 중에는 `--force`보다 안전한 선택입니다.

이제 B의 PR을 열어 A와 같은 방식으로 리뷰와 승인을 거쳐 `main`에 병합하고, 6단계와 같은 방법으로 `feature/list-task` 브랜치를 로컬과 원격에서 정리합니다.

```bash
git switch main
git pull origin main
git branch -d feature/list-task
git push origin --delete feature/list-task
```

## 8단계: 태그를 찍어 릴리스하기

`add`와 `list` 기능이 모두 `main`에 병합되었으니, 이 시점을 첫 번째 릴리스로 표시해 보겠습니다. 13장에서 다룬 태그를 사용합니다.

```bash
git switch main
git pull origin main
```

```bash
cat > CHANGELOG.md << 'EOF'
# 변경 이력

## v1.0.0
- 할 일 추가(add) 기능
- 할 일 목록 조회(list) 기능
EOF
git add CHANGELOG.md
git commit -m "docs: v1.0.0 변경 이력 추가"
git push origin main
```

```bash
git tag -a v1.0.0 -m "todo-app 첫 번째 릴리스: add, list 기능"
git push origin v1.0.0
```

`git tag -a`로 만든 태그는 주석이 달린(annotated) 태그로, 태그를 만든 사람과 날짜, 메시지가 함께 기록되어 릴리스를 표시하는 용도에 적합합니다. `git push origin v1.0.0`처럼 태그 이름을 명시해야 원격 저장소로 전송됩니다. 일반적인 `git push`는 커밋만 보낼 뿐 태그는 함께 보내지 않는다는 점을 기억해 두세요.

GitHub에서는 이렇게 푸시된 태그를 기반으로 "Release" 페이지를 만들어 다운로드 링크나 릴리스 노트를 함께 제공할 수도 있습니다.

## 지금까지의 흐름 정리

여덟 단계를 거치는 동안 A와 B는 다음과 같은 흐름을 따랐습니다.

1. 저장소 초기화, 원격 연결(`git init`, `git remote add`, `git push -u`)
2. 각자 기능 브랜치 생성(`git switch -c`)
3. 독립적인 커밋과 푸시(`git add`, `git commit`, `git push`)
4. Pull Request와 리뷰(플랫폼 기능 + 리뷰 반영 커밋)
5. 승인과 병합(Squash and merge)
6. 브랜치 정리(`git branch -d`, `git push origin --delete`)
7. rebase 중 충돌 발생과 해결(`git rebase`, 충돌 표시 수정, `git rebase --continue`, `--force-with-lease`)
8. 릴리스 태깅(`git tag -a`, `git push origin <태그>`)

이 흐름은 팀의 규모나 프로젝트의 성격에 따라 세부적으로는 달라질 수 있지만, "기능 브랜치에서 작업 → 리뷰 → 병합 → 정리"라는 뼈대 자체는 GitHub Flow를 사용하는 대부분의 프로젝트에서 공통적으로 나타납니다.

## 확장 아이디어

이번 장에서 다룬 흐름에 다음과 같은 요소를 더하면 더 실전에 가까운 협업 환경을 만들 수 있습니다. 구체적인 설정 방법은 이 책의 범위를 벗어나지만, 앞으로 학습을 이어갈 때 어떤 이름을 검색해 보면 좋을지 소개합니다.

- **GitHub Actions**를 이용한 CI(지속적 통합) 연동. PR이 열릴 때마다 자동으로 테스트를 실행하고, 결과를 PR 화면에 표시할 수 있습니다. 17장에서 다룬 `pre-push` 훅이 "내 컴퓨터에서" 하던 검사를, 팀 전체 차원에서 서버가 대신 해 주는 셈입니다.
- **브랜치 보호 규칙(branch protection rules)**. `main` 브랜치에 직접 푸시를 막고, 반드시 PR과 최소 1인 이상의 승인을 거치도록 강제할 수 있습니다.
- **필수 상태 검사(required status checks)**. CI 테스트가 통과하지 않으면 병합 버튼 자체가 비활성화되도록 설정할 수 있습니다.
- **이슈 트래커와 프로젝트 보드 연동**. 커밋 메시지나 PR 설명에 이슈 번호를 적어 두면, 이슈와 PR을 자동으로 연결해 진행 상황을 추적할 수 있습니다.

## 요약

- 실전 협업은 브랜치 생성, 커밋, 푸시, PR, 리뷰, 병합, 브랜치 정리라는 흐름이 반복되는 과정이며, 이 책에서 배운 명령어들은 이 흐름 곳곳에서 함께 쓰입니다.
- PR에 새 커밋을 푸시하면 별도 조작 없이도 열려 있는 PR에 자동으로 반영되므로, 리뷰 코멘트를 반영하는 과정이 자연스럽게 이어집니다.
- 병합 후에는 `git branch -d`와 `git push origin --delete`로 로컬과 원격의 기능 브랜치를 모두 정리하는 습관을 들이는 것이 좋습니다.
- 여러 브랜치가 같은 부분을 수정하면 rebase나 merge 도중 충돌이 발생할 수 있으며, 충돌 표시를 직접 수정하고 `git add` 후 작업을 이어가는 절차는 병합과 리베이스 모두 공통적으로 적용됩니다.
- rebase로 이미 푸시했던 커밋 이력이 바뀌었다면 `git push --force-with-lease`로 안전하게 덮어쓸 수 있습니다.
- 릴리스 시점에는 `git tag -a`로 주석 태그를 만들고 `git push origin <태그>`로 명시적으로 태그를 전송해야 원격 저장소에 반영됩니다.

## 연습문제

1. 이번 장의 시나리오에서 A와 B가 각자의 기능 브랜치를 만들 때 왜 `main`을 먼저 `pull`한 뒤에 브랜치를 만들었는지 설명해 보세요.
2. PR에 새 커밋을 추가로 푸시했을 때 별도로 새 PR을 만들지 않아도 되는 이유를 설명해 보세요.
3. `git branch -d`가 삭제를 거부하는 상황이 어떤 경우인지 설명하고, 이런 경우에 사용할 수 있는 다른 명령을 적어 보세요.
4. rebase 도중 발생한 충돌을 해결한 뒤 원격 브랜치로 다시 푸시할 때 `--force` 대신 `--force-with-lease`를 권장하는 이유를 설명해 보세요.
5. 일반 `git push`만으로는 태그가 원격 저장소에 전송되지 않는 이유와, 태그를 전송하기 위한 올바른 명령을 적어 보세요.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 17장. Git Hooks와 자동화](ch17-GitHooks와자동화.md) | [다음: 부록 ▶](appendix-부록.md)

# 6장. 원격 저장소와 GitHub

📖 [◀ 목차](00-목차.md) | [◀ 이전: 5장. 병합(merge)과 충돌 해결](ch05-병합과충돌해결.md) | [다음: 7장. 커밋 히스토리 탐색 ▶](ch07-커밋히스토리탐색.md)

---

지금까지 다룬 모든 명령은 여러분 컴퓨터 안, 즉 로컬(local) 저장소 안에서만 벌어지는 일이었습니다. 실제 협업에서는 이 이력을 다른 사람과, 또는 다른 컴퓨터와 주고받아야 합니다. 이 장에서는 로컬 저장소가 아닌 다른 곳에 있는 저장소인 **원격 저장소(remote repository)**를 다루는 방법과, 원격 저장소를 무료로 호스팅해 주는 대표적인 서비스인 **GitHub**의 역할을 살펴봅니다.

## 6.1 원격 저장소란

Git은 분산 버전 관리 시스템입니다. 즉, 중앙 서버 없이도 각자의 컴퓨터에 저장소 전체(모든 커밋 이력 포함)를 온전히 복제해 둘 수 있습니다. 원격 저장소란 내 컴퓨터가 아닌 다른 위치 — 회사 서버일 수도 있고, GitHub 같은 호스팅 서비스일 수도 있고, 심지어 같은 컴퓨터의 다른 폴더일 수도 있습니다 — 에 있는 Git 저장소를 가리키는 이름입니다. 여러 사람이 같은 원격 저장소를 공유하면, 각자 자기 컴퓨터에서 커밋을 쌓다가 필요할 때 원격 저장소를 통해 서로의 변경 내용을 주고받을 수 있습니다.

## 6.2 원격 저장소 등록하기

이미 만들어 둔 로컬 저장소에 원격 저장소를 연결하려면 `git remote add`를 사용합니다.

```bash
git remote add origin https://github.com/username/repo.git
```

`origin`은 원격 저장소를 가리키는 이름(별칭)입니다. 반드시 `origin`이어야 하는 것은 아니지만, 관례적으로 가장 먼저 연결한(또는 clone의 출처가 된) 원격 저장소를 `origin`이라고 부릅니다.

등록된 원격 저장소 목록은 `git remote -v`로 확인합니다.

```bash
git remote -v
```

```text
origin  https://github.com/username/repo.git (fetch)
origin  https://github.com/username/repo.git (push)
```

같은 이름이 fetch용과 push용으로 각각 표시되는데, 대부분의 경우 두 URL은 같습니다. 저장소 하나에 원격을 여러 개 등록할 수도 있습니다. 예를 들어 회사 내부 서버와 개인 백업용 서버를 동시에 연결해 둘 수 있습니다.

```bash
git remote add backup https://example.com/backup/repo.git
```

원격 저장소 이름을 바꾸거나 제거해야 할 때는 각각 다음 명령을 씁니다.

```bash
git remote rename backup old-backup
git remote remove old-backup
```

## 6.3 원격 저장소 복제하기: `git clone`

이미 존재하는 원격 저장소를 통째로 내려받아 로컬 저장소로 만들려면 `git clone`을 사용합니다.

```bash
git clone https://github.com/username/repo.git
```

이 한 줄로 다음 일들이 한꺼번에 일어납니다.

- 현재 디렉터리 아래 `repo`라는 폴더가 생성됩니다.
- 원격 저장소의 모든 커밋 이력, 브랜치, 태그가 내려받아집니다.
- 원격 저장소의 기본 브랜치(보통 `main`)가 로컬에 체크아웃됩니다.
- `origin`이라는 이름으로 원격 저장소가 자동 등록됩니다.

즉 `git clone` 이후에는 `git remote add`를 따로 할 필요가 없습니다. 폴더 이름을 직접 지정하고 싶다면 두 번째 인자를 추가합니다.

```bash
git clone https://github.com/username/repo.git my-folder
```

## 6.4 원격으로 올리기: `git push`

로컬에서 만든 커밋을 원격 저장소에 반영하려면 `git push`를 사용합니다.

```bash
git push origin main
```

이 명령은 "origin이라는 원격 저장소의 main 브랜치로, 내 로컬 main 브랜치를 밀어 올려라"는 뜻입니다. 처음으로 어떤 브랜치를 원격에 올릴 때는 `-u`(`--set-upstream`) 옵션을 함께 주는 것이 좋습니다.

```bash
git push -u origin main
```

```text
Enumerating objects: 5, done.
...
To https://github.com/username/repo.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

`-u`는 로컬 `main` 브랜치가 원격 `origin/main` 브랜치를 **추적(tracking)**하도록 설정합니다. 한 번 추적 관계를 맺어 두면, 이후로는 브랜치와 원격 이름을 생략하고 다음처럼 짧게 쓸 수 있습니다.

```bash
git push
git pull
```

새로 만든 브랜치를 처음 원격에 올릴 때도 같은 방식입니다.

```bash
git switch -c feature-x
# ... 작업 및 커밋 ...
git push -u origin feature-x
```

이렇게 하면 원격 저장소에 `feature-x` 브랜치가 새로 생기고, 로컬 `feature-x`가 그것을 추적하게 됩니다.

## 6.5 `git fetch`와 `git pull`의 차이

다른 사람이 원격 저장소에 새 커밋을 올렸다면, 내 로컬 저장소에도 그 변경 내용을 받아와야 합니다. 이때 쓰는 두 명령, `git fetch`와 `git pull`은 자주 헷갈리지만 하는 일이 분명히 다릅니다.

`git fetch`는 원격 저장소의 새 커밋과 브랜치 정보를 **내려받기만** 합니다. 로컬의 현재 브랜치(`main` 등)나 작업 디렉터리 파일은 전혀 건드리지 않습니다.

```bash
git fetch origin
```

```text
remote: Enumerating objects: 4, done.
...
From https://github.com/username/repo.git
   a1b2c3d..e4f5g6h  main       -> origin/main
```

내려받은 내용은 `origin/main`이라는 이름으로 확인할 수 있습니다. 아직 내 `main` 브랜치와 합쳐지지 않았으므로, 무엇이 달라졌는지 먼저 살펴볼 수 있습니다.

```bash
git log origin/main
git diff main origin/main
```

살펴본 뒤 반영하고 싶다면 그때 직접 병합합니다.

```bash
git merge origin/main
```

반면 `git pull`은 **fetch와 merge를 한 번에** 수행합니다.

```bash
git pull origin main
```

이 명령은 내부적으로 `git fetch origin`을 실행한 뒤, 곧바로 `git merge origin/main`까지 실행합니다. 즉 `git pull`은 다음 두 줄을 줄여 쓴 것과 사실상 같습니다.

```bash
git fetch origin
git merge origin/main
```

정리하면, `git fetch`는 "받아오기만 하고 판단은 내가 한다"는 신중한 방식이고, `git pull`은 "받아오는 즉시 바로 합친다"는 빠른 방식입니다. 협업 중 변경 내용을 먼저 검토하고 싶다면 `fetch` 후 `diff`나 `log`로 확인하는 습관이 유용합니다.

> 참고: `git pull --rebase`처럼 merge 대신 rebase 방식으로 합치는 옵션도 있지만, rebase는 이력을 다시 쓰는 별도의 개념이므로 이 책의 뒤쪽 장에서 다룹니다.

## 6.6 인증 방식: HTTPS와 SSH

원격 저장소, 특히 GitHub 같은 서비스는 아무나 push하지 못하도록 인증을 요구합니다. Git이 지원하는 대표적인 원격 URL 방식은 두 가지입니다.

| 방식 | URL 형태 | 인증 방법 |
|---|---|---|
| HTTPS | `https://github.com/username/repo.git` | 사용자 이름 + 비밀번호 대신 개인 액세스 토큰(PAT), 또는 credential helper에 저장된 자격 증명 |
| SSH | `git@github.com:username/repo.git` | 미리 등록해 둔 SSH 키 쌍(공개키/개인키)으로 인증 |

**HTTPS 방식**은 URL이 익숙하고 방화벽 환경에서도 잘 동작한다는 장점이 있습니다. 다만 최근 GitHub를 비롯한 대부분의 호스팅 서비스는 계정 비밀번호를 그대로 쓰는 방식을 막고 있어서, 별도로 발급한 토큰을 비밀번호 자리에 입력해야 합니다. 한 번 인증하면 운영체제의 자격 증명 관리자(credential helper)가 토큰을 기억해 두어 매번 입력할 필요는 없습니다.

**SSH 방식**은 컴퓨터에 미리 만들어 둔 SSH 키 쌍 중 공개키를 GitHub 계정에 등록해 두고, push나 pull을 할 때 개인키로 자동 인증하는 방식입니다. 키 생성에는 `ssh-keygen` 명령을 사용하는데, 구체적인 생성·등록 절차는 운영체제와 서비스마다 조금씩 달라 이 책에서는 다루지 않고 이름만 소개합니다. 한 번 설정해 두면 이후로는 별도의 로그인 없이 원격 저장소를 사용할 수 있다는 점만 기억해 두면 충분합니다.

두 방식 중 무엇을 쓰든 원격 저장소를 다루는 Git 명령(`clone`, `fetch`, `pull`, `push`) 자체는 동일합니다. 차이는 오직 원격 URL의 형태와 최초 인증 절차뿐입니다.

## 6.7 GitHub의 역할

GitHub는 Git 저장소를 인터넷에 호스팅해 주는 대표적인 서비스입니다(비슷한 서비스로 GitLab, Bitbucket 등도 있습니다). GitHub 자체는 Git이 아니라 Git 저장소를 웹에서 편리하게 쓸 수 있게 해 주는 부가 서비스에 가깝습니다. 지금까지 배운 `git remote add`, `git clone`, `git push`, `git pull` 같은 명령은 원격 저장소가 GitHub든 GitLab이든 개인 서버든 상관없이 동일하게 동작합니다.

GitHub가 제공하는 대표적인 기능은 다음과 같습니다.

- 원격 저장소를 웹에서 보고, 커밋 이력과 파일 변경 내용을 시각적으로 확인
- 이슈(issue)로 버그나 할 일을 추적
- **Fork**: 다른 사람의 저장소를 내 계정으로 복사해 자유롭게 실험할 수 있게 해 주는 기능
- **Pull Request(PR)**: 내가 고친 내용을 원본 저장소에 반영해 달라고 제안하고, 코드 리뷰를 거쳐 병합하는 절차

Fork와 Pull Request는 오픈소스 프로젝트에 기여하거나 팀 단위로 코드 리뷰를 진행할 때 핵심이 되는 절차입니다. 이 책에서는 11장에서 GitHub를 이용한 협업 흐름을 다루면서 Fork와 Pull Request를 자세히 설명합니다. 지금은 "원격 저장소를 호스팅해 주는 서비스이며, 그 위에 협업을 돕는 기능들이 얹혀 있다" 정도로 이해하고 넘어가면 충분합니다.

## 6.8 push가 거부될 때

여러 사람이 같은 원격 브랜치를 사용하다 보면, 내가 push하기 전에 다른 사람이 먼저 push해서 원격 브랜치가 나보다 앞서 있는 상황이 자주 생깁니다. 이럴 때 `git push`는 실패합니다.

```bash
git push origin main
```

```text
To https://github.com/username/repo.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'https://github.com/username/repo.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
```

`[rejected]`와 `(fetch first)`가 핵심입니다. 원격에는 내가 아직 갖고 있지 않은 커밋이 있으니, 그걸 먼저 받아오라는 뜻입니다. Git이 힌트에서 알려주는 대로 처리하면 됩니다.

```bash
git pull origin main
```

`git pull`은 앞서 배운 대로 원격의 새 커밋을 받아온 뒤 내 브랜치와 자동으로 병합을 시도합니다. 병합이 충돌 없이 끝나면 곧바로 다시 push할 수 있습니다.

```bash
git push origin main
```

만약 `git pull` 도중 5장에서 다룬 것과 같은 **충돌**이 발생한다면, 그 충돌부터 해결해야 합니다. 파일을 직접 수정하고, `git add`로 해결을 표시하고, `git commit`으로 병합을 마무리한 뒤에 다시 push를 시도합니다.

```bash
git pull origin main
# 충돌 발생 시:
#   1. 충돌 파일 수정
#   2. git add <파일>
#   3. git commit
git push origin main
```

정리하면, push가 거부되는 것은 오류라기보다 "원격에 있는 최신 내용을 먼저 반영하고 오라"는 정상적인 안전장치입니다. 항상 **pull로 최신 내용을 먼저 받아온 뒤 push한다**는 순서를 기억해 두면 됩니다.

## 요약

- 원격 저장소는 내 컴퓨터가 아닌 다른 곳에 있는 Git 저장소이며, `git remote add <이름> <URL>`로 등록하고 `git remote -v`로 확인합니다.
- `git clone <URL>`은 원격 저장소 전체를 복제하면서 `origin`이라는 이름으로 원격 등록까지 자동으로 해 줍니다.
- `git push -u origin <브랜치>`로 브랜치를 처음 올리면서 추적 관계를 설정해 두면, 이후로는 `git push`, `git pull`만으로 충분합니다.
- `git fetch`는 원격의 새 커밋을 내려받기만 하고, `git pull`은 fetch에 이어 자동으로 merge까지 수행합니다.
- 원격 저장소 인증은 HTTPS(토큰 기반)와 SSH(키 쌍 기반) 두 방식이 있으며, 둘 다 이후의 Git 명령 사용법에는 차이를 주지 않습니다.
- GitHub는 Git 저장소를 호스팅하고 이슈, Fork, Pull Request 같은 협업 기능을 제공하는 서비스입니다. Fork와 Pull Request는 11장에서 자세히 다룹니다.
- 원격이 나보다 앞서 있으면 `git push`가 `[rejected]`로 거부되며, 이때는 `git pull`로 최신 내용을 받아 병합(필요하면 충돌 해결)한 뒤 다시 push합니다.

## 연습문제

1. `git remote add`와 `git clone`은 둘 다 원격 저장소를 로컬에 연결하지만 하는 일이 다릅니다. 두 명령의 차이를 설명하시오.
2. `git push -u origin main`에서 `-u` 옵션이 하는 역할을 설명하고, 이 옵션을 사용한 이후 `git push`만 입력해도 되는 이유를 서술하시오.
3. `git fetch`와 `git pull`의 차이를 설명하고, 협업 중 원격의 변경 내용을 먼저 검토하고 싶을 때 어떤 명령을 쓰는 것이 더 적합한지 서술하시오.
4. HTTPS 인증 방식과 SSH 인증 방식의 차이를 간단히 설명하시오.
5. `git push` 실행 시 `! [rejected] (fetch first)` 오류가 발생했다. 이 오류의 원인과, 해결을 위해 실행해야 할 명령들을 순서대로 서술하시오.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 5장. 병합(merge)과 충돌 해결](ch05-병합과충돌해결.md) | [다음: 7장. 커밋 히스토리 탐색 ▶](ch07-커밋히스토리탐색.md)

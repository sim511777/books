# 17장. gem과 Bundler

[◀ 이전: 16장. 메타프로그래밍 기초](ch16-메타프로그래밍기초.md) | [📖 목차](00-목차.md) | [다음: 18장. 좋은 Ruby 코드 작성 습관 ▶](ch18-좋은Ruby코드작성습관.md)


지금까지는 표준 라이브러리(Ruby를 설치하면 기본으로 딸려 오는 기능들)만으로 예제를 작성해왔습니다. 하지만 실제 프로젝트를 진행하다 보면 "이미 누군가 잘 만들어 놓은 기능"을 가져다 쓰고 싶은 순간이 자주 찾아옵니다. 날짜를 다루는 기능, HTTP 요청을 보내는 기능, 웹 애플리케이션 프레임워크 등을 매번 직접 구현하는 것은 비효율적이기 때문입니다. 이 장에서는 Ruby 생태계에서 이런 재사용 가능한 코드 묶음을 어떻게 설치하고 관리하는지, 그리고 나만의 코드 묶음을 어떻게 만들 수 있는지를 살펴봅니다.

## gem이란 무엇인가

**gem**은 Ruby로 작성된 코드를 다른 사람이 쉽게 설치하고 사용할 수 있도록 하나로 묶어 배포하는 패키지 단위입니다. 클래스, 모듈, 메서드의 모음을 파일 하나(또는 몇 개)로 압축해두고, 여기에 이름과 버전, 의존성 같은 메타데이터를 붙인 것이라고 생각하면 됩니다.

이런 gem들을 모아두고 검색하고 내려받을 수 있게 해주는 중앙 저장소가 **RubyGems**(rubygems.org)입니다. Ruby를 설치하면 `gem`이라는 명령줄 도구가 함께 설치되는데, 이 도구를 이용해 RubyGems 저장소에서 원하는 gem을 내려받아 내 컴퓨터에 설치할 수 있습니다.

터미널에서 현재 설치된 gem 목록은 다음 명령으로 확인할 수 있습니다.

```bash
gem list
```

## gem install로 패키지 설치하기

가장 기본적인 gem 설치 명령은 `gem install`입니다. 예를 들어 문자열 색상을 다루는 `colorize`라는 gem을 설치하고 싶다면 다음과 같이 실행합니다.

```bash
gem install colorize
```

이 명령을 실행하면 RubyGems 저장소에서 `colorize`의 최신 버전을 내려받아 설치합니다. 설치가 끝나면 어떤 Ruby 스크립트에서든 `require` 키워드로 그 gem을 불러와 사용할 수 있습니다.

```ruby
require "colorize"

puts "성공했습니다!".colorize(:green)
puts "실패했습니다!".colorize(:red)
```

특정 버전을 지정해서 설치하고 싶다면 `-v` 옵션을 사용합니다.

```bash
gem install colorize -v 0.8.1
```

설치된 gem을 제거할 때는 `gem uninstall`을, 설치된 gem의 정보를 확인할 때는 `gem info`를 사용합니다.

```bash
gem uninstall colorize
gem info colorize
```

`gem install`은 "지금 이 컴퓨터에 전역으로 하나 설치해두기"에는 편리하지만, 프로젝트마다 여러 gem이 필요하고 각 gem의 버전까지 맞춰야 하는 상황에서는 한계가 있습니다. 예를 들어 프로젝트 A는 어떤 gem의 1.0 버전이 필요하고 프로젝트 B는 같은 gem의 2.0 버전이 필요하다면, `gem install`만으로는 두 프로젝트를 동시에 깔끔하게 관리하기 어렵습니다. 이런 문제를 해결해주는 도구가 바로 **Bundler**입니다.

## Gemfile과 Bundler

Bundler는 "이 프로젝트가 어떤 gem들을 필요로 하는지"를 하나의 파일에 명시해두고, 그 파일을 기준으로 필요한 gem들을 정확한 버전으로 설치·관리해주는 도구입니다. 이 명세 파일을 **Gemfile**이라고 부릅니다.

최신 Ruby에는 Bundler가 기본으로 함께 설치되어 있어, 별도의 설치 과정 없이 바로 `bundle` 명령을 사용할 수 있는 경우가 대부분입니다. 혹시 설치되어 있지 않다면 다음 명령으로 설치할 수 있습니다.

```bash
gem install bundler
```

### bundle init: Gemfile 만들기

새 프로젝트 폴더에서 다음 명령을 실행하면 기본 틀을 갖춘 `Gemfile`이 자동으로 생성됩니다.

```bash
bundle init
```

생성된 `Gemfile`을 열어보면 다음과 비슷한 내용이 들어 있습니다.

```ruby
# frozen_string_literal: true

source "https://rubygems.org"

# gem "rails"
```

`source`는 gem을 어디서 내려받을지를 지정합니다. 이제 이 파일에 프로젝트가 필요로 하는 gem을 한 줄씩 추가하면 됩니다. 예를 들어 앞서 사용한 `colorize`와, HTTP 요청을 보낼 때 자주 쓰이는 `httparty`가 필요하다고 해봅시다.

```ruby
# frozen_string_literal: true

source "https://rubygems.org"

gem "colorize"
gem "httparty", "~> 0.21.0"
```

버전 뒤에 붙은 `~>`는 **비관적 버전 제약(pessimistic version constraint)**이라고 부르는 표기법으로, "0.21.0 이상이지만 0.22.0 미만"처럼 특정 범위 안에서만 버전이 올라가도록 제한하는 역할을 합니다. 버전을 명시하지 않으면 항상 최신 버전을 설치하려 시도합니다.

### bundle install: 의존성 설치

`Gemfile`을 작성한 뒤 다음 명령을 실행하면, Bundler가 `Gemfile`에 적힌 모든 gem과 그 gem들이 다시 의존하는 다른 gem들까지 한꺼번에 찾아 설치합니다.

```bash
bundle install
```

### Gemfile.lock: 설치된 버전을 고정하기

`bundle install`이 끝나면 프로젝트 폴더에 `Gemfile.lock`이라는 파일이 함께 생성됩니다. 이 파일에는 실제로 설치된 모든 gem의 **정확한 버전**이 기록됩니다.

`Gemfile`이 "0.21.x 버전대라면 무엇이든 괜찮다"는 식의 느슨한 범위를 나타낸다면, `Gemfile.lock`은 "이번에 정확히 0.21.2 버전이 설치되었다"는 사실을 못박아 둔 기록입니다. 이 파일 덕분에 나중에 같은 프로젝트를 다른 컴퓨터(동료의 컴퓨터나 서버)에서 `bundle install`을 다시 실행해도, 매번 완전히 동일한 버전의 gem들이 설치됩니다. 만약 `Gemfile.lock` 없이 버전 범위만 믿는다면, 시간이 지나 gem의 새 버전이 나왔을 때 사람마다 서로 다른 버전이 설치되어 "내 컴퓨터에서는 되는데 다른 컴퓨터에서는 안 되는" 문제가 생길 수 있습니다.

그래서 실무에서는 `Gemfile.lock`을 (다른 소스 코드처럼) 버전 관리 시스템에 함께 커밋해서, 팀원 전체가 항상 동일한 버전의 gem으로 작업하도록 하는 것이 일반적인 관례입니다.

`Gemfile`에 새 gem을 추가하고 싶을 때는 파일을 직접 수정한 뒤 다시 `bundle install`을 실행하면 됩니다. 이미 설치된 gem들의 버전을 범위 내에서 최신으로 업데이트하고 싶다면 다음 명령을 사용합니다.

```bash
bundle update
```

### bundle exec: 올바른 버전으로 실행하기

한 컴퓨터에 여러 프로젝트가 있고 각 프로젝트가 같은 gem의 서로 다른 버전을 사용한다면, 스크립트를 실행할 때 "이 프로젝트의 `Gemfile.lock`에 적힌 버전"을 확실히 사용하도록 다음처럼 실행하는 것이 안전합니다.

```bash
bundle exec ruby app.rb
```

`bundle exec`는 실행 전에 `Gemfile.lock`을 확인해서, 시스템에 여러 버전이 설치되어 있더라도 그 프로젝트에 고정된 정확한 버전을 사용하도록 보장해줍니다.

## 자주 쓰이는 유명 gem들

Ruby 생태계에는 수만 개의 gem이 존재합니다. 이 중 이름 정도는 알아두면 좋은 대표적인 gem 몇 가지를 간단히 소개합니다. 상세한 사용법은 이 책의 범위를 벗어나므로, "이런 이름의 gem이 있고 대략 이런 용도로 쓰인다" 정도로만 기억해두어도 충분합니다.

- **rails**: Ruby로 작성된 대표적인 웹 애플리케이션 프레임워크입니다. 데이터베이스 처리, 라우팅, 화면 렌더링 등 웹 서비스에 필요한 기능 대부분을 한 번에 제공합니다. 1장에서 언급했듯 Ruby를 세계적으로 유명하게 만든 주역입니다.
- **rspec**: Ruby 진영에서 가장 널리 쓰이는 테스트 프레임워크 중 하나입니다. 사람이 읽는 문장에 가까운 형태로 테스트 코드를 작성할 수 있게 해줍니다. 이 gem은 18장에서 조금 더 살펴봅니다.
- **rubocop**: 코드 스타일을 점검해주는 정적 분석 도구입니다. 들여쓰기, 따옴표 사용, 줄 길이 같은 관례를 자동으로 검사하고 지적해줍니다. 18장에서 함께 다룹니다.
- **puma**: Ruby 웹 애플리케이션을 실제로 구동시켜주는 웹 서버 gem으로, Rails 프로젝트에서 기본 서버로 널리 쓰입니다.
- **nokogiri**: HTML이나 XML 문서를 읽고 원하는 정보를 뽑아내는(파싱하는) 데 특화된 gem으로, 웹 스크래핑 작업에 자주 사용됩니다.
- **sidekiq**: 시간이 오래 걸리는 작업을 백그라운드에서 비동기로 처리해주는 gem입니다.
- **pry**: irb를 대체할 수 있는 더 강력한 대화형 콘솔로, 디버깅 시 특히 유용합니다.

## 나만의 gem 만들기: 구조 개요

지금까지는 남이 만든 gem을 가져다 쓰는 방법을 살펴봤습니다. 이번에는 반대로, 여러 프로젝트에서 재사용하고 싶은 코드를 직접 gem으로 만드는 흐름을 개괄적으로 살펴봅니다. 실제로 RubyGems에 공개 배포까지 하는 전 과정은 이 책의 범위를 넘어서므로, 여기서는 "gem 하나가 어떤 구조로 이루어져 있는가"에 초점을 맞춥니다.

Bundler는 gem의 기본 뼈대를 자동으로 만들어주는 명령도 제공합니다.

```bash
bundle gem greeting_kit
```

이 명령을 실행하면 다음과 비슷한 디렉터리 구조가 생성됩니다.

```
greeting_kit/
├── greeting_kit.gemspec
├── Gemfile
├── lib/
│   ├── greeting_kit.rb
│   └── greeting_kit/
│       └── version.rb
├── sig/
├── spec/
└── README.md
```

### lib 디렉터리: 실제 코드가 담기는 곳

`lib/` 디렉터리에는 gem이 제공하는 실제 기능(클래스, 모듈, 메서드)을 작성합니다. 관례적으로 gem 이름과 같은 이름의 파일이 진입점 역할을 합니다.

```ruby
# lib/greeting_kit.rb
require_relative "greeting_kit/version"

module GreetingKit
  def self.greet(name)
    "안녕하세요, #{name}님! GreetingKit이 인사드립니다."
  end
end
```

```ruby
# lib/greeting_kit/version.rb
module GreetingKit
  VERSION = "0.1.0"
end
```

이렇게 만든 gem을 다른 프로젝트에서 사용할 때는 다른 gem과 마찬가지로 `require "greeting_kit"` 후 `GreetingKit.greet("지수")`처럼 호출하게 됩니다.

### .gemspec: gem의 명세서

`.gemspec` 파일은 이 gem이 누구를 위한 것인지, 이름과 버전은 무엇인지, 어떤 다른 gem에 의존하는지를 정의하는 명세서입니다.

```ruby
# greeting_kit.gemspec
require_relative "lib/greeting_kit/version"

Gem::Specification.new do |spec|
  spec.name        = "greeting_kit"
  spec.version     = GreetingKit::VERSION
  spec.authors     = ["작성자 이름"]
  spec.summary     = "다양한 언어로 인사말을 생성해주는 간단한 gem"
  spec.files       = Dir["lib/**/*.rb"]
  spec.required_ruby_version = ">= 3.0.0"

  spec.add_dependency "colorize", "~> 1.0"
  spec.add_development_dependency "rspec", "~> 3.0"
end
```

`add_dependency`는 이 gem을 사용하는 사람이 실행 시점에 함께 설치해야 하는 다른 gem을 지정합니다. `add_development_dependency`는 이 gem 자체를 개발하고 테스트할 때만 필요한 gem(예를 들면 rspec)을 지정합니다. 즉 `.gemspec`은 하나의 gem이 다른 gem에 대해 갖는 의존 관계를, `Gemfile`은 하나의 애플리케이션이 여러 gem에 대해 갖는 의존 관계를 기술한다고 정리할 수 있습니다.

이렇게 구조를 갖춘 뒤 `gem build greeting_kit.gemspec` 명령으로 실제 `.gem` 패키지 파일을 만들 수 있고, 이를 `gem push`로 RubyGems에 공개할 수도 있습니다. 반드시 공개 배포를 하지 않더라도, 여러 사내 프로젝트에서 공통으로 쓰는 코드를 이런 구조의 gem으로 뽑아 관리하면 코드 재사용과 버전 관리가 훨씬 수월해집니다.

## 요약

- gem은 Ruby 코드를 재사용 가능한 형태로 묶어 배포하는 패키지 단위이며, RubyGems는 이런 gem들을 모아둔 중앙 저장소입니다.
- `gem install`로 gem을 시스템에 직접 설치할 수 있지만, 여러 프로젝트의 버전 충돌을 관리하기에는 한계가 있습니다.
- Bundler는 `Gemfile`에 프로젝트가 필요로 하는 gem과 버전 범위를 명시하고, `bundle install`로 이를 일괄 설치·관리하는 도구입니다.
- `Gemfile.lock`은 실제로 설치된 gem의 정확한 버전을 기록해, 여러 컴퓨터에서 항상 동일한 환경을 재현할 수 있게 해줍니다.
- rails, rspec, rubocop, puma, nokogiri, sidekiq, pry 등은 Ruby 생태계에서 자주 쓰이는 대표적인 gem입니다.
- 나만의 gem은 `lib/` 디렉터리에 실제 코드를, `.gemspec` 파일에 이름·버전·의존성 같은 메타데이터를 담는 구조로 만들 수 있습니다.

## 연습문제

1. `gem install`과 `bundle install`의 차이를 자신의 언어로 한 문단으로 설명해보세요.
2. `Gemfile`과 `Gemfile.lock`이 각각 어떤 역할을 하는지, 그리고 왜 `Gemfile.lock`을 버전 관리 시스템에 함께 커밋하는 것이 권장되는지 설명해보세요.
3. 자신의 컴퓨터에서 새 폴더를 만들고 `bundle init`을 실행해 `Gemfile`을 생성한 뒤, `colorize` gem을 추가하고 `bundle install`까지 실행해보세요.
4. 본문에서 소개한 gem(rails, rspec, rubocop, puma, nokogiri, sidekiq, pry) 중 두 가지를 골라, 각각 어떤 상황에서 필요할지 자신의 말로 설명해보세요.
5. `.gemspec` 파일의 `add_dependency`와 `add_development_dependency`의 차이를 예를 들어 설명해보세요.

---

[◀ 이전: 16장. 메타프로그래밍 기초](ch16-메타프로그래밍기초.md) | [📖 목차](00-목차.md) | [다음: 18장. 좋은 Ruby 코드 작성 습관 ▶](ch18-좋은Ruby코드작성습관.md)

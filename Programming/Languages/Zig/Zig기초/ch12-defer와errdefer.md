# 12장. defer와 errdefer

[◀ 이전: 11장. 에러 처리](ch11-에러처리.md) | [📖 목차](00-목차.md) | [다음: 13장. 슬라이스 ▶](ch13-슬라이스.md)


11장에서는 함수가 실패할 수 있다는 사실을 `!T`라는 타입으로 명시적으로 드러내는 방법을 배웠습니다. 그런데 실패가 일어날 수 있는 함수일수록 한 가지 골치 아픈 문제가 따라옵니다. 함수 중간에서 이미 확보해 둔 자원(열린 파일, 할당된 메모리 등)을 어떻게 확실히 정리할 것인가 하는 문제입니다. 정상적으로 끝나든, 중간에 에러로 빠져나가든, 확보한 자원은 반드시 정리되어야 합니다.

Zig는 이 문제를 해결하기 위해 `defer`와 `errdefer`라는 두 개의 문법을 제공합니다. 둘 다 "지금 실행하지 말고, 이 스코프를 벗어날 때 실행하라"고 예약하는 문법이지만 조건이 다릅니다. `defer`는 스코프를 벗어나는 모든 경우(정상 종료든 에러든)에 실행되고, `errdefer`는 에러로 스코프를 벗어날 때만 실행됩니다. 이 장에서는 두 문법의 정확한 동작 방식과, 자원 관리에서 이들을 어떻게 관용적으로 사용하는지 살펴봅니다.

## defer: 스코프를 벗어날 때 실행되는 코드

`defer` 뒤에 오는 문장(또는 블록)은 현재 스코프가 어떤 방식으로든 끝나는 순간에 실행되도록 예약됩니다. "어떤 방식으로든"이란 함수가 정상적으로 `return`하는 경우, 에러를 `return`하는 경우, 블록의 끝에 도달하는 경우를 모두 포함합니다.

```zig
const std = @import("std");

fn greet() void {
    std.debug.print("함수 시작\n", .{});
    defer std.debug.print("함수 종료 직전\n", .{});
    std.debug.print("함수 본문 실행 중\n", .{});
}
```

이 함수를 호출하면 다음 순서로 출력됩니다.

```
함수 시작
함수 본문 실행 중
함수 종료 직전
```

`defer`가 적힌 위치와 무관하게, 그 뒤에 예약된 코드는 함수를 실제로 빠져나가는 시점에 실행됩니다. 이 특성 덕분에 "자원을 할당한 바로 다음 줄에 해제 코드를 적어 둘 수 있다"는 강력한 관용구가 성립합니다.

### 자원 해제 관용구: 할당과 해제를 나란히 두기

C 언어처럼 수동으로 자원을 관리하는 언어에서 자주 발생하는 버그는, 함수 중간에 `return`을 추가하면서 그 위에 있던 해제 코드를 실행하지 못하고 지나쳐 버리는 것입니다. 자원을 여는 코드와 닫는 코드가 함수의 서로 다른 위치에 떨어져 있으면, 둘 사이에 새로운 return 경로가 추가될 때마다 해제 코드를 빠뜨릴 위험이 커집니다.

Zig에서는 자원을 확보하는 줄 바로 다음 줄에 `defer`로 해제 코드를 적는 것이 관용구입니다.

```zig
fn read_config(allocator: std.mem.Allocator, path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    const stat = try file.stat();
    const contents = try allocator.alloc(u8, stat.size);
    errdefer allocator.free(contents);

    _ = try file.readAll(contents);
    return contents;
}
```

`file`을 여는 줄 바로 다음 줄에 `defer file.close();`를 적어 두었기 때문에, 이 함수 안에서 이후에 어떤 이유로 함수를 빠져나가든(정상 반환이든, `try`로 인한 에러 전파든) `file.close()`가 반드시 호출됩니다. 코드를 읽는 사람은 "이 파일은 이 함수를 벗어나는 순간 자동으로 닫힌다"는 사실을 파일을 여는 줄만 보고도 확신할 수 있습니다. 할당과 해제 코드가 시각적으로 붙어 있으므로, 나중에 이 함수에 새로운 `try`나 `return`이 추가되어도 해제 코드가 누락될 걱정이 없습니다.

### 여러 defer의 실행 순서: LIFO

한 스코프 안에 여러 개의 `defer`가 있으면, 이들은 선언된 순서의 반대 순서로 실행됩니다. 즉 나중에 선언된 `defer`가 먼저 실행되는 후입선출(LIFO, Last In First Out) 순서입니다.

```zig
fn demo() void {
    defer std.debug.print("첫 번째 defer (가장 먼저 선언)\n", .{});
    defer std.debug.print("두 번째 defer\n", .{});
    defer std.debug.print("세 번째 defer (가장 나중에 선언)\n", .{});
    std.debug.print("함수 본문\n", .{});
}
```

출력 순서는 다음과 같습니다.

```
함수 본문
세 번째 defer (가장 나중에 선언)
두 번째 defer
첫 번째 defer (가장 먼저 선언)
```

이 LIFO 순서는 우연이 아니라 자원 관리의 자연스러운 요구를 그대로 반영한 것입니다. 자원 A를 확보한 뒤 자원 A에 의존하는 자원 B를 확보했다면, 해제할 때는 반드시 B를 먼저 해제하고 그다음 A를 해제해야 합니다. `defer`를 확보한 순서대로 나란히 적어 두기만 하면, 해제는 자동으로 올바른 역순으로 이루어집니다.

```zig
fn open_two_files(dir: std.fs.Dir) !void {
    const a = try dir.openFile("a.txt", .{});
    defer a.close();

    const b = try dir.openFile("b.txt", .{});
    defer b.close();

    // ... a와 b를 사용하는 코드 ...
    // 함수를 빠져나갈 때 b가 먼저 닫히고, 그다음 a가 닫힙니다.
}
```

## errdefer: 에러로 빠져나갈 때만 실행되는 코드

`defer`는 정상 종료든 에러든 항상 실행됩니다. 하지만 종종 "함수가 정상적으로 끝날 때는 자원을 그대로 유지하고, 에러 때문에 중간에 실패해서 빠져나갈 때만 지금까지 확보한 것을 되돌리고 싶다"는 상황이 생깁니다. 이럴 때 쓰는 것이 `errdefer`입니다.

`errdefer` 뒤에 예약된 코드는 함수가 에러 값을 반환하며 종료될 때만 실행되고, 함수가 정상적인(에러가 아닌) 값을 반환하며 종료될 때는 실행되지 않습니다.

### 예제: 구조체 초기화 중 일부 필드만 실패했을 때

두 개의 버퍼를 필드로 가지는 구조체를 초기화하는 함수를 생각해 봅시다. 첫 번째 버퍼 할당에는 성공했지만 두 번째 버퍼 할당에서 실패한다면, 이미 할당된 첫 번째 버퍼는 반드시 해제해 주어야 메모리 누수가 생기지 않습니다.

```zig
const Buffers = struct {
    first: []u8,
    second: []u8,
};

fn init_buffers(allocator: std.mem.Allocator, size: usize) !Buffers {
    const first = try allocator.alloc(u8, size);
    errdefer allocator.free(first);

    const second = try allocator.alloc(u8, size);
    errdefer allocator.free(second);

    return Buffers{ .first = first, .second = second };
}
```

이 함수가 어떻게 동작하는지 경우를 나누어 살펴보겠습니다.

- `first` 할당이 실패하면, 아직 `errdefer allocator.free(first);`가 등록되기 전이므로 아무것도 해제할 필요가 없습니다. 함수는 그냥 에러를 반환합니다.
- `first` 할당은 성공했지만 `second` 할당이 실패하면, `first`에 대한 `errdefer`만 등록된 상태에서 함수를 에러로 빠져나갑니다. 이때 `allocator.free(first)`가 실행되어 이미 확보했던 `first` 메모리가 해제됩니다. `second`에 대한 `errdefer`는 아직 등록되지 않았으므로 실행되지 않습니다(할당에 실패했으니 해제할 것도 없습니다).
- 두 할당이 모두 성공하면 함수는 `Buffers` 값을 정상적으로 `return`합니다. 이 경우 등록되어 있던 두 개의 `errdefer`는 모두 실행되지 않습니다. 정상적으로 반환된 `first`와 `second`는 호출한 쪽이 소유권을 가지고 계속 사용해야 하므로, 여기서 해제되어서는 안 됩니다.

이 패턴이 바로 `errdefer`가 존재하는 이유입니다. `defer`만 사용했다면 함수가 정상적으로 끝날 때도 `allocator.free`가 실행되어, 막 반환한 값을 즉시 해제해 버리는 심각한 버그가 됩니다. `errdefer`는 "에러일 때만" 되돌리므로, 정상 경로에서는 소유권이 안전하게 호출자에게 넘어갑니다.

### defer와 errdefer를 함께 사용하기

실제 코드에서는 두 문법을 함께 쓰는 경우가 많습니다. 앞서 본 `read_config` 예제를 다시 보면, 파일 핸들은 함수의 성공/실패와 무관하게 항상 닫아야 하므로 `defer`를 쓰고, 할당한 버퍼는 함수가 실패했을 때만 해제해야 하므로(성공하면 호출자에게 소유권이 넘어가므로) `errdefer`를 씁니다.

```zig
fn read_config(allocator: std.mem.Allocator, path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close(); // 성공/실패 무관하게 항상 닫힘

    const stat = try file.stat();
    const contents = try allocator.alloc(u8, stat.size);
    errdefer allocator.free(contents); // 에러로 실패할 때만 해제

    _ = try file.readAll(contents);
    return contents; // 정상 반환: contents의 소유권이 호출자에게 넘어감
}
```

`file`은 이 함수 안에서만 쓰이는 자원이므로 함수를 나갈 때 항상 닫혀야 하지만(`defer`), `contents`는 성공 시 함수의 반환값 자체가 되어 호출자의 소유가 되므로 성공했을 때는 해제되면 안 됩니다(`errdefer`). 두 문법의 차이가 정확히 "소유권이 누구에게 있는가"라는 질문에 대한 답과 맞닿아 있음을 알 수 있습니다.

### errdefer도 여러 개면 LIFO

`errdefer` 역시 `defer`와 마찬가지로 같은 스코프 안에서 여러 개가 등록되면 나중에 등록된 것이 먼저 실행되는 LIFO 순서를 따릅니다. 또한 `defer`와 `errdefer`가 같은 스코프에 섞여 있을 때도, 실행 시점이 되면 등록된 순서의 역순으로 실행됩니다(단, `errdefer`는 에러로 빠져나갈 때만 그 대열에 포함됩니다).

```zig
fn complex_init(allocator: std.mem.Allocator) !void {
    const a = try allocator.alloc(u8, 10);
    errdefer allocator.free(a);
    defer std.debug.print("complex_init 종료\n", .{});

    const b = try allocator.alloc(u8, 20);
    errdefer allocator.free(b);

    return error.SomeFailure;
    // 실행 순서: allocator.free(b) → "complex_init 종료" 출력 → allocator.free(a)
}
```

이 예제에서 `return error.SomeFailure`에 도달하면, 그 시점까지 등록된 예약 코드들이 등록된 순서의 반대로 실행됩니다. `b`에 대한 `errdefer`, `defer` 출력문, `a`에 대한 `errdefer` 순서로 실행되어 자원이 안전하게 정리됩니다.

## 요약

- `defer 문장;`은 현재 스코프를 벗어나는 순간(정상 종료, 에러 종료 모두 포함) 그 문장을 실행하도록 예약합니다.
- 자원을 확보하는 줄 바로 다음 줄에 해제용 `defer`를 적어 두는 것이 관용구입니다. 예: `const file = try ...openFile(...); defer file.close();`. 할당과 해제를 시각적으로 나란히 두면 해제 누락을 방지할 수 있습니다.
- `errdefer 문장;`은 함수가 에러를 반환하며 스코프를 벗어날 때만 그 문장을 실행합니다. 정상적으로(에러가 아닌 값으로) 종료될 때는 실행되지 않습니다.
- 함수 도중 자원을 여러 개 확보하다가 뒤쪽에서 실패하면, 앞서 성공한 자원들은 `errdefer`로 등록해 두었던 해제 코드에 의해 안전하게 되돌려집니다. 반면 함수가 최종적으로 성공하면, 자원의 소유권은 호출자에게 넘어가므로 `errdefer`는 실행되지 않아야 합니다.
- 같은 스코프에 여러 개의 `defer`/`errdefer`가 있으면 나중에 선언된 것이 먼저 실행되는 LIFO 순서를 따릅니다. 이는 나중에 확보한 자원이 앞서 확보한 자원에 의존할 수 있다는 점을 감안한 자연스러운 순서입니다.

## 연습문제

1. 두 개의 파일을 순서대로 여는 함수를 작성하고, 각 파일을 열자마자 바로 다음 줄에 `defer`로 닫는 코드를 적어 보세요. 두 파일이 어떤 순서로 닫히는지 설명해 보세요.
2. `defer`와 `errdefer`를 하나씩 등록한 함수를 만들고, 함수가 (a) 정상적으로 끝나는 경우와 (b) 에러를 반환하며 끝나는 경우 각각 어떤 예약 코드가 실행되는지 표로 정리해 보세요.
3. 세 개의 메모리 블록을 순서대로 할당하면서 각 할당 직후 `errdefer allocator.free(...)`를 등록하는 함수를 작성하세요. 세 번째 할당에서 실패했을 때 어떤 순서로 해제가 일어나는지 설명해 보세요.
4. 다음 코드에서 출력 순서를 예측해 보고, 실제로 실행해서 확인해 보세요.
   ```zig
   fn puzzle() void {
       defer std.debug.print("A\n", .{});
       {
           defer std.debug.print("B\n", .{});
           std.debug.print("C\n", .{});
       }
       std.debug.print("D\n", .{});
   }
   ```
5. `errdefer` 대신 실수로 `defer`를 사용해서 성공 경로에서도 자원을 해제해 버리는 버그를 하나 만들어 보고, 이 버그가 실제로 어떤 문제(예: 반환된 슬라이스를 사용할 때 해제된 메모리를 참조)를 일으키는지 설명해 보세요.

---

[◀ 이전: 11장. 에러 처리](ch11-에러처리.md) | [📖 목차](00-목차.md) | [다음: 13장. 슬라이스 ▶](ch13-슬라이스.md)

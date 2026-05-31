# Server Sent Events

[Server-Sent Events](https://html.spec.whatwg.org/multipage/comms.html#the-eventsource-interface) 명세는 서버와 커넥션을 유지하고 서버에서 보내는 이벤트를 받을 수 있게 해주는 내장 클래스 `EventSource`를 설명합니다.

`WebSocket`과 마찬가지로 커넥션은 지속적으로 유지됩니다.

하지만 몇 가지 중요한 차이점이 있습니다.

| `WebSocket` | `EventSource` |
|-------------|---------------|
| 양방향: 클라이언트와 서버가 모두 메시지를 주고받을 수 있음 | 단방향: 서버만 데이터를 전송함 |
| 바이너리 데이터와 텍스트 데이터 | 텍스트만 |
| WebSocket 프로토콜 | 일반 HTTP |

`EventSource`는 `WebSocket`보다 서버와 통신하는 기능이 제한적입니다.

그렇다면 왜 `EventSource`를 사용할까요?

주된 이유는 더 단순하기 때문입니다. 많은 애플리케이션에서는 `WebSocket`의 강력한 기능이 오히려 과할 수 있습니다.

채팅 메시지나 시세 등 서버에서 보내는 데이터 스트림을 받아야 하는 경우가 있습니다. 바로 이런 작업에 `EventSource`가 잘 맞습니다. 또한 `WebSocket`에서는 직접 구현해야 하는 자동 재연결을 지원합니다. 게다가 새로운 프로토콜이 아니라 오래전부터 사용하던 평범한 HTTP입니다.

## 메시지 받기

메시지 수신을 시작하려면 `new EventSource(url)`을 생성하기만 하면 됩니다.

브라우저는 `url`에 연결하고 이벤트를 기다리며 커넥션을 열린 상태로 유지합니다.

서버는 상태 코드 200과 `Content-Type: text/event-stream` 헤더로 응답해야 합니다. 그런 다음 커넥션을 유지하면서 다음과 같은 특별한 형식으로 메시지를 써 내려갑니다.

```
data: 메시지 1

data: 메시지 2

data: 메시지 3
data: 두 줄짜리 메시지
```

- 메시지 텍스트는 `data:` 뒤에 옵니다. 콜론 뒤의 공백은 선택 사항입니다.
- 메시지는 줄바꿈 두 번 `\n\n`으로 구분됩니다.
- 줄 바꿈 `\n`을 전송하려면 바로 이어서 `data:`를 하나 더 보내면 됩니다. 위 예시의 세 번째 메시지가 이에 해당합니다.

실제로는 복잡한 메시지를 보통 JSON으로 인코딩해 전송합니다. 줄 바꿈은 JSON 안에서 `\n`으로 인코딩되므로 여러 줄로 된 `data:` 메시지는 필요하지 않습니다.

예시:

```js
data: {"user":"John","message":"첫 번째 줄*!*\n*/!* 두 번째 줄"}
```

따라서 `data:` 하나가 정확히 하나의 메시지를 담는다고 볼 수 있습니다.

이런 메시지마다 `message` 이벤트가 생성됩니다.

```js
let eventSource = new EventSource("/events/subscribe");

eventSource.onmessage = function(event) {
  console.log("새 메시지", event.data);
  // 위 데이터 스트림에 대해 3번 로그를 출력합니다
};

// 또는 eventSource.addEventListener('message', ...)
```

### 크로스 오리진 요청

`EventSource`는 `fetch` 등과 마찬가지로 어떤 URL로든 크로스 오리진 요청을 보낼 수 있습니다.

```js
let source = new EventSource("https://another-site.com/events");
```

원격 서버는 `Origin` 헤더를 받고, 요청을 계속 진행하려면 `Access-Control-Allow-Origin`으로 응답해야 합니다.

자격 증명을 함께 전달하려면 다음과 같이 `withCredentials` 옵션을 추가로 설정해야 합니다.

```js
let source = new EventSource("https://another-site.com/events", {
  withCredentials: true
});
```

크로스 오리진 헤더에 대한 자세한 내용은 <info:fetch-crossorigin> 챕터를 참고하시기 바랍니다.


## 재연결

`new EventSource`가 생성되면 서버에 연결하고, 커넥션이 끊어지면 다시 연결합니다.

이 과정을 신경 쓰지 않아도 되므로 아주 편리합니다.

재연결 사이에는 기본적으로 몇 초 정도의 짧은 지연 시간이 있습니다.

서버는 응답에서 `retry:`를 사용해 권장 지연 시간을 밀리초 단위로 설정할 수 있습니다.

```js
retry: 15000
data: 안녕하세요, 재연결 지연 시간을 15초로 설정했습니다
```

`retry:`는 데이터와 함께 올 수도 있고, 단독 메시지로 올 수도 있습니다.

브라우저는 재연결하기 전에 지정된 밀리초만큼 대기해야 합니다. 만약 운영체제를 통해 현재 네트워크 연결이 끊긴 것을 감지하면, 연결이 다시 복구될 때까지 기다렸다가 재시도하는 등 대기 시간이 더 길어질 수 있습니다.

- 서버가 브라우저의 재연결을 중단시키고 싶다면 HTTP 상태 코드 204로 응답해야 합니다.
- 브라우저가 커넥션을 닫고 싶다면 `eventSource.close()`를 호출해야 합니다.

```js
let eventSource = new EventSource(...);

eventSource.close();
```

또한 응답의 `Content-Type`이 올바르지 않거나 HTTP 상태 코드가 301, 307, 200, 204 중 어느 것에도 해당하지 않는 경우에도 재연결하지 않습니다. 이런 경우 `"error"` 이벤트가 발생하고 브라우저는 재연결하지 않습니다.

```smart
커넥션이 완전히 닫히면 이를 "다시 열" 방법은 없습니다. 다시 연결하고 싶다면 새 `EventSource`를 생성하면 됩니다.
```

## 메시지 id

네트워크 문제로 커넥션이 끊어지면 어느 쪽도 어떤 메시지를 받았고 어떤 메시지를 받지 못했는지 확신할 수 없습니다.

커넥션을 올바르게 재개하려면 각 메시지에 다음과 같이 `id` 필드가 있어야 합니다.

```
data: 메시지 1
id: 1

data: 메시지 2
id: 2

data: 메시지 3
data: 두 줄짜리 메시지
id: 3
```

`id:`가 있는 메시지를 받으면 브라우저는 다음 작업을 수행합니다.

- 프로퍼티 `eventSource.lastEventId`를 해당 값으로 설정합니다.
- 재연결할 때 헤더 `Last-Event-ID`에 해당 `id`를 담아 전송합니다. 그러면 서버는 그 뒤의 메시지를 다시 보낼 수 있습니다.

```smart header="`id:`를 `data:` 뒤에 두세요"
주의: 서버는 메시지 `data` 아래에 `id`를 붙입니다. 메시지를 받은 뒤 `lastEventId`가 갱신되도록 보장하기 위해서입니다.
```

## 커넥션 상태: readyState

`EventSource` 객체에는 `readyState` 프로퍼티가 있으며, 세 값 중 하나를 가집니다.

```js no-beautify
EventSource.CONNECTING = 0; // 연결 중이거나 재연결 중
EventSource.OPEN = 1;       // 연결됨
EventSource.CLOSED = 2;     // 커넥션 닫힘
```

객체가 생성되었거나 커넥션이 끊어진 상태라면 항상 `EventSource.CONNECTING`(`0`)입니다.

이 프로퍼티를 조회하면 `EventSource`의 상태를 알 수 있습니다.

## 이벤트 종류

기본적으로 `EventSource` 객체는 세 가지 이벤트를 생성합니다.

- `message` -- 메시지를 받았을 때 발생하며, 메시지는 `event.data`로 사용할 수 있습니다.
- `open` -- 커넥션이 열렸을 때 발생합니다.
- `error` -- 서버가 HTTP 500 상태 코드를 반환하는 경우처럼 커넥션을 만들 수 없을 때 발생합니다.

서버는 이벤트 시작 부분에 `event: ...`를 지정해 다른 종류의 이벤트를 명시할 수 있습니다.

예시:

```
event: join
data: Bob

data: 안녕하세요

event: leave
data: Bob
```

커스텀 이벤트를 처리하려면 `onmessage`가 아니라 `addEventListener`를 사용해야 합니다.

```js
eventSource.addEventListener('join', event => {
  alert(`입장: ${event.data}`);
});

eventSource.addEventListener('message', event => {
  alert(`메시지: ${event.data}`);
});

eventSource.addEventListener('leave', event => {
  alert(`퇴장: ${event.data}`);
});
```

## 전체 예시

다음은 `1`, `2`, `3` 메시지를 보낸 뒤 `bye`를 보내고 커넥션을 끊는 서버입니다.

그러면 브라우저는 자동으로 재연결합니다.

[codetabs src="eventsource"]

## 요약

`EventSource` 객체는 지속적인 커넥션을 자동으로 만들고, 서버가 그 커넥션을 통해 메시지를 보낼 수 있게 합니다.

`EventSource`는 다음 기능을 제공합니다.
- 조정 가능한 `retry` 타임아웃을 사용하는 자동 재연결
- 이벤트를 재개하기 위한 메시지 id. 마지막으로 받은 식별자는 재연결 시 `Last-Event-ID` 헤더로 전송됩니다.
- 현재 상태를 담는 `readyState` 프로퍼티

이런 기능 덕분에 `EventSource`는 `WebSocket`의 유용한 대안이 될 수 있습니다. `WebSocket`은 더 저수준이고, 이런 내장 기능이 없습니다. 물론 직접 구현할 수는 있습니다.

실무의 많은 애플리케이션에서는 `EventSource`의 기능만으로도 충분합니다.

IE를 제외한 모든 모던 브라우저에서 지원합니다.

문법은 다음과 같습니다.

```js
let source = new EventSource(url, [credentials]);
```

두 번째 인수에는 `{ withCredentials: true }` 한 가지 옵션만 사용할 수 있습니다. 이 옵션을 사용하면 크로스 오리진 자격 증명을 전송할 수 있습니다.

전반적인 크로스 오리진 보안 방식은 `fetch` 및 다른 네트워크 메서드와 동일합니다.

### `EventSource` 객체의 프로퍼티

`readyState`
: 현재 커넥션 상태입니다. 값은 `EventSource.CONNECTING (=0)`, `EventSource.OPEN (=1)`, `EventSource.CLOSED (=2)` 중 하나입니다.

`lastEventId`
: 마지막으로 받은 `id`입니다. 재연결 시 브라우저는 이 값을 `Last-Event-ID` 헤더로 전송합니다.

### 메서드

`close()`
: 커넥션을 닫습니다.

### 이벤트

`message`
: 메시지를 받았을 때 발생하며, 데이터는 `event.data`에 있습니다.

`open`
: 커넥션이 만들어졌을 때 발생합니다.

`error`
: 커넥션 유실처럼 자동 재연결이 가능한 에러와 치명적인 에러를 포함해 에러가 발생했을 때 발생합니다. `readyState`를 확인하면 재연결이 시도되고 있는지 알 수 있습니다.

서버는 `event:`에 커스텀 이벤트 이름을 설정할 수 있습니다. 이런 이벤트는 `on<event>`가 아니라 `addEventListener`를 사용해 처리해야 합니다.

### 서버 응답 형식

서버는 `\n\n`으로 구분된 메시지를 전송합니다.

메시지에는 다음 필드가 들어갈 수 있습니다.

- `data:` -- 메시지 본문입니다. 여러 `data`가 연속되면 부분 사이에 `\n`이 들어간 하나의 메시지로 해석됩니다.
- `id:` -- `lastEventId`를 갱신합니다. 재연결 시 `Last-Event-ID`로 전송됩니다.
- `retry:` -- 재연결을 위한 재시도 지연 시간을 밀리초 단위로 권장합니다. 자바스크립트에서는 이 값을 설정할 방법이 없습니다.
- `event:` -- 이벤트 이름입니다. 반드시 `data:`보다 앞에 와야 합니다.

메시지는 하나 이상의 필드를 어떤 순서로든 포함할 수 있지만, 보통 `id:`는 마지막에 둡니다.

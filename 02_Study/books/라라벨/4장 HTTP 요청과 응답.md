### 1. 요청 핸들링

### 1-1) 요청 취득

```jsx
$response = tap($kernel->handle(
    $request = Request::capture()
))->send();
```

- 라라벨에서 HTTP 요청은 **`Illuminate\\Http\\Request`** 인스턴스로 다룬다.
- 엔트리 포인트(`public/index.php`)에서 `Request::capture()`로 요청을 캡처하고, HTTP 커널이 이를 받아 애플리케이션 흐름을 진행한다.
    - 예: `$kernel->handle($request = Request::capture())`

요청 객체가 담는 대표 정보:

- `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_SERVER`

### 1-2) Request 퍼사드로 요청 값 읽기

```jsx
use Illuminate\\Support\\Facades\\Request;

// "name" 키를 통해 요청으로부터 값을 얻는다
$name = Request::get('name');
// "name" 키가 없으면 guest를 반환한다
$name = Request::get('name', 'guest');
```

- `Illuminate\\Support\\Facades\\Request`를 통해 **어디서든 간단히 요청 값 접근**이 가능하다.
- 내부적으로는 서비스 컨테이너에서 `'request'`를 리졸브해서 실제 Request 객체를 가져오는 구조(퍼사드).

자주 쓰는 예:

- `Request::get('name')`, 기본값 제공: `Request::get('name', 'guest')`
- 모든 입력: `Request::all()`
- 일부만: `Request::only(['name','age'])`
- 업로드 파일: `Request::file('material')`
- 쿠키/헤더/서버정보: `Request::cookie()`, `Request::header()`, `Request::server()`

### 1-3) Request 객체를 주입해서 사용하기

```jsx
<?php

declare(strict_types=1);

namespace App\\Http\\Controllers;

use Illuminate\\Http\\Request; // Request 클래스 임포트

class UserController extends Controller
{
    // 인수로 Requst 클래스의 인스턴스를 전달함
    public function register(Request $request)
    {
        // 인스턴스에 대해 값을 문의함
        $name = $request->get('name');
        $age = $request->get('age');
...
    }
}
```

- 컨트롤러(또는 액션)의 **메서드 인젝션/컨스트럭터 인젝션**으로 `Illuminate\\Http\\Request`를 직접 받는다.
- 서비스 컨테이너가 자동으로 인스턴스를 주입한다.

### JSON 요청 다루기

- `Content-Type: application/json`(또는 `+json`)이면 일반 요청처럼 `get()`으로 접근 가능.
- Content-Type이 확실치 않거나 JSON 전용 처리가 필요하면 `json('key')` 사용.

### 1-4) 폼 요청(Form Request)

```jsx
public function boot()
{
    $this->app->afterResolving(ValidatesWhenResolved::class, function ($resolved) {
        $resolved->validateResolved();
    });

    $this->app->resolving(FormRequest::class, function ($request, $app) {
        $request = FormRequest::createFrom($app['request'], $request);

        $request->setContainer($app)->setRedirector($app->make(Redirector::class));
    });
}
```

- `Illuminate\\Foundation\\Http\\FormRequest`를 상속한 클래스로
    - **입력값 취득 + 밸리데이션 + 권한(authorize)** 를 한곳에 모아 관리한다.
- 장점: 컨트롤러에서 밸리데이션 로직이 빠져 **결합도가 낮아지고 유지보수 쉬움**.
- artisan: `php artisan make:request UserRegistPost`
- 핵심 메서드:
    - `authorize()` : 요청 허용 여부
    - `rules()` : 밸리데이션 규칙
    - (선택) `messages()` : 에러 메시지 커스터마이즈

```jsx
<?php

namespace App\\Http\\Requests;

use Illuminate\\Foundation\\Http\\FormRequest;

class UserRegistPost extends FormRequest
{
    public function authorize(): bool
    {
//        return false;
        return true; // 요청 취득을 기준으로 확인하므로 실행 허가로 변경
    }

    public function rules(): array
    {
        return [
            //
        ];
    }
}
```

```jsx
<?php

declare(strict_types=1);

namespace App\\Http\\Controllers;

use App\\Http\\Requests\\UserRegistPost;

final class UserController extends Controller
{
    // 인수에 UserRegistPost 클래스의 인스턴스를 전달함
    public function register(UserRegistPost $request)
    {
        // 인스턴스에 대해 값을 문의
        $name = $request->get('name');
        $age = $request->get('age');
    }
}
```

---

### 2. 밸리데이션

```jsx
<?php

declare(strict_types=1);

namespace App\\Http\\Controllers;

use App\\Service\\UserPurchaseService;
use Illuminate\\Http\\Request;
use Illuminate\\Support\\Facades\\Validator;

final class UserController extends Controller
{
    public function register(Request $request)
    {
        // 모든 입력값을 얻어 $inputs에 저장한다
        $inputs = $request->all();

        // 1. 밸리데이션 규칙 정의
        // name 키 값은 필수, age는 정숫값
        $rules = [
            'name' => 'required',
            'age' => 'integer',
        ];

        // 2. 밸리데이션 클래스 인스턴스 취득
        $validator = Validator::make($inputs, $rules);

        if ($validator->fails()) { // 3
            // 값이 에러인 경우의 처리
        }
        // 값이 정상인 경우의 처리
    }
}
```

### 2-1) 규칙 지정 방법

- 입력 필드명 => 규칙 문자열 또는 배열
- 여러 규칙은 **배열 방식 권장** (정규표현식 등에서 `|` 충돌 가능)

예:

- `['email' => ['required','email'], 'age' => 'required|integer']`

### 2-2) 주요 규칙 분류

1. 값 존재/필수 체크

- `required`, `present`, `filled`

1. 타입/포맷

- `numeric`, `alpha`, `email`, `ip`

1. 자릿수/길이/크기/범위

- `digits:4`, `in:a,b,c`, `size`, `between`, `min`, `max`
- `size`는 조합에 따라 의미가 달라짐
    - numeric/integer면 “값 자체”, 문자열이면 “문자 길이”, 배열이면 “요소 수”, 파일이면 “KB”

1. 비교

- `confirmed` (예: `email`과 `email_confirmation`)
- `unique:table,column,...` (DB 중복 체크)

1. 처리 흐름 제어

- `bail` : 한 항목에서 에러 나면 이후 규칙 검사 중단

### 2-3) 밸리데이션 적용 방식

1. 컨트롤러에서 간단히:

- `$this->validate($request, $rules)`
- 실패 시 보통 **직전 페이지로 리다이렉트 + 에러/입력값 세션 저장**

1. Validator 인스턴스로 직접 제어:

- `Validator::make(...)->fails()`로 분기 처리 가능

1. 폼 요청으로 분리:

- 컨트롤러 파라미터 타입을 FormRequest로 바꾸면
    
    **컨트롤러 진입 전에 자동 검증 + 실패 시 자동 리다이렉트**
    

```jsx
<?php

namespace App\\Http\\Requests;

use Illuminate\\Foundation\\Http\\FormRequest;

class UserRegistPost extends FormRequest
{
    public function authorize(): bool
    {
//        return false;
        return true; // 요청 취득을 기준으로 확인하므로 실행 허가로 변경
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'max:20'],
            'email' => ['required', 'email', 'max:255'],
        ];
    }
}
```

```jsx
<?php

declare(strict_types=1);

namespace App\\Http\\Controllers;

use App\\Http\\Requests\\UserRegistPost;

final class UserController extends Controller
{
    public function register(UserRegistPost $request)
    {
        // 여기에 도달할 때까지 밸리데이션 판정을 수행한다

        // 밸리데이션 통과 후의 처리
        $name = $request->get('name');
    }
}
```

### 2-4) 실패 처리와 에러 출력

- 에러는 `MessageBag`에 저장되며, 뷰에서 기본적으로 `$errors`로 접근 가능
- 예: `$errors->all()`, `$errors->has('name')`, `$errors->first('name')`

```jsx
<?php

namespace App\\Http\\Requests;

use Illuminate\\Foundation\\Http\\FormRequest;

class UserRegistPost extends FormRequest
{
    public function authorize(): bool
    {
//        return false;
        return true; // 요청 취득을 기준으로 확인하므로 실행 허가로 변경
    }

    public function messages()
    {
        return [
            'name.required' => '이름을 입력해 주십시오',
            'name.max' => '이름은 최대 20문자까지 입력할 수 있습니다',
            'email.required' => '메일주소를 입력해 주십시오',
            'email.email' => '메일주소 형식이 올바르지 않습니다',
            'email.max' => '메일주소는 최대 255문자까지 입력할 수 있습니다',
        ];
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'max:20'],
            'email' => ['required', 'email', 'max:255'],
        ];
    }
}
```

### 2-5) 규칙 커스터마이즈

1. 규칙 추가: `Validator::extend('rule', function (...) { ... })`
2. 조건부 규칙: `sometimes(field, rule, 조건클로저)`

```jsx
<?php

declare(strict_types=1);

namespace App\\Http\\Controllers;

use Illuminate\\Http\\Request;
use Illuminate\\Support\\Facades\\Validator;

final class UserController extends Controller
{
    public function register(Request $request)
    {
        // 'name'의 밸리데이션 규칙에 'ascii_alpha' 추가
        $rules = [
            'name' => ['required', 'max:20', 'ascii_alpha'],
            'email' => ['required', 'email', 'max:255'],
        ];

        $inputs = $request->all();

        // 밸리데이션 규칙에 'ascii_alpha' cnrk
        Validator::extend('ascii_alpha', function ($attribute, $value, $parameters) {
            // 영대소문자이면 true(밸리데이션 OK)
            return preg_match('/^[a-zA-Z]+$/', $value);
        });

        $validator = Validator::make($inputs, $rules);

        if ($validator->fails()) {
            // 밸리데이션 에러 시 처리
        }

        // 밸리데이션 통과 후 처리
        $name = $request->get('name');
    }
}
```

---

### 3. 응답(Response)

### 3-1) 다양한 응답 만들기

라라벨 응답은 주로 `Illuminate\\Http\\Response` 및 파생 클래스가 담당하며,

내부적으로 Symfony의 Response 계열을 기반으로 한다.

1. 문자열 응답
    - `response('hello')` 또는 `Response::make('hello')`
    - 헤더/콘텐츠 타입 조정 가능
2. 뷰(템플릿) 응답
    - `view('file')` 또는 `response(view('file'), status)`
    - 상태코드/헤더까지 제어하려면 `response()` 조합이 유용
3. JSON / JSONP
    - `response()->json([...])`
    - `response()->jsonp('callback', [...])`
    - `content-type`도 커스터마이즈 가능(예: `application/vnd...+json`)
4. 다운로드
    - `response()->download(path, filename?, headers?)`
5. 리다이렉트
    - `redirect('/')`
    - `withInput()` : 입력값 전달
    - `with()` : 플래시 메시지 등 1회성 데이터 전달
6. Server-Sent Events(SSE)
    - `response()->stream(function(){ ... }, status, headers)`
    - `content-type: text/event-stream`, 버퍼링 off 등 헤더 설정 중요

---

### 4. 미들웨어

### 4-1) 미들웨어 기본

- 컨트롤러 실행 전/후에 **요청 필터링, 응답 변경, 인증/세션/암호화 등**을 수행.
- `Illuminate\\Pipeline\\Pipeline`이 미들웨어 체인을 실행.

### 4-2) 기본 제공 미들웨어 (구성 위치: `app/Http/Kernel.php`)

1. 글로벌 미들웨어 (`$middleware`)
    - 모든 요청에 적용, 라우트 정보가 확정되기 전에 실행되는 성격
2. 그룹 미들웨어 (`$middlewareGroups`)
    - 대표적으로 `web`, `api`
    - `routes/web.php`는 기본적으로 `web` 그룹 적용
    - `routes/api.php`는 `api` 그룹 적용(보통 prefix도 `/api`)
3. 별칭(알리아스) 미들웨어 (`$middlewareAliases`)
    - `auth`, `throttle`, `verified` 등 이름으로 라우트/컨트롤러에 적용

### 4-3) 커스텀 미들웨어 구현 흐름

1. 생성: `php artisan make:middleware HeaderDumper`
2. `handle(Request $request, Closure $next)`에서
    - 요청 처리 전 로그/검사
    - `$response = $next($request)`로 다음 단계 실행
    - 응답 처리 후 로그/변경
3. `Kernel.php`에 등록(글로벌/그룹/별칭 중 목적에 맞게)

```jsx
// 요청 헤더 로그를 찍는 커스텀 미들웨어
<?php

namespace App\\Http\\Middleware;

use Closure;
use Illuminate\\Http\\Request;
use Psr\\Log\\LoggerInterface;
use Symfony\\Component\\HttpFoundation\\Response;

class HeaderDumper
{
    private $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    /**
     * Handle an incoming request.
     *
     * @param \\Closure(\\Illuminate\\Http\\Request): (\\Symfony\\Component\\HttpFoundation\\Response) $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        $this->logger->info(
            'request',
            [
                'header' => strval($request->headers)
            ]
        );
        // 헬퍼 함수 이용 시에는 다음과 같다
        // info('request', ['header' => strval($request->headers)]);
        $response = $next($request);
        $this->logger->info(
            'response',
            [
                'header' => strval($response->headers)
            ]
        );
        return $response;
    }
}
```

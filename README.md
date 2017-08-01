# Go Styleguide

이 가이드라인은 다년간의 경험과 컨퍼런스등에서 나온 영감 및 아이디어에 기반해 [Effective Go](https://golang.org/doc/effective_go.html)를 보충하는 내용을 담았습니다.

## 목차

- [에러에 컨텍스트를 추가하세요](#에러에-컨텍스트를-추가하세요)
- [의존성 관리](#의존성-관리)
  - [dep을 사용하세요](#dep을-사용하세요)
  - [Semantic Versioning을 사용하세요](#semantic-versioning을-사용하세요)
  - [gopkg.in 사용을 지양하세요](#gopkgin-사용을-지양하세요)
- [구조적 로깅](#구조적-로깅)
- [전역 변수를 피하세요](#전역-변수를-피하세요)
- [테스팅](#테스팅)
  - [Assert 라이브러리를 사용하세요](#assert-라이브러리를-사용하세요)
  - [테이블 기반 테스트를 하세요](#테이블-기반-테스트를-하세요)
  - [모킹을 지양하세요](#모킹을-지양하세요)
  - [DeepEqual 사용을 지양하세요](#deepequal-사용을-지양하세요)
  - [노출되지 않은 함수를 테스팅하지 마세요](#노출되지-않은-함수를-테스팅하지-마세요)
- [Linter를 사용하세요](#linter를-사용하세요)
- [gofmt를 사용하세요](#gofmt를-사용하세요)
- [사이드 이펙트를 피하세요](#사이드-이펙트를-피하세요)
- [순수 함수를 지향하세요](#순수-함수를-지향하세요)
- [과한 인터페이스는 피하세요](#과한-인터페이스는-피하세요)
- [거대한 패키징은 피하세요](#거대한-패키징은-피하세요)
- [시그널 처리](#시그널-처리)
- [임포트를 나누세요](#임포트를-나누세요)
- [빈 반환문 사용은 지양하세요](#빈-반환문-사용은-지양하세요)
- [패키지 주석을 사용하세요](#패키지-주석을-사용하세요)
- [빈 인터페이스 사용은 지양하세요](#빈-인터페이스-사용은-지양하세요)
- [메인 함수를 맨 위에 두세요](#메인-함수를-맨-위에-두세요)
- [내부 패키지를 사용하세요](#내부-패키지를-사용하세요)
- [helper/util 같은 이름은 피하세요](#helperutil-같은-이름은-피하세요)
- [바이너리 데이터를 임베딩 하세요](#바이너리-데이터를-임베딩-하세요)
- [데코레이터 패턴을 사용하세요](#데코레이터-패턴을-사용하세요)

## 에러에 컨텍스트를 추가하세요

**비권장:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

위의 방법을 사용하면 컨텍스트가 누락되어 에러 메시지가 불분명해질 수 있습니다.

**권장:**
```go
import "github.com/pkg/errors" // for example

// ...

file, err := os.Open("foo.txt")
if err != nil {
	return errors.Wrap(err, "open foo.txt failed")
}
```

커스텀 메시지로 에러를 래핑하면 메시지가 에러 스택으로 전파되면서 컨텍스트를 전달합니다.

이는 항상 의미있는건 아닙니다. 

만약 반환되는 에러의 컨텍스트가 항상 충분하다고 확신할 수 없는 경우에 래핑하세요.

타입 체킹을 위해 루트 에러에 여전히 접근이 가능한지 확인하세요.

## 의존성 관리

### dep을 사용하세요
[dep](https://github.com/golang/dep)을 사용하세요. 이는 프로덕션 준비 상태이며, 곧 툴체인의 일부로 들어갈 예정입니다.

– [Sam Boyer at GopherCon 2017](https://youtu.be/5LtMb090AZI?t=27m57s)

### Semantic Versioning을 사용하세요
`dep`은 버전 관리가 가능하므로 [Semantic Versioning](http://semver.org)을 사용해 패키지를 태그하세요.

### gopkg.in 사용을 지양하세요
[gopkg.in](http://labix.org/gopkg.in)은 훌륭한 툴이며 실제로 유용했지만, 이는 하나의 버전을 태그하며 이는 `dep`과 함께 사용할 수 없습니다.

직접적인 임포트를 선호하고 `Gopkg.toml`에 버전을 명시하세요.

## 구조적 로깅

**비권장:**
```go
log.Printf("Listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**권장:**
```go
import "github.com/uber-go/zap" // for example

// ...

logger, _ := zap.NewProduction()
defer logger.Sync()
logger.Info("Server started",
	zap.Int("port", port),
	zap.String("env", env),
)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","ts":1501326297.511464,"caller":"Desktop/structured.go:17","msg":"Server started","port":80,"env":"production"}
```

구조적 로깅은 디버깅과 로그 파싱을 쉽게 만들어 줍니다.

## 전역 변수를 피하세요

**비권장:**
```go
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```

전역 변수는 테스팅과 가독성을 어렵게 만들며 모든 메서드에서 접근을 가능하게 만듭니다. (메서드들이 이를 필요로하지 않음에도)

**권장:**
```go
func main() {
	db := // ...
	http.HandleFunc("/drop", DropHandler(db))
	// ...
}

func DropHandler(db *sql.DB) http.HandleFunc {
	return func (w http.ResponseWriter, r *http.Request) {
		db.Exec("DROP DATABASE prod")
	}
}
```

전역 변수 대신 고차원 함수를 사용해 의존성을 적절히 주입하세요.

## 테스팅

### Assert 라이브러리를 사용하세요

**비권장:**
```go
func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	if (actual != expected) {
		t.Errorf("Expected %d, but got %d", expected, actual)
	}
}
```

**권장:**
```go
import "github.com/stretchr/testify/assert" // for example

func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	assert.Equal(t, expected, actual)
}
```

테스트를 보다 읽기 쉽게 하기위해 assert 라이브러리를 사용하세요. 이는 더 적은 코드로도 테스트 작성이 가능하며 일관성 있는 에러 메시지를 제공합니다.

### 테이블 기반 테스트를 하세요

**비권장:**
```go
func TestAdd(t *testing.T) {
	assert.Equal(t, 1+1, 2)
	assert.Equal(t, 1+-1, 0)
	assert.Equal(t, 1, 0, 1)
	assert.Equal(t, 0, 0, 0)
}
```

이 방식은 간단해 보이지만 실패 케이스를 찾기가 어려우며, 특히 케이스가 많은 경우엔 더더욱 어렵습니다.

**권장:**
```go
func TestAdd(t *testing.T) {
	cases := []struct {
		A, B, Expected int
	}{
		{1, 1, 2},
		{1, -1, 0},
		{1, 0, 1},
		{0, 0, 0},
	}

	for _, tc := range cases {
		t.Run(fmt.Sprintf("%d + %d", tc.A, tc.B), func(t *testing.T) {
			assert.Equal(t, t.Expected, tc.A+tc.B)
		})
	}
}
```

하위 테스트와 함께 테이블 기반 테스트를 사용하면 어떤 케이스가 실패했고 성공했는지를 직접 파악할 수 있습니다.

– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=7m34s)

### 모킹을 지양하세요

**비권장:**
```go
func TestRun(t *testing.T) {
	mockConn := new(MockConn)
	run(mockConn)
}
```

**권장:**
```go
func TestRun(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	t.AssertNil(t, err)

	var server net.Conn
	go func() {
		defer ln.Close()
		server, err := ln.Accept()
		t.AssertNil(t, err)
	}()

	client, err := net.Dial("tcp", ln.Addr().String())
	t.AssertNil(err)

	run(client)
}
```

다른 가능한 대안이 전혀 없는 경우에만 모킹을 사용하고, 가급적 실제 구현체 사용을 지향하세요. – [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=26m51s)

### DeepEqual 사용을 지양하세요

**비권장:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	assert.True(t, reflect.DeepEqual(expected, actual))
}
```

**권장:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func (m *myType) testString() string {
	return fmt.Sprintf("%d.%s", m.id, m.name)
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	if actual.testString() != expected.testString() {
		t.Errorf("Expected '%s', got '%s'", expected.testString(), actual.testString())
	}
	// or assert.Equal(t, actual.testString(), expected.testString())
}
```

구조체를 비교하기 위해 `testString()`을 사용하면 동등 비교와 관련 없는 여러개의 필드가 있는 복잡한 구조체를 비교할 때 유용합니다.

이 방법은 매우 크거나 트리 형태의 구조체에 대해서만 유의미합니다.

– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=30m45s)

### 노출되지 않은 함수를 테스팅하지 마세요

노출된 함수로 접근이 불가능한 경우엥만 노출되지 않은 함수를 테스트하세요. 이들은 노출된 함수가 아니기 때문에 코드가 변경되기 쉽습니다.

## Linter를 사용하세요

커밋 전에 프로젝트를 linting 하기 위해 linter (예: [gometalinter](https://github.com/alecthomas/gometalinter))를 사용하세요.

## gofmt를 사용하세요

gofmt를 거친 파일들만 커밋하세요. `-s`를 사용해 코드를 깔끔하게 만드세요.

## 사이드 이펙트를 피하세요

**비권장:**
```go
func init() {
	someStruct.Load()
}
```

사이드 이펙트는 오직 특정 상황에서만 허용됩니다. (예: cmd에서 플래그 파싱) 다른 방법을 못 찾겠다면, 다시 생각하고 리팩토링을 해보세요.

## 순수 함수를 지향하세요

> 컴퓨터 프로그래밍에서, 함수에 대한 다음의 문장들이 모두 성립하면 그 함수는 순수한 함수로 간주될 수 있습니다:
>
> 1. 함수는 항상 같은 인자에 대해선 같은 결과값을 반환한다. 함수의 반환값은 그 어떤 숨겨진 정보나 프로그램의 실행시점이나 다른 프로그램의 실행중에 변할 수 있는 상태에 의존해서는 안되며 또한 I/O 디바이스로부터의 그 어떤 외부 입력에 의존해서도 안된다.
> 2. 결과의 수행은 변경 가능한 객체의 변형이나 I/O 디바이스로의 출력과 같은 의미상 관찰 가능한 사이드 이펙트나 출력이 발생하지 않는다.

– [Wikipedia](https://en.wikipedia.org/wiki/Pure_function)

**비권장:**
```go
func MarshalAndWrite(some *Thing) error {
	b, err := json.Marshal(some)
	if err != nil {
		return err
	}

	return ioutil.WriteFile("some.thing", b, 0644)
}
```

**권장:**
```go
// Marshal is a pure func (even though useless)
func Marshal(some *Thing) ([]bytes, error) {
	return json.Marshal(some)
}

// ...
```

이는 알다시피 모든 경우에 대해 가능하지는 않지만, 가능한한 순수한 함수를 작성하여 코드를 이해하기 쉽고 디버깅 하기 쉽도록 만드세요.

## 과한 인터페이스는 피하세요

**비권장:**
```go
type Server interface {
	Serve() error
	Some() int
	Fields() float64
	That() string
	Are([]byte) error
	Not() []string
	Necessary() error
}

func debug(srv Server) {
	fmt.Println(srv.String())
}

func run(srv Server) {
	srv.Serve()
}
```

**권장:**
```go
type Server interface {
	Serve() error
}

func debug(v fmt.Stringer) {
	fmt.Println(v.String())
}

func run(srv Server) {
	srv.Serve()
}
```

작은 인터페이스 사용을 지향하고 함수는 함수가 필요로 하는 인터페이스만 받도록 만드세요.

## 거대한 패키징은 피하세요

패키지를 삭제하고 합치는건 큰 패키지를 분리하는 것보다 훨씬 쉽습니다. 패키지를 분리할 수 있는지 확신이 서지 않을 땐 분리하세요.

## 시그널 처리

**비권장:**
```go
func main() {
	for {
		time.Sleep(1 * time.Second)
		ioutil.WriteFile("foo", []byte("bar"), 0644)
	}
}
```

**권장:**
```go
func main() {
	logger := // ...
	sc := make(chan os.Signal)
	done := make(chan bool)

	go func() {
		for {
			select {
			case s := <-sc:
				logger.Info("Received signal, stopping application",
					zap.String("signal", s.String()))
				break
			default:
				time.Sleep(1 * time.Second)
				ioutil.WriteFile("foo", []byte("bar"), 0644)
			}
		}
		done <- true
	}()

	signal.Notify(sc, os.Interrupt, os.Kill)
	<-done // Wait for go-routine
}
```

시그널 처리를 통해 서버를 정상적으로 종료하고 열린 파일 및 연결을 닫을 수 있으므로, 다른 것들 사이에서의 파일 손상을 방지할 수 있습니다.

## 임포트를 나누세요

**비권장:**
```go
import (
	"encoding/json"
	"github.com/some/external/pkg"
	"fmt"
	"github.com/this-project/pkg/some-lib"
	"os"
)
```

**권장:**
```go
import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/some/external/pkg"

	"github.com/this-project/pkg/some-lib"
)
```

표준, 외부 그리고 내부 패키지 임포트를 분리하면 가독성이 높아집니다.

## 빈 반환문 사용은 지양하세요

**비권장:**
```go
func run() (n int, err error) {
	// ...
	return
}
```

**권장:**
```go
func run() (n int, err error) {
	// ...
	return n, err
}
```

명명된 반환문은 문서화에 용이하며, 빈 반환문은 가독성에도 안좋으며 에러를 발생시키기가 쉽습니다.

## 패키지 주석을 사용하세요

**비권장:**
```go
package sub
```

**권장:**
```go
package sub // import "github.com/my-package/pkg/sth/else/sub"
```

패키지에 주석을 추가하여 패키지에 컨텍스트를 추가하고 임포트 하기를 쉽게 만들도록 하세요.

## 빈 인터페이스 사용은 지양하세요

**비권장:**
```go
func run(foo interface{}) {
	// ...
}
```

빈 인터페이스는 코드를 더욱 복잡하고 덜 깔끔하게 만드므로 가능하다면 지양하세요.

## 메인 함수를 맨 위에 두세요

**비권장:**
```go
package main // import "github.com/me/my-project"

func someHelper() int {
	// ...
}

func someOtherHelper() string {
	// ...
}

func Handler(w http.ResponseWriter, r *http.Reqeust) {
	// ...
}

func main() {
	// ...
}
```

**권장:**
```go
package main // import "github.com/me/my-project"

func main() {
	// ...
}

func Handler(w http.ResponseWriter, r *http.Reqeust) {
	// ...
}

func someHelper() int {
	// ...
}

func someOtherHelper() string {
	// ...
}
```

`main()`을 맨 위에 두면 파일을 읽기가 더욱 수월해집니다. `init()` 함수만이 메인 함수 위에 위치하도록 하세요.

## 내부 패키지를 사용하세요

cmd를 만들고 있다면 라이브러리들을 `internal/`로 옮겨 불안정한 임포트나 패키지 변경을 방지하세요.

## helper/util 같은 이름은 피하세요

간단명료한 이름을 사용하도록 하며 `helper.go`나 `utils.go`와 같은 이름을 갖는 파일 혹은 패키지들은 지양하세요 

## 바이너리 데이터를 임베딩 하세요

단일 바이너리 배포를 위해선 템플릿과 기타 정적 애셋들을 바이너리에 추가할 수 있는 툴을 사용하세요.

(예: [github.com/jteeuwen/go-bindata](https://github.com/jteeuwen/go-bindata)).

## 데코레이터 패턴을 사용하세요

```go
type Config struct {
	port    int
	timeout time.Duration
}

type ServerOpt func(*Config)

func WithPort(port int) ServerOpt {
	return func(cfg *Config) {
		cfg.port = port
	}
}

func WithTimeout(timeout time.Duration) ServerOpt {
	return func(cfg *Config) {
		cfg.timeout = timeout
	}
}

func startServer(opts ...ServerOpt) {
	cfg := new(Config)
	for _, fn := range opts {
		fn(cfg)
	}

	// ...
}

func main() {
	// ...
	startServer(
		WithPort(8080),
		WithTimeout(1 * time.Second),
	)
}
```

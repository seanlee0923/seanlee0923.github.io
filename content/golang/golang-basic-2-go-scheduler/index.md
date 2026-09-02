---
date: '2026-08-26T16:33:40+09:00'
draft: false
title: 'Golang 기초 다지기 2편 - Go 언어 스케줄러 분석하기'

tags:
  - Go
  - Golang
---

#### 서론

중요! 이 글은 1.27.0 버전을 기준으로 작성한 글이다.

[1.27.0 버전 릴리즈 노트](https://go.dev/doc/go1.27)

[소스코드](https://github.com/golang/go/tree/go1.27.0)

올것이 와버렸다.

지난 글에서는 Go 런타임이 무엇인지, 그리고 프로그램이 `main.main()`에 도달하기까지 어떤 순서로 초기화되는지를 살펴봤다.

Scheduler에 대해서는 `G`, `M`, `P`를 중심으로 동작한다는 것과 Local Run Queue, Work Stealing 같은 이름들이 있다는 정도만 확인하고 넘어갔다.

이번 글에서는 그 부분을 `runtime/proc.go`와 `runtime/runtime2.go` 파일 같은 실제 고 파일을 보면서 따라가보려고 한다.

```go
go doSomething()
```

코드는 `go` 키워드 하나뿐이다.

이 한 줄이 실제로 CPU 위에서 실행되기까지 런타임 안에서 어떤 일들이 일어나는지를 확인하는 것이 이번 글의 목표다.

---

## 1. Goroutine Scheduler는 무슨 일을 할까?

우선 `runtime/proc.go`의 시작 부분을 다시 살펴보자.

Go Runtime에서는 Scheduler의 역할을

> 실행할 준비가 된 Goroutine을 Worker Thread에 분배하는 것

으로 설명한다.

그리고 Scheduler의 핵심 개념으로 `G`, `M`, `P`를 이야기한다.

지난 글에서는 이를 다음과 같이 정리했다.

```text
G = 고루틴
M = 워커 쓰레드
P = Go 코드를 실행하기 위해 필요한 자원
```

특히 중요한 부분은 **M이 Go 코드를 실행하려면 P와 연결되어 있어야 한다는 것**이다.

그러면 Scheduler가 해결해야 하는 문제도 어느 정도 보인다.

```text
실행 가능한 G
      │
      ▼
어디에 보관할까?
      │
      ▼
어떤 M이 가져갈까?
      │
      ▼
어떤 P에서 실행할까?
      │
      ▼
     CPU
```

결국 이번 글에서 알아보고 싶은 것은 크게 두 가지다

> **실행 가능한 G는 어디에서 기다리는가?**

런타임에서 전역 대기열 같은거를 관리하는걸까??

그리고

> **M은 실행할 G를 어떻게 찾아서 실행하는가?**

이 두 의문들을 중심으로 코드를 따라가 보자.

---

## 2. G, M, P를 조금 더 자세히 보기

지난 글에서는 G, M, P의 개념만 간단히 살펴봤다.

이번에는 실제 구현에서 서로 어떻게 연결되어 있는지를 조금만 더 살펴보자.

G, M, P를 나타내는 주요 구조체는 `runtime/runtime2.go`에 정의되어 있다.

모든 필드를 살펴보면 너무 많기 때문에 Scheduler를 이해하기 위해 필요한 부분만 보려고 한다.

개념적으로는 다음과 같다.

```text
G
│
└─ 실제 실행할 Goroutine

M
│
├─ 현재 실행 중인 G
└─ 현재 연결된 P

P
│
├─ 현재 연결된 M
├─ Local Run Queue
└─ runnext
```

이제 실제 구조체를 하나씩 확인해보자.

먼저 G(고루틴) 구조체다.

구조체에 주석, 필드가 상당히 많기 때문에 주요 필드를 위주로 확인하겠다. 전체 구조체가 궁금한 사람은 `runtime/runtime2.go`의 471번 Line을 직접 확인하면 된다.

```go
type g struct {
	// 이하 생략
	m         *m      // current m

	sched     gobuf

	atomicstatus atomic.Uint32

	schedlink    guintptr

	waitreason   waitReason // if status==Gwaiting
}
```

G 구조체를 확인시 아래와 같은 필드를 확인할 수 있다.(모든 필드들이 중요한데 g-m-p 기준으로 중요 필드들을 말하는것이다.)

- `m` - 이 G를 실행중인 M을 가리키는 포인터
- `schedlink` - G들을 리스트 형태로 엮을 때 사용하는 포인터
- `sched` - G의 상태를 저장하고 나중에 복원하기 위해 사용하는 값
- `atomicstatus` - G의 상태 그 자체
- `waitreason` - 만약 status가 기다리는(`_Gwaiting`) 상태일 때 기다리는 이유

이젠 M을 확인해보자.

M 또한 주요 필드를 위주로 확인하겠다. 실제 코드를 확인할 사람은 `runtime/runtime2.go`의 616번 Line부터 확인하면 된다.

```go
type m struct {
	// 이하 생략
	g0      *g     // goroutine with scheduling stack

	gsignal    *g       // signal-handling g
	curg       *g       // current running goroutine
	// p is the currently attached P for executing Go code, nil if not executing user Go code.
	//
	// A non-nil p implies exclusive ownership of the P, unless curg is in _Gsyscall.
	// In _Gsyscall the scheduler may mutate this instead. The point of synchronization
	// is the _Gscan bit on curg's status. The scheduler must arrange to prevent curg
	// from transitioning out of _Gsyscall if it intends to mutate p.
	p puintptr

	nextp           puintptr // The next P to install before executing. Implies exclusive ownership of this P.
	oldp            puintptr // The P that was attached before executing a syscall.

	spinning        bool // m is out of work and is actively looking for work
	blocked         bool // m is blocked on a note
	newSigstack     bool // minit on C thread called sigaltstack
}
```

- `g0` - 스케줄링 스택을 가지고 있는 G. 진짜 고루틴이 아니라 이 스택을 담기 위해 만들어진 껍데기 G다
- `gsignal` - 시그널 처리에 사용하는 G. 이것도 껍데기 G다
- `curg` - 현재 실행중인 G를 가리키는 포인터
- `p` - 지금 이 M에 연결되어 있는 P. Go 코드를 실행하고 있지 않다면 nil이다
- `nextp` - 실행을 시작하기 전에 붙일 P를 미리 넣어두는 자리
- `oldp` - syscall에 들어가기 직전에 붙어 있던 P를 저장해두는 자리
- `spinning` - M이 실행할 G를 찾지 못해서 일거리를 찾아 돌아다니고 있는 상태
- `blocked` - M이 note에 블록되어 있는 상태. `note`는 런타임 내부에서 쓰는 일회성 통지 수단이다
- `newSigstack` - `minit`이 시그널 스택을 직접 깔았는지 여부. 나중에 되돌리기 위해 기록해둔다

여기서 한 가지 눈에 띄는 것이 있다. `m` 구조체에는 `g` 포인터가 세 개나 있다. `g0`, `gsignal`, `curg`다.

G가 세 개 필요해서가 아니라 **스택이 세 개 필요해서**다. Go에서 스택은 `g`에 매달려 있기 때문에 스택을 바꾸려면 `g`를 바꿔야 한다.

`runtime/HACKING.md`의 Stacks 항목에서 이를 설명하고 있다.

```text
Every M has a *system stack* associated with it (also known as the M's
"g0" stack because it's implemented as a stub G) and, on Unix
platforms, a *signal stack* (also known as the M's "gsignal" stack).
```

- `curg` - 유저 스택. 지금 실행 중인 goroutine의 스택이고 필요에 따라 늘어난다
- `g0` - 시스템 스택. Scheduler 코드가 실행되는 곳이다. 크기가 고정되어 있고 GC가 스캔하지 않는다
- `gsignal` - 시그널 스택. 시그널 핸들러 전용이다

`schedule()`이나 `findRunnable()` 같은 Scheduler 코드가 유저 G의 스택 위에서 실행되면 곤란하다. G를 교체하는 코드가 그 G의 스택에 얹혀 있는 셈이 되기 때문이다.

그래서 M은 스케줄링을 위한 별도의 스택을 따로 가지고 있고, 그것을 담기 위한 껍데기 G가 `g0`다.

마지막으로 P 구조체이다.

P 또한 직접 확인할 사람은 `runtime/runtime2.go` 파일의 774 번 Line 부터 확인하면 된다.

```go
type p struct {
	// 이하 생략
	m           muintptr   // back-link to associated m (nil if idle)

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	//
	// Note that while other P's may atomically CAS this to zero,
	// only the owner P can CAS it to a valid G.
	runnext guintptr
}
```

- `m` - 관련 m 에 대한 백링크, idle 상태면 nil 을 가진다.
- `runqhead` - runq 에서 G를 꺼내는 위치를 나타내는 값
- `runqtail` - runq 에 G를 넣는 위치를 나타내는 값
- `runq` - 실행 가능한 G들을 담아두는 크기 256의 배열. 이렇게 runq, head, tail 은 lock 제어 없이 접근이 가능하다.
- `runnext` - 만약 nil 이 아닌 경우, 현재 G 에 의해 준비 상태가 되어서 runq 에 있는 고루틴 대신 다음 바로 실행되어야 하는 runnable 상태의 G 이다.

그런데 `runq`를 처음 봤을 때 좀 헷갈렸다.

```go
runq     [256]guintptr
```

`[]guintptr`이 아니라 `[256]guintptr`이다.

슬라이스가 아니라 **크기가 256으로 고정된 배열**이고, P 구조체 안에 통째로 들어있다.

그래서 앞에서 G 구조체를 볼 때 나왔던 `schedlink`는 여기에서 쓰이지 않는다. 배열에 그대로 담기 때문에 G끼리 서로 엮을 필요가 없다.

배열이면 그냥 인덱스로 접근하면 될 것 같은데 왜 `runqhead`와 `runqtail`을 따로 두고 큐처럼 사용할까?

`runqput()`에서 실제로 G를 배열에 넣는 부분을 보면 힌트가 있다. (`runtime/proc.go`의 7529번 Line)

```go
pp.runq[t%uint32(len(pp.runq))].set(gp)
```

`t`를 배열 크기로 나눈 **나머지**를 인덱스로 사용한다.

즉 `runqhead`와 `runqtail`은 배열의 인덱스가 아니라 **계속 증가하기만 하는 카운터**다.

값이 256을 넘어가도 나머지 연산 때문에 자연스럽게 배열의 앞쪽으로 돌아온다.

흔히 **링 버퍼(원형 큐)** 라고 부르는 방식이다.

```text
runq [256]guintptr

 인덱스  0    1    2    3   ...  254  255
      ┌────┬────┬────┬────┬───┬────┬────┐
      │    │ G  │ G  │ G  │   │    │    │
      └────┴────┴────┴────┴───┴────┴────┘
             ▲              ▲
             │              │
        head % 256      tail % 256
        (여기서 꺼냄)    (여기에 넣음)
```

이렇게 카운터로 두면 큐의 상태를 뺄셈 하나로 알 수 있다.

```text
tail - head == 0     → 큐가 비어있음
tail - head == 256   → 큐가 가득 참
tail - head          → 지금 큐에 들어있는 G의 개수
```

그리고 여기서 한 가지 사실이 나온다.

> **하나의 P가 가지는 Local Run Queue에는 최대 256개의 G만 들어갈 수 있다.**

그럼 256개가 이미 꽉 차 있는데 새로운 G가 또 만들어지면 어떻게 될까?

그리고 lock 없이 접근한다고 했는데, 여러 M이 동시에 같은 Queue를 건드리면 문제가 없을까?

이 두 가지는 조금 뒤에 다시 살펴보자.

특히 P에는 실행 가능한 Goroutine을 저장하는 **Local Run Queue**가 존재한다.

Scheduler 전체에는 별도의 **Global Run Queue**도 존재한다. Go Runtime의 scheduler state에는 global runnable queue가 있고, P는 자신의 local runnable queue를 가진다.

그래서 처음에 생각했던

```text
G
↓
Run Queue
↓
P
```

보다는 사실

```text
            Global Run Queue
                   │
                   │
              ┌────┴────┐
              │         │
             P0         P1
              │         │
         Local RunQ Local RunQ
              │         │
            G G G      G G
```

와 같은 구조에 더 가깝다.

즉 모든 Goroutine이 하나의 Queue에서 기다리는 구조가 아니다.

각 P가 자신의 Local Run Queue를 가지고 있고, 필요에 따라 Global Run Queue도 사용한다.

그렇다면 Goroutine을 하나 만들어서 이 Queue에 넣는 과정부터 살펴보자.

---

## 3. `go f()`를 호출하면 어떻게 될까?

다음과 같은 코드가 있다고 해보자.

```go
func main() {
    go doSomething()
}
```

Go 1.27의 `proc.go`에서 `newproc()` 위의 주석을 보면 컴파일러가 `go` statement를 이 함수 호출로 바꾼다고 설명하고 있다.
newproc()함수는 `runtime/proc.go`의 5334 Line에서 확인 가능하다.

실제로 그런지는 어셈블리를 뽑아보면 바로 확인할 수 있다.

```bash
GOOS=linux GOARCH=amd64 go tool compile -S main.go
```

```text
LEAQ	main.doSomething·f(SB), AX
CALL	runtime.newproc(SB)
```

`go` 키워드는 사라지고 실행할 함수를 넘기면서 `runtime.newproc`을 호출하는 코드만 남는다.

즉 Runtime Scheduler 관점에서 새로운 Goroutine 생성의 시작점 중 하나가 `newproc()`이다.

코드를 단순하게 보면 다음과 같은 흐름을 가진다.

```text
go doSomething()
       │
       ▼
   newproc()
       │
       ▼
   newproc1()
       │
       ▼
 새로운 G 생성
       │
       ▼
   runqput()
       │
       ▼
     wakep()
```

실제 `newproc()`의 핵심 부분 역시 `newproc1()`에서 새로운 G를 만들고 현재 P를 가져와 `runqput()`을 호출하는 구조다. 이후 main이 시작된 상태라면 `wakep()`도 호출한다.

여기서 눈에 들어오는 것이 하나 있다.

새로운 G를 만든 다음 바로 실행하지 않는다.

```text
new G
  │
  X 바로 실행
  │
  ▼
runqput()
```

즉 새로운 Goroutine은 우선 **실행 가능한 상태로 만들어지고 Scheduler가 관리하는 Queue에 들어간다.**

이 부분이 중요하다.

> **Goroutine 생성과 Goroutine 실행은 별개의 과정이다.**

G를 생성했다고 해당 G를 생성한 OS Thread에서 곧바로 실행하는 것이 아니다.

Scheduler에게

> "실행할 수 있는 Goroutine이 하나 생겼다."

라고 알려주는 쪽에 가깝다.

그리고 `newproc1()`을 조금 더 들여다보면 재미있는 부분이 있다.

새로운 G가 필요한데 곧바로 할당부터 하지 않는다. (`runtime/proc.go`의 5359번 Line)

```go
newg := gfget(pp)
if newg == nil {
	newg = malg(stackMin)
	casgstatus(newg, _Gidle, _Gdead)
	allgadd(newg)
}
```

`gfget()`은 gfree 리스트에서 G를 가져오는 함수다.

즉 `go` 키워드를 쓸 때마다 항상 새로운 G를 만드는 것이 아니라 **이미 사용이 끝난 G가 있다면 그것을 재사용한다.**

그리고 함수 뒷부분에서 이 G를 실행 가능한 상태로 바꾼다. (5438번 Line)

```go
casgstatus(newg, _Gdead, status)
```

여기서 `status`는 `_Grunnable`이다.

새로 만들어진 G가 곧바로 `_Grunnable`이 되는 것이 아니라 `_Gdead`를 한 번 거친다.

```text
_Gidle → _Gdead → _Grunnable
```

그런데 이 상태값들이 무엇인지는 아직 제대로 보지 않았다.

G의 상태부터 정리하고 넘어가자.

---

## 4. Goroutine의 상태

3장에서 `newproc1()`을 따라가다 보니 `_Gidle`, `_Gdead`, `_Grunnable` 같은 값들이 나왔다.

2장에서 G 구조체를 볼 때 `atomicstatus`라는 필드가 있었다. 여기에 들어가는 값들이 `runtime2.go` 맨 위에 상수로 정의되어 있다.
구조체보다 위쪽에 있어서 그냥 스크롤을 내리면 지나치기 쉽다. (`runtime/runtime2.go`의 17번 Line부터)

```text
_Gidle             0    갓 할당되어 아직 초기화되지 않음
_Grunnable         1    run queue에 들어있음
_Grunning          2    실행 중
_Gsyscall          3    system call 실행 중
_Gwaiting          4    런타임에서 블록됨
_Gmoribund_unused  5    현재 사용하지 않음
_Gdead             6    지금 아무도 사용하지 않는 상태
_Genqueue_unused   7    현재 사용하지 않음
_Gcopystack        8    스택을 옮기는 중
_Gpreempted        9    preempt 되어 스스로 멈춤
_Gleaked          10    GC가 찾아낸 누수된 goroutine
_Gdeadextra       11    extra M에 붙어있는 _Gdead
```

생각보다 많다.

여기에 더해서 `_Gscan`이라는 값도 있는데 이건 별개의 상태가 아니라 다른 상태와 함께 사용하는 비트 플래그다. GC가 해당 goroutine의 스택을 스캔하는 중이라는 표시다.

3장에서 봤던 `_Gdead`도 목록에 있다.

이름만 보면 죽은 goroutine처럼 보이는데 주석을 읽어보면 조금 다르다.

```go
// _Gdead means this goroutine is currently unused. It may be
// just exited, on a free list, or just being initialized.
```

세 가지 경우를 이야기하고 있다. 방금 종료된 경우, free list에 들어가 재사용을 기다리는 경우, 이제 막 초기화되는 경우다.

3장에서 `gfget()`으로 G를 가져오고 `_Gdead`에서 `_Grunnable`로 올리는 코드를 봤는데, 그게 두 번째와 세 번째 경우에 해당한다.

즉 `_Gdead`는 "종료된 상태"보다는 **"지금 아무도 쓰지 않는 상태"** 에 가깝다.

Scheduler를 이해하기 위해 우선 다음 정도만 생각하려고 한다.

```text
_Grunnable
_Grunning
_Gwaiting
_Gsyscall
```

이름 그대로 보면 어느 정도 의미를 알 수 있다.

그런데 상수 블록 맨 위의 주석을 보면 상태가 단순한 표시값이 아니라는 이야기가 나온다.

```go
// Beyond indicating the general state of a G, the G status
// acts like a lock on the goroutine's stack (and hence its
// ability to execute user code).
```

G의 상태는 지금 무엇을 하고 있는지를 나타내는 것을 넘어서 **그 goroutine의 스택에 대한 lock처럼 동작한다**는 것이다.

그래서인지 각 상수의 주석을 읽어보면 전부 스택 소유권 이야기가 같이 붙어있다.

### `_Grunnable`

실행할 준비가 된 상태다.

아직 CPU에서 실행되고 있는 것은 아니지만 Scheduler가 선택하면 실행할 수 있다.

주석에서는 run queue에 들어있으며 유저 코드를 실행하고 있지 않고, **스택을 소유하지 않는다**고 설명한다.

### `_Grunning`

실제로 실행 중인 상태다.

M과 연결되어 Go 코드를 실행하고 있다.

이때는 **스택을 이 goroutine이 소유한다.** run queue에는 들어있지 않고, M이 할당되어 있으며(`g.m`이 유효하다) 보통은 P도 가지고 있다.

### `_Gwaiting`

어떤 이유로 실행을 기다리고 있는 상태다.

Channel, lock, sleep 등 여러 이유로 Goroutine이 기다릴 수 있다.

run queue에는 없지만 나중에 다시 깨울 수 있어야 하기 때문에 **어딘가에는 기록되어 있어야 한다.** 주석에서는 channel의 wait queue를 예로 들고 있다.

### `_Gsyscall`

System Call을 실행하고 있는 상태다.

유저 코드를 실행하는 것은 아니지만 **스택은 이 goroutine이 소유한다.** M도 할당되어 있다.

그리고 주석에 이런 문장이 있다.

```go
// It may have a P attached, but it does not own it.
```

P가 붙어있을 수는 있지만 소유하지는 않는다.

syscall 중인 goroutine에게서 P를 떼어내 다른 M이 사용할 수 있다는 이야기인데, 이 부분은 뒤에서 다시 살펴보자.

---

우선 이번 글에서 가장 중요한 흐름만 단순화하면 다음과 같다.

```text
_Grunnable (실행 가능한 상태)
     │
 Scheduler가 선택
     ▼
 _Grunning (실행중)
     │
     │ block
     ▼
 _Gwaiting (실행 대기)
     │
     │ 다시 실행 가능
     ▼
_Grunnable (실행 가능한 상태)
```

결국 Scheduler가 주로 관심을 가지는 대상은 **현재 실행 가능한 `_Grunnable` G**라고 볼 수 있다.

그리고 이 상태값은 아무렇게나 바꿀 수 있는 것이 아니다.

2장에서 `atomicstatus`가 그냥 `uint32`가 아니라 `atomic.Uint32`인 것을 봤는데, 상태를 변경할 때도 `casgstatus()`라는 전용 함수를 거친다. (`runtime/proc.go`의 1290번 Line)

상태가 스택에 대한 lock처럼 동작한다면 그 값을 바꾸는 것 역시 lock을 잡고 푸는 일과 같기 때문일 것이다.

그럼 이 `_Grunnable` G는 어디에 저장되는 걸까?

---

## 5. Local Run Queue와 Global Run Queue

새로운 G를 만든 `newproc()` 코드를 다시 보자.

```text
newproc()
   │
   ▼
newproc1()
   │
   ▼
runqput()
```

이름 그대로 `runqput()`은 G를 Run Queue에 넣는 함수다.

Go 1.27의 주석에서도 `runqput`은 G를 P의 local runnable queue에 넣으려고 시도한다고 설명한다.

`next`가 false라면 Local Run Queue의 뒤쪽에 넣고, true라면 `runnext`를 먼저 사용한다. Queue가 가득 찬 경우에는 일부 작업이 Global Run Queue로 넘어갈 수 있다.

대략 다음과 같이 생각할 수 있다.

```text
            P
     ┌──────┴──────┐
     │             │
  runnext       Local RunQ
     │          [ G G G G ]
     │
     G
```

`runnext`는 말 그대로 다음 실행 후보를 빠르게 지정할 수 있는 별도의 공간이라고 우선 생각하면 된다.

그리고 Local Run Queue가 가득 찬 상황에서는 `runqputslow()` 경로를 통해 일부 G를 Global Run Queue로 이동시킨다. 실제 구현에서도 Local Queue에서 batch를 구성한 뒤 global queue에 넣는다.

그런데 여기서 계속 나오는 Global Run Queue는 도대체 어디에 있는 걸까?

Local Run Queue는 2장에서 본 것처럼 `p` 구조체 안에 있었다. 하지만 Global Run Queue는 어느 P에도 속하지 않으니 다른 곳에 있어야 한다.

`runtime2.go`의 전역 변수 블록에 있다. (`runtime/runtime2.go`의 1408번 Line)

```go
var (
	// 이하 생략
	sched         schedt
)
```

이 `sched` 하나가 Scheduler 전체의 전역 상태다. 앞으로 계속 나올 `sched.lock`도 이 변수다.

타입인 `schedt` 구조체는 `p` 구조체 바로 다음에 정의되어 있다. (932번 Line)

```go
type schedt struct {
	// 이하 생략
	lock mutex

	// Global runnable queue.
	runq gQueue
}
```

`sched.runq`가 Global Run Queue다.

여기서 타입이 눈에 띈다. Local Run Queue는 `[256]guintptr` 배열이었는데 Global은 `gQueue`다. (`runtime/proc.go`의 7802번 Line)

```go
type gQueue struct {
	head guintptr
	tail guintptr
	size int32
}
```

배열이 아니라 **링크드 리스트**다. G를 꺼내는 코드를 보면 확실하다.

```go
func (q *gQueue) pop() *g {
	gp := q.head.ptr()
	if gp != nil {
		q.head = gp.schedlink
```

2장에서 봤던 `g` 구조체의 `schedlink`가 여기서 쓰인다. Local Run Queue는 배열이라 G들을 엮을 필요가 없었는데, Global Run Queue는 리스트라서 G가 다음 G를 가리켜야 한다.

그리고 접근 방식도 다르다.

```go
func globrunqput(gp *g) {
	assertLockHeld(&sched.lock)
	sched.runq.pushBack(gp)
}
```

`globrunqput()`의 첫 줄이 `assertLockHeld(&sched.lock)`이다. `sched.lock`을 잡지 않고 호출하면 죽는다. 꺼내는 `globrunqget()`도 마찬가지다.

2장에서 P 구조체에 붙어있던 `// Queue of runnable goroutines. Accessed without lock.` 주석과 정확히 반대다.

```text
Local  (p.runq)       [256]guintptr 배열     lock 없음
Global (sched.runq)   gQueue 링크드 리스트    sched.lock 필수
```

그래서 Scheduler의 Run Queue 구조를 조금 더 정확하게 표현하면 다음과 같다.

```text
                     Global Run Queue
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
        P0                P1                P2
         │                 │                 │
    Local RunQ        Local RunQ        Local RunQ
    [G G G G]         [G G]             [G G G]
```

왜 이런 구조를 사용할까?

만약 모든 P가 하나의 Global Run Queue만 사용한다면 여러 P가 계속 같은 Queue에 접근해야 한다.

방금 본 것처럼 Global Run Queue를 건드리려면 매번 `sched.lock`을 잡아야 한다. G를 하나 넣고 뺄 때마다 모든 P가 이 lock 하나를 두고 경쟁하게 된다.

반대로 P마다 Local Run Queue를 가지고 있으면 자신의 Queue에서 대부분의 작업을 처리할 수 있다.

그런데 여기서 또 다른 문제가 생긴다.

```text
P0                 P1
G G G G G G        없음
```

P0에는 실행할 G가 많은데 P1에는 아무것도 없을 수도 있다.

그럼 P1은 놀고 있어야 할까?

조금 뒤 Work Stealing에서 다시 살펴보자.

우선 Queue에 들어간 G를 Scheduler가 어떻게 선택하는지부터 보자.

---

## 6. Scheduler의 중심 `schedule()`

`proc.go`를 보다 보면 이름부터 굉장히 직접적인 함수가 나온다.

```go
func schedule()
```

주석 역시 굉장히 직접적이다.

`한 번의 scheduler 동작에서 runnable goroutine을 찾아 실행한다`는 역할을 가지고 있으며 함수는 반환하지 않는다.

복잡한 코드를 일단 모두 제외하고 보면 핵심 흐름은 다음처럼 보인다.

```text
schedule()
    │
    ▼
findRunnable()
    │
    ▼
실행할 G 선택
    │
    ▼
execute()
```

실제 `schedule()`에서도 `findRunnable()`을 호출해 실행할 G를 가져오는 코드가 존재한다.

이걸 보고 나니까 Scheduler가 하는 일이 조금 단순하게 보이기 시작한다.

```text
while (...)
    실행 가능한 G를 찾는다.
    찾은 G를 실행한다.
```

물론 실제 구현은 이렇게 간단하지 않다.

GC가 동작 중일 수도 있고,
Timer도 확인해야 하고,
Network Poller도 확인해야 하고,
다른 P에서 G를 가져올 수도 있고,
실행할 일이 없다면 M을 대기시켜야 할 수도 있다.

그래서 실제로 복잡한 부분은 `schedule()` 자체라기보다는

```go
findRunnable()
```

에 많이 들어있다.

결국 이런 질문이다.

> **그래서 실행할 G는 어디서 찾지?**

---

## 7. `findRunnable()` — 실행할 G 찾기

`findRunnable()`의 주석을 보면 이 함수가 무엇을 하는지 비교적 잘 설명되어 있다.

실행 가능한 Goroutine을 찾으면서 다른 P에서 steal하기도 하고, Local / Global Queue를 확인하고, Network Polling도 수행한다.

코드를 처음 열었을 때 솔직히 꽤 당황했다.

함수 하나를 읽는데 GC, timer, netpoll, spinning M 등 지금까지 제대로 살펴보지 않은 내용들이 한꺼번에 나온다.

`runtime/proc.go`의 3404번 Line부터 3813번 Line까지, 함수 하나가 400줄이 넘는다.

그래서 모든 조건문을 이해하려고 하기보다는 **어디를 어떤 순서로 찾는지**만 따라가 보기로 했다.

```text
findRunnable()
      │
      ▼
① 61번에 한 번 Global Run Queue      proc.go:3458
      │ 없으면
      ▼
② 현재 P의 Local Run Queue           proc.go:3484
      │ 없으면
      ▼
③ Global Run Queue                   proc.go:3491
      │ 없으면
      ▼
④ Network Poller (논블로킹)          proc.go:3511
      │ 없으면
      ▼
⑤ 다른 P에서 Work Stealing           proc.go:3537
      │ 그래도 없으면
      ▼
⑥ P를 반납하고 마지막으로 재확인
      │
      ▼
   stopm() 으로 M을 재운다            proc.go:3811
```

Local Run Queue, Global Run Queue, Network Poller를 한꺼번에 확인하는 것이 아니라 순서가 정해져 있다. 앞쪽에서 G를 찾으면 뒤쪽은 아예 보지 않는다.

여기서 ①이 눈에 띈다. Local Run Queue보다 **먼저** Global Run Queue를 확인하는 경우가 있다.

```go
// Check the global runnable queue once in a while to ensure fairness.
// Otherwise two goroutines can completely occupy the local runqueue
// by constantly respawning each other.
if pp.schedtick%61 == 0 && !sched.runq.empty() {
```

주석이 이유를 그대로 설명하고 있다.

Local Run Queue만 계속 확인하면 서로를 계속 깨우는 두 개의 goroutine이 Local Queue를 독점해버릴 수 있다. 그러면 Global Run Queue에 있는 G들은 영원히 실행되지 않는다.

그래서 61번에 한 번은 Global Run Queue를 먼저 본다.

5장에서 Queue를 Local과 Global로 나누면 동기화 비용이 줄어든다고 했는데, 그 대가로 이런 공평성 문제가 생긴다. 여기서 그것을 보정하고 있는 셈이다.

그리고 ⑥도 재미있다.

일을 찾지 못했다고 M이 곧바로 잠드는 것이 아니라 **P를 반납한 뒤에** 한 번 더 확인한다.

이때 2장에서 봤던 `allpSnapshot`이 등장한다. P를 놓은 상태에서는 전역 `allp`를 그냥 읽을 수 없기 때문에 P를 놓기 직전에 스냅샷을 떠둔다. (`proc.go`의 3602번 Line)

그리고 정말 할 일이 없으면 `stopm()`으로 M을 재운다.

```go
	stopm()
	goto top
```

나중에 M이 다시 깨어나면 `top`으로 돌아가 처음부터 다시 찾는다.

중요한 것은 Scheduler가 단순히 현재 P의 Local Run Queue 하나만 확인하고 끝나는 것이 아니라는 점이다.

현재 P에 실행할 G가 없다면 다른 곳에서도 일을 찾는다.

그중 하나가 Work Stealing이다.

---

## 8. Work Stealing

다시 다음 상황을 생각해보자.

```text
P0                      P1
Local Run Queue         Local Run Queue
G G G G G G G           empty
```

만약 P1이 자기 Local Run Queue만 확인한다면 P1은 실행할 일이 없다.

CPU를 사용할 수 있음에도 P0에 있는 G들이 끝날 때까지 기다려야 한다.

이런 상황에서 다른 P가 가지고 있는 작업을 가져오는 방식을 **Work Stealing**이라고 한다.

Go Runtime에는 이를 위한 `runqsteal()` 등의 구현이 존재한다.

개념적으로는 다음과 같다.

```text
P0                           P1
G G G G G G
│ │ │
│ │ └──────────────────────▶ G
│ └────────────────────────▶ G
└──────────────────────────▶ G
```

P1에 일이 없다면 다른 P의 Run Queue에서 일부 runnable G를 가져올 수 있다.

그래서 P별 Local Run Queue를 사용하면서도 특정 P에 작업이 몰리는 문제를 줄일 수 있다.

여기까지 오면

```text
Local Run Queue를 왜 P마다 둘까?
```

와

```text
그럼 P별 작업량이 달라지면 어떻게 하지?
```

라는 두 질문이 연결된다.

```text
Global Queue 하나만 사용
        ↓
공유 자원 접근 증가

P마다 Local Run Queue 사용
        ↓
각 P에서 대부분 독립적으로 처리
        ↓
하지만 작업 불균형 발생 가능
        ↓
Work Stealing
```

Scheduler의 여러 구현이 각각 따로 존재하는 것처럼 보였는데 하나씩 따라가다 보니 서로 연결되어 있다.

---

## 9. 찾은 G를 실행하기 — `execute()`

`findRunnable()`에서 실행할 G를 하나 찾았다고 해보자.

이제 실제로 해당 G를 실행해야 한다.

이때 등장하는 함수가 `execute()`다.

`execute()`는 `runtime/proc.go`의 3346번 Line에 있다. 핵심은 이 부분이다.

```go
// Assign gp.m before entering _Grunning so running Gs have an M.
mp.curg = gp
gp.m = mp
gp.syncSafePoint = false
casgstatus(gp, _Grunnable, _Grunning)
```

M과 G가 서로를 가리키게 만든 다음 상태를 `_Grunnable`에서 `_Grunning`으로 바꾼다.

주석이 순서를 지켜야 하는 이유를 설명하고 있다. 상태를 먼저 `_Grunning`으로 바꿔버리면 그 사이에 이 G를 들여다보는 쪽에서는 실행 중인데 M이 없는 G를 보게 된다.

그리고 4장에서 이야기했던 `casgstatus()`가 여기서 나온다.

이름 그대로 CAS(Compare-And-Swap)로 상태를 바꾸는 함수다. (`runtime/proc.go`의 1290번 Line)

```go
for i := 0; !gp.atomicstatus.CompareAndSwap(oldval, newval); i++ {
```

지금 상태가 `_Grunnable`일 때만 `_Grunning`으로 바꾸고, 아니라면 성공할 때까지 반복한다.

4장에서 G의 상태가 스택에 대한 lock처럼 동작한다는 주석을 봤는데, 그래서 상태를 바꾸는 것도 단순한 대입이 아니라 lock을 잡는 일에 가깝다. 다른 쪽에서 이 G의 상태를 쥐고 있다면 놓을 때까지 기다린다.

실제 `casgstatus()`는 106줄짜리 함수이고 GC가 스캔 중인 경우의 처리 등이 더 들어있지만 우선은 이 정도만 보고 넘어가려고 한다.

그리고 `execute()`의 마지막 줄이 이것이다.

```go
gogo(&gp.sched)
```

2장에서 G 구조체를 볼 때 나왔던 `sched` 필드다. 저장해두었던 실행 문맥을 복원해서 그 G의 코드로 점프한다.

그래서 `execute()`는 값을 반환하지 않는다. 함수 위의 주석에도 `Never returns.` 라고 적혀있다.

즉 개념적으로는 다음 흐름이다.

```text
findRunnable()
       │
       ▼
   Runnable G
       │
       ▼
    execute()
       │
       ▼
G와 M을 연결
       │
       ▼
_Grunnable
       │
       ▼
 _Grunning
       │
       ▼
     실행
```

여기까지 오면 처음에 그렸던 그림을 조금 더 구체적으로 바꿀 수 있다.

처음에는 단순히

```text
G
↓
Run Queue
↓
P
↓
M
↓
CPU
```

정도로 생각했다.

실제 Scheduler 흐름을 조금 더 반영하면 다음에 더 가깝다.

```text
               G 생성
                 │
                 ▼
             _Grunnable
                 │
                 ▼
        P Local Run Queue
                 │
                 ▼
            schedule()
                 │
                 ▼
         findRunnable()
                 │
                 ▼
             execute()
                 │
                 ▼
        G + M + P 연결
                 │
                 ▼
            _Grunning
                 │
                 ▼
                CPU
```

물론 이것도 실제 구현 전체를 표현한 것은 아니다.

하지만 Goroutine이 만들어진 뒤 실행되기까지의 가장 기본적인 Scheduler 흐름은 어느 정도 보이기 시작한다.

---

## 10. 새로운 G가 생겼는데 실행할 M이 없다면?

`newproc()` 코드를 다시 보면 마지막에 이런 흐름이 있었다.

```text
newproc
   │
   ▼
runqput
   │
   ▼
wakep
```

`runqput()`이 Queue에 G를 넣는 함수라는 것은 이제 알았다.

그럼 `wakep()`는 뭘까?

Go Runtime의 주석에서는 `wakep()`를 G가 runnable 상태가 되었을 때 실행을 위한 P를 하나 더 추가하려고 시도하는 함수로 설명한다. `newproc`과 `ready`가 대표적인 호출 경로다.

생각해보면 Run Queue에 G를 넣는 것만으로는 충분하지 않다.

```text
Run Queue
 G G G G
```

실행할 G가 아무리 많아도 이를 처리할 M이 없다면 실행되지 않는다.

그렇다고 새로운 G가 하나 생길 때마다 OS Thread를 하나씩 만들거나 깨우는 것도 비효율적이다.

그래서 Go Scheduler에는 Worker Thread를 park/unpark하고 spinning 상태를 관리하기 위한 구현도 존재한다.

이 부분은 `proc.go` 맨 위의 `Worker thread parking/unparking` 주석에서 굉장히 길게 설명하고 있다.

처음 읽었을 때는 왜 Scheduler 주석 시작부터 Thread를 깨우고 재우는 얘기를 이렇게 길게 하나 싶었는데, 지금까지 흐름을 보고 나니 어느 정도 이유가 보인다.

Scheduler는 단순히

> **어떤 G를 실행할까?**

만 결정하는 것이 아니다.

> **그 G를 실행하기 위해 몇 개의 Worker Thread를 동작시켜야 하는가?**

도 관리해야 한다.

`wakep()`가 실제로 하는 일을 따라가보면 `startm()`으로 이어진다. (`runtime/proc.go`의 3050번 Line)

그리고 `startm()` 안에 이런 코드가 있다.

```go
nmp := mget()
if nmp == nil {
	// No M is available, we must drop sched.lock and call newm.
	...
	newm(fn, pp, id)
```

`mget()`으로 쉬고 있는 M을 하나 가져온다. 2장에서 M 구조체를 볼 때 나왔던 `idleNode`로 엮여있는 `sched.midle` 리스트에서 꺼내는 것이다.

가져올 M이 있으면 그 M을 깨워서 사용하고, 없으면 `newm()`으로 새로 만든다.

```text
wakep()
   │
   ▼
startm()
   │
   ├─ mget() 성공 → 자고 있는 M을 깨운다
   │
   └─ mget() 실패 → newm() 으로 새로 만든다
```

그러면 자연스럽게 다음 질문이 생긴다.

> **`newm()`은 OS Thread를 어떻게 만드는 걸까?**

---

## 11. M은 어디에서 만들어질까?

이 글의 첫 질문은 이것이었다.

> **수많은 Goroutine을 소수의 OS Thread에서 도대체 어떻게 실행하는 걸까?**

그런데 지금까지 OS Thread라고 불러온 M이 실제로 어떻게 만들어지는지는 한 번도 보지 않았다.

`newm()`부터 따라가보자. (`runtime/proc.go`의 2875번 Line)

```go
func newm(fn func(), pp *p, id int64) {
	// allocm adds a new M to allm, but they do not start until created by
	// the OS in newm1 or the template thread.
	// 이하 생략
	mp := allocm(pp, fn, id)
	mp.nextp.set(pp)
	mp.sigmask = initSigmask
	// 이하 생략
	newm1(mp)
}
```

주석이 중요한 이야기를 하고 있다.

`allocm`은 새로운 M을 `allm`에 추가하지만, 그 M이 실제로 시작되는 것은 `newm1`에서 OS가 만들어준 다음이라는 것이다.

즉 **M을 만드는 일이 두 단계로 나뉘어 있다.**

```text
allocm()     m 구조체와 g0 스택을 만든다   (아직 OS Thread가 아님)
   ↓
newm1()      실제 OS Thread를 만든다
```

그리고 `mp.nextp.set(pp)`도 눈에 띈다. 2장에서 봤던 그 `nextp`다. 아직 태어나지도 않은 M에게 사용할 P를 미리 넣어두고 있다.

### `allocm()` — M 구조체 만들기

`allocm()`은 `runtime/proc.go`의 2287번 Line에 있다.

여기서 눈여겨볼 부분은 이 두 줄이다.

```go
mp.g0 = malg(16384 * sys.StackGuardMultiplier)
mp.g0.m = mp
```

2장에서 봤던 `g0`가 여기서 만들어진다. 16KB 크기의 스케줄링 전용 스택이다.

그리고 `mp.g0.m = mp`로 g0가 자기 M을 가리키게 한다.

이 시점까지는 아직 **메모리 위의 구조체일 뿐** OS Thread가 아니다.

### `newm1()` — 진짜 OS Thread 만들기

`runtime/proc.go`의 2924번 Line이다.

```go
func newm1(mp *m) {
	if iscgo && _cgo_thread_start != nil {
		// 이하 생략
		asmcgocall(_cgo_thread_start, unsafe.Pointer(&ts))
		return
	}
	execLock.rlock()
	newosproc(mp)
	execLock.runlock()
}
```

여기서 길이 두 갈래로 갈린다.

cgo를 사용하는 경우에는 `_cgo_thread_start`를 호출한다. 이쪽을 따라가면 `runtime/cgo/pthread_unix.c`에서 `pthread_create`를 부른다.

그런데 cgo를 쓰지 않는 순수 Go 프로그램은 `newosproc()`으로 간다.

### `newosproc()` — `clone()` 시스템 콜

`newosproc()`은 OS별로 다른 파일에 들어있다. 리눅스는 `runtime/os_linux.go`의 170번 Line이다.

```go
func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
	// 이하 생략
	ret := retryOnEAGAIN(func() int32 {
		r := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0),
			unsafe.Pointer(abi.FuncPCABI0(mstart)))
	// 이하 생략
```

libc 의 `pthread_create`가 아니라 `clone` 시스템 콜을 직접 호출한다.

즉 순수 Go 프로그램에서 M은 pthread가 아니다. Go Runtime이 리눅스 커널에게 직접 스레드를 요청한다.

넘기는 인자를 하나씩 보면 지금까지 본 것들이 모두 나온다.

- `stk` = `mp.g0.stack.hi` — 방금 `allocm()`이 만든 g0 스택이다. 새 스레드는 처음부터 이 스택 위에서 실행된다
- `mp`, `mp.g0` — 새 스레드가 자신의 M과 g0를 알 수 있도록 넘긴다
- `mstart` — 새 스레드가 시작할 지점

`cloneFlags`도 같은 파일에 있다. (156번 Line)

```go
cloneFlags = _CLONE_VM | /* share memory */
	_CLONE_FS | /* share cwd, etc */
	_CLONE_FILES | /* share fd table */
	_CLONE_SIGHAND | /* share sig handler table */
	_CLONE_SYSVSEM |
	_CLONE_THREAD
```

리눅스에서 프로세스와 스레드를 나누는 것이 결국 이 플래그 조합이다. 메모리 공간, 파일 디스크립터 테이블, 시그널 핸들러를 전부 공유하는 실행 흐름을 만든다.

`pthread_create` 역시 내부적으로는 결국 `clone`을 호출한다. Go는 libc를 거치지 않고 직접 호출할 뿐이다.


- glibc 는 버전이 오르면서 `pthread_create` 에서 clone(2) -> clone3(2) 를 호출하도록 변경이 되었는데
CGO 없이 직접 시스콜 할때는 clone3(2) 가 아니라 clone(2) 을 기본으로 쓰레드를 만든다. 그리고 CgroupFD 랑 time namespace 플래그가 있는 경우만 clone3 를 호출하도록 되어있다.


### 새로 만들어진 스레드는 어디로 갈까

`clone()`에 넘긴 진입점이 `mstart`였다.

`mstart0()`(1875번 Line)에서 시작해 `mstart1()`(1917번 Line)로 이어지고, `mstart1()`의 마지막 줄이 이것이다.

```go
schedule()
```

6장에서 본 그 `schedule()`이다.

새로 태어난 OS Thread가 하는 일은 결국 하나다. **Scheduler를 실행해서 자신이 실행할 G를 찾는 것.**

```text
clone()
   │
   ▼
mstart
   │
   ▼
mstart0() → mstart1()
   │
   ▼
schedule()
   │
   ▼
findRunnable() 로 실행할 G 찾기
```

### 그래서 스레드가 무한정 늘어나지는 않는다

여기까지만 보면 `go` 키워드를 쓸 때마다 OS Thread가 생기는 것처럼 보일 수 있는데 그렇지 않다.

10장에서 본 것처럼 `startm()`은 `newm()`을 부르기 전에 항상 `mget()`을 먼저 시도한다.

```text
G가 하나 생김
   │
   ▼
wakep()
   │
   ▼
자고 있는 M이 있나?
   │
   ├─ 있다 → mget() 으로 꺼내서 깨운다     (OS Thread 재사용)
   │
   └─ 없다 → newm() → clone()             (OS Thread 신규 생성)
```

일이 없어진 M은 `stopm()`으로 `sched.midle`에 들어가 잠들고, 나중에 필요해지면 다시 깨어난다.

goroutine을 수만 개 만들어도 OS Thread가 수만 개 생기지 않는 이유가 이것이다.

그리고 Go 코드를 실제로 동시에 실행할 수 있는 M의 개수는 P의 개수, 즉 `GOMAXPROCS`로 제한된다. M이 Go 코드를 실행하려면 P가 필요하기 때문이다.

1장에서 봤던

> **M이 Go 코드를 실행하려면 P와 연결되어 있어야 한다**

는 문장이 여기서 다시 의미를 가진다.

---

## 12. 기다리던 Goroutine이 다시 실행 가능해지면?

지금까지는 새로운 Goroutine이 생기는 과정만 살펴봤다.

그런데 Goroutine은 실행 중 여러 이유로 기다릴 수 있다.

```text
_Grunning
    │
    │ block
    ▼
_Gwaiting
```

예를 들어 Channel에서 데이터를 기다리거나, lock을 기다리거나, timer를 기다릴 수 있다.

이후 기다리던 조건이 해결되면 다시 실행할 수 있어야 한다.

이때 `ready()`라는 함수가 등장한다.

`ready()`의 구현을 보면 waiting 상태의 G를 runnable 상태로 변경한 뒤 다시 `runqput()`을 호출하고 `wakep()`을 수행한다.

즉 다음과 같다.

```text
_Gwaiting
    │
    ▼
  ready()
    │
    ▼
_Grunnable
    │
    ▼
 runqput()
    │
    ▼
 Run Queue
```

여기서 재미있는 부분이 있다.

새로운 Goroutine을 만드는 경우를 다시 보면

```text
newproc()
    │
    ▼
new G
    │
    ▼
runqput()
```

기다리던 Goroutine이 깨어나는 경우에는

```text
ready()
    │
    ▼
기존 G
    │
    ▼
runqput()
```

이다.

두 상황은 서로 다르지만 Scheduler 입장에서 보면 결국 같다.

> **실행할 수 있는 G가 하나 생겼다.**

그 G를 runnable 상태로 만들고 Run Queue에 넣으면 Scheduler가 이후 실행할 수 있다.

---

## 13. 그러면 P는 왜 필요할까?

여기까지 보면 한 가지 의문이 또 생긴다.

```text
G = 실행할 코드
M = 실제 OS Thread
```

라면 그냥 G와 M만 연결하면 되는 것 아닐까?

왜 중간에 P라는 개념이 하나 더 존재할까?

P는 단순히 G와 M 사이를 이어주는 객체가 아니다.

Local Run Queue와 같은 Scheduler 상태를 가지고 있으며, Go 코드를 실행하는 데 필요한 여러 Runtime resource와 연결되어 있다.

또 M이 System Call에 Block되는 경우에도 P를 반드시 같이 붙잡고 있을 필요는 없다.

`proc.go`에는 이런 상황에서 P를 다른 M이 사용할 수 있도록 넘기는 코드도 존재한다.

```text
평소
G
│
M
│
P
│
CPU
```

M이 System Call에서 Block됐다고 가정해보자.

```text
G + M
  │
syscall
  │
 Block
```

이때 P까지 계속 해당 M에 묶여 있다면 P가 가지고 있는 실행 능력 역시 함께 놀게 된다.

그래서 Scheduler는 상황에 따라 M과 P의 연결을 변경할 수 있다.

```text
Blocked M
       P
       │
    다른 M
       │
   다른 G 실행
```

이 부분까지 들어가면 `handoffp()`, syscall 진입/복귀 과정 등을 같이 봐야 하기 때문에 이번에는 전체 Scheduler 흐름을 이해하는 정도까지만 보고, 이후 좀 더 자세하게 살펴보려고 한다.

(분명 Scheduler 하나만 보면 될 줄 알았는데 계속 새로운 함수가 튀어나온다...)

---

## 14. 전체 흐름 다시 보기

이제 처음 질문으로 돌아가 보자.

> **수많은 Goroutine을 소수의 OS Thread에서 어떻게 실행할까?**

지금까지 살펴본 가장 기본적인 흐름을 연결하면 다음과 같다.

```text
go f()
 │
 ▼
newproc()
 │
 ▼
newproc1()
 │
 ▼
새로운 G 생성
 │
 ▼
_Grunnable
 │
 ▼
runqput()
 │
 ▼
P Local Run Queue
 │
 ▼
schedule()
 │
 ▼
findRunnable()
 │
 ├──────── Local Run Queue
 │
 ├──────── Global Run Queue
 │
 ├──────── Network Poller
 │
 └──────── 다른 P에서 Work Stealing
 │
 ▼
실행할 G 선택
 │
 ▼
execute()
 │
 ▼
G + M 연결
 │
 ▼
_Grunning
 │
 ▼
CPU에서 실행
```

그리고 실행 중 기다려야 한다면 다시

```text
_Grunning
    │
    ▼
_Gwaiting
    │
    │ 조건 만족
    ▼
  ready()
    │
    ▼
_Grunnable
    │
    ▼
 Run Queue
```

로 돌아간다.

그래서 처음의 질문에 지금 단계에서 답해본다면 다음과 같이 정리할 수 있을 것 같다.

Go Runtime은 실행 가능한 Goroutine을 P의 Local Run Queue와 Global Run Queue 등을 통해 관리한다.

P를 가진 M은 Scheduler를 실행하면서 `findRunnable()`을 통해 실행할 G를 찾는다.

현재 P에 일이 없다면 다른 P에서 Work Stealing을 시도하거나 Network Poller 등 다른 실행 가능한 작업도 확인한다.

실행할 G를 찾으면 `execute()`를 통해 M과 G를 연결하고 실제 Go 코드를 실행한다.

결국 굉장히 단순화해서 생각하면 Scheduler는 계속해서

```text
Runnable G가 생김
       ↓
Run Queue에 저장
       ↓
실행할 G를 찾음
       ↓
M이 G를 실행
       ↓
다시 다음 G를 찾음
```

이라는 일을 반복하고 있다고 볼 수 있다.

---

## 15. 정리

지난 글에서는 Go Runtime 전체를 살펴보면서 Scheduler가

```text
G
P
M
```

이라는 구조를 사용한다는 것까지만 확인했다.

이번에는 `runtime/proc.go`의 구현을 조금 더 따라가면서 Goroutine이 생성되고 실제 실행되기까지의 흐름을 살펴봤다.

가장 중요한 함수만 다시 정리하면 다음과 같다.

```text
newproc()
    ↓
새로운 Goroutine 생성

runqput()
    ↓
Runnable G를 Run Queue에 추가

schedule()
    ↓
Scheduler의 한 번의 scheduling 과정

findRunnable()
    ↓
실행할 Goroutine 탐색

execute()
    ↓
선택된 G를 M에서 실행
```

1편에서 `proc.go`를 처음 열었을 때는 `schedinit()`, `newproc()`, `schedule()`, `findRunnable()`, `execute()` 같은 함수 이름만 눈에 익혀두고 넘어갔다.

이번에는 그 함수들이 따로 존재하는 것이 아니라 하나의 흐름으로 연결되어 있다는 것을 확인했다.

물론 실제 Scheduler는 이것보다 훨씬 복잡하다.

Worker Thread의 parking/unparking, spinning M, timer, network poller, syscall, preemption 등 아직 제대로 살펴보지 않은 내용이 많이 남아 있다.

그래도 처음부터 `proc.go`의 모든 조건문과 동기화 코드를 이해하려고 하기보다는

> **"Runnable G가 생기고, Queue에 들어가고, Scheduler가 이를 찾아 M에서 실행한다."**

라는 큰 흐름을 먼저 이해하고 나니 코드가 조금은 덜 막막하게 보이는 것 같다.

내가 그냥 `go` 딸깍 하면서 편하게 고루틴을 사용하고 있었는데 뒤에서는 이런 일이 일어나고 있었다 ㅋㅋ.

다음에는 Memory Allocator를 보기 전에 Scheduler에서 남은 부분을 조금 더 살펴볼지 고민해봐야겠다.

(벌써 `proc.go` 하나만으로 글을 몇 개 쓸 수 있을 것 같다.)

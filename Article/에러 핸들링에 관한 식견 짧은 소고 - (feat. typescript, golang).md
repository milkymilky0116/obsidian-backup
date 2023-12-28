
학부생 시절까지만 해도 에러 핸들링에 관해서 그다지 관심을 두지 않았었다. 프로그램은 요구사항 대로 잘 돌아가기 만 하면 장땡이라고 생각했고, "안정성 있는 코드"에 대해 깊게 생각하지 못했다. 하지만 최근 몇년간 백엔드, 조금의 프론트를 공부하고 프로젝트를 해보면서 "어떻게 에러를 잘 처리해야 하는지" 에도 관심이 많이 생기게 되었다. 그리고 이 글은 나름대로 여러 언어들로 개발하면서 겪었던 '에러 핸들링'에 관한 경험을 소개하려는 취지로 작성되었다. 물론 식견이 짧은 주니어 개발자의 글이라서 여러 부분에서 틀린 점이 존재할 수도 있으니 '이런 생각을 가지고 있구나~' 정도로 보고 넘어가주시면 되겠다.

### try-catch 와 err != nil

Python/Typescript 개발자가 Golang을 처음 접했을때, Golang의 에러 핸들링 방식을 보고 충격받았었다. 그 방식이 슬쩍 보기에는 너무나도 단순하고 예뻐보이지 않았기 때문이다. 그래서 이런 의문이 들었다. 왜 golang은 try-catch 를 도입하지 않았을까?

우선 아래와 같은 상황을 가정하자.
1. `foo()` 와 `bar()` 라는 함수 두가지를 써야하는 상황이고, 두 함수 모두 에러가 생길수 있는 여지가 있다.
2. `foo()` 와 `bar()`는 integer(혹은 number) 를 리턴한다.
3. 에러를 최대한 잘 핸들링 하고 싶다.

Typescript로 코드를 작성하면 이런 형태일 것이다.

```typescript
function foo():number {
	//...
	return 1
}

function bar():number {
	//...
	return 2
}

async function main() {
	try {
		const foo = await foo();
		const bar = await bar();
	}catch(error){
		// some error handling...
	}
}
```

하지만 Golang 으로 작성한다면 이런 형태일것이다.

```go
func foo() (error,int) {
	//...
	return nil, 1
}

func bar() (error,int) {
	return nil, 2
}

func main() {
	err, foo := foo()
	
	if err != nil {
		// some error handling...
	}
	
	err, bar := bar()
	
	if err != nil {
		// some error handling...
	}
}
```


언뜻 보기에 Golang의 방식은 전혀 우아하지 않다. error를 인자로 넘겨주는 Golang의 방식보다 try-catch 블록으로 함수를 감싸는 Typescript 의 방식이 더 간결해보이고 언어-디자인 적으로 봤을때 꽤 예뻐보일지도 모르겠다.

Golang의 에러 핸들링 방식은 try-catch에 익숙해져 있는 개발자가 보기에 많은 의문을 남긴다. 만약 에러가 발생할 수 있는 함수들이 수십 가지라면? 코드 전체가 `if err != nil` 로 채워질것이고 코드는 꽤.. 못생겨보일 것이다.

```go

func main() {
	_, err := somefunctionA()
	
	if err != nil {
		//some error handling..
	}
	
	_, err = somefunctionB()
	
	if err != nil {
		//some error handling..
	}
	
	_, err = somefunctionC()
	
	if err != nil {
		//some error handling..
	}
	
	_, err = somefunctionD()
	
	if err != nil {
		//some error handling..
	}
	
	//...
}
```


![[스크린샷 2023-12-28 오후 7.36.08.png]]
*Golang의 에러 핸들링 방식은 전혀 우아해보이지 않고, 난잡해 보이기 까지 한다.*

### err != nil 은 못생겼다. 하지만...

상황을 하나 더 추가해보자. 이번에는 `foo()` 와 `bar()` 에서 발생할 수 있는 에러를 각각 다르게 처리하고 싶다. 그리고 `foo()`와 `bar()`의 값을 합한 값을 리턴하고 싶다.

Typescript 로 해결하려고 보니, 점점 머리가 아파오기 시작한다. try-catch 블록을 나눠서 해결하려고 하니, foo와 bar의 값을 합친 값을 리턴할 수가 없다. 왜냐하면 try-catch 블록 안에서 선언된 변수는 해당 scope 에서만 존재할 수 있기 때문이다. 

```typescript
function foo():number {
	//...
	return 1
}

function bar():number {
	//...
	return 2
}

async function main():Promise<number> {

	try {
		const foo : Promise<number> = await foo();
	}catch(error){
		// some foo error handling...
	}
	
	try {
		const bar : Promise<number> = await bar();
	}catch(error){
		// some bar error handling...
	}
	
	return foo + bar; // error: foo와 bar가 선언되지 않음
}
```

그렇다면 이렇게 해야 하는 걸까?

```typescript
function foo():number {
	//...
	return 1
}

function bar():number {
	//...
	return 2
}

async function main():Promise<number> {
	let foo : Promise<number | undefined> = undefined;
	let bar : Promise<number | undefined> = undefined;
	
	try {
		foo = await foo();
	}catch(error){
		// some foo error handling...
	}
	
	try {
		bar = await bar();
	}catch(error){
		// some bar error handling...
	}
	
	return foo + bar;
}
```

상황이 점점 좋지 않게 흘러간다. try-catch 블록 바깥에서 변수를 사용하기 위해 복잡한 타입을 가진 변수를 선언해야 하고, 그 변수가 number도 될 수 있고 undefined도 될 수 있기 때문에 만일 

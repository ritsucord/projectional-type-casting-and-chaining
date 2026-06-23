# projectional-type-casting-and-chaining

기성의 컴파일러/인터프리터들은 변수 대입에서 l-value의 타입에 r-value의 타입이 들어맞는지, 단순 일치 검사를 통해 식의 타당성 검사 및 업캐스팅을 수행한다.

그러나 r-value는 l-value의 타입에 무관하게 실행 시점에 이미 평가된다. 우리는 평가 시점에 우항이 어떤 타입으로 존재하는지에 전적으로 의존할 수밖에 없다. 이는 당연하다. 다운캐스팅은 손실된 정보를 요구하기 때문에 새롭게 객체를 구축하고, 빈 값을 채워넣는 과정이 필연적이며, 더욱이나 이러한 구조는 메타데이터를 추출할 수 없다.

템플릿 프로그래밍의 용이성을 위해, 모든 객체 및 원시값들을 메타 정보가 포함된 무거운 객체로 래핑한다는 것은 현실적으로 불가능하다. 모든 종류의 값에 그런 오버헤드를 붙일 수는 없다. 그러나 동시에 메타 프로그래밍은 별도의 문법을 요구하고, 완전히 다른 층위에서 사고하게 만드는 인지적 부하를 동반한다.

이 문제는 우리가 대입문을 너무 정적으로 바라보기에 생긴다. 만약 우변의 평가가 좌변에 할당되는 그 순간까지 실행되지 않는다면 어떻게 될까? 일반적인 값 소비에서 우변이 무거운 메타데이터를 들고 있을 필요가 없다. 동시에 좌변에서 추가적인 정보를 요구한다면 컴파일러는 이면에서 적절한 데이터를 매핑하여 넘겨준다.

```javascript
const v = 3;

const simple_num: Number = v;
const meta_info: Reflection = v;
```

위와 같은 코드에서, 컴파일러는 항상 `v`의 평가마다 무거운 정보를 덧붙일 필요가 없다. 좌변의 타입을 곧 해당 타입으로의 사영의 명시로 바라본다면, 넘겨줘야 할 정보와 버려도 되는 정보의 구분이 명확해진다. 그렇다면 컴파일러는 다음과 같이 최적화를 수행할 수 있다.

```javascript
// after compilation
const simple_num = v;
const meta_info = v.constructor;
```

결과적으로 메타 프로그래밍에 `template <typename T>`와 같은 별도의 문법의 사용이 필요 없게 된다. 동시에 이는 컴파일 언어와 인터프리터 언어 모두에서 잘 동작하고 지연된 평가를 제공하며, 최적화될 수 있다.

```cpp
void print(std::union<int, std::string> v) {
	std::type_info(std::string type_name) = v;
	if (type_name == "int")
		std::cout << "It's number, " << v << std::endl;
	else std::cout << "It's string, " << v << std::endl;
}
```

---

제네레이터는 순서가 보장된 병렬 처리를 가능하게끔 해주는 훌륭한 문법 설탕이다. 그러나 모든 곳에 제네레이터 객체를 사용하는 건 현실적으로 불가능한데, 값을 소비할 때 상당한 양의 코드를 추가적으로 작성해야 하기 때문이다.

```javascript
const a = v; // v is a simple value
const b = g.next().value; // g is a generator object
```

값이 부재할 때의 예외 처리까지 생각한다면 복잡성이 더욱 증대한다.

타입 캐스팅이 사영이라면, 사영은 중첩될 수 있고, 규칙을 적절하게 정의하여 다음과 같이 체이닝시킬 수 있을 것이다.

```javascript
const a: String: Temporal.ToEpochMilliSeconds:
Temporal.Instant = Temporal.Now.instant();
// assume 'Temporal.ToEpochMilliSeconds' as virtual type
// that ensures automatic type cast
// as alias of 'Temporal.Instant.epochMilliSeconds'
```

변수명이 곧 해당 변수에 대입되는 리터럴에 대한 타입 별칭이라고 생각할 수 있다. 즉, 타입 체이닝의 중간에 넣을 수 있고, 타입 체이닝이 파이프로 기능한다는 점을 느낄 수 있을 것이다.

```javascript
(const is_not_zero): Boolean: (const a): Number = "123456";
a == 123456; // true
is_not_zero == true; // true
```

위에 동의한다면, 다음과 같이 간결한 제네레이터 순회 설계가 가능하다.

```javascript
let g: Generator<Number> = makeNumberGenerator();

while ((const a): Generator.Value<Number, null>:
(g): Generator.Next<Number> = g)
// if there is nothing as value,
// 'Generator.Value<Number, null>' returns null,
// hence suspending while loop.
	console.log(a);
```


# 목표
> 크롬의 V8 엔진이 스크립트를 다운로드하는 동안 벌어지는 일에 대해 간단하게 알아봅니다.

컴파일은 고수준의 언어를 저수준 언어로 번역하고 문법이나 타입 체크, 최적화 등의 작업을 합니다. JavaScript는 동적 타입 언어로서 실행 중 최적화를 위해 JIT 컴파일을 합니다.  

정적 타입 언어 : Java, TypeScript처럼 실행 전에 타입을 지정.  
동적 타입 언어 : JavaScript처럼 실행 중에 타입을 결정, 추론.  

### JIT(Just-In-Time)
JS는 runtime에 컴파일됩니다. 웬만한 언어는 다들 컴파일을 합니다. 왜일까요? 똑같은 코드를 10000번 실행한다고 쳤을 때 아래와 같은 차이가 발생하기 때문입니다.  
인터프리터: 똑같은 내용을 처음부터 끝까지 10000번 일일이 읽는다.  
컴파일: 최적화해서 저장해둔 코드 10000번 출력.  

> V8은 JS를 실행할 때 JIT 컴파일을 합니다. 스크립트를 실행하기 직전 파싱 & 컴파일하기 때문에 오버헤드가 많아질 수 있는데요. 이를 방지하기 위해 코드를 캐싱합니다. ‌‌스크립트를 처음 컴파일할 땐 캐시 데이터를 만들고 저장합니다. 나중에 V8이 똑같은 스크립트를 컴파일하려고 하면 인스턴스에서 캐시를 가져와서 결과를 출력합니다. 따라서 스크립트를 더 빠르게 실행할 수 있고요.‌‌  
...	‌‌ 
캐시 생성에는 계산과 메모리 비용이 발생하는데요. 그래서 크롬은 이틀 내에 동일한 스크립트를 봤을 때 캐시를 만듭니다. 이렇게 하면 스크립트 파일을 평균 2배 이상 빠른 코드로 바꾸기 때문에 페이지 로딩 시 유저의 시간을 아낄 수 있습니다.  
-V8 블로그-

JIT 컴파일은 실행 속도가 빠릅니다. 이는 메모리 차지가 있음을 뜻하고, 메모리 접근이 상대적으로 쉽다는 뜻이 됩니다. 즉, 보안 이슈가 있습니다.  

>‌‌	어떨 땐 메모리 할당 없이 실행하는 게 바람직할 수 있는데요. 일부 플랫폼(iOS, 스마트 TV, 게임 콘솔 등)은 가용 메모리에 접근할 수 없습니다.  
V8을 사용할 수 없단 얘기죠. 가용 메모리 작성을 금지하면 어플 악용을 위한 공격 표면(attack surface)을 줄일 수 있습니다.  
.‌‌..‌  
V8의 JIT-less 모드는 이를 위한 모드입니다. jitless 플래그로 V8을 시작하면 런타임에 가용 메모리를 할당하지 않고 사용합니다.  
-V8 블로그-

메모리에 접근하기 때문에 공격 표면이 넓어지고, 속도가 빠른들 많은 메모리와 CPU 사이클을 요구해서 무겁습니다. 이 점 때문에 크롬이 욕을 많이 먹습니다.  
JIT 컴파일과 대비되는 개념으로 AOT(Ahead-of-Time) 컴파일이 있습니다. 컴파일을 브라우저에서 하느냐(JIT), 서버에서 하느냐(AOT)이며 각자 장단점이 있습니다. 여담으로 파이썬보다 빠르다는 PyPy도 JIT 방식입니다.  

>‌‌ PyPy는 JIT 컴파일러를 쓰기 때문에 대체로 파이썬보다 빠릅니다.‌‌  
-위키피디아-

### runtime

런타임은 여러 뜻이 있습니다.
- runtime (실행중)
- runtime libaray (컴파일러, 가상 머신이 쓰는 라이브러리)
- runtime environment (runtime system, 실행할 수 있도록 돕는 프로그램)

> 자바스크립트 엔진은 파싱과 스크립트 코드를 실행하고, 런타임은 자바스크립트 엔진을 포함합니다. 런타임은 메모리에 접근하는 법, 운영 체제와 상호작용하는 법, 올바른 프로그램 구문 규칙 등을 제공합니다. 각 브라우저마다 JavaScript용 런타임 환경이 있습니다.  
-apps script-

- 엔진: 코드를 파싱하고 실행 가능 명령어로 변환, 실행.
- 런타임: 외부와 상호작용하기 위해 JavaScript에 객체 제공.

Chrome 브라우저와 Node.js는 둘 다 V8 엔진을 쓰지만 런타임이 다릅니다. Chrome은 window, DOM 객체 등을 쓰고, Node에선 buffer, require, processes 등을 제공합니다. 	
브라우저에서 개발자 도구를 켜고 require를 입력해보면 찾을 수 없는 반면, Node.js를 설치한 VSCode에서 require를 쓰면 해당 API를 찾을 수 있습니다.

### Lexical Analysis

렉싱(lexing), 스캐닝(scanning)이라고도 부르며 이 단계를 담당하는 곳을 어휘 분석기(lexical analyzer), 스캐너(scanner), 렉서(lexer), 토크나이저(tokenizer) 등으로 부릅니다.	
JS 코드는 처음엔 긴 문자열로 전달되고, 이후로 여러 단계를 거쳐 코드로 변환되고 실행됩니다.

```javascript
// 개발 코드
const count = 0;
function handleClick(){
  count++;
  console.log(count);
}

// 클라이언트가 받는 스크립트 문자열
"const count=0;\n function handleClick(){\n count++;\n console.log(count);\n}\n"
```

렉서는 문자열을 스캔해서 토큰(시스템 문법적으로 가장 작은 단위의 낱말)들로 분류합니다. 간단한 예시로 확인해봅시다.  

```javascript
const a = b / 10 + "a";
```

위 코드는 렉서를 통해 아래와 같은 토큰들로 변환됩니다.
```text
const
a
=
b 
/ 
10 
+ 
"a"
```

토큰은 일반 토큰과 특수 토큰 등, 분류가 있습니다. C같은 언어는 아래와 같이 분류합니다.

 <table style="background-color: #2d2d2d; color:#ccc;" >
<thead>
<tr >
<th style="text-align: center">일반 토큰</th>
<th style="text-align: center">특수 토큰(예약어)</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center">a (식별자)</td>
<td style="text-align: center">const (지정어)</td>
</tr>
<tr>
<td style="text-align: center">b (식별자)</td>
<td style="text-align: center">= (연산자)</td>
</tr>
<tr>
<td style="text-align: center">10 (리터럴)</td>
<td style="text-align: center">/ (연산자)</td>
</tr>
<tr>
<td style="text-align: center">"a" (리터럴)</td>
<td style="text-align: center">+ (연산자)</td>
</tr>
</tbody>
</table>

JavaScript는 아래와 같이 분류합니다.

<table style="background-color: #2d2d2d; color:#ccc;" >
<thead>
<tr>
<th style="text-align: center">일반 토큰</th>
<th style="text-align: center">특수 토큰</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center">const (예약어)</td>
<td style="text-align: center"></td>
</tr>
<tr>
<td style="text-align: center">a (식별자)</td>
<td style="text-align: center"></td>
</tr>
<tr>
<td style="text-align: center">= (부호)</td>
<td style="text-align: center"></td>
</tr>
<tr>
<td style="text-align: center">b (식별자)</td>
<td style="text-align: center">/ (나누기 부호)</td>
</tr>
<tr>
<td style="text-align: center">10 (숫자 리터럴)</td>
<td style="text-align: center"></td>
</tr>
<tr>
<td style="text-align: center">+ (부호)</td>
<td style="text-align: center"></td>
</tr>
<tr>
<td style="text-align: center">"a" (문자열 리터럴)</td>
<td style="text-align: center"></td>
</tr>
</tbody>
</table>

- 특수 토큰은 나누기, 정규 표현식 리터럴// 등이 있는데 소수이고, 나머지는 전부 일반 토큰입니다. 
- 예약어는 식별자 명으로 사용 못합니다.
- 지정어(keyword)는 그 언어체계에서 특별한 의미를 가진 단어입니다. (if, while 등은 지정어 & 예약어. 하지만 지정어 === 예약어는 아닙니다.)
- 일부 예약어는 식별자 명으로 사용 가능합니다. (await는 async 함수 밖에서는 변수 이름으로 사용 가능)
- let은 예약어가 아닙니다.
- 예약어가 아니지만 strict 모드에서 식별자 명으로 못 쓰는 말이 있습니다(let, static, implements, interface, package, private, protected, public 등...)
- implements, private 같은 몇몇 단어는 future reserved words(예약 예정어)입니다.

```javascript
// strict 모드가 아니면 사용 OK
var let = 123;

// 사용 불가
let let = 123;
const let = 123;
```

### lexical scope

렉싱 단계에서는 식별자의 lexical scope를 정의합니다.  

> 식별자가 선언된 곳을 기준으로 정의한 식별자의 유효 범위. 	 
정적 영역(static scope)이라고도 부르는데, 호출한 곳을 기준으로 스코프를 정의하는 동적 영역(dynamic scope)과 대비됩니다. 대부분 현대 언어들은 정적 영역을 사용합니다.

렉싱 단계에서 토큰은 고유 번호와 값을 부여받고 파서로 전달됩니다.  

### Syntax Analysis for AST

문법 분석(Syntax Analysis) 과정은 흔히 파싱(parsing)이라고 부릅니다. 이 단계에서는 문법을 검사하고 오류 발견 시 `SyntaxError`를 출력합니다.  
검사가 무사히 통과하면 핵심 정보만 담은 문법 구조체를 만드는데 이를 AST(Abstract Syntax Tree, 추상 구문 트리), 혹은 의미 트리(semantic tree)라고 부릅니다. 옛날에는 CST(구체적 구문 트리, Concrete Syntax Tree)를 사용했으나 현대에는 AST를 많이 사용합니다.  
많이들 쓰는 `TypeSCript`, `Prettier`, `ESLint`도 내부적으로 AST를 사용합니다. 트리가 계층 구조를 잘 표현하고 탐색도 용이하기 때문이죠.  

파싱이 끝나면 정적 타입 언어는 중간 언어(Intermediate Representation)를 생성합니다. JavaScript는 동적 타입 언어이기 때문에 이 단계를 건너뜁니다.(엄밀히 밝혀두자면, JS는 중간 언어를 생성하는 시기가 다릅니다. 정적 타입 언어는 이 때 생성하고, JS는 런타임 이후 최적화를 위해 별도의 과정을 거쳐서 중간 언어를 만듭니다.)

### Bytecode Generation

![202302080006](https://github.com/hamelln/javascript-dive-notes/assets/39308313/e0de8419-536e-4c21-b11f-76002598db4b)

AST는 바로 기계어로 번역하기 전에 Bytecode로 번역합니다. 기계어보다 메모리 차지율이 적고 JS보다 번역이 빠르기 때문에 성능과 최적화를 하기 적합하기 때문입니다.  
이 작업을 담당하는 곳은 Ignition입니다. 흔히 인터프리터로서 소개되지만 V8에선 "컴파일러"라고 표현할 때도 많습니다. 이후 챕터에서 이그니션에 대해 다시 언급될 텐데 이그니션의 역할을 보면 이해가 될 것입니다.  
이렇게 번역한 Bytecode는 실행시 직렬화해서 `Heap`에 보관했다가 동일한 스크립트를 만나면 역직렬화해서 가져온 다음 실행합니다. 참고로 바이트코드 생성 중에 클로저 로직 준비를 같이 합니다.

>	“bytecode 생성 중에 BytecodeGenerator는 context object pointers(클로저 전반에 걸쳐 상태 유지에 사용) 등을 위해 함수 레지스터 파일에 레지스터를 할당합니다.”  
-Ignition: V8 Interpreter 문서-

### Compile

- Turbofan: 최적화 및 여러 복잡한 작업으로 느림
- Sparkplug: 빠른 실행

한 번만 실행할 코드는 최적화가 필요할까요? 그런 코드까지 최적화하는 건 비효율적입니다. 이 때문에 최적화를 안 하고 bytecode를 바로 기계어로 번역하는 Sparkplug가 도입됐습니다.

> 저희는 인터프리터 최적화의 한계에 부딪혔습니다. V8의 인터프리터는 최적화가 매우 잘 되어있고, 아주 빠릅니다.‌‌ 하지만 인터프리터는 해결할 수 없는 고유한 오버헤드를 발생시킵니다. 예를 들면 바이트코드 디코딩 오버헤드나 디스패치 오버헤드는 인터프리터 기능의 본질적인 부분이거든요.   
.‌‌..  
현재의 2-컴파일러 모델로는 더 빠른 최적화를 계층화할 수 없습니다.‌‌ 최적화 속도를 올리도록 할 수는 있지만, 최고 퍼포먼스를 깎고 최적화 경로를 제거해야 속도를 올릴 수 있는 상황입니다.  
.‌‌..  
스파크플러그 도입 : 이그니션 인터프리터와 터보팬 최적화 컴파일러 사이에 넣은, V8 9.1버전부터 도입된 비최적화 JS 컴파일러‌‌  
...	 ‌‌
스파크플러그는 빠르게 컴파일하도록 디자인했습니다. 엄청 빨리요. 너무 빠른 나머지 우리가 원하면 언제든 컴파일할 수 있기 때문에 터보팬 코드보다 더 적극적으로 코드를 계층화할 수 있습니다.  
-V8 블로그-


### Optimization

![202302050001](https://github.com/hamelln/javascript-dive-notes/assets/39308313/6a8f80d3-890e-4936-a178-c4ec71b1da5e)

- 첫 번째: 컴파일하면서 브라우저 캐시에 스크립트 태그를 저장합니다.
- 두 번째: 브라우저 캐시에서 스크립트를 가져와 다시 컴파일합니다. 이 때 직렬화해서 metadata로 첨부합니다. (warm run)
- 세 번째~: 브라우저 캐시에서 스크립트, metadata를 가져온다. metadata를 역직렬화해서 heap에서 매칭되는 코드를 찾아옵니다. (hot runs)

![202302140006](https://github.com/hamelln/javascript-dive-notes/assets/39308313/9f18d3dc-8f1f-4b28-96b1-678c2df8b8f5)

### Main, Background Thread

JavaScript 자체는 싱글 스레드 언어지만 V8 엔진은 멀티 스레드 환경입니다.  

> 크롬 41 버전부터 V8은 StreamedSource API로 백그라운드 스레드의 JS 파싱을 지원했습니다.‌‌ 덕분에 네트워크에서 첫 번째 청크를 다운로드하는 순간부터 파싱하고 스트리밍하는 동안에도 병렬 파싱할 수 있었죠.‌‌ 다운로드가 끝날 쯤에 파싱도 거의 동시에 끝나서 로딩 시간이 단축됐습니다.  
...	‌‌ 
이그니션은 **멀티 스레딩**을 염두하고 제작했습니다. 따라서 백그라운드 컴파일을 시키기 위해 컴파일 파이프라인을 전반적으로 손봤습니다.  
...	 
이번에 저희는 파이프라인을 이그니션 + 터보팬으로 전환하면서 Bytecode 컴파일도 백그라운드 스레드로 넘겨줄 수 있게 되었고, 덕분에 메인 스레드는 더 부드럽고 응답성이 좋은 브라우징 환경을 제공하게 됐습니다.  
-V8 블로그-

![202302100001](https://github.com/hamelln/javascript-dive-notes/assets/39308313/e6170c84-5062-46b4-bdac-602929cf943c)

## 마무리

> 이 포스팅에선 V8이 어떻게 JavaScript를 처리하는지에 대해 가볍게 알아보았습니다. 크롬은 사용자가 많지만 무겁다는 단점이 있는데요. 이 포스팅과 주제가 달라 작성하지 않았지만 네이버 웨일이 크롬과 달리 어떻게 JS엔진에 접근해서 성능을 개선하는지 참조 링크를 걸어두었습니다. 상당히 흥미로우니 한 번 읽어보는 것을 추천합니다. 

### references

- 오세만(2010.08). 컴파일러 입문. 정익사.
- [이성원, 조명진(2017.09). 웨일브라우저의 성능 및 메모리 최적화. NAVER](https://www.slideshare.net/deview/ss-80845715)
- [V8 blog(2018.09). Improving DataView performance in V8](https://v8.dev/blog/dataview)
- [V8 blog(2019.04). Code caching for JavaScript developers](https://v8.dev/blog/code-caching-for-devs)
- [V8 blog(2018.03). Background compilation](https://v8.dev/blog/background-compilation)
- [Google Workspace - V8 Runtime Overview](https://developers.google.com/apps-script/guides/v8-runtime)
- [Ignition Design Doc - Ignition: V8 Interpreter](https://docs.google.com/document/d/11T2CRex9hXxoJwbYqVQ32yIPMh0uouUZLdyrtmMoL44)
- [ECMA-262, 12th edition](https://www.ecma-international.org/wp-content/uploads/ECMA-262_12th_edition_june_2021.pdf)
- [JavaScript AST Explorer](https://astexplorer.net)
- [TypeScript AST Viewer](https://ts-ast-viewer.com)
- [Daniel Rosenwasser(2022.03). A Proposal For Type Syntax in JavaScript](https://devblogs.microsoft.com/typescript/a-proposal-for-type-syntax-in-javascript)
- [Jack Halon(2022.11). Chrome Browser Exploitation, Part 2: Introduction to Ignition, Sparkplug and JIT Compilation via TurboFan](https://jhalon.github.io/chrome-browser-exploitation-2)
- [Fedor Indutny(2015.09). Sea of Nodes](https://darksi.de/d.sea-of-nodes)
- [georg(2015.03). What is the difference between JavaScript Engine and JavaScript Runtime Environment](https://stackoverflow.com/questions/29027845/what-is-the-difference-between-javascript-engine-and-javascript-runtime-environm)

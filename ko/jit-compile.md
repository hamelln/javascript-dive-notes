# 목표
> 크롬의 V8 엔진이 스크립트를 다운로드하는 동안 벌어지는 일에 대해 간단하게 알아봅니다.

컴파일은 고수준의 언어를 저수준 언어로 번역하고 문법이나 타입 체크, 최적화 등의 작업을 합니다. JavaScript는 동적 타입 언어로서 실행 중 최적화를 위해 JIT 컴파일을 합니다.  

정적 타입 언어 : Java, TypeScript처럼 실행 전에 타입을 지정.  
동적 타입 언어 : JavaScript처럼 실행 중에 타입을 결정, 추론.  

### JIT(Just-In-Time)
JS는 runtime에 컴파일됩니다. 웬만한 언어는 다들 컴파일을 합니다. 왜일까요? 똑같은 코드를 10000번 실행한다고 쳤을 때 아래와 같은 차이가 발생하기 때문입니다.  
인터프리터: 똑같은 내용을 처음부터 끝까지 10000번 일일이 읽는다.  
컴파일: 최적화해서 저장해둔 코드 10000번 출력.  

> V8은 JS를 실행할 때 JIT 컴파일을 합니다. 스크립트를 실행하기 직전 파싱 & 컴파일하기 때문에 오버헤드가 많아질 수 있는데요. 	
이를 방지하기 위해 코드를 캐싱합니다. ‌‌스크립트를 처음 컴파일할 땐 캐시 데이터를 만들고 저장합니다. 나중에 V8이 똑같은 스크립트를 컴파일하려고 하면 인스턴스에서 캐시를 가져와서 결과를 출력합니다.  
따라서 스크립트를 더 빠르게 실행할 수 있고요.‌‌..	‌‌
캐시 생성에는 계산과 메모리 비용이 발생하는데요. 그래서 크롬은 이틀 내에 동일한 스크립트를 봤을 때 캐시를 만듭니다. 이렇게 하면 스크립트 파일을 평균 2배 이상 빠른 코드로 바꾸기 때문에 페이지 로딩 시 유저의 시간을 아낄 수 있습니다.  
-V8 블로그-

JIT 컴파일은 실행 속도가 빠릅니다. 이는 메모리 차지가 있음을 뜻하고, 메모리 접근이 상대적으로 쉽다는 뜻이 됩니다. 즉, 보안 이슈가 있습니다.
>‌‌	어떨 땐 메모리 할당 없이 실행하는 게 바람직할 수 있는데요. 일부 플랫폼(iOS, 스마트 TV, 게임 콘솔 등)은 가용 메모리에 접근할 수 없습니다.  
V8을 사용할 수 없단 얘기죠. 가용 메모리 작성을 금지하면 어플 악용을 위한 공격 표면(attack surface)을 줄일 수 있습니다.‌‌..‌  
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

### 어휘 분석(Lexical Analysis)
> “파싱을 위해 낱말로 나누다.”

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

렉서는 문자열을 스캔해서 토큰(문법적으로 유의미한 가장 작은 단위의 말)들로 분류합니다. 자세한 예시로 확인해봅시다.  

```javascript
const a = b / 10 + "a";
```

위 코드는 문자열로 변환 후, 렉서를 통해 아래와 같은 토큰들로 변환됩니다.
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


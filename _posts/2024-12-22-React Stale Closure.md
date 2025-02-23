---
title: React Stale Closure
categories: [ Frontend, React ]
tags: [ frontend, react ]  # TAG names should always be lowercase
toc: true
comments: true
---

> React 버전 18.3에서 작성 및 테스트한 글입니다.
{: .prompt-info }

사내에서 솔루션의 테스트 화면을 개발하던 중에 경험한 Stale Closure 이슈와 해결 방법에 대해 살펴보고자 한다.

<br>

## 문제 상황

문제 상황을 이해하기 위해서는 테스트 화면과 서버의 매커니즘에 대해 알아야 한다.

### 화면 관점

화면에서는 테스트가 종료된 경우 실행 버튼이 표시되고, 테스트를 실행 중인 경우 강제 종료 버튼이 표시된다.
사용자가 실행 버튼을 클릭하면 테스트를 수행할 수 있고, 강제 종료 버튼을 클릭하면 실행 중인 테스트를 강제 종료할 수 있다.

> 단, 이 포스팅의 예제에서는 테스트의 상태와 관계없이 두 버튼을 모두 표시하도록 하였다.
{: .prompt-tip }

### 서버 관점

서버는 사용자로부터 받은 실행, 강제 종료 요청이 유효한지를 검사한 후, 해당 요청에 대한 처리 로직을 수행한다.
처리 로직은 크게 두 가지 부분으로 나뉘며, **(1) 테스트에 대한 정보를 테스트 테이블에 넣거나 업데이트**하는 부분과 **(2) 실제적인 요청에 대한 처리를 수행**하는 부분이다.

**(1) 테스트 테이블에 넣거나 업데이트**: 한 번의 실행은 테스트 테이블에서 하나의 로우로 취급되며, 해당 로우에는 테스트의 상태값이 포함된다.
따라서, 실행 로직에서는 테스트 상태값을 `RUNNING`으로 새로운 로우를 INSERT하며, 강제 종료 로직에서는 앞서 INSERT한 로우를 찾아 테스트 상태값을 `TERMINATE`로 UPDATE한다.

**(2) 실제적인 요청에 대한 처리 수행**: 앞의 처리가 정상적으로 완료되면, 실행이나 강제 종료 등의 요청에 대한 처리가 비동기로 수행된다. 처리가 비동기로 이루어지기 때문에 사용자가 실행 중인 상태를 Near
Real-Time으로 확인할 수 있어야 한다는 요건을 충족하기 위해 화면에서는 상태값을 10초에 한 번씩 주기적으로 폴링한다.
만약 처리 중 File I/O 등과 같은 오류가 발생하면 로우의 테스트 상태값을 `ERROR`로 변경한다.

자, 이제 화면과 서버의 동작에 대해 간단하게 이해했으니, 문제 상황을 재현해보도록 하겠다.

### 상황 재현

상황 재현을 위해 아래와 같은 시나리오를 설정했다.

1. 사용자가 화면에서 실행 버튼을 클릭한다.
2. 서버는 실행 요청을 받고, 해당 요청이 유효한지 검증한 뒤 테이블에 INSERT를 성공적으로 완료하면 200 응답을 리턴한다.
3. 서버가 실제적인 실행 요청에 대한 처리를 비동기적으로 수행하던 중 오류가 발생해 상태값을 `ERROR`로 변경한다.
4. 화면은 사용자에게 Near Real-Time으로 테스트의 상태를 제공하기 위해 최신 상태값을 폴링해온다.
5. 사용자가 화면에서 강제 종료 버튼을 클릭한다.

4, 5번이 거의 동시에 이루어진다고 가정한다. 위 시나리오를 기억하면서 코드를 살펴보자. 어떤 결과가 나올 것이라고 생각하는가? (서버와의 통신은 따로 구현하지 않고 모킹하였다.)

```tsx
import {useState} from "react";
import {Button} from "antd";

interface TestInfo {
  status: string;
  startFile?: string;
  endFile?: string;
}

const defaultTestInfo: TestInfo = {
  status: "",
  startFile: "",
  endFile: ""
}

const StaleClosure = () => {
  const [testInfo, setTestInfo] = useState<TestInfo>(defaultTestInfo);
  const runningState: TestInfo = {status: 'RUNNING'}
  const errorState: TestInfo = {
    status: 'ERROR',
    startFile: 'start-file.json',
    endFile: 'end-file.json'
  };
  const terminateState: TestInfo = {status: 'TERMINATE'};

  const getRunningState = () => {
    return runningState;
  }

  const getTerminateState = () => {
    return terminateState;
  }

  // 서버 요청 모킹을 위한 함수
  const sendRequest = async (getState: any): Promise<any> => {
    let promise = new Promise((resolve, reject) => {
      setTimeout(() => resolve(getState), 1000)
    });
    return await promise;
  }

  function polling() {
    setTestInfo(errorState);
  }

  setTimeout(polling, 3000);

  const startTest = () => {
    const result = sendRequest(getRunningState());
    result.then((result: TestInfo) => {
      setTestInfo({
        ...defaultTestInfo,
        status: result.status
      });
    })
  }

  const terminateTest = async () => {
    const result = sendRequest(getTerminateState());
    result.then(result => {
      // 만약 폴링된 status가 이미 ERROR면 TERMINATE로 변경하지 않도록 조건문 추가
      if (testInfo.status !== 'ERROR') {
        setTestInfo({...testInfo, status: result.status});
      }
    });
  }

  return (
    <>
      <div>
        <div>{`상태: ${testInfo.status}`}</div>
        <div>{`시작 파일: ${testInfo.startFile}`}</div>
        <div>{`종료 파일: ${testInfo.endFile}`}</div>
        <Button onClick={startTest}>테스트 시작</Button>
        <Button onClick={terminateTest}>테스트 종료</Button>
      </div>
    </>
  )
}

export default StaleClosure;
```

사용자는 실행 버튼을 클릭한 뒤 바로 강제 종료 버튼을 클릭했지만, 서버에서는 실행 요청을 처리하는 과정에서 이미 오류가 발생했다고 가정해보자. 아래 두 가지 선택지 중 몇 번이 위 코드를 실행시킨 결과일까?

| 1번                                                                            | 2번                                                                            |
|-------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| ![선택지 1](/assets/gif/241222/React%20Stale%20Closure%20선택지%201.gif){: w="300"} | ![선택지 2](/assets/gif/241222/React%20Stale%20Closure%20선택지%202.gif){: w="300"} |

<br>

정답은…

<br>

바로 **2번**이다. 화면에서 폴링한 상태값이 `ERROR`면 사용자가 강제 종료 요청을 보내도 상태값이 변경되지 않도록 `terminateTest()` 함수 내부에 조건 분기를 추가해두었는데... 왜 실행 결과는 2번을 가리킬까? 
이는 Stale Closure와 관련된 이슈로, 문제 해결 방법으로 넘어가기 전 클로저에 대한 개념부터 살펴보자.

<br>

## 클로저란?

MDN의 공식 문서에 작성된 클로저의 정의는 아래와 같다.

> 주변 상태(어휘적 환경)에 대한 참조와 함께 묶인(포함된) 함수의 조합입니다. 즉, 클로저는 내부 함수에서 외부 함수의 범위에 대한 접근을 제공합니다. JavaScript에서 클로저는 함수 생성 시 함수가 생성될
> 때마다 생성됩니다.


무슨 소린지 잘 모르겠다. 예시를 보자.

```tsx
function makeFunc() {
  const name = "Mozilla";

  function displayName() {
    console.log(name);
  }

  return displayName;
}

const myFunc = makeFunc(); // (1)
myFunc();  // (2)
```

(1)에서 `makeFunc()`은 내부에 선언된 `displayName()`을 리턴하고, (2)에서는 (1)에서 리턴된 `displayName()`을 호출한다. 
(2)의 `displayName()` 함수 자체에는 name 변수가 선언되어 있지 않기 때문에 undefined로 결과가 출력된다고 생각할 수 있지만, Javascript에서는 클로저가 형성되기 때문에 콘솔에서는 Mozilla가 출력된다.

클로저는 함수와 함수가 선언된 어휘적 환경의 조합인데, 여기서 어휘적 환경이란 클로저가 생성된 시점의 유효 범위 내에 있는 모든 지역 변수를 의미한다. 
즉, 예시에서 `myFunc`은 `makeFunc()`이 실행될 때 생성된 `displayName()` 함수의 인스턴스에 대한 참조 뿐만 아니라 해당 시점의 name 변수에 대해서도 참조를 유지한다.

정리하자면, 클로저란 **함수 생성 시점의 함수와 해당 함수에서 사용된 외부 변수들에 대해 사진을 찍어두는 것**이라고 생각하면 된다.

<br>

## Stale Closure

클로저를 이해했으니 Stale Closure에 대해 알아보자. 
Stale Closure는 React Hook을 사용하면 마주칠 수 있는 문제이며, 클로저에서 참조하는 외부 변수에 변화가 생겨도 이를 감지하지 못하고 예전 값을 바라보는 것이다.

### 문제 상황 다시보기

이전의 문제 상황으로 돌아가보자. 화면 상에서 이상했던 점은 크게 두 가지이다.

1. 사용자가 실행 버튼을 누르고 강제 종료 버튼을 눌렀지만, 서버에서는 이미 실행 요청을 처리하던 중 오류가 발생한 상황이므로 정상 동작하는 화면에서는 `RUNNING` -> `ERROR`로 상태값이 표시되어야 한다. 그러나, 화면에서는 `RUNNING` -> `ERROR` -> `TERMINATE` -> `ERROR` 순으로 표시된다. 
2. 화면에서 상태값이 `TERMINATE`인 시점에 시작 파일, 종료 파일이 표시되지 않는다.

![선택지 2](/assets/gif/241222/React%20Stale%20Closure%20선택지%202.gif){: w="300"}

왜 이런 현상이 발생할까? 원인 파악을 위해 실행, 강제 종료, 폴링 시점의 상태값을 확인할 수 있는 코드를 추가하였다. 

```tsx
const StaleClosure = () => {
  const [testInfo, setTestInfo] = useState<TestInfo>(defaultTestInfo);
  const errorState: TestInfo = {
    status: 'ERROR',
    startFile: 'start-file.json',
    endFile: 'end-file.json'
  };
  const terminateState: TestInfo = {status: 'TERMINATE'};

  const sendRequest = async (getState: any): Promise<any> => {
    let promise = new Promise((resolve, reject) => {
      setTimeout(() => resolve(getState), 1000)
    });
    return await promise;
  }

  function polling() {
    console.log("polling");  // 콘솔 로그 출력
    setTestInfo(errorState);
  }

  setTimeout(polling, 3000);

  console.log("current test status is ", testInfo.status);  // 콘솔 로그 출력

  const startTest = () => {
    const result = sendRequest(getRunningState());
    result.then((result: TestInfo) => {
      console.log("start test / current test status is ", testInfo.status);  // 콘솔 로그 출력
      setTestInfo({
        ...defaultTestInfo,
        status: result.status
      });
    })
  }

  const terminateTest = async () => {
    const result = sendRequest(getTerminateState());
    result.then(result => {
      console.log("terminate test / current test status is ", testInfo.status);  // 콘솔 로그 출력
      if (testInfo.status !== 'ERROR') {
        setTestInfo({...testInfo, status: result.status});
      }
    });
  }

  return (
    <>
      <div>
        <div>{`상태: ${testInfo.status}`}</div>
        <div>{`시작 파일: ${testInfo.startFile}`}</div>
        <div>{`종료 파일: ${testInfo.endFile}`}</div>
        <Button onClick={startTest}>테스트 시작</Button>
        <Button onClick={terminateTest}>테스트 종료</Button>
      </div>
    </>
  )
}
```

이제 위 코드로 문제 상황을 동일하게 재현해보자. 이때, 콘솔 창을 열어 화면과 콘솔을 함께 확인해보았다.

![콘솔 출력](/assets/gif/241222/React%20Stale%20Closure%20Console.gif){: w="600"}

사용자가 강제 종료 버튼을 클릭하기 전, 이미 폴링에 의해 최신 상태값인 `ERROR`로 업데이트 된 것으로 보아, 여기까지는 문제가 없다는 것을 알 수 있다. 
그러나, 사용자가 강제 종료 버튼을 클릭하는 시점의 로그를 살펴보면 `ERROR`여야 할 상태값이 `RUNNING`임을 확인할 수 있다. 
즉, **`terminateTest()` 실행 시점에 `testInfo`의 최신 상태(`ERROR`)가 아닌 과거 상태(`RUNNING`)을 바라보기** 때문에 문제가 발생한 것이었다. 
이는 앞에서 살펴본 클로저와 관련되어 있다. 

상태값이 빈 문자열에서 `RUNNING`으로 변하면 컴포넌트가 재렌더링되는데, 이때 생성된 `terminateTest()` 함수는 해당 시점의 `testInfo`를 바라본다. 
이후 폴링으로 인해 상태값이 ERROR로 변경되며 컴포넌트가 재렌더링되지만, 사용자가 재렌더링이 완료되기 직전에 강제 종료 버튼을 클릭했기 때문에 `terminateTest()` 함수는 `RUNNING` 시점의 `testInfo`를 바라보게 되는 것이다.

> 물론, 강제 종료 버튼을 클릭하기 전 시간을 충분히 두면 이런 현상이 발생하지 않는다. 그러나, 사용자가 클릭 시점에 의존해 정상 동작 여부가 결정된다는 점에서 이는 버그이고 수정할 필요가 있다.
{: .prompt-tip }

### 조치 과정 및 결과

자, 그럼 기능이 문제 없이 돌아가도록 하려면 코드를 어떻게 수정해야 할까?
아래와 같이 useState의 set 함수에 함수를 인자로 넘기면 Stale Closure 이슈를 쉽게 해결할 수 있다.

```tsx
// 기존 코드
const terminateTest = async () => {
  const result = sendRequest(getTerminateState());
  result.then(result => {
    if (testInfo.status !== 'ERROR') {
      setTestInfo({...testInfo, status: result.status});
    }
  });
}

// 새로운 코드
const newTerminateTest = async () => {
  const result = sendRequest(getTerminateState());
  result.then(result => {
    setTestInfo(testInfo => {
      if (testInfo.status !== 'ERROR') {
        return {...testInfo, status: result.status}
      }
      return testInfo;
    });
  })
}
```

이렇게 React State를 사용하며 발생할 수 있는 Stale Closure 문제와 해결 방법에 대해 알아보았다. 
결국 클로저가 바라보는 외부 변수의 상태값이 너무 오래되어 발생한 문제였다. 이해를 돕기 위해 다른 Stale Closure 예시를 살펴보는 것으로 포스팅을 마무리 짓도록 하겠다.

### 예시

어떤 결과가 나올 것이라 예상하는가? 화면과 콘솔에 찍힐 결과를 예상해보자.

```tsx
import {useEffect, useState} from "react";

function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setInterval(() => {
      console.log(count);
      setCount(count + 1);
    }, 1000);
  }, []);

  return (
    <h1>{count}</h1>
  )
}

export default App;
```

<br>

결과는 바로…

<br>

![image1.png](/assets/img/241222/image1.png){: w="400"}

화면에서는 0에서 1로 업데이트되는 것을 볼 수 있지만, 콘솔에서는 1초마다 0이 출력된다. 이는 첫 번째 렌더링 시점에 setInterval 함수가 클로저를 형성하기 때문이며, 이로 인해 count는 업데이트되지
않는 상황이 발생한다. 이를 해결하려면 아래와 같이 작성할 수 있다.

```tsx
import {useEffect, useState} from "react";

function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);
      setCount(count + 1);
    }, 1000);

    return () => clearInterval(id);
  }, [count]);

  return (
    <h1>{count}</h1>
  )
}

export default App;
```

![image2.png](/assets/img/241222/image2.png){: w="400"}

<br>

## Reference

- <https://react.dev/reference/react/useState#setstate>
- <https://legacy.reactjs.org/docs/hooks-faq.html#why-am-i-seeing-stale-props-or-state-inside-my-function>
- <https://developer.mozilla.org/ko/docs/Web/JavaScript/Closures>
- <https://javascript.plainenglish.io/stale-closures-in-react-afb0dda37f0b>

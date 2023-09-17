# node.js 멀티 스레딩

## node.js란

node.js는 js 실행하기 위한 런타임 환경

js는 single-thread 언어이기 때문에 js 실행에 있어서 메인 프로그램은 싱글 스레드로 동작하지만, node.js 자체는 multi-thread로 동작

## node.js 아키텍처 및 동작

![Node_Architecture](https://github.com/code-roasters/2023-study/assets/59640337/863ddbd1-3603-497c-a359-f90bec3896b1)

“Single Threaded Event Loop” 아키텍처를 가짐

event-based 플랫폼, node.js에서 일어나는 모든 일들은 이벤트에 대한 반응으로 이루어지며, event loop를 통해 수행됨

event loop는 node.js가 active하는 동안 계속해서 요청을 수신, 처리, 클라이언트에 응답

I/O 작업 요청의 경우, 작업을 system 커널에 오프로딩하여 백그라운드에서 처리 → node.js에서는 i/o작업은 메인 스레드 블라킹 없이 처리(C++로 작성된 libuv 라이브러리를 통해 처리)

커널이 node.js에 작업 완료를 알리면 해당 작업에 대한 적절한 callback을 event queue에 추가

- event loop mechanism
  ![Event_Loop_Mechanism](https://github.com/code-roasters/2023-study/assets/59640337/10420d17-6ac8-4b8d-8f49-0e1bbd8c3bf5)
  - js 프로그램이 실행
    js는 js엔진에 의해 싱글 스레드 방식으로 실행, 실행은 하나의 call stack으로 관리
  - 비동기 api를 callback과 함께 호출
    event loop에 의해 백그라운드에서 수행, 작업이 완료 시 callback 은 callback queue(혹은 event queue) 에 추가됨
    비동기 api는 http request, 파일입출력 등의 i/o 요청을 백그라운드에서 수행할 수 있는 메서드, 브라우저 환경에서는 web api 형태로 제공, node에서는 내부적으로 제공하는 C++ 라이브러리로 제공
  - event loop는 call stack이 비어있으면 event queue에 있는 callback을 callstack에 추가
    메인 스레드에서 callback을 실행할 수 있게 됨

event loop 메커니즘을 통해 I/O 작업을 백그라운드에서 수행하여 main 스레드를 blocking하지 않고 여러 작업들을 concurrent하게 처리할 수 있게 됨

## node.js의 숨겨진 threads

- js 실행을 위한 main thread (1)
- libuv 라이브러리에서 제공하는 4개 threads (4)
  non-blocking I/O 작업들을 처리하기 위해, js를 실행하는 main thread와 별도로 실행됨.
- v8 엔진에 의해 실행되는 2개 threads (2)
  자동 가비지 컬렉션 등을 수행

총 7 treads로 구성

## js에서의 CPU-intensive 작업

js는 싱글 스레드로 실행되지만, node.js는 멀티 스레드로 동작하기 때문에 js를 실행하는 메인 스레드 작업, 백그라운드에서 실행되는 i/o작업을 blocking없이 병렬적으로 처리할 수 있음

하지만, js로 CPU-intensive 작업(complex calculations, image resizing, video compression 등) 실행하게 되면 여전히 blocking 문제 발생, CPU-intensive 작업은 main thread를 블라킹하기 때문에 그 다음에 수행되어야 할 작업들의 실행이 지연됨

이러한 문제를 해결하기 위해 node.js 10.5.0 이상부터 js로 다중 threads를 만들어 병렬로 실행할 수 있는 worker thread 모듈을 지원하고 있음

worker thread 모듈을 사용하여 CPU-intensive 작업을 다른 스레드에 오프로드하여 main 스레드의 blocking을 해결하는 방법 살펴보기

### 1. main 스레드 blocking

```jsx
const express = require("express");

const app = express();
const port = process.env.PORT || 3000;

app.get("/non-blocking/", (req, res) => {
  res.status(200).send("This page is non-blocking");
});

app.get("/blocking", (req, res) => {
  let counter = 0;
  for (let i = 0; i < 20_000_000_000; i++) {
    counter++;
  }
  res.status(200).send(`result is ${counter}`);
});

app.listen(port, () => {
  console.log(`App listening on port ${port}`);
});
```

CPU-intensive 작업을 수행하는 간단한 node.js app, express 프레임 워크를 사용하여 HTTP server 어플리케이션 구축

/blocking, /non-blocking 두 endpoints 가지고 있으며, blocking route는 CPU-intensive 작업을 수행합니다. for loop를 200억번 돌며 count variable 값을 1씩 증가, CPU에서 실행되며, 몇십초 걸립니다.

GET /blocking 요청(1) 후 즉시 GET /non-blocking 요청(2) 시,

> 1. main thread에서 for loop 작업 실행

    main thread는 blocking됨

2. for loop 작업 완료 후 /blocking 요청에 대한 응답 client에게 전송
3. /non-blocking 요쳥에 대한 응답을 client에게 전송
   >

/non-blocking 요청(2)을 보낸 client는 긴 시간을 대기하게 됨

## 2. promise 사용

js에서 CPU-bound 작업의 blocking 문제를 마주하게 되면, js 개발자들은 code를 non-blocking으로 만들기 위해 promise로 전환하는 경우가 종종 있습니다. 그 이유는 readFile()메서드 writeFile( )메서드와 같은 프로미스 기반 non-blocking i/o 메서드 사용에 대한 지식에서 비롯되기 때문인데요.

하지만, 앞에서 언급한 것처럼 i/o작업은 cpu-bound 작업과 다르게 node.js의 숨겨진 스레드를 사용하며, 결국에는 cpu-intensive 작업을 promise로 래핑해도 main 스레드의 blocking 문제는 해결되지 않습니다.

예를 들어, 다음과 같이 promise를 사용해서 CPU-intensive 작업 처리해보겠습니다.

```jsx
const express = require("express");

const app = express();
const port = process.env.PORT || 3000;

app.get("/non-blocking/", (req, res) => {
  res.status(200).send("This page is non-blocking");
});

function calculateCount() {
  return new Promise((resolve, reject) => {
    let counter = 0;
    for (let i = 0; i < 20_000_000_000; i++) {
      counter++;
    }
    resolve(counter);
  });
}

app.get("/blocking", async (req, res) => {
  const counter = await calculateCount();
  res.status(200).send(`result is ${counter}`);
});

app.listen(port, () => {
  console.log(`App listening on port ${port}`);
});
```

for loop 작업을 promise로 래핑하여 calculateCount 함수로 추출하였고, /blocking route가 수행할 callback을 async callback으로 변경하여 promise가 resolve 될 때까지 기다렸다가 결과를 응답하도록 하였습니다.

GET /blocking 요청(1) 후 즉시 GET /non-blocking 요청(2) 시,

> 1. calculateCount 함수가 호출되면서 for loop 작업에 대한 promise 객체를 생성
>    생성 과정에서 for loop 작업이 실행되기 때문에 main thread가 blocking됨
> 2. for loop 작업이 완료되면 promise가 reslove됨
> 3. reslove된 counter 값과 함께 요청에 대한 응답을 client에게 보내는 작업이 event queue에 추가됨
> 4. /non-blocking route가 응답을 client에게 전송
> 5. 이벤트 루프에 의해 event queue에 있는 작업이 main thread에서 실행되어 /blocking 요청에 대한 응답이 client에게 전송

이전 예제의 결과와 다른 점은 /non-blocking 요청에 대한 응답을 먼저 보내고 나서 /blocking 응답을 전송합니다. 하지만, for-loop 작업은 여전히 메인 스레드에서 실행되기 때문에 /non-blocking 요청을 보낸 client는 여전히 긴 시간을 대기하게 됩니다.

즉, promise를 사용해도 CPU-bound 작업을 오프로드하여 non-blocking하게 만들 수 없습니다.

### 3. worker thread 사용

worker-threads 모듈을 사용하면 cpu-intensive task를 다른 스레드로 오프로드하여 메인 스레드를 blocking하는 것을 막을 수 있습니다.

- CPU-intensive 작업을 포함할 worker.js 파일 만들기

  ```jsx
  const { parentPort } = require("worker_threads");

  let counter = 0;
  for (let i = 0; i < 20_000_000_000; i++) {
    counter++;
  }

  parentPort.postMessage(counter);
  ```

  worker.js 파일에 cpu-intensive 작업을 정의하고, worker-thread 모듈에서 제공하는 parentPort class의 postMessage 메서드를 호출하여 cpu-intensive 작업의 결과를 포함하는 message를 전송

- index.js 파일에서 worker-threads 모듈을 사용하여 스레드를 실행하여 메인 스레드와 병렬로 실행하기

  ```jsx
  const express = require("express");
  const { Worker } = require("worker_threads");

  const app = express();
  const port = process.env.PORT || 3000;

  app.get("/non-blocking/", (req, res) => {
    res.status(200).send("This page is non-blocking");
  });

  app.get("/blocking", async (req, res) => {
    const worker = new Worker("./worker.js");
    worker.on("message", (data) => {
      res.status(200).send(`result is ${data}`);
    });
    worker.on("error", (msg) => {
      res.status(404).send(`An error occurred: ${msg}`);
    });
  });

  app.listen(port, () => {
    console.log(`App listening on port ${port}`);
  });
  ```

  - index.js 파일에서 worker-threads 모듈을 사용하여 스레드를 초기화하고 메인 스레드와 병렬로 실행되도록 worker.js 파일에 정의된 작업을 시작
    /blocking route 콜백 내부에서 Worker 인스턴스를 생성하면, 새로운 스레드가 만들어지고 생성된 스레드는 worker instance 생성자 함수에 전달된 path에 정의되어 있는 작업을 다른 multi-core에서 실행합니다.
  - 작업이 완료되면 worker thread는 결과가 포함된 메시지를 메인 스레드로 다시 보냄
    worker instance의 on method를 호출하여 message event를 listening 해주면 이벤트를 수신할 수 있음. worker.js 작업의 결과를 포함하는 message를 수신하게 되면 on 메서드의 callback에 매개 변수로 메세지가 전달되어 실행되고, 응답을 client에게 반환합니다.
    앱 실행 시, CPU-bound 작업은 다른 스레드로 오프로드되고 메인 스레드는 들어오는 모든 요청을 처리하게 됩니다.
    GET /blocking 요청(1) 후 즉시 GET /non-blocking 요청(2) 시, 요청(1)에 대한 응답이 처리되는 것을 기다리지 않고 즉시 요청 (2)에 대한 응답을 받을 수 있습니다.

### 4. 4개 worker thread 사용하여 최적화

> >

CPU-intensive 작업을 4개의 worker thread로 분할하여 /blocking 라우트의 요청 처리 시간을 단축시켜보겠습니다.

200억번 for-loop 작업을 50억번의 for-loop 작업 4개로 분할

각 스레드는 50억번 반복하고 카운터를 1씩 증가시킴, 완료되면 결과가 포함된 메세지를 main 스레드에 전송, main thread가 네 개의 스레드에서 각각 메세지를 받으면 결과를 합쳐서 사용자에게 응답을 전송

(이 접근 방식은 큰 배열을 반복하는 작업에서도 적용해볼 수 있음. 예를 들어 directory에 있는 800개의 이미지 크기를 조정하기 위해 모든 이미지 파일 경로를 포함하는 배열을 만들고, 1번 스레드는 배열 인덱스 0~199, 2번 스레드는 200~399까지, 총 4개의 스레드로 작업을 분할하여 처리할 수 있습니다.)

- four_workers.js

  ```jsx
  const { workerData, parentPort } = require("worker_threads");

  let counter = 0;
  for (let i = 0; i < 20_000_000_000 / workerData.thread_count; i++) {
    counter++;
  }

  parentPort.postMessage(counter);
  ```

  WorkerData 객체를 추출해서 thread_count변수를 사용하여 for-loop 반복 횟수를 정의
  WorkerData는 worker thread가 초기화될 때 메인 스레드에서 전달된 데이터를 포함

- index_four_workers.js

  ```jsx
  const express = require("express");
  const { Worker } = require("worker_threads");

  const app = express();
  const port = process.env.PORT || 3000;
  const THREAD_COUNT = 4;

  function createWorker() {
    return new Promise(function (resolve, reject) {
      const worker = new Worker("./four_workers.js", {
        workerData: { thread_count: THREAD_COUNT },
      });
      worker.on("message", (data) => {
        resolve(data);
      });
      worker.on("error", (msg) => {
        reject(`An error ocurred: ${msg}`);
      });
    });
  }

  app.get("/blocking", async (req, res) => {
    const workerPromises = [];
    for (let i = 0; i < THREAD_COUNT; i++) {
      workerPromises.push(createWorker());
    }
    const thread_results = await Promise.all(workerPromises);
    const total = thread_results.reduce((sum, currValue) => sum + currValue, 0);

    res.status(200).send(`result is ${total}`);
  });

  app.listen(port, () => {
    console.log(`App listening on port ${port}`);
  });
  ```

- 생성할 스레드 수를 THREAD_COUNT 상수에 정의
  나중에 서버에 코어가 더 많아지면, 사용하려는 스레드 수에 맞게 THREAD_COUNT 상수 값을 변경하여 확장할 수 있음.
- createWorker() 함수
  프로미스를 생성, 반환. 프로미스 콜백 내에서는 Worker 인스턴스를 생성하여 새 스레드를 초기화하는 과정에서 첫 번째 인자에 four_workers.js 파일의 파일 경로를 전달. 그런 다음 두 번째 인수로 객체를 전달. 이 객체를 통해 main thread에서 워커 스레드로 데이터를 전달할 수 있음. workerData 객체는 앞서 workers.js 파일에서 참조한 객체.
  worker thread가 메인 스레드로 메시지를 보내면, 프로미스는 데이터를 resolve.
- 스레드 생성
  THREAD_COUNT 만큼 createWorker() 함수를 호출하여 워커 프로미스 배열을 구성.
  Promise.all() 메서드는 배열의 모든 프로미스가 resolve될 때까지 기다림, 모든 프로미스가 resolve되면 thread_results 변수에는 resolve된 값이 포함.
  4개의 워커 스레드로부터 받은 값을 더한 값을 client에게 반환.

GET /blocking 요청시, 응답 속도가 worker thread 1개로 작업을 처리한 경우보다 약 70% 단축.

## node.js 멀티스레딩의 함정

- worker thread는 일반적으로 알려진 스레드와 다름
  - 일반 멀티-스레드 환경
    여러 스레드를 병렬적으로 실행하는 과정엥서 메모리 영역을 통해 같은 state를 공유하기 때문에 race-condition 막기 위한 메모리 관리 필요
  - node.js의 멀티-스레드 환경(worker threads)
    각각의 worker thread 생성은 노드의 V8 자바스크립트 런타임의 격리된 인스턴스를 생성하는 방식으로 작동
    새로운 런타임 환경은 main event loop 에 의존하지 않고 개별 event loop 메커니즘에서 실행
    메인 프로세스와 worker thread간 state sharing 일어나지 않음
    대신, ‘event-based messaging system’을 통해 데이터 공유. 혹은 main 스레드에서 명시적으로 share memory(SharedArrayBuffer)를 선언, 스레드 생성 시점에 전달하여 공유
- worker threads 생성 비용
  process forking보다는 덜 들지만, 일반 멀티-스레드 언어에서 스레드를 생성하는 것보다는 비용이 더 많이 듦.
  생성된 각 worker thread는 V8 JavaScript 엔진의 자체 인스턴스를 실행 → 너무 많은 워커를 사용하면 호스트에서 상당한 리소스를 소비
  즉, 워커 스레드를 생성하는 것은 상대적으로 비용이 많이 드는 작업이기 때문에 가벼운 작업에는 적합하지 않음, CPU-intensive 작업을 병렬으로 처리하는 것과 같이 ‘성능 향상 효과’가 ‘워커 스레드 생성 비용’보다 더 큰 경우에만 worker thread를 사용하는 것이 적합, **[Piscina](https://snyk.io/advisor/npm-package/piscina),** **[Poolifier](https://snyk.io/advisor/npm-package/poolifier)** 라이브러리를 통해 worker threads pool을 재사용하는 방법도 있음
  - i/o작업은 낭비: worker thread 생성 및 관리에 드는 비용보다 node.js 에 내장되어 있는 asynchronous i/o 작업 처리 메커니즘이 훨씬 효율적

## 참고자료

- https://www.digitalocean.com/community/tutorials/how-to-use-multithreading-in-node-js
- https://snyk.io/blog/node-js-multithreading-worker-threads-pros-cons/
- https://soshace.com/advanced-node-js-a-hands-on-guide-to-event-loop-child-process-and-worker-threads-in-node-js/

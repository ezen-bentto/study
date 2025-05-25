# express 를 이용하여 백엔드 예제

node.js + express.js 를 이용하여 백엔드를 해보자

## 개요

1. 기능 기반 모듈 구조 설명 (단일책임 원칙, SRP)

2. yarn 설치 및 라이브러리 세팅

3. Express 세팅

4. mariaDB 연결

5. fnc1 구현(GET 메소드)

6. fnc2 구현(POST 메소드)

7. 에러 핸들러 사용법

### ✅ 참고

- 본 예제는 팀 ezen-bentto의 [node-backend 레포지토리](https://github.com/ezen-bentto/node-backend) 참고용으로 제작된 로직을 기준으로 진행한다.

- 예제에 사용된 핵심 기술 스택은 아래와 같다.
  - typeScript
  - express.js
  - node.js
  - mariaDB

## 1. 기능 기반 모듈 구조 설명 (단일책임 원칙, SRP)

본 프로젝트는 기능별로 폴더를 나누고 파일에 단 **하나**의 역활만 수행한다.

```dos
├── controllers/                     # 요청 핸들러
│   ├── demo/
│   │   ├── fnc1.controller.ts        # demo fnc1 처리
│   │   └── fnc2.controller.ts       # demo fnc2 처리
│   └── demo.controller.ts           # demo 관련 컨트롤러 통합
```

예시로

커뮤니티를 구현하다고 생각해보자

커뮤니티에 필요한 API 가 대충 

- 게시물 보여주는 API 
- 상세글 API 
- 수정 API 

등이 있을 것이다.

그것을 community 폴더에 각각 api 별로 모듈화하고 그것을 

그리고 그것을 `community.controller.ts ` 에서 통합한다.

```ts
import { fnc1 } from '@/controllers/demo/get.controller';
import { fnc2 } from '@/controllers/demo/post.controller';

const DemoController = { fnc1, fnc2 };

export default DemoController;
```

이처럼 최대한 모듈화하여서 진행하도록한다.

목표는 한 파일에 `50`줄을 넘기지 않도록 해보자.

물론 목표일뿐, 50줄 넘어간다고 줄빠따 때릴 생각은 없다

의식하고 코딩하라는 소리다.

## 2. yarn 설치

해당 프로젝트는 yarn 을 이용하여 프로젝트를 진행한다.

### yarn 설치 명령어

```
npm install -g yarn
```

### 라이브러리 세팅

백엔드 세팅은 이미 되어 있기 때문에

[node-backend 레포지토리](https://github.com/ezen-bentto/node-backend) 에서 그대로 clone 하여 의존성 설치 해주면 된다.

```
yarn install
```

이럼 끝

직접 백엔드 세팅한 것은 별도의 글로 게시할 생각이다

## 3. Express 세팅

`node-backend` 프로젝트는 이미 어느정도 세팅이 되어 있어서 로직 구현만 하면 된다.

하지만 Express 이해를 위해 설명하도록 하겠다.

우선 Express 홈페이지 docs 에 있는 예제를 가져왔다.

자세한 설명은 주석을 보면 된다.

```js
// index.js
// Express 모듈을 불러온다.
const express = require('express');

// Express 애플리케이션 객체 생성
const app = express();

// 사용할 포트 번호 지정
const port = 3000;

// 기본 라우트 설정: 루트 경로("/")로 GET 요청이 오면 "Hello World!"를 응답
app.get('/', (req, res) => {
  // 클라이언트에 문자열 응답 전송
  res.send('Hello World!');
});

// 서버 실행: 지정한 포트에서 서버를 실행하고, 실행되면 로그를 출력
app.listen(port, () => {
  console.log(`Example app listening on port ${port}`);
});
```

이것만 하면 실제로 실행이 된다.

물론 우리는 이대로 안 쓴다.

CORS 정책 설정이나 라우터, json, error 설정 등을 해줄 것이다.

그러면 코드가 길어지므로 미들웨어와 route로 나눴고

그것을 app.ts 에서 통합한 다음에

```ts
// app.ts
import express from 'express';
import { registerMiddlewares } from '@/middlewares/index.middleware';
import { notFoundHandler } from '@/middlewares/error.middleware';
import DemoRouter from '@/routes/demo.routes';

const app = express();

// ✅ 1. 공통 미들웨어 설정
registerMiddlewares(app);

// ✅ 2. 라우터 연결
app.use('/api/demo', DemoRouter);

// ✅ 3. 에러 핸들러 등록
app.use(errorHandler);

export default app;
```

이렇게 index.ts 에 넣었다.

```ts
// index.ts
import { env } from 'node:process';
import app from './app';

const PORT = env.port || 4000;

app.listen(PORT, () => {
  console.log(`Server is listening on port : ${PORT}`);
});
```

해당 미들웨어의 경우 (`index.middleware.ts`) 주석을 달아놨다.

**핵심은**

> 미들웨어로 express 설정을 주고 <br>
> 라우터는 주소를 연결한다. <br>
> app.ts 은 이것들을 통합한 파일이다.

## 4. mariaDB 연결

딱히 설명할게 없다.

그냥 이럼 된다.

```ts
// db.config.ts
import mariadb from 'mariadb';
import { ENV } from './env.config';

let pool: mariadb.Pool;

export const getDBConnection = () => {
  if (!pool) {
    pool = mariadb.createPool(ENV.db);
  }
  return pool;
};
```

```ts
// env.config.ts
import dotenv from 'dotenv';
dotenv.config();

export const ENV = {
  //   .... 생략
  db: {
    host: process.env.DB_HOST as string,
    port: Number(process.env.DB_PORT as string),
    user: process.env.DB_USER as string,
    password: process.env.DB_PASSWORD as string,
    database: process.env.DB_DATABASE as string,
  },
  //   .... 생략
};
```

.env 파일을 생성해서 mariaDB 정보를 박아주자

현재 내 .env 파일은 이렇다.

```
# ────────────── 서버 설정 ──────────────
PORT=3000
NODE_ENV=development                # development | production

# ────────────── 데이터베이스 ──────────────
DB_HOST=localhost
DB_PORT=3306
DB_USER=님꺼USER
DB_PASSWORD=님꺼password
DB_DATABASE=demo

# ────────────── JWT 인증 ──────────────
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=1d                  # or 3600s, 7d 등

# ────────────── AWS S3 ──────────────
AWS_REGION=ap-northeast-2
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key
AWS_S3_BUCKET_NAME=your_s3_bucket

# ────────────── 기타 설정 ──────────────
CORS_ORIGIN=http://localhost:5173  # 허용할 프론트엔드 주소
LOG_LEVEL=debug                    # debug | info | warn | error
```

예제를 해보기 위해 DB 에 demo 테이블을 생성해주도록 하자.

```mysql
CREATE TABLE demo (
  id CHAR(36) PRIMARY KEY,
  name VARCHAR(10) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE
);
```

## 5. fnc1 구현(GET 메소드)

솔직히 별거 없고 이미 프로젝트에 죄다 적혀 있다.

그냥 파일 찾기 귀찮을까봐

순서대로 나열한 수준이니 적당히 글만 읽고 쭉쭉 내리면 된다.

### 5-1. route

클라이언트에서 `{{base_url}}/api/demo` 로 요청이 오면

route 에서 주소와 방식(GET, POST) 에 맞게 진행이 된다.

본 예제는 임의로 기능 이름을 get, post 로 지었다.

```ts
// ✅ 1. app.ts 파일 내용
import DemoRouter from '@/routes/demo.routes';
app.use('/api/demo', DemoRouter);
// ---------------------------------------------//
// ✅2. @/routesdemo.routes.ts 파일 내용
import DemoController from '@/controllers/demo.controller';
import { Router } from 'express';

const router = Router();

router.get('/', DemoController.get);
router.post('/', DemoController.post);

export default router;
```

### 5-2. controller

controller, req, res 선언 해주고

serivce 실행해줘서 값을 받는다

해당 값을 `zod` 로 검증해주고 검증 결과에 따라 res 를 반환해주었다.

사실 `fnc1`은 `req`를 아예 안쓰고,

`data` 도 관계형DB 에서 나온 select 값이라 data 검증도 필요 없다.

```ts
// 다른 import 들은 생략함
import { DemoResponseSchema } from '@/schemas/demo.schema';
import { z } from 'zod/v4';

export const get: RequestHandler = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const data = await DemoService.get();
    const parsed = z.array(DemoResponseSchema).safeParse(data);
    if (!parsed.success) {
      throw new AppError(INTERNAL_SERVER_ERROR, StatusCodes.INTERNAL_SERVER_ERROR);
    }

    res.status(StatusCodes.OK).json({ data: parsed.data });
  } catch (err) {
    next(err);
  }
```

**그러나** 외부 시스템에서 데이터를 가져오는 경우 (ex: Raw SQL 쿼리, API 통신, `크롤링` 등) 의 경우에는

`res` 를 검증 해주어야 한다고 한다.

공모전 data, 정책 data 때문에 박아 넣었다.

### 5-3 Zod

Zod는 스키마 선언 및 데이터 검증 라이브러리다.

기본적인 사용법은

```ts
해당스키마.parse(검증값);
```

위의 `fnc1` 의 경우 검증값이 배열형태이기에 이렇게 된다.

```ts
z.array(해당스키마).parse(검증값);
```

하지만 예제는 `safeParse` 를 사용하였는데

`safeParse` 는 검증이 맞으면 `success: true` 를 반환하기에 분기처리하기 쉬워서 `safeParse` 를 사용했다.

또한 `Type` 이나 `interface` 대신 타입 선언 할 수 있다.

본 프로젝트는 웬만한 경우는 type도 zod를 이용하도록 하자.

```ts
import { z } from 'zod/v4';

// post
export const DemoCreateSchema = z.object({
  name: z.string().min(1).max(10),
  email: z.email(),
});

// get
export const DemoResponseSchema = z.object({
  id: z.uuid(),
  name: z.string().min(1).max(10),
  email: z.email(),
});

export type DemoCreate = z.infer<typeof DemoCreateSchema>; // 타입 선언
export type DemoResponse = z.infer<typeof DemoResponseSchema>; // 타입 선언
```

### 5-4 service

```ts
// 다른 import 생략
import { DemoModel } from '@/models/demo.model';
import { DemoResponse } from '@/schemas/demo.schema';

// 반환 타입 명시
export const get = async (): Promise<DemoResponse[]> => {
  const data = await DemoModel.findAll();
  if (data.length === 0) {
    throw new AppError(NOT_FOUND_POSTS, StatusCodes.NOT_FOUND);
  }
  return data;
};
```

zod 로 타입선언한 것을 반환값으로 박아주자

### 5-5 model

```ts
import { getDBConnection } from '@/config/db.config';

export const findAll = async () => {
  const db = getDBConnection();
  const row = await db.query('SELECT id, name, email FROM demo');
  return row;
};
```

이럼 끝

실행해주면 당연히 테이블에 값이 없기에 없다고 뜬다

## 6. fnc2 구현(POST 메소드)

값을 넣어보고 실제 DB에 박히는지 보자

### 6-1. controller

```ts
// import 생략
export const post = async (req: Request, res: Response, next: NextFunction) => {
  const parsed = DemoCreateSchema.safeParse(req.body);
  if (!parsed.success) {
    // 조건절로 error 보낼땐 next
    next(new AppError(BAD_REQUEST_VALUE, StatusCodes.BAD_REQUEST));
    return;
  }

  try {
    await DemoService.post(parsed.data);
    res.status(StatusCodes.CREATED).json({ message: OK_JOIN });
  } catch (err) {
    next(err);
  }
};
```

### 6-2. service

웬만에서는 `zod` 를 쓰자고 했지만

`InsertResult` 는 `interface` 다

이유는 DB 반환값이 DB 데이터 값이 아니라

```
{ affectedRows: 1, insertId: 0n, warningStatus: 0 }
```

로 이렇게 schema 가 없는 경우에는 직접 interface 를 지정해주도록 하자

```ts
// service
import { InsertResult } from '@/types/db/response.type';

export const post = async (data: DemoCreate): Promise<InsertResult> => {
  const res = await DemoModel.create(data);
  if (res.affectedRows !== 1) {
    throw new AppError(INTERNAL_SERVER_ERROR, StatusCodes.INTERNAL_SERVER_ERROR);
  }
  return res;
};
```

```ts
// /types/db/response.type.ts
export interface InsertResult {
  affectedRows: number;
  insertId: number | bigint;
  warningStatus: number;
}
```

### 6-4 model

```ts
import { getDBConnection } from '@/config/db.config';
import { DemoCreate } from '@/schemas/demo.schema';
import { v4 as uuidv4 } from 'uuid';

export const create = async (data: DemoCreate) => {
  const { name, email } = data;
  //  uuid 로 id 값 지정
  const id = uuidv4();

  const db = getDBConnection();
  const result = await db.query('INSERT INTO demo (id, name, email) VALUES (?, ?, ?)', [
    id,
    name,
    email,
  ]);
  return result;
};
```

참고로 예시다. 실제로 uuid 를 id 값으로 넣지 말도록 하자.

포스트맨으로 body에 name 값과 email 값을 지정해 주고

[POST] `{{base_url}}/api/demo` 쏴보도고 직접 들어갔는지 확인해 보자

들어갔을 것이다.

값이 들어간 것을 확인했다면

[GET] `{{base_url}}/api/demo` 을 다시 해보자 해당 값이 나올 것이다.

안나온다면 님이

- URL

- env.config.ts 의 db 설정값이 이상하던가

- DB 에 테이블을 안만들었거나

셋 중 하나임

여기서 더 나아가려면 TEST 도구를 쓰면 좋을 것이다.

하지만 그런건 알아서 해라

## 7. 에러 핸들러 사용법

간단하게 에러 핸들러를 세팅했다.

그냥

```ts
new AppError(
  StatusCodes.NOT_FOUND,
  `error.constant.ts 에 존재하는 에러 변수`
);
```

이렇게 박아서 넣어주면 된다.

근데 메세지 변수가 필요할까 고민 중

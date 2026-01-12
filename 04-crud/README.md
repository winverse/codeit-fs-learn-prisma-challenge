# 2-4. Prisma Client로 CRUD 구현하기 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: 상수 파일 생성

**src/constants/http-status.js 생성:**

```javascript
// HTTP 상태 코드 상수
export const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  NO_CONTENT: 204,
  BAD_REQUEST: 400,
  NOT_FOUND: 404,
  CONFLICT: 409,
  INTERNAL_SERVER_ERROR: 500,
};
```

**src/constants/errors.js 생성:**

```javascript
// Prisma 에러 코드 상수
export const PRISMA_ERROR = {
  UNIQUE_CONSTRAINT: "P2002", // Unique constraint 위반
  RECORD_NOT_FOUND: "P2025", // 레코드를 찾을 수 없음
};

// 에러 메시지 상수
export const ERROR_MESSAGE = {
  // User 관련
  USER_NOT_FOUND: "User not found",
  EMAIL_REQUIRED: "Email is required",
  EMAIL_ALREADY_EXISTS: "Email already exists",
  FAILED_TO_FETCH_USERS: "Failed to fetch users",
  FAILED_TO_FETCH_USER: "Failed to fetch user",
  FAILED_TO_CREATE_USER: "Failed to create user",
  FAILED_TO_UPDATE_USER: "Failed to update user",
  FAILED_TO_DELETE_USER: "Failed to delete user",
};
```

**src/constants/index.js 생성:**

```javascript
export { HTTP_STATUS } from "./http-status.js";
export { PRISMA_ERROR, ERROR_MESSAGE } from "./errors.js";
```

---

### □ 2단계: package.json에 import alias 추가

**package.json의 imports 섹션에 추가:**

```json
{
  "imports": {
    "#generated/*": "./generated/*",
    "#config": "./src/config/config.js",
    "#db/*": "./src/db/*",
    "#constants": "./src/constants/index.js",
    "#repository": "./src/repository/index.js"
  }
}
```

---

### □ 3단계: User Repository 생성

**src/repository/users.repository.js 생성:**

```javascript
import { prisma } from "#db/prisma.js";

function createUser(data) {
  return prisma.user.create({ data });
}

function findUserById(id) {
  return prisma.user.findUnique({
    where: { id: Number(id) },
  });
}

function findAllUsers() {
  return prisma.user.findMany();
}

function updateUser(id, data) {
  return prisma.user.update({
    where: { id: Number(id) },
    data,
  });
}

function deleteUser(id) {
  return prisma.user.delete({
    where: { id: Number(id) },
  });
}

export const userRepository = {
  createUser,
  findUserById,
  findAllUsers,
  updateUser,
  deleteUser,
};
```

**src/repository/index.js 생성:**

```javascript
export { userRepository } from "./users.repository.js";
```

---

### □ 4단계: User Router 생성

**src/routes/users.routes.js 생성:**

```javascript
import express from "express";
import { userRepository } from "#repository";
import {
  HTTP_STATUS,
  PRISMA_ERROR,
  ERROR_MESSAGE,
} from "#constants";

export const userRouter = express.Router();

// GET /api/users - 모든 사용자 조회
userRouter.get("/", async (req, res) => {
  try {
    const users = await userRepository.findAllUsers();
    res.json(users);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_FETCH_USERS });
  }
});

// GET /api/users/:id - 특정 사용자 조회
userRouter.get("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const user = await userRepository.findUserById(id);

    if (!user) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.USER_NOT_FOUND });
    }

    res.json(user);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_FETCH_USER });
  }
});

// POST /api/users - 새 사용자 생성
userRouter.post("/", async (req, res) => {
  try {
    const { email, name } = req.body;

    if (!email) {
      return res
        .status(HTTP_STATUS.BAD_REQUEST)
        .json({ error: ERROR_MESSAGE.EMAIL_REQUIRED });
    }

    const newUser = await userRepository.createUser({
      email,
      name,
    });
    res.status(HTTP_STATUS.CREATED).json(newUser);
  } catch (error) {
    if (error.code === PRISMA_ERROR.UNIQUE_CONSTRAINT) {
      return res.status(HTTP_STATUS.CONFLICT).json({
        error: ERROR_MESSAGE.EMAIL_ALREADY_EXISTS,
      });
    }
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_CREATE_USER });
  }
});

// PATCH /api/users/:id - 사용자 정보 수정
userRouter.patch("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const { email, name } = req.body;

    const updatedUser = await userRepository.updateUser(
      id,
      { email, name }
    );
    res.json(updatedUser);
  } catch (error) {
    if (error.code === PRISMA_ERROR.RECORD_NOT_FOUND) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.USER_NOT_FOUND });
    }
    if (error.code === PRISMA_ERROR.UNIQUE_CONSTRAINT) {
      return res.status(HTTP_STATUS.CONFLICT).json({
        error: ERROR_MESSAGE.EMAIL_ALREADY_EXISTS,
      });
    }
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_UPDATE_USER });
  }
});

// DELETE /api/users/:id - 사용자 삭제
userRouter.delete("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    await userRepository.deleteUser(id);
    res.status(HTTP_STATUS.NO_CONTENT).send();
  } catch (error) {
    if (error.code === PRISMA_ERROR.RECORD_NOT_FOUND) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.USER_NOT_FOUND });
    }
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_DELETE_USER });
  }
});
```

---

### □ 5단계: 라우터 통합

**src/routes/index.js 생성:**

```javascript
import express from "express";
import { userRouter } from "./users.routes.js";

export const router = express.Router();

router.use("/users", userRouter);
```

---

### □ 6단계: server.js 수정

**src/server.js 수정:**

```javascript
import express from "express";
import { prisma } from "#db/prisma.js";
import { config } from "#config";
import { router as apiRouter } from "./routes/index.js";

const app = express();

// JSON 파싱
app.use(express.json());

// API 라우터 등록
app.use("/api", apiRouter);

app.listen(config.PORT, () => {
  console.log(
    `[${config.NODE_ENV}] Server running at http://localhost:${config.PORT}`
  );
});

process.on("SIGINT", async () => {
  await prisma.$disconnect();
  process.exit(0);
});
```

---

### □ 7단계: 서버 실행 및 API 테스트

```bash
npm run dev
```

**API 테스트 (Thunder Client 또는 curl 사용):**

```bash
# 1. 모든 사용자 조회
GET http://localhost:5001/api/users

# 2. 특정 사용자 조회
GET http://localhost:5001/api/users/1

# 3. 새 사용자 생성
POST http://localhost:5001/api/users
Content-Type: application/json

{
  "email": "test@example.com",
  "name": "Test User"
}

# 4. 사용자 정보 수정
PATCH http://localhost:5001/api/users/1
Content-Type: application/json

{
  "name": "Updated Name"
}

# 5. 사용자 삭제
DELETE http://localhost:5001/api/users/1
```

---

## 프로젝트 구조

```
src/
├── server.js
├── config/
│   └── config.js
├── db/
│   └── prisma.js
├── constants/
│   ├── index.js
│   ├── http-status.js
│   └── errors.js
├── repository/
│   ├── index.js
│   └── users.repository.js
└── routes/
    ├── index.js
    └── users.routes.js
```

---

## 주요 개념

### Repository 패턴

```javascript
// Repository: 데이터베이스 로직만
function findUserById(id) {
  return prisma.user.findUnique({
    where: { id: Number(id) },
  });
}

// Router: HTTP 요청/응답만
router.get("/:id", async (req, res) => {
  const user = await userRepository.findUserById(
    req.params.id
  );
  res.json(user);
});
```

### HTTP 상태 코드

- `200 OK`: 성공
- `201 Created`: 생성 성공
- `204 No Content`: 삭제 성공
- `400 Bad Request`: 잘못된 요청
- `404 Not Found`: 찾을 수 없음
- `409 Conflict`: 중복 (이메일 등)
- `500 Internal Server Error`: 서버 에러

### Prisma 에러 코드

- `P2002`: Unique constraint 위반 (이메일 중복)
- `P2025`: 레코드를 찾을 수 없음

---

## 완료 확인

✅ 모든 상수 파일이 생성되었나요?
✅ Repository 파일이 생성되었나요?
✅ Router 파일이 생성되었나요?
✅ 서버가 정상적으로 실행되나요?
✅ API가 정상적으로 작동하나요?

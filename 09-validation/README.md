# 2-9. 에러 핸들링과 유효성 검사 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: Zod 설치

```bash
npm install zod
```

---

### □ 2단계: Validation 에러 상수 추가

**src/constants/errors.js에 추가:**
 
```javascript
export const ERROR_MESSAGE = {
  // ... 기존 에러들 ...

  // Validation 관련 (추가)
  INVALID_INPUT: "Invalid input",
  VALIDATION_FAILED: "Validation failed",
};
```

---

### □ 3단계: 중앙 에러 핸들러 구현/개선

**src/middlewares/error-handler.middleware.js 수정:**

```javascript
import { Prisma } from "#generated/prisma/client.ts";
import { HTTP_STATUS, PRISMA_ERROR } from "#constants";

export const errorHandler = (err, req, res, _next) => {
  console.error(err.stack);

  // Prisma 에러 처리
  if (err instanceof Prisma.PrismaClientKnownRequestError) {
    if (err.code === PRISMA_ERROR.UNIQUE_CONSTRAINT) {
      const field = err.meta?.target?.[0];
      return res.status(HTTP_STATUS.CONFLICT).json({
        success: false,
        message: `${field}가 이미 사용 중입니다.`,
      });
    }

    if (err.code === PRISMA_ERROR.RECORD_NOT_FOUND) {
      return res.status(HTTP_STATUS.NOT_FOUND).json({
        success: false,
        message: "요청한 리소스를 찾을 수 없습니다.",
      });
    }
  }

  res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
    success: false,
    message: "서버 내부 오류가 발생했습니다.",
  });
};
```

---

### □ 4단계: Auth 검증 스키마 생성

**src/routes/auth/auth.schemas.js 생성:**

```javascript
import { z } from "zod";

export const signUpSchema = z.object({
  email: z.email("유효한 이메일 형식이 아닙니다."),
  password: z
    .string({ error: "비밀번호는 필수입니다." })
    .min(6, "비밀번호는 6자 이상이어야 합니다."),
  name: z
    .string()
    .min(2, "이름은 2자 이상이어야 합니다.")
    .optional(),
});

export const loginSchema = z.object({
  email: z.email("유효한 이메일 형식이 아닙니다."),
  password: z
    .string({ error: "비밀번호는 필수입니다." })
    .min(1, "비밀번호를 입력해주세요."),
});
```

---

### □ 5단계: 유효성 검사 미들웨어 생성

**src/middlewares/validation.middleware.js 생성:**

```javascript
import { isProduction } from "#config";
import { flattenError } from "zod";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";

export const validate = (schema) => (req, res, next) => {
  const result = schema.safeParse(req.body);

  if (!result.success) {
    const { fieldErrors } = flattenError(result.error);

    if (isProduction) {
      return res.status(HTTP_STATUS.BAD_REQUEST).json({
        success: false,
        message: ERROR_MESSAGE.INVALID_INPUT,
      });
    }

    return res.status(HTTP_STATUS.BAD_REQUEST).json({
      success: false,
      message: ERROR_MESSAGE.VALIDATION_FAILED,
      details: fieldErrors,
    });
  }

  req.body = result.data;
  next();
};
```

---

### □ 6단계: usersRepository에 이메일 조회 메서드 추가

**src/repository/users.repository.js에 추가:**

```javascript
function findUserByEmail(email) {
  return prisma.user.findUnique({
    where: { email },
  });
}

export const usersRepository = {
  createUser,
  findUserById,
  findAllUsers,
  updateUser,
  deleteUser,
  findUserByEmail, // 추가
};
```

---

### □ 7단계: Auth 라우터에 validate 적용

**src/routes/auth/auth.routes.js 수정:**

```javascript
import express from "express";
import { usersRepository } from "#repository";
import {
  hashPassword,
  comparePassword,
} from "#utils/hash.util.js";
import { generateTokens } from "#utils/jwt.util.js";
import { setAuthCookies } from "#utils/cookie.util.js";
import { validate } from "#middlewares/validation.middleware.js";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";
import {
  signUpSchema,
  loginSchema,
} from "./auth.schemas.js";

export const authsRouter = express.Router();

authsRouter.post(
  "/signup",
  validate(signUpSchema),
  async (req, res, next) => {
    try {
      const { email, password, name } = req.body;

      const hashedPassword = await hashPassword(password);

      const user = await usersRepository.createUser({
        email,
        password: hashedPassword,
        name,
      });

      const tokens = generateTokens(user);
      setAuthCookies(res, tokens);

      const { password: _, ...userWithoutPassword } = user;
      res
        .status(HTTP_STATUS.CREATED)
        .json(userWithoutPassword);
    } catch (error) {
      next(error);
    }
  }
);

authsRouter.post(
  "/login",
  validate(loginSchema),
  async (req, res, next) => {
    try {
      const { email, password } = req.body;

      const user = await usersRepository.findUserByEmail(
        email
      );

      if (!user) {
        return res.status(HTTP_STATUS.UNAUTHORIZED).json({
          error: ERROR_MESSAGE.INVALID_CREDENTIALS,
        });
      }

      const isPasswordValid = await comparePassword(
        password,
        user.password
      );

      if (!isPasswordValid) {
        return res.status(HTTP_STATUS.UNAUTHORIZED).json({
          error: ERROR_MESSAGE.INVALID_CREDENTIALS,
        });
      }

      const tokens = generateTokens(user);
      setAuthCookies(res, tokens);

      const { password: _, ...userWithoutPassword } = user;
      res.status(HTTP_STATUS.OK).json(userWithoutPassword);
    } catch (error) {
      next(error);
    }
  }
);
```

---

### □ 8단계: auth/index.js 생성 + 전체 라우터에 연결

**src/routes/auth/index.js 생성:**

```javascript
import express from "express";
import { authsRouter } from "./auth.routes.js";

export const authRouter = express.Router();

authRouter.use("/", authsRouter);
```

**src/routes/index.js 수정:**

```javascript
import express from "express";
import { userRouter } from "./users/index.js";
import { postRouter } from "./posts/index.js";
import { authRouter } from "./auth/index.js";

export const router = express.Router();

router.use("/users", userRouter);
router.use("/posts", postRouter);
router.use("/auth", authRouter);
```

---

### □ 9단계: server.js에 에러 핸들러 적용 (순서 중요)

**src/server.js 수정:**

```javascript
import express from "express";
import cookieParser from "cookie-parser";
import { config } from "#config";
import { router } from "./routes/index.js";
import { errorHandler } from "./middlewares/error-handler.middleware.js";

const app = express();

app.use(express.json());
app.use(cookieParser());

app.use("/api", router);

// 반드시 라우터 뒤에 위치
app.use(errorHandler);

app.listen(config.PORT, () => {
  console.log(
    `Server is running on http://localhost:${config.PORT}`
  );
});
```

---

## 완료 확인

✅ validate 미들웨어가 400 응답을 반환하나요?
✅ auth signup/login에 validate가 적용됐나요?
✅ Prisma 에러(P2002, P2025)가 중앙 에러 핸들러에서 처리되나요?
✅ server.js에서 에러 핸들러가 라우터 뒤에 있나요?

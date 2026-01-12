# 2-9. 인증 기능 구현 (유틸리티와 미들웨어) 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: 패키지 설치

```bash
npm install jsonwebtoken bcrypt cookie-parser
```

---

### □ 2단계: 환경 변수 추가

**.env 파일에 JWT 시크릿 키 추가:**

```env
# 기존 환경 변수...
DATABASE_URL="postgresql://..."
PORT=5001

# ✅ JWT 시크릿 키 추가 (최소 32자 이상)
JWT_ACCESS_SECRET="your-super-secret-access-key-min-32-chars"
JWT_REFRESH_SECRET="your-super-secret-refresh-key-min-32-chars"
```

**config/config.js 수정:**

```javascript
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z
    .enum(["development", "production", "test"])
    .default("development"),
  PORT: z.coerce
    .number()
    .min(1000)
    .max(65535)
    .default(5001),
  DATABASE_URL: z.url(),
  JWT_ACCESS_SECRET: z.string().min(32), // ✅ 추가
  JWT_REFRESH_SECRET: z.string().min(32), // ✅ 추가
});

const parseEnvironment = () => {
  try {
    return envSchema.parse({
      NODE_ENV: process.env.NODE_ENV,
      PORT: process.env.PORT,
      DATABASE_URL: process.env.DATABASE_URL,
      JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET, // ✅ 추가
      JWT_REFRESH_SECRET: process.env.JWT_REFRESH_SECRET, // ✅ 추가
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      console.error("환경 변수 검증 실패:", error.errors);
    }
    process.exit(1);
  }
};

export const config = parseEnvironment();

export const isDevelopment =
  config.NODE_ENV === "development";
export const isProduction =
  config.NODE_ENV === "production";
export const isTest = config.NODE_ENV === "test";
```

---

### □ 3단계: 시간 상수 정의

**src/constants/time.js 생성:**

```javascript
// 시간 단위 변환 상수
export const SECOND_IN_MS = 1000;
export const MINUTE_IN_MS = 60 * SECOND_IN_MS;
export const HOUR_IN_MS = 60 * MINUTE_IN_MS;
export const DAY_IN_MS = 24 * HOUR_IN_MS;
```

**src/constants/index.js에 export 추가:**

```javascript
export * from "./http-status.js";
export * from "./errors.js";
export * from "./time.js";
```

---

### □ 4단계: 비밀번호 해싱 유틸리티

**src/utils/hash.util.js 생성:**

```javascript
import bcrypt from "bcrypt";

export const hashPassword = async (password) => {
  const saltRounds = 10;
  try {
    const hashed = await bcrypt.hash(password, saltRounds);
    return hashed;
  } catch {
    throw new Error(
      "비밀번호 해싱 중 오류가 발생했습니다."
    );
  }
};

export const comparePassword = async (
  password,
  hashedPassword
) => {
  try {
    return await bcrypt.compare(password, hashedPassword);
  } catch {
    return false;
  }
};
```

---

### □ 5단계: JWT 유틸리티

**src/utils/jwt.util.js 생성:**

```javascript
import jwt from "jsonwebtoken";
import { config } from "#config";

// Access Token 생성 (15분 유효)
export const generateAccessToken = (user) => {
  return jwt.sign(
    {
      userId: user.id,
      name: user.name,
    },
    config.JWT_ACCESS_SECRET,
    {
      expiresIn: "15m",
    }
  );
};

// Refresh Token 생성 (7일 유효)
export const generateRefreshToken = (user) => {
  return jwt.sign(
    { userId: user.id },
    config.JWT_REFRESH_SECRET,
    {
      expiresIn: "7d",
    }
  );
};

// 두 토큰 동시 생성
export const generateTokens = (user) => {
  const accessToken = generateAccessToken(user);
  const refreshToken = generateRefreshToken(user);
  return { accessToken, refreshToken };
};

// 토큰 검증
export const verifyToken = (
  token,
  tokenType = "access"
) => {
  try {
    const secret =
      tokenType === "access"
        ? config.JWT_ACCESS_SECRET
        : config.JWT_REFRESH_SECRET;
    return jwt.verify(token, secret);
  } catch (error) {
    console.error(
      "Token verification error:",
      error.message
    );
    return null;
  }
};
```

---

### □ 6단계: 쿠키 유틸리티

**src/utils/cookie.util.js 생성:**

```javascript
import { MINUTE_IN_MS, DAY_IN_MS } from "#constants";
import { config } from "#config";

// 인증 쿠키 설정
export const setAuthCookies = (res, tokens) => {
  const { accessToken, refreshToken } = tokens;

  res.cookie("accessToken", accessToken, {
    httpOnly: true,
    secure: config.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 15 * MINUTE_IN_MS,
    path: "/",
  });

  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: config.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 7 * DAY_IN_MS,
    path: "/",
  });
};

// 인증 쿠키 삭제
export const clearAuthCookies = (res) => {
  res.clearCookie("accessToken", { path: "/" });
  res.clearCookie("refreshToken", { path: "/" });
};
```

---

### □ 7단계: 상수 업데이트

**src/constants/http-status.js에 UNAUTHORIZED 추가:**

```javascript
export const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  NO_CONTENT: 204,
  BAD_REQUEST: 400,
  UNAUTHORIZED: 401, // ✅ 추가
  NOT_FOUND: 404,
  CONFLICT: 409,
  INTERNAL_SERVER_ERROR: 500,
};
```

**src/constants/errors.js에 인증 에러 추가:**

```javascript
export const ERROR_MESSAGE = {
  // User 관련
  USER_NOT_FOUND: "User not found",
  EMAIL_REQUIRED: "Email is required",
  EMAIL_ALREADY_EXISTS: "Email already exists",
  // ... 기존 코드 ...

  // Auth 관련 (추가)
  NO_AUTH_TOKEN: "No authentication token provided",
  INVALID_TOKEN: "Invalid or expired token",
  USER_NOT_FOUND_FROM_TOKEN: "User not found from token",
  AUTH_FAILED: "Authentication failed",
  INVALID_CREDENTIALS: "Invalid email or password",
};
```

---

### □ 8단계: 인증 미들웨어

**src/middlewares/auth.middleware.js 생성:**

```javascript
import { verifyToken } from "../utils/jwt.util.js";
import { prisma } from "#db";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";

export const authMiddleware = async (req, res, next) => {
  try {
    // 1. 쿠키에서 Access Token 추출
    const { accessToken } = req.cookies;

    if (!accessToken) {
      return res.status(HTTP_STATUS.UNAUTHORIZED).json({
        error: ERROR_MESSAGE.NO_AUTH_TOKEN,
      });
    }

    // 2. 토큰 검증
    const payload = verifyToken(accessToken, "access");

    if (!payload) {
      return res.status(HTTP_STATUS.UNAUTHORIZED).json({
        error: ERROR_MESSAGE.INVALID_TOKEN,
      });
    }

    // 3. 사용자 조회
    const user = await prisma.user.findUnique({
      where: { id: payload.userId },
      select: { id: true, email: true, name: true },
    });

    if (!user) {
      return res.status(HTTP_STATUS.UNAUTHORIZED).json({
        error: ERROR_MESSAGE.USER_NOT_FOUND_FROM_TOKEN,
      });
    }

    // 4. req.user에 사용자 정보 저장
    req.user = user;

    // 5. 다음 미들웨어/핸들러로 진행
    next();
  } catch (error) {
    console.error("인증 미들웨어 오류:", error);
    return res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({
        error: ERROR_MESSAGE.AUTH_FAILED,
      });
  }
};
```

**src/middlewares/index.js에 추가:**

```javascript
export { errorHandler } from "./error-handler.middleware.js";
export { authMiddleware } from "./auth.middleware.js"; // ✅ 추가
```

---

### □ 9단계: Express에 cookie-parser 설정

학생용에서는 `src/app.js` 예시로 설명하지만, 프로젝트 엔트리 파일이 `src/server.js`라면 그 파일에 동일하게 적용하세요.

```javascript
import express from "express";
import { prisma } from "./db/prisma.js";
import { config } from "#config";
import { router as apiRouter } from "./routes/index.js";
import cookieParser from "cookie-parser";

const app = express();

// JSON 파싱
app.use(express.json());

// 쿠키 파싱 (중요!)
app.use(cookieParser());

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

### □ 10단계: 미들웨어 적용 테스트(예시)

아래처럼 특정 라우트에 `authMiddleware`를 붙여서 `req.user`가 설정되는지 확인하세요.

```javascript
router.get("/me", authMiddleware, async (req, res) => {
  res.json({ user: req.user });
});
```

---

## 완료 확인

✅ Access/Refresh Token이 정상 생성되나요?
✅ 쿠키가 `httpOnly`로 설정되나요?
✅ 인증 미들웨어가 `req.user`를 설정하나요?
✅ 토큰이 없거나/유효하지 않으면 401이 반환되나요?

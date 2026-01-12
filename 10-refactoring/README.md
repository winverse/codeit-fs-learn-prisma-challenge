# 2-10. 코드 리팩토링: 커스텀 에러와 검증 개선 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 0단계: Pool + Graceful Shutdown 준비

> 이 단계는 “코드 리팩토링”과 직접 관련은 없지만, 이번 챕터(학생용 2-10)에서 **필수로 선행**하는 준비 단계입니다.

**1) pg 설치**

```bash
npm install pg
```

**2) src/db/prisma.js 수정 (Pool 사용)**

```javascript
import { PrismaClient } from "#generated/prisma/client.ts";
import { PrismaPg } from "@prisma/adapter-pg";
import pg from "pg";
import { config } from "#config";

// ✅ 커넥션 풀 생성
const pool = new pg.Pool({
  connectionString: config.DATABASE_URL,
});

// ✅ Pool을 어댑터에 전달
const adapter = new PrismaPg(pool);

// ✅ Prisma 클라이언트 생성
export const prisma = new PrismaClient({ adapter });
```

**3) src/utils/graceful-shutdown.util.js 생성**

```javascript
/**
 * Graceful shutdown 설정
 */
export const setupGracefulShutdown = (server, prisma) => {
  const shutdown = async (signal) => {
    console.log(
      `\n${signal} 신호를 받았습니다. 서버를 종료합니다...`
    );

    // 1. HTTP 서버 종료 (새 요청 거부, 기존 요청 완료 대기)
    server.close((err) => {
      if (err) {
        console.error("서버 종료 중 에러:", err);
        process.exit(1);
      }
      console.log("서버가 종료되었습니다.");
    });

    try {
      // 2. Prisma 연결 종료 (Pool 포함)
      await prisma.$disconnect();
      console.log("데이터베이스 연결이 종료되었습니다.");
      process.exit(0);
    } catch (error) {
      console.error("데이터베이스 종료 중 에러:", error);
      process.exit(1);
    }
  };

  process.on("SIGINT", () => shutdown("SIGINT"));
  process.on("SIGTERM", () => shutdown("SIGTERM"));
};
```

**4) src/utils/index.js에 export 추가**

```javascript
export {
  hashPassword,
  comparePassword,
} from "./hash.util.js";
export { setAuthCookies } from "./cookie.util.js";
export {
  generateTokens,
  generateAccessToken,
  generateRefreshToken,
  verifyToken,
  shouldRefreshToken,
  refreshTokens,
} from "./jwt.util.js";
export { setupGracefulShutdown } from "./graceful-shutdown.util.js"; // ✅ 추가
```

**5) src/server.js 수정 (setupGracefulShutdown 호출)**

```javascript
import express from "express";
import { prisma } from "./db/prisma.js";
import { config } from "#config";
import { router as apiRouter } from "./routes/index.js";
import cookieParser from "cookie-parser";
import { errorHandler } from "#middlewares";
import { setupGracefulShutdown } from "#utils"; // ✅ 추가

const app = express();

app.use(express.json());
app.use(cookieParser());

app.use("/api", apiRouter);
app.use(errorHandler);

// ✅ 서버 인스턴스를 변수에 저장
const server = app.listen(config.PORT, () => {
  console.log(
    `[${config.NODE_ENV}] Server running at http://localhost:${config.PORT}`
  );
});

// ✅ Graceful shutdown 설정
setupGracefulShutdown(server, prisma);
```

### □ 1단계: 에러 메시지 상수 추가

**src/constants/errors.js에 추가:**

```javascript
export const ERROR_MESSAGE = {
  // 기존 에러들...

  // Validation
  INVALID_INPUT: "Invalid input",
  VALIDATION_FAILED: "Validation failed",

  // 일반 에러 (Exception 기본값) ✅ 추가
  RESOURCE_NOT_FOUND: "리소스를 찾을 수 없습니다.",
  BAD_REQUEST: "잘못된 요청입니다.",
  RESOURCE_CONFLICT: "이미 존재하는 데이터입니다.",
  INTERNAL_SERVER_ERROR: "서버 내부 오류가 발생했습니다.",
};
```

---

### □ 2단계: 커스텀 에러 클래스 생성

**src/exceptions/http.exception.js (기본 클래스) 생성:**

```javascript
export class HttpException extends Error {
  constructor(statusCode, message, details = null) {
    super(message);
    this.statusCode = statusCode;
    this.details = details;
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}
```

**src/exceptions/not-found.exception.js 생성:**

```javascript
import { HttpException } from "./http.exception.js";
import { ERROR_MESSAGE } from "#constants";

export class NotFoundException extends HttpException {
  constructor(
    message = ERROR_MESSAGE.RESOURCE_NOT_FOUND,
    details = null
  ) {
    super(404, message, details);
  }
}
```

**src/exceptions/bad-request.exception.js 생성:**

```javascript
import { HttpException } from "./http.exception.js";
import { ERROR_MESSAGE } from "#constants";

export class BadRequestException extends HttpException {
  constructor(
    message = ERROR_MESSAGE.BAD_REQUEST,
    details = null
  ) {
    super(400, message, details);
  }
}
```

**src/exceptions/unauthorized.exception.js 생성:**

```javascript
import { HttpException } from "./http.exception.js";
import { ERROR_MESSAGE } from "#constants";

export class UnauthorizedException extends HttpException {
  constructor(
    message = ERROR_MESSAGE.UNAUTHORIZED,
    details = null
  ) {
    super(401, message, details);
  }
}
```

**src/exceptions/forbidden.exception.js 생성:**

```javascript
import { HttpException } from "./http.exception.js";
import { ERROR_MESSAGE } from "#constants";

export class ForbiddenException extends HttpException {
  constructor(
    message = ERROR_MESSAGE.FORBIDDEN,
    details = null
  ) {
    super(403, message, details);
  }
}
```

**src/exceptions/conflict.exception.js 생성:**

```javascript
import { HttpException } from "./http.exception.js";
import { ERROR_MESSAGE } from "#constants";

export class ConflictException extends HttpException {
  constructor(
    message = ERROR_MESSAGE.RESOURCE_CONFLICT,
    details = null
  ) {
    super(409, message, details);
  }
}
```

**src/exceptions/index.js 생성:**

```javascript
export { HttpException } from "./http.exception.js";
export { NotFoundException } from "./not-found.exception.js";
export { BadRequestException } from "./bad-request.exception.js";
export { UnauthorizedException } from "./unauthorized.exception.js";
export { ForbiddenException } from "./forbidden.exception.js";
export { ConflictException } from "./conflict.exception.js";
```

---

### □ 2-1단계: #exceptions alias 추가

**package.json에 추가:**

```json
  "imports": {
    // 기존 코드...
    "#middlewares": "./src/middlewares/index.js",
    "#exceptions": "./src/exceptions/index.js"  // 추가
  }
```

---

### □ 3단계: 에러 핸들러 개선

**src/middlewares/error-handler.middleware.js 수정:**

```javascript
import { Prisma } from "#generated/prisma/client.ts";
import {
  HTTP_STATUS,
  PRISMA_ERROR,
  ERROR_MESSAGE,
} from "#constants";
import {
  HttpException,
  UnauthorizedException,
} from "#exceptions";
import jwt from "jsonwebtoken";

export const errorHandler = (err, req, res, _next) => {
  console.error(err.stack);

  // 1. JWT 에러 처리 (✅ 추가)
  if (err instanceof jwt.JsonWebTokenError) {
    const error = new UnauthorizedException(
      ERROR_MESSAGE.INVALID_TOKEN
    );
    return res.status(error.statusCode).json({
      success: false,
      message: error.message,
    });
  }

  // 2. 커스텀 에러 처리 (✅ 추가)
  if (err instanceof HttpException) {
    return res.status(err.statusCode).json({
      success: false,
      message: err.message,
      ...(err.details && { details: err.details }),
    });
  }

  // 3. Prisma 에러 처리 (기존)
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
        message: ERROR_MESSAGE.RESOURCE_NOT_FOUND,
      });
    }
  }

  // 4. 기본 에러 응답
  res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
    success: false,
    message: ERROR_MESSAGE.INTERNAL_SERVER_ERROR,
  });
};
```

---

### □ 4단계: URL 파라미터 검증 미들웨어

**src/middlewares/validation.middleware.js 수정 (validate 함수 확장):**

```javascript
import { isProduction } from "#config";
import { flattenError } from "zod";
import { ERROR_MESSAGE } from "#constants";
import { BadRequestException } from "#exceptions";

/**
 * 범용 검증 미들웨어
 * @param {string} target - 검증할 대상 ('body', 'params', 'query')
 * @param {ZodSchema} schema - Zod 스키마
 */
export const validate = (target, schema) => {
  if (!["body", "query", "params"].includes(target)) {
    throw new Error(
      `[validate middleware] Invalid target: "${target}". Expected "body", "query", or "params".`
    );
  }
  return (req, res, next) => {
    try {
      const result = schema.safeParse(req[target]);

      if (!result.success) {
        const { fieldErrors } = flattenError(result.error);

        if (isProduction) {
          throw new BadRequestException(
            ERROR_MESSAGE.INVALID_INPUT
          );
        }

        throw new BadRequestException(
          ERROR_MESSAGE.VALIDATION_FAILED,
          fieldErrors
        );
      }

      req[target] = result.data;
      next();
    } catch (error) {
      next(error);
    }
  };
};
```

---

### □ 5단계: 스키마 파일 생성 (users/posts/auth)

**src/routes/users/users.schema.js 생성:**

```javascript
import { z } from "zod";

// ID 파라미터 검증 스키마
export const idParamSchema = z.object({
  id: z.coerce.number().int().positive({
    message: "ID는 양수여야 합니다.",
  }),
});

// 사용자 생성 스키마
export const createUserSchema = z.object({
  email: z.email("유효한 이메일 형식이 아닙니다."),
  name: z
    .string()
    .min(2, "이름은 2자 이상이어야 합니다.")
    .optional(),
});

// 사용자 수정 스키마
export const updateUserSchema = z.object({
  email: z
    .email("유효한 이메일 형식이 아닙니다.")
    .optional(),
  name: z
    .string()
    .min(2, "이름은 2자 이상이어야 합니다.")
    .optional(),
});
```

**src/routes/posts/posts.schema.js 생성:**

```javascript
import { z } from "zod";

// ID 파라미터 검증 스키마
export const idParamSchema = z.object({
  id: z.coerce.number().int().positive({
    message: "ID는 양수여야 합니다.",
  }),
});

// 페이지네이션 쿼리 스키마
export const paginationQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce
    .number()
    .int()
    .positive()
    .max(100)
    .default(10),
});

// 검색 쿼리 스키마
export const searchQuerySchema = z.object({
  q: z.string().min(1, "검색어를 입력해주세요."),
});

// 게시글 생성 스키마
export const createPostSchema = z.object({
  title: z.string().min(1, "제목은 필수입니다."),
  content: z.string().optional(),
  published: z.boolean().default(false),
});

// 게시글 수정 스키마
export const updatePostSchema = z.object({
  title: z.string().min(1, "제목은 필수입니다.").optional(),
  content: z.string().optional(),
  published: z.boolean().optional(),
});

// 게시글과 댓글 함께 생성 스키마
export const createPostWithCommentSchema = z.object({
  title: z.string().min(1, "제목은 필수입니다."),
  content: z.string().optional(),
  commentContent: z
    .string()
    .min(1, "댓글 내용은 필수입니다."),
});

// 여러 게시글 일괄 생성 스키마
export const batchCreatePostsSchema = z.object({
  posts: z
    .array(
      z.object({
        title: z.string().min(1, "제목은 필수입니다."),
        content: z.string().optional(),
        published: z.boolean().default(false),
        authorId: z.coerce.number().int().positive({
          message: "작성자 ID는 양수여야 합니다.",
        }),
      })
    )
    .min(1, "최소 1개의 게시글이 필요합니다."),
});
```

**src/routes/auth/auth.schemas.js 수정(또는 생성):**

```javascript
import { z } from "zod";

export const signUpSchema = z.object({
  email: z.email("유효한 이메일 형식이 아닙니다."),
  password: z
    .string({ error: "비밀번호는 필수입니다." })
    .min(6, "비밀번호는 6자 이상이어야 합니다."),
  name: z.string().min(2, "이름은 2자 이상이어야 합니다."),
});

export const loginSchema = z.object({
  email: z.email("유효한 이메일 형식이 아닙니다."),
  password: z
    .string({ error: "비밀번호는 필수입니다." })
    .min(1, "비밀번호를 입력해주세요."),
});

// ID 파라미터 검증 스키마
export const idParamSchema = z.object({
  id: z.coerce.number().int().positive({
    message: "ID는 양수여야 합니다.",
  }),
});
```

---

### □ 6단계: User 라우터 리팩토링

**src/routes/users/users.routes.js 수정:**

```javascript
import express from "express";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";
import { usersRepository } from "#repository";
import { authMiddleware, validate } from "#middlewares";
import {
  createUserSchema,
  idParamSchema,
  updateUserSchema,
} from "./users.schema.js";
import {
  ForbiddenException,
  NotFoundException,
} from "#exceptions";

export const usersRouter = express.Router();

// GET /api/users - 모든 사용자 조회
usersRouter.get("/", async (req, res, next) => {
  try {
    const users = await usersRepository.findAllUsers();
    res.status(HTTP_STATUS.OK).json(users);
  } catch (error) {
    next(error);
  }
});

// GET /api/users/:id - 특정 사용자 조회
usersRouter.get(
  "/:id",
  validate("params", idParamSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;
      const user = await usersRepository.findUserById(id);

      if (!user) {
        throw new NotFoundException(
          ERROR_MESSAGE.USER_NOT_FOUND
        );
      }

      res.status(HTTP_STATUS.OK).json(user);
    } catch (error) {
      next(error);
    }
  }
);

// POST /api/users - 새 사용자 생성
usersRouter.post(
  "/",
  validate("body", createUserSchema),
  async (req, res, next) => {
    try {
      const { email, name } = req.body;

      const newUser = await usersRepository.createUser({
        email,
        name,
      });

      res.status(HTTP_STATUS.CREATED).json(newUser);
    } catch (error) {
      next(error);
    }
  }
);

// PATCH /api/users/:id - 사용자 정보 수정
usersRouter.patch(
  "/:id",
  authMiddleware,
  validate("params", idParamSchema),
  validate("body", updateUserSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;
      const { email, name } = req.body;

      if (req.user.id !== id) {
        throw new ForbiddenException(
          ERROR_MESSAGE.FORBIDDEN_RESOURCE
        );
      }

      const existingUser =
        await usersRepository.findUserById(id);
      if (!existingUser) {
        throw new NotFoundException(
          ERROR_MESSAGE.USER_NOT_FOUND
        );
      }

      const updatedUser = await usersRepository.updateUser(
        id,
        { email, name }
      );

      res.status(HTTP_STATUS.OK).json(updatedUser);
    } catch (error) {
      next(error);
    }
  }
);

// DELETE /api/users/:id - 사용자 삭제
usersRouter.delete(
  "/:id",
  authMiddleware,
  validate("params", idParamSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;

      if (req.user.id !== id) {
        throw new ForbiddenException(
          ERROR_MESSAGE.FORBIDDEN_RESOURCE
        );
      }

      const existingUser =
        await usersRepository.findUserById(id);
      if (!existingUser) {
        throw new NotFoundException(
          ERROR_MESSAGE.USER_NOT_FOUND
        );
      }

      await usersRepository.deleteUser(id);

      res.sendStatus(HTTP_STATUS.NO_CONTENT);
    } catch (error) {
      next(error);
    }
  }
);
```

---

### □ 7단계: Auth 라우터 리팩토링

**src/routes/auth/auth.routes.js 수정:**

```javascript
import express from "express";
import {
  generateTokens,
  setAuthCookies,
  hashPassword,
  comparePassword,
} from "#utils";
import { validate } from "#middlewares";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";
import {
  signUpSchema,
  loginSchema,
} from "./auth.schemas.js";
import { usersRepository } from "#repository";
import { UnauthorizedException } from "#exceptions";

export const authRouter = express.Router();

authRouter.post(
  "/signup",
  validate("body", signUpSchema),
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

authRouter.post(
  "/login",
  validate("body", loginSchema),
  async (req, res, next) => {
    try {
      const { email, password } = req.body;

      const user = await usersRepository.findUserByEmail(
        email
      );

      if (!user) {
        throw new UnauthorizedException(
          ERROR_MESSAGE.INVALID_CREDENTIALS
        );
      }

      const isPasswordValid = await comparePassword(
        password,
        user.password
      );
      if (!isPasswordValid) {
        throw new UnauthorizedException(
          ERROR_MESSAGE.INVALID_CREDENTIALS
        );
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

### □ 8단계: Post 라우터 리팩토링

**src/routes/posts/posts.routes.js 수정:**

```javascript
import express from "express";
import { postRepository } from "#repository";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";
import { authMiddleware, validate } from "#middlewares";
import {
  batchCreatePostsSchema,
  createPostSchema,
  createPostWithCommentSchema,
  idParamSchema,
  paginationQuerySchema,
  searchQuerySchema,
  updatePostSchema,
} from "./posts.schema.js";
import {
  ForbiddenException,
  NotFoundException,
} from "#exceptions";

export const postsRouter = express.Router();

// GET /api/posts - 모든 게시글 조회 (작성자 포함)
postsRouter.get(
  "/",
  validate("query", paginationQuerySchema),
  async (req, res, next) => {
    try {
      const { page, limit } = req.query;

      const result =
        await postRepository.getPostsWithPagination(
          page,
          limit
        );
      res.status(HTTP_STATUS.OK).json(result);
    } catch (error) {
      next(error);
    }
  }
);

// GET /api/posts/search?q=검색어 - 게시글 검색
postsRouter.get(
  "/search",
  validate("query", searchQuerySchema),
  async (req, res, next) => {
    try {
      const { q: search } = req.query;

      const posts = await postRepository.searchPosts(
        search
      );
      res.status(HTTP_STATUS.OK).json({ posts });
    } catch (error) {
      next(error);
    }
  }
);

// GET /api/posts/published - 공개 게시글만 조회
postsRouter.get("/published", async (req, res, next) => {
  try {
    const posts = await postRepository.getPublishedPosts();
    res.status(HTTP_STATUS.OK).json({ posts });
  } catch (error) {
    next(error);
  }
});

// GET /api/posts/:id - 특정 게시글 조회 (작성자 포함)
postsRouter.get(
  "/:id",
  validate("params", idParamSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;
      const post = await postRepository.findPostById(id, {
        author: true,
      });

      if (!post) {
        throw new NotFoundException(
          ERROR_MESSAGE.POST_NOT_FOUND
        );
      }

      res.status(HTTP_STATUS.OK).json(post);
    } catch (error) {
      next(error);
    }
  }
);

// POST /api/posts - 새 게시글 생성
postsRouter.post(
  "/",
  authMiddleware,
  validate("body", createPostSchema),
  async (req, res, next) => {
    try {
      const { title, content, published } = req.body;

      const newPost = await postRepository.createPost({
        title,
        content,
        published: published ?? false,
        authorId: req.user.id,
      });

      res.status(HTTP_STATUS.CREATED).json(newPost);
    } catch (error) {
      next(error);
    }
  }
);

// PATCH /api/posts/:id - 게시글 정보 수정
postsRouter.patch(
  "/:id",
  authMiddleware,
  validate("params", idParamSchema),
  validate("body", updatePostSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;
      const { title, content, published } = req.body;

      const existingPost =
        await postRepository.findPostById(id);
      if (!existingPost) {
        throw new NotFoundException(
          ERROR_MESSAGE.POST_NOT_FOUND
        );
      }

      if (existingPost.authorId !== req.user.id) {
        throw new ForbiddenException(
          ERROR_MESSAGE.FORBIDDEN_RESOURCE
        );
      }

      const updatedPost = await postRepository.updatePost(
        id,
        {
          title,
          content,
          published,
        }
      );

      res.status(HTTP_STATUS.OK).json(updatedPost);
    } catch (error) {
      next(error);
    }
  }
);

// DELETE /api/posts/:id - 게시글 삭제
postsRouter.delete(
  "/:id",
  authMiddleware,
  validate("params", idParamSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;
      const existingPost =
        await postRepository.findPostById(id);
      if (!existingPost) {
        throw new NotFoundException(
          ERROR_MESSAGE.POST_NOT_FOUND
        );
      }

      if (existingPost.authorId !== req.user.id) {
        throw new ForbiddenException(
          ERROR_MESSAGE.FORBIDDEN_RESOURCE
        );
      }

      await postRepository.deletePost(id);
      res.sendStatus(HTTP_STATUS.NO_CONTENT);
    } catch (error) {
      next(error);
    }
  }
);

// POST /api/posts/with-comment - 게시글과 댓글 함께 생성
postsRouter.post(
  "/with-comment",
  authMiddleware,
  validate("body", createPostWithCommentSchema),
  async (req, res, next) => {
    try {
      const { title, content, commentContent } = req.body;

      const result = await postRepository.createWithComment(
        req.user.id,
        { title, content, published: true },
        commentContent
      );

      res.status(HTTP_STATUS.CREATED).json({
        message: "게시글과 댓글이 함께 생성되었습니다.",
        ...result,
      });
    } catch (error) {
      next(error);
    }
  }
);

// POST /api/posts/batch - 여러 게시글 일괄 생성
postsRouter.post(
  "/batch",
  authMiddleware,
  validate("body", batchCreatePostsSchema),
  async (req, res, next) => {
    try {
      const { posts } = req.body;

      const result = await postRepository.createMultiple(
        posts
      );

      res.status(HTTP_STATUS.CREATED).json({
        message: `${result.length}개의 게시글이 생성되었습니다.`,
        posts: result,
      });
    } catch (error) {
      next(error);
    }
  }
);

// DELETE /api/posts/:id/with-comments - 게시글과 댓글 함께 삭제
postsRouter.delete(
  "/:id/with-comments",
  authMiddleware,
  validate("params", idParamSchema),
  async (req, res, next) => {
    try {
      const { id } = req.params;

      const existingPost =
        await postRepository.findPostById(id);
      if (!existingPost) {
        throw new NotFoundException(
          ERROR_MESSAGE.POST_NOT_FOUND
        );
      }

      if (existingPost.authorId !== req.user.id) {
        throw new ForbiddenException(
          ERROR_MESSAGE.FORBIDDEN_RESOURCE
        );
      }

      const result =
        await postRepository.deleteWithComments(id);

      res.json({
        message: "게시글과 댓글이 삭제되었습니다.",
        ...result,
      });
    } catch (error) {
      next(error);
    }
  }
);
```

---

### □ 9단계: JWT 유틸 추가 (자동 갱신)

**src/utils/jwt.util.js에 import 추가:**

```javascript
import jwt from "jsonwebtoken";
import { config } from "#config";
import { MINUTE_IN_SECONDS } from "#constants";
import { usersRepository } from "#repository";
```

**src/utils/jwt.util.js에 함수 추가:**

```javascript
export const shouldRefreshToken = (payload) => {
  if (!payload || !payload.exp) return false;

  const expiresIn =
    payload.exp - Math.floor(Date.now() / 1000);

  return expiresIn < MINUTE_IN_SECONDS * 5 && expiresIn > 0;
};

export const refreshTokens = async (refreshToken) => {
  const payload = verifyToken(refreshToken, "refresh");

  if (!payload) {
    return null;
  }

  const user = await usersRepository.findUserById(
    payload.userId
  );
  if (!user) {
    return null;
  }

  return generateTokens({ id: user.id, name: user.name });
};
```

---

### □ 10단계: Auth 미들웨어 개선

**src/middlewares/auth.middleware.js 수정:**

```javascript
import {
  verifyToken,
  shouldRefreshToken,
  refreshTokens,
  setAuthCookies,
} from "#utils";
import { ERROR_MESSAGE } from "#constants";
import { UnauthorizedException } from "#exceptions";

export const authMiddleware = async (req, res, next) => {
  try {
    const { accessToken, refreshToken } = req.cookies;

    if (!accessToken) {
      throw new UnauthorizedException(
        ERROR_MESSAGE.NO_AUTH_TOKEN
      );
    }

    const payload = verifyToken(accessToken, "access");
    if (!payload) {
      throw new UnauthorizedException(
        ERROR_MESSAGE.INVALID_TOKEN
      );
    }

    req.user = { id: payload.userId };

    if (shouldRefreshToken(payload) && refreshToken) {
      const newTokens = await refreshTokens(refreshToken);
      if (newTokens) {
        setAuthCookies(res, newTokens);
      }
    }

    next();
  } catch (error) {
    next(error);
  }
};
```

---

### □ 11단계: 서버 실행 및 테스트

```bash
npm run dev
```

**테스트:**

```bash
# 1. 잘못된 ID 형식 (문자열)
GET http://localhost:5001/api/users/abc
# 응답: { "success": false, "message": "Validation failed", "details": { "id": [...] } }

# 2. 음수 ID
GET http://localhost:5001/api/users/-1
# 응답: { "success": false, "message": "Validation failed", "details": { "id": ["ID는 양수여야 합니다."] } }

# 3. 존재하지 않는 ID
GET http://localhost:5001/api/users/99999
# 응답: { "success": false, "message": "User not found" }

# 4. 정상 요청
GET http://localhost:5001/api/users/1
# 응답: { "id": 1, "email": "...", "name": "..." }
```

---

## 완료 확인

✅ 커스텀 Exception 클래스가 작동하나요?
✅ URL 파라미터 검증이 작동하나요?
✅ 모든 에러 응답이 일관된 형식인가요?
✅ 잘못된 ID 요청이 사전에 차단되나요?
✅ 코드가 간결해졌나요?

# 2-5. 효율적인 관계 쿼리 (N+1 문제 해결) 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: User Repository에 관계 쿼리 함수 추가 (+ export 이름 변경)

**src/repository/users.repository.js 수정:**

```javascript
import { prisma } from "#db/prisma.js";

// ... 기존 함수들 ...

// 사용자와 게시글 함께 조회
function findUserWithPosts(id) {
  return prisma.user.findUnique({
    where: { id: Number(id) },
    include: {
      posts: true,
    },
  });
}

// 모든 사용자와 게시글 함께 조회
function findAllUsersWithPosts() {
  return prisma.user.findMany({
    include: {
      posts: true,
    },
  });
}

// ✅ userRepository → usersRepository로 변경
export const usersRepository = {
  createUser,
  findUserById,
  findAllUsers,
  updateUser,
  deleteUser,
  findUserWithPosts,
  findAllUsersWithPosts,
};
```

---

### □ 2단계: Post Repository 생성

**src/repository/posts.repository.js 생성:**

```javascript
import { prisma } from "#db/prisma.js";

// 게시글 생성
function createPost(data) {
  return prisma.post.create({
    data,
  });
}

// 특정 게시글 조회
function findPostById(id, include = null) {
  return prisma.post.findUnique({
    where: { id: Number(id) },
    ...(include && { include }),
  });
}

// 모든 게시글 조회
function findAllPosts(include = null) {
  return prisma.post.findMany({
    ...(include && { include }),
  });
}

// 게시글 정보 수정
function updatePost(id, data) {
  return prisma.post.update({
    where: { id: Number(id) },
    data,
  });
}

// 게시글 삭제
function deletePost(id) {
  return prisma.post.delete({
    where: { id: Number(id) },
  });
}

export const postRepository = {
  createPost,
  findPostById,
  findAllPosts,
  updatePost,
  deletePost,
};
```

---

### □ 3단계: Repository export 통합 업데이트

**src/repository/index.js 수정:**

```javascript
export { usersRepository } from "./users.repository.js";
export { postRepository } from "./posts.repository.js";
```

---

### □ 4단계: Post 에러 메시지 상수 추가

**src/constants/errors.js 수정:**

```javascript
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

  // Post 관련
  POST_NOT_FOUND: "Post not found",
  TITLE_REQUIRED: "Title is required",
  AUTHOR_ID_REQUIRED: "Author ID is required",
  FAILED_TO_FETCH_POSTS: "Failed to fetch posts",
  FAILED_TO_FETCH_POST: "Failed to fetch post",
  FAILED_TO_CREATE_POST: "Failed to create post",
  FAILED_TO_UPDATE_POST: "Failed to update post",
  FAILED_TO_DELETE_POST: "Failed to delete post",
  FAILED_TO_FETCH_USER_WITH_POSTS:
    "Failed to fetch user with posts",
};
```

---

### □ 5단계: Post Router 폴더 구조 생성

#### 1) posts 폴더 생성

```bash
mkdir -p src/routes/posts
```

#### 2) Post CRUD 라우터 생성

**src/routes/posts/posts.routes.js 생성:**

```javascript
import express from "express";
import { postRepository } from "#repository";
import {
  HTTP_STATUS,
  PRISMA_ERROR,
  ERROR_MESSAGE,
} from "#constants";

export const postsRouter = express.Router();

// GET /api/posts - 모든 게시글 조회 (작성자 포함)
postsRouter.get("/", async (req, res) => {
  try {
    const posts = await postRepository.findAllPosts({
      author: true,
    });
    res.json(posts);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_FETCH_POSTS });
  }
});

// GET /api/posts/:id - 특정 게시글 조회 (작성자 포함)
postsRouter.get("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const post = await postRepository.findPostById(id, {
      author: true,
    });

    if (!post) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.POST_NOT_FOUND });
    }

    res.json(post);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_FETCH_POST });
  }
});

// POST /api/posts - 새 게시글 생성
postsRouter.post("/", async (req, res) => {
  try {
    const { title, content, published, authorId } =
      req.body;

    if (!title) {
      return res
        .status(HTTP_STATUS.BAD_REQUEST)
        .json({ error: ERROR_MESSAGE.TITLE_REQUIRED });
    }

    if (!authorId) {
      return res
        .status(HTTP_STATUS.BAD_REQUEST)
        .json({ error: ERROR_MESSAGE.AUTHOR_ID_REQUIRED });
    }

    const newPost = await postRepository.createPost({
      title,
      content,
      published: published ?? false,
      authorId: Number(authorId),
    });

    res.status(HTTP_STATUS.CREATED).json(newPost);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_CREATE_POST });
  }
});

// PATCH /api/posts/:id - 게시글 정보 수정
postsRouter.patch("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const { title, content, published } = req.body;

    const updatedPost = await postRepository.updatePost(
      id,
      {
        title,
        content,
        published,
      }
    );

    res.json(updatedPost);
  } catch (error) {
    if (error.code === PRISMA_ERROR.RECORD_NOT_FOUND) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.POST_NOT_FOUND });
    }
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_UPDATE_POST });
  }
});

// DELETE /api/posts/:id - 게시글 삭제
postsRouter.delete("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    await postRepository.deletePost(id);
    res.status(HTTP_STATUS.NO_CONTENT).send();
  } catch (error) {
    if (error.code === PRISMA_ERROR.RECORD_NOT_FOUND) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.POST_NOT_FOUND });
    }
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_DELETE_POST });
  }
});
```

#### 3) posts/index.js 생성 (라우터 통합)

**src/routes/posts/index.js 생성:**

```javascript
import express from "express";
import { postsRouter } from "./posts.routes.js";

export const postRouter = express.Router();

// Post CRUD 라우트 연결
postRouter.use("/", postsRouter);
```

---

### □ 6단계: User 라우터를 폴더 구조로 변경 (+ export 이름 변경)

#### 1) users 폴더 생성 및 파일 이동

```bash
mkdir -p src/routes/users
mv src/routes/users.routes.js src/routes/users/users.routes.js
```

#### 2) users/index.js 생성

**src/routes/users/index.js 생성:**

```javascript
import express from "express";
import { usersRouter } from "./users.routes.js";

export const userRouter = express.Router();

// User CRUD 라우트 연결
userRouter.use("/", usersRouter);
```

#### 3) users.routes.js 수정 (export 및 변수명 변경)

**src/routes/users/users.routes.js 수정:**

```javascript
import express from "express";
import { usersRepository } from "#repository";
import {
  HTTP_STATUS,
  PRISMA_ERROR,
  ERROR_MESSAGE,
} from "#constants";

export const usersRouter = express.Router();

// GET /api/users - 모든 사용자 조회
usersRouter.get("/", async (req, res) => {
  try {
    const users = await usersRepository.findAllUsers();
    res.json(users);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_FETCH_USERS });
  }
});

// GET /api/users/:id - 특정 사용자 조회
usersRouter.get("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const user = await usersRepository.findUserById(id);

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
usersRouter.post("/", async (req, res) => {
  try {
    const { email, name } = req.body;

    if (!email) {
      return res
        .status(HTTP_STATUS.BAD_REQUEST)
        .json({ error: ERROR_MESSAGE.EMAIL_REQUIRED });
    }

    const newUser = await usersRepository.createUser({
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
usersRouter.patch("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const { email, name } = req.body;

    const updatedUser = await usersRepository.updateUser(
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
usersRouter.delete("/:id", async (req, res) => {
  try {
    const { id } = req.params;
    await usersRepository.deleteUser(id);
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

### □ 7단계: Nested Resource (/users/:id/posts) 추가

#### 1) users/posts 폴더 생성

```bash
mkdir -p src/routes/users/posts
```

#### 2) user-posts.routes.js 생성

**src/routes/users/posts/user-posts.routes.js 생성:**

```javascript
import express from "express";
import { usersRepository } from "#repository";
import { HTTP_STATUS, ERROR_MESSAGE } from "#constants";

export const userPostsRouter = express.Router({
  mergeParams: true,
});

// GET /api/users/:id/posts - 사용자와 게시글 함께 조회
userPostsRouter.get("/", async (req, res) => {
  try {
    const { id } = req.params;
    const user = await usersRepository.findUserWithPosts(
      Number(id)
    );

    if (!user) {
      return res
        .status(HTTP_STATUS.NOT_FOUND)
        .json({ error: ERROR_MESSAGE.USER_NOT_FOUND });
    }

    res.json(user);
  } catch (_) {
    res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
      error: ERROR_MESSAGE.FAILED_TO_FETCH_USER_WITH_POSTS,
    });
  }
});
```

#### 3) users/posts/index.js 생성

**src/routes/users/posts/index.js 생성:**

```javascript
import express from "express";
import { userPostsRouter as router } from "./user-posts.routes.js";

export const userPostsRouter = express.Router({
  mergeParams: true,
});

// User의 Posts 라우트 연결
userPostsRouter.use("/", router);
```

#### 4) users/index.js 수정

**src/routes/users/index.js 수정:**

```javascript
import express from "express";
import { usersRouter } from "./users.routes.js";
import { userPostsRouter } from "./posts/index.js";

export const userRouter = express.Router();

// User 기본 CRUD 라우트 연결
userRouter.use("/", usersRouter);

// Nested resource: /users/:id/posts
userRouter.use("/:id/posts", userPostsRouter);
```

---

### □ 8단계: 전체 라우터 통합 업데이트

**src/routes/index.js 수정:**

```javascript
import express from "express";
import { userRouter } from "./users/index.js";
import { postRouter } from "./posts/index.js";

export const router = express.Router();

router.use("/users", userRouter);
router.use("/posts", postRouter);
```

---

### □ 9단계: 서버 실행 및 API 테스트

```bash
npm run dev
npm run seed
```

```http
# 사용자와 게시글 함께 조회 (Nested Resource)
GET http://localhost:5001/api/users/1/posts

# 모든 게시글 조회 (작성자 포함)
GET http://localhost:5001/api/posts

# 특정 게시글 조회 (작성자 포함)
GET http://localhost:5001/api/posts/1
```

---

## 완료 확인

✅ usersRepository에 관계 쿼리가 추가되었나요?
✅ postsRepository/postsRouter가 폴더 구조로 추가되었나요?
✅ users 라우터가 폴더 구조로 변경되었나요?
✅ /api/users/:id/posts가 정상 동작하나요?

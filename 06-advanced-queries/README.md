# 2-6. 고급 쿼리: 필터링, 정렬, 페이지네이션 실습 가이드

아래 체크리스트에 따라 파일을 수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: Post Repository에 고급 쿼리 함수 추가

**src/repository/posts.repository.js 수정:**

```javascript
import { prisma } from "#db/prisma.js";

// ... 기존 CRUD 함수들 ...

// 1. 검색 기능
function searchPosts(search) {
  return prisma.post.findMany({
    where: {
      OR: [
        {
          title: {
            contains: search,
            mode: "insensitive",
          },
        },
        {
          author: {
            name: {
              contains: search,
              mode: "insensitive",
            },
          },
        },
      ],
    },
    orderBy: { createdAt: "desc" },
    include: {
      author: {
        select: { name: true, email: true },
      },
    },
  });
}

// 2. 페이지네이션
async function getPostsWithPagination(
  page = 1,
  limit = 10
) {
  const skip = (page - 1) * limit;

  const [posts, totalCount] = await Promise.all([
    prisma.post.findMany({
      skip,
      take: limit,
      orderBy: { createdAt: "desc" },
      include: {
        author: {
          select: { name: true, email: true },
        },
      },
    }),
    prisma.post.count(),
  ]);

  return {
    posts,
    pagination: {
      currentPage: page,
      totalPages: Math.ceil(totalCount / limit),
      totalCount,
      hasNext: page < Math.ceil(totalCount / limit),
      hasPrev: page > 1,
    },
  };
}

// 3. 공개 게시글만 조회
function getPublishedPosts() {
  return prisma.post.findMany({
    where: { published: true },
    orderBy: { createdAt: "desc" },
    include: {
      author: {
        select: { name: true, email: true },
      },
    },
  });
}

export const postRepository = {
  createPost,
  findPostById,
  findAllPosts,
  updatePost,
  deletePost,
  searchPosts,
  getPostsWithPagination,
  getPublishedPosts,
};
```

---

### □ 2단계: 에러 상수 추가

**src/constants/errors.js 수정:**

```javascript
// Prisma 에러 코드 상수
export const PRISMA_ERROR = {
  UNIQUE_CONSTRAINT: "P2002",
  RECORD_NOT_FOUND: "P2025",
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

  // Post 관련
  POST_NOT_FOUND: "Post not found",
  TITLE_REQUIRED: "Title is required",
  AUTHOR_ID_REQUIRED: "Author ID is required",
  SEARCH_QUERY_REQUIRED: "Search query is required",
  FAILED_TO_FETCH_POSTS: "Failed to fetch posts",
  FAILED_TO_FETCH_POST: "Failed to fetch post",
  FAILED_TO_CREATE_POST: "Failed to create post",
  FAILED_TO_UPDATE_POST: "Failed to update post",
  FAILED_TO_DELETE_POST: "Failed to delete post",
  FAILED_TO_SEARCH_POSTS: "Failed to search posts",
  FAILED_TO_FETCH_PUBLISHED_POSTS:
    "Failed to fetch published posts",
  FAILED_TO_FETCH_USER_WITH_POSTS:
    "Failed to fetch user with posts",
};
```

---

### □ 3단계: posts 라우터에 엔드포인트 추가 (+ 라우트 순서 주의)

**src/routes/posts/posts.routes.js 수정:**

```javascript
import express from "express";
import { postRepository } from "#repository";
import {
  HTTP_STATUS,
  PRISMA_ERROR,
  ERROR_MESSAGE,
} from "#constants";

export const postsRouter = express.Router();

// GET /api/posts - 모든 게시글 조회 (페이지네이션)
postsRouter.get("/", async (req, res) => {
  try {
    const page = Number(req.query.page) || 1;
    const limit = Number(req.query.limit) || 10;

    const result =
      await postRepository.getPostsWithPagination(
        page,
        limit
      );
    res.json(result);
  } catch (_) {
    res
      .status(HTTP_STATUS.INTERNAL_SERVER_ERROR)
      .json({ error: ERROR_MESSAGE.FAILED_TO_FETCH_POSTS });
  }
});

// GET /api/posts/search - 게시글 검색 (⚠️ /:id보다 위에)
postsRouter.get("/search", async (req, res) => {
  try {
    const { q: search } = req.query;

    if (!search) {
      return res.status(HTTP_STATUS.BAD_REQUEST).json({
        error: ERROR_MESSAGE.SEARCH_QUERY_REQUIRED,
      });
    }

    const posts = await postRepository.searchPosts(search);
    res.json({ posts });
  } catch (_) {
    res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
      error: ERROR_MESSAGE.FAILED_TO_SEARCH_POSTS,
    });
  }
});

// GET /api/posts/published - 공개 게시글만 조회 (⚠️ /:id보다 위에)
postsRouter.get("/published", async (req, res) => {
  try {
    const posts = await postRepository.getPublishedPosts();
    res.json({ posts });
  } catch (_) {
    res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
      error: ERROR_MESSAGE.FAILED_TO_FETCH_PUBLISHED_POSTS,
    });
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

---

### □ 4단계: 서버 실행 및 API 테스트

```bash
npm run dev
```

```http
# 1) 페이지네이션 (GET /api/posts?page=1&limit=10)
GET http://localhost:5001/api/posts?page=1&limit=10

# 2) 게시글 검색 (GET /api/posts/search?q=JavaScript)
GET http://localhost:5001/api/posts/search?q=JavaScript

# 3) 공개 게시글만 조회
GET http://localhost:5001/api/posts/published
```

---

## 완료 확인

✅ postRepository에 검색/페이지네이션/공개글 함수가 추가되었나요?
✅ postsRouter에 /search, /published가 /:id보다 위에 있나요?
✅ GET /api/posts가 페이지네이션 응답(posts + pagination)으로 동작하나요?

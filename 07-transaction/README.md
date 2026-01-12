# 2-8. (심화) 트랜잭션(Transactions) 처리 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: Comment 모델 추가

**prisma/schema.prisma 수정:**

```prisma
model User {
  id        Int       @id @default(autoincrement())
  email     String    @unique
  name      String?
  posts     Post[]
  comments  Comment[] // 추가
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Post {
  id        Int       @id @default(autoincrement())
  title     String
  content   String?
  published Boolean   @default(false)
  author    User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId  Int
  comments  Comment[] // 추가
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId  Int
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId    Int
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

**마이그레이션 + Prisma Client 재생성:**

```bash
npm run prisma:migrate
# 마이그레이션 이름: add_comment_model

npm run prisma:generate
```

---

### □ 2단계: 에러 메시지 상수 추가

**src/constants/errors.js 수정(추가):**

```javascript
export const ERROR_MESSAGE = {
  // ... 기존 에러들 ...

  // 트랜잭션 관련 (추가)
  FAILED_TO_DELETE_POST_WITH_COMMENTS:
    "Failed to delete post with comments",
  FAILED_TO_CREATE_POST_WITH_COMMENT:
    "Failed to create post with comment",
  FAILED_TO_CREATE_MULTIPLE_POSTS:
    "Failed to create multiple posts",
  POSTS_ARRAY_REQUIRED: "Posts array is required",
  INVALID_POSTS_ARRAY: "Posts must be an array",

  // Comment 관련 (추가)
  COMMENT_NOT_FOUND: "Comment not found",
  COMMENT_CONTENT_REQUIRED: "Comment content is required",
  FAILED_TO_CREATE_COMMENT: "Failed to create comment",
  FAILED_TO_DELETE_COMMENT: "Failed to delete comment",
};
```

---

### □ 3단계: Post Repository에 트랜잭션 함수 추가

**src/repository/posts.repository.js 수정(함수 추가):**

```javascript
import { prisma } from "#db/prisma.js";

// ... 기존 CRUD 함수들 ...

// 트랜잭션: 게시글과 댓글 안전하게 삭제
async function deleteWithComments(postId) {
  return await prisma.$transaction(async (tx) => {
    // 1. 댓글 수 확인 (로깅용)
    const commentCount = await tx.comment.count({
      where: { postId: Number(postId) },
    });

    // 2. 댓글 삭제
    await tx.comment.deleteMany({
      where: { postId: Number(postId) },
    });

    // 3. 게시글 삭제
    const deletedPost = await tx.post.delete({
      where: { id: Number(postId) },
    });

    return {
      deletedPost,
      deletedCommentsCount: commentCount,
    };
  });
}

// 트랜잭션: 게시글과 첫 댓글 함께 생성
async function createWithComment(
  authorId,
  postData,
  commentContent
) {
  return await prisma.$transaction(async (tx) => {
    // 1. 게시글 생성
    const post = await tx.post.create({
      data: {
        ...postData,
        authorId: Number(authorId),
      },
    });

    // 2. 첫 댓글 생성
    const comment = await tx.comment.create({
      data: {
        content: commentContent,
        authorId: Number(authorId),
        postId: post.id,
      },
    });

    return {
      post,
      comment,
    };
  });
}

// 트랜잭션: 여러 게시글 일괄 생성 (배치 작업)
async function createMultiple(posts) {
  return await prisma.$transaction(
    posts.map((post) =>
      prisma.post.create({
        data: {
          title: post.title,
          content: post.content,
          published: post.published ?? false,
          authorId: Number(post.authorId),
        },
      })
    )
  );
}

export const postRepository = {
  // ... 기존 export들 ...
  deleteWithComments,
  createWithComment,
  createMultiple,
};
```

---

### □ 4단계: Post Router에 트랜잭션 엔드포인트 추가

> 💡 `/with-comment`, `/batch` 라우트는 `/:id`보다 **위에** 정의해야 합니다.

**src/routes/posts/posts.routes.js 수정(엔드포인트 추가):**

```javascript
import express from "express";
import { postRepository } from "#repository";
import {
  HTTP_STATUS,
  PRISMA_ERROR,
  ERROR_MESSAGE,
} from "#constants";

export const postsRouter = express.Router();

// ... 기존 CRUD 엔드포인트들 ...

// POST /api/posts/with-comment - 게시글과 댓글 함께 생성
postsRouter.post("/with-comment", async (req, res) => {
  try {
    const { authorId, title, content, commentContent } =
      req.body;

    if (!authorId) {
      return res.status(HTTP_STATUS.BAD_REQUEST).json({
        error: ERROR_MESSAGE.AUTHOR_ID_REQUIRED,
      });
    }

    if (!title) {
      return res
        .status(HTTP_STATUS.BAD_REQUEST)
        .json({ error: ERROR_MESSAGE.TITLE_REQUIRED });
    }

    if (!commentContent) {
      return res.status(HTTP_STATUS.BAD_REQUEST).json({
        error: ERROR_MESSAGE.COMMENT_CONTENT_REQUIRED,
      });
    }

    const result = await postRepository.createWithComment(
      authorId,
      { title, content, published: true },
      commentContent
    );

    res.status(HTTP_STATUS.CREATED).json({
      message: "게시글과 댓글이 함께 생성되었습니다.",
      ...result,
    });
  } catch (_) {
    res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
      error:
        ERROR_MESSAGE.FAILED_TO_CREATE_POST_WITH_COMMENT,
    });
  }
});

// POST /api/posts/batch - 여러 게시글 일괄 생성
postsRouter.post("/batch", async (req, res) => {
  try {
    const { posts } = req.body;

    if (!posts) {
      return res.status(HTTP_STATUS.BAD_REQUEST).json({
        error: ERROR_MESSAGE.POSTS_ARRAY_REQUIRED,
      });
    }

    if (!Array.isArray(posts)) {
      return res.status(HTTP_STATUS.BAD_REQUEST).json({
        error: ERROR_MESSAGE.INVALID_POSTS_ARRAY,
      });
    }

    const result = await postRepository.createMultiple(
      posts
    );

    res.status(HTTP_STATUS.CREATED).json({
      message: `${result.length}개의 게시글이 생성되었습니다.`,
      posts: result,
    });
  } catch (_) {
    res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
      error: ERROR_MESSAGE.FAILED_TO_CREATE_MULTIPLE_POSTS,
    });
  }
});

// DELETE /api/posts/:id/with-comments - 게시글과 댓글 함께 삭제
postsRouter.delete(
  "/:id/with-comments",
  async (req, res) => {
    try {
      const { id } = req.params;
      const result =
        await postRepository.deleteWithComments(id);

      res.json({
        message: "게시글과 댓글이 삭제되었습니다.",
        ...result,
      });
    } catch (error) {
      if (error.code === PRISMA_ERROR.RECORD_NOT_FOUND) {
        return res
          .status(HTTP_STATUS.NOT_FOUND)
          .json({ error: ERROR_MESSAGE.POST_NOT_FOUND });
      }
      res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json({
        error:
          ERROR_MESSAGE.FAILED_TO_DELETE_POST_WITH_COMMENTS,
      });
    }
  }
);
```

---

### □ 5단계: 시드에 Comment 데이터 추가

**scripts/seed.js 수정(전체):**

```javascript
import { PrismaClient } from "#generated/prisma/client.ts";
import { PrismaPg } from "@prisma/adapter-pg";
import { faker } from "@faker-js/faker";

const NUM_USERS_TO_CREATE = 5;

const xs = (n) =>
  Array.from({ length: n }, (_, i) => i + 1);

const pickRandom = (array) =>
  array[
    faker.number.int({ min: 0, max: array.length - 1 })
  ];

const makeUserInput = () => ({
  email: faker.internet.email(),
  name: faker.person.fullName(),
});

const makePostInputsForUser = (userId, count) =>
  xs(count).map(() => ({
    title: faker.lorem.sentence({ min: 3, max: 8 }),
    content: faker.lorem.paragraphs(
      { min: 2, max: 5 },
      "\n\n"
    ),
    published: faker.datatype.boolean(),
    authorId: userId,
  }));

const makeCommentInputsForPost = (postId, users, count) =>
  xs(count).map(() => ({
    content: faker.lorem.sentence({ min: 1, max: 3 }),
    postId,
    authorId: pickRandom(users).id,
  }));

// transaction
const resetDb = (prisma) =>
  prisma.$transaction([
    prisma.comment.deleteMany(),
    prisma.post.deleteMany(),
    prisma.user.deleteMany(),
  ]);

const seedUsers = async (prisma, count) => {
  const data = xs(count).map(makeUserInput);

  return await prisma.user.createManyAndReturn({
    data,
    select: { id: true },
  });
};

const seedPosts = async (prisma, users) => {
  const data = users
    .map((u) => ({
      id: u.id,
      count: faker.number.int({ min: 2, max: 5 }),
    }))
    .flatMap(({ id, count }) =>
      makePostInputsForUser(id, count)
    );

  return await prisma.post.createManyAndReturn({
    data,
    select: { id: true },
  });
};

const seedComments = async (prisma, posts, users) => {
  const data = posts.flatMap((post) => {
    const commentCount = faker.number.int({
      min: 1,
      max: 4,
    });
    return makeCommentInputsForPost(
      post.id,
      users,
      commentCount
    );
  });

  await prisma.comment.createMany({ data });
  return data.length;
};

async function main(prisma) {
  if (process.env.NODE_ENV !== "development") {
    throw new Error(
      "⚠️  프로덕션 환경에서는 시딩을 실행하지 않습니다"
    );
  }

  if (!process.env.DATABASE_URL?.includes("localhost")) {
    throw new Error(
      "⚠️  localhost 데이터베이스에만 시딩을 실행할 수 있습니다"
    );
  }

  console.log("🌱 시딩 시작...");

  await resetDb(prisma);
  console.log("✅ 기존 데이터 삭제 완료");

  const users = await seedUsers(
    prisma,
    NUM_USERS_TO_CREATE
  );
  console.log(
    `✅ ${users.length}명의 유저가 생성되었습니다`
  );

  const posts = await seedPosts(prisma, users);
  console.log(
    `✅ ${posts.length}개의 게시글이 생성되었습니다`
  );

  const commentCount = await seedComments(
    prisma,
    posts,
    users
  );
  console.log(
    `✅ ${commentCount}개의 댓글이 생성되었습니다`
  );

  console.log("✅ 데이터 시딩 완료");
}

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL,
});

const prisma = new PrismaClient({ adapter });

main(prisma)
  .catch((e) => {
    console.error("❌ 시딩 에러:", e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

```bash
npm run seed
```

---

### □ 6단계: API 테스트

```http
# 게시글 + 댓글 함께 생성
POST http://localhost:5001/api/posts/with-comment
Content-Type: application/json

{
  "authorId": 1,
  "title": "트랜잭션 테스트",
  "content": "게시글 내용",
  "commentContent": "첫 댓글입니다!"
}

# 게시글 + 댓글 함께 삭제
DELETE http://localhost:5001/api/posts/1/with-comments
```

---

## 완료 확인

✅ Comment 모델/마이그레이션/Client 재생성이 완료되었나요?
✅ postRepository에 트랜잭션 함수 3개가 추가되었나요?
✅ postsRouter에 /with-comment, /batch, /:id/with-comments가 추가되었나요?
✅ scripts/seed.js로 댓글 데이터까지 시딩되나요?

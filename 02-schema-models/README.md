# 2-2. Prisma 스키마 모델과 관계 정의 실습 가이드

아래 체크리스트에 따라 파일을 수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: User 모델 추가

**prisma/schema.prisma 수정:**

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

---

### □ 2단계: Post 모델 추가

**prisma/schema.prisma에 추가:**

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId  Int
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

> 💡 `onDelete: Cascade`: User 삭제 시 관련 Post도 자동 삭제

---

### □ 3단계: 마이그레이션 생성 및 적용

```bash
npm run prisma:migrate
```

**프롬프트에서 마이그레이션 이름 입력:**

```
? Enter a name for the new migration: › init
```

**실행 결과 확인:**

```
✔ Enter a name for the new migration: … init
Applying migration `20251231081058_init`

The following migration(s) have been created and applied from new schema changes:

prisma/migrations/
  └─ 20251231081058_init/
    └─ migration.sql
```

---

### □ 4단계: Prisma Studio로 테이블 확인

```bash
npm run prisma:studio
```

**브라우저에서 확인:**

- 터미널에 표시된 URL 접속 (예: http://localhost:51212)
- User, Post 테이블이 생성되었는지 확인

---

## 완성된 스키마

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId  Int
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

---

## 완료 확인

✅ User, Post 모델이 스키마에 추가되었나요?
✅ 마이그레이션이 성공적으로 적용되었나요?
✅ Prisma Studio에서 두 테이블이 보이나요?

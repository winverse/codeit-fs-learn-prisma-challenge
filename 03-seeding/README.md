# 2-3. 마이그레이션과 시딩 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: Faker.js 설치

```bash
npm install -D @faker-js/faker
```

---

### □ 2단계: seed.js 파일 생성

**scripts/seed.js 생성:**

```javascript
import { PrismaClient } from "#generated/prisma/client.ts";
import { PrismaPg } from "@prisma/adapter-pg";
import { faker } from "@faker-js/faker";

const NUM_USERS_TO_CREATE = 5;

const xs = (n) =>
  Array.from({ length: n }, (_, i) => i + 1);

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
    authorId: userId,
  }));

const resetDb = (prisma) =>
  prisma.$transaction([
    prisma.post.deleteMany(),
    prisma.user.deleteMany(),
  ]);

const seedUsers = async (prisma, count) => {
  const data = xs(count).map(makeUserInput);
  const emails = data.map((u) => u.email);

  await prisma.user.createMany({ data });
  return prisma.user.findMany({
    where: { email: { in: emails } },
    select: { id: true },
  });
};

const seedPosts = async (prisma, users) => {
  const data = users
    .map((u) => ({
      id: u.id,
      count: faker.number.int({ min: 1, max: 3 }),
    }))
    .flatMap(({ id, count }) =>
      makePostInputsForUser(id, count)
    );

  await prisma.post.createMany({ data });
};

async function main(prisma) {
  if (process.env.NODE_ENV !== "development") {
    throw new Error(
      "⚠️  프로덕션 환경에서는 시딩을 실행하지 않습니다"
    );
  }

  console.log("🌱 시딩 시작...");

  await resetDb(prisma);
  console.log("✅ 기존 데이터 삭제 완료");

  const users = await seedUsers(
    prisma,
    NUM_USERS_TO_CREATE
  );
  await seedPosts(prisma, users);

  console.log(
    `✅ ${users.length}명의 유저가 생성되었습니다`
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

---

### □ 3단계: package.json에 스크립트 추가

**package.json의 scripts 섹션에 추가:**

```json
{
  "scripts": {
    "dev": "nodemon --env-file=./env/.env.development src/server.js",
    "prod": "node --env-file=./env/.env.production src/server.js",
    "prisma:migrate": "dotenv -e ./env/.env.development -- npx prisma migrate dev",
    "prisma:studio": "dotenv -e ./env/.env.development -- npx prisma studio",
    "prisma:generate": "dotenv -e ./env/.env.development -- npx prisma generate",
    "seed": "node --env-file=./env/.env.development scripts/seed.js",
    "format": "npx prettier --write .",
    "format:check": "npx prettier --check ."
  }
}
```

---

### □ 4단계: 시딩 실행

```bash
npm run seed
```

**실행 결과 확인:**

```
🌱 시딩 시작...
✅ 기존 데이터 삭제 완료
✅ 5명의 유저가 생성되었습니다
✅ 데이터 시딩 완료
```

---

### □ 5단계: Prisma Studio로 데이터 확인

```bash
npm run prisma:studio
```

**확인 사항:**

- User 테이블에 5명의 사용자가 생성되었는지 확인
- Post 테이블에 각 사용자의 게시글이 생성되었는지 확인
- User를 클릭하면 해당 사용자의 posts 관계 확인 가능

---

## 완료 확인

✅ Faker.js가 설치되었나요?
✅ seed.js 파일이 생성되었나요?
✅ package.json에 seed 스크립트가 추가되었나요?
✅ 시딩이 성공적으로 실행되었나요?
✅ Prisma Studio에서 데이터를 확인할 수 있나요?

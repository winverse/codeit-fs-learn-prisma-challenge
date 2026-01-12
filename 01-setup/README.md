# 2-1. Setup 실습 가이드

아래 체크리스트에 따라 파일을 생성/수정하고, 코드 블록을 그대로 작성하세요.

---

## 체크리스트

### □ 1단계: 프로젝트 생성

```bash
mkdir my-blog-api
cd my-blog-api
npm init -y
```

**package.json 수정:**

```json
{
  "name": "my-blog-api",
  "version": "1.0.0",
  "type": "module",
  "imports": {
    "#generated/*": "./generated/*",
    "#config": "./src/config/config.js",
    "#db/*": "./src/db/*"
  },
  "engines": {
    "node": ">=22.0.0"
  },
  "scripts": {
    "dev": "nodemon --env-file=./env/.env.development src/server.js",
    "prod": "node --env-file=./env/.env.production src/server.js",
    "prisma:migrate": "dotenv -e ./env/.env.development -- npx prisma migrate dev",
    "prisma:studio": "dotenv -e ./env/.env.development -- npx prisma studio",
    "prisma:generate": "dotenv -e ./env/.env.development -- npx prisma generate",
    "format": "npx prettier --write .",
    "format:check": "npx prettier --check ."
  }
}
```

---

### □ 2단계: 라이브러리 설치

```bash
npm install express @prisma/client @prisma/adapter-pg pg dotenv dotenv-cli zod
npm install -D nodemon prisma eslint prettier @eslint/js
```

---

### □ 3단계: 코드 스타일 설정

**eslint.config.js 생성:**

```javascript
import js from "@eslint/js";

export default [
  js.configs.recommended,
  {
    languageOptions: {
      ecmaVersion: 2024,
      sourceType: "module",
      globals: {
        console: "readonly",
        process: "readonly",
      },
    },
    rules: {
      "no-unused-vars": [
        "warn",
        { argsIgnorePattern: "^_" },
      ],
      "no-console": "off",
      "prefer-const": "error",
      "no-var": "error",
    },
  },
];
```

**.prettierrc 생성:**

```json
{
  "printWidth": 80,
  "bracketSpacing": true,
  "trailingComma": "all",
  "semi": true,
  "singleQuote": true
}
```

---

### □ 4단계: PostgreSQL 데이터베이스 생성

- Window면  작업 표시 줄에 “psql”검색, "SQL Shell(psql)" 오픈

- MacOS는 아래로 진행
```bash
psql
```

```sql
CREATE DATABASE my_blog_db;
\l
\q
```

---

### □ 5단계: Prisma 스키마 파일 생성

**prisma/schema.prisma 생성:**

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

---

### □ 6단계: 환경 변수 설정

**env/.env.example 생성:**

```env
# 환경 변수 예시 파일
NODE_ENV=development
PORT=5001
DATABASE_URL="postgresql://username:password@localhost:5432/my_blog_db"
```

**env/.env.development 생성:**

```env
NODE_ENV=development
PORT=5001
DATABASE_URL="postgresql://username:password@localhost:5432/my_blog_db"
```

> ⚠️ username, password를 실제 PostgreSQL 계정 정보로 변경하세요!

**env/.env.production 생성:**

```env
NODE_ENV=production
PORT=5001
DATABASE_URL="postgresql://username:password@production-host:5432/my_blog_db"
```

**.gitignore 생성:**

```
node_modules
env/*
!env/.env.example
generated/
```

---

### □ 7단계: 환경 변수 검증 설정

**src/config/config.js 생성:**

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
});

const parseEnvironment = () => {
  try {
    return envSchema.parse({
      NODE_ENV: process.env.NODE_ENV,
      PORT: process.env.PORT,
      DATABASE_URL: process.env.DATABASE_URL,
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

### □ 8단계: Prisma 설정 파일

**prisma.config.js 생성:**

```javascript
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
});
```

---

### □ 9단계: jsconfig.json 설정

**jsconfig.json 생성:**

```json
{
  "compilerOptions": {
    "target": "esnext", // 최신 JavaScript 문법 사용 (async/await, optional chaining 등)
    "module": "nodenext", // 최신 Node.js ESM 모듈 시스템 사용
    "moduleResolution": "nodenext" // package.json의 imports 필드 인식
  },
  "include": ["./generated/", "./src/"] // VS Code가 자동완성할 폴더 지정
}
```

> 💡 이 설정으로 `#generated`, `#config`, `#db` 같은 path alias 자동완성이 작동합니다.

---

### □ 10단계: Prisma Client 설정

**src/db/prisma.js 생성:**

```javascript
import { PrismaClient } from "#generated/prisma/client.ts";
import { PrismaPg } from "@prisma/adapter-pg";
import { config } from "#config";

const adapter = new PrismaPg({
  connectionString: config.DATABASE_URL,
});

export const prisma = new PrismaClient({ adapter });
```

---

### □ 11단계: Express 서버 생성

**src/server.js 생성:**

```javascript
import express from "express";
import { prisma } from "#db/prisma.js";
import { config } from "#config";

const app = express();

app.use(express.json());

app.get("/", (req, res) => {
  res.send("Hello, Prisma!");
});

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

### □ 12단계: 서버 실행 테스트

```bash
npm run dev
```

**브라우저에서 확인:**

- http://localhost:5001 접속
- "Hello, Prisma!" 메시지 확인

**터미널 출력 확인:**

```
[development] Server running at http://localhost:5001
```

---

## 최종 프로젝트 구조

```
my-blog-api/
├── prisma/
│   └── schema.prisma
├── env/
│   ├── .env.example
│   ├── .env.development
│   └── .env.production
├── src/
│   ├── config/
│   │   └── config.js
│   ├── db/
│   │   └── prisma.js
│   └── server.js
├── prisma.config.js
├── .prettierrc
├── eslint.config.js
├── jsconfig.json
├── .gitignore
└── package.json
```

---

## 완료 확인

✅ 모든 파일이 생성되었나요?
✅ 서버가 정상적으로 실행되나요?
✅ http://localhost:5001 에서 메시지가 보이나요?

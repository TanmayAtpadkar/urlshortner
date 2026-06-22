# URL Shortener — Express + TypeScript Setup

Day-one scaffold. Goal: a correct, typed skeleton you build on — not a finished app.

---

## 1. Folder structure

```
url-shortener/
├── src/
│   ├── server.ts            # entry point — starts the HTTP server
│   ├── app.ts               # builds the Express app (routes, middleware)
│   ├── config/
│   │   └── env.ts           # reads + validates environment variables
│   ├── routes/
│   │   └── url.routes.ts    # /shorten and /:code routes
│   ├── controllers/
│   │   └── url.controller.ts # request handlers (the logic per route)
│   ├── services/
│   │   └── url.service.ts    # business logic: make code, save, look up
│   ├── db/
│   │   └── index.ts          # database connection (pg pool)
│   ├── middleware/
│   │   └── errorHandler.ts   # central error handler
│   └── types/
│       └── index.ts          # shared interfaces/types
├── .env                      # secrets + config (NOT committed)
├── .env.example              # template of .env (committed)
├── .gitignore
├── tsconfig.json
├── package.json
└── README.md
```

Why this shape: it's the same separation you already use at work — routes are the gateway,
controllers/services are the microservice logic, db is your DB layer. Routes stay thin;
real work lives in services. This separation is also what interviewers like to see.

---

## 2. Initial commands

```bash
mkdir url-shortener && cd url-shortener
npm init -y

# runtime deps
npm install express pg dotenv

# dev deps (types + tooling)
npm install -D typescript ts-node-dev @types/node @types/express

# generate tsconfig
npx tsc --init
```

- `express` — web framework
- `pg` — Postgres client
- `dotenv` — loads `.env` into process.env
- `typescript` — the TS compiler
- `ts-node-dev` — runs TS directly + auto-restarts on save (dev only)
- `@types/*` — type definitions; THIS is what powers autocomplete for libraries

---

## 3. package.json scripts

Replace the "scripts" block with:

```json
"scripts": {
  "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
  "build": "tsc",
  "start": "node dist/server.js"
}
```

`npm run dev` → live-reloading dev server. `npm run build` → compiles to `dist/`.
`npm start` → runs the compiled JS (what you'll use on the server later).

---

## 4. tsconfig.json

Replace the generated file with this (sane strict defaults):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

`"strict": true` is the important one — it's what makes TS actually teach you. Don't turn it off.

---

## 5. .gitignore

```
node_modules/
dist/
.env
```

---

## 6. Minimal working skeleton

These four files give you a server that starts cleanly. Build features on top.

**src/config/env.ts**
```typescript
import dotenv from "dotenv";
dotenv.config();

export const env = {
  port: Number(process.env.PORT) || 3000,
  databaseUrl: process.env.DATABASE_URL || "",
};
```

**src/middleware/errorHandler.ts**
```typescript
import { Request, Response, NextFunction } from "express";

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void {
  console.error(err);
  res.status(500).json({ error: "Internal server error" });
}
```

**src/app.ts**
```typescript
import express, { Request, Response } from "express";
import { errorHandler } from "./middleware/errorHandler";

export function buildApp() {
  const app = express();
  app.use(express.json());

  app.get("/health", (_req: Request, res: Response) => {
    res.json({ status: "ok" });
  });

  // url routes will mount here later, e.g. app.use(urlRoutes);

  app.use(errorHandler);
  return app;
}
```

**src/server.ts**
```typescript
import { buildApp } from "./app";
import { env } from "./config/env";

const app = buildApp();
app.listen(env.port, () => {
  console.log(`Server running on http://localhost:${env.port}`);
});
```

Run `npm run dev`, hit `http://localhost:3000/health`, see `{"status":"ok"}`. That's your
green light. Everything else (DB, /shorten, /:code) gets added on top of this.

---

## 7. VS Code setup

### Extensions to install (Extensions panel, Ctrl+Shift+X)
1. **ESLint** (dbaeumer.vscode-eslint) — flags problems as you type
2. **Prettier** (esbenp.prettier-vscode) — auto-formats on save
3. **Error Lens** (usernamehw.errorlens) — shows the error inline on the line, not just underlined
4. **Pretty TypeScript Errors** (yoavbls.pretty-ts-errors) — makes TS's scary error messages readable

You do NOT need a "TypeScript extension" — VS Code has TS support built in.

### Workspace settings
Create `.vscode/settings.json` in the project:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib"
}
```
The last line tells VS Code to use your project's TS version, not its bundled one — keeps
editor and compiler in sync.

---

## 8. The autocomplete thing (IntelliSense)

What you described — start typing and suggestions pop up — is called **IntelliSense**, and
in a TypeScript project it's already on. No setup needed beyond the above. Here's how to use it:

- **Suggestions appear automatically** as you type. If they don't, press **Ctrl+Space** to force them.
- Type an object then `.` (a dot) → it lists every method/property that exists, with types.
  e.g. type `res.` and it shows `.json()`, `.status()`, `.send()`...
- **Tab** or **Enter** to accept the highlighted suggestion.
- **Hover** over any variable/function → shows its type and docs.
- **Ctrl+Click** (or F12) on anything → jumps to where it's defined.

Why it works so well in TS: the `@types/*` packages you installed describe every library's
shape, so the editor knows exactly what `express` and `pg` expose. This is a big reason TS
is worth learning — the editor becomes a live reference and catches mistakes before you run.

**Tip while learning:** when autocomplete shows you a type you don't recognize, hover it and
read it. That's TypeScript teaching you the API in real time — better than any tutorial.

---

## Next steps (not tonight)
- Postgres table: `urls(id SERIAL PK, code VARCHAR UNIQUE, long_url TEXT, created_at)`
- `/shorten` → generate code, INSERT, return short URL
- `/:code` → SELECT by code, 302 redirect (404 if not found)
- Then: base62 vs random (the collision trade-off you already spotted), Redis cache, auth.

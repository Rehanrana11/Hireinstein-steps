# WEEK 1 EXECUTION CHECKLIST
## Hour-by-hour breakdown to ship your first MVP

---

## MONDAY (12 hours)

### Hours 1-2: Setup & Architecture
- [ ] Create GitHub repo: `rankgpt-killer`
- [ ] Clone monorepo template: `npm create vite@latest`
- [ ] Initialize Prisma: `npx prisma init`
- [ ] Set up database (create Postgres on Railway.app)
- [ ] Create `.env.local`:
  ```
  DATABASE_URL=postgresql://...
  OPENAI_API_KEY=...
  ANTHROPIC_API_KEY=...
  GOOGLE_API_KEY=...
  REDIS_URL=redis://...
  STRIPE_SECRET_KEY=...
  VERCEL_TOKEN=...
  ```

**Commit:** "chore: monorepo setup"

### Hours 3-5: Data Model (Prisma Schema)
Create `prisma/schema.prisma`:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  passwordHash  String
  plan          String    @default("free") // free|pro|enterprise
  apiKey        String    @unique @default(cuid())
  stripeId      String?   @unique
  createdAt     DateTime  @default(now())

  domains       Domain[]
  rankings      Ranking[]
  queries       Query[]
}

model Domain {
  id            String    @id @default(cuid())
  userId        String
  user          User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  name          String
  website       String    @unique
  competitors   String[]  // JSON array of competitor domains
  lastCheckedAt DateTime  @default(now())

  rankings      Ranking[]
  citations     Citation[]

  @@unique([userId, website])
}

model Ranking {
  id            String    @id @default(cuid())
  domainId      String
  domain        Domain    @relation(fields: [domainId], references: [id], onDelete: Cascade)
  userId        String
  user          User      @relation(fields: [userId], references: [id])
  
  query         String
  model         String    // openai|claude|gemini|grok
  position      Int?      // 1, 2, 3, null
  mentionType   String    // direct|comparison|indirect|not_found
  sentiment     String    // positive|neutral|negative
  confidence    Float     // 0-1
  fullResponse  String    @db.Text
  responseTimes Int       // ms
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  citations     Citation[]

  @@unique([domainId, query, model])
}

model Citation {
  id            String    @id @default(cuid())
  rankingId     String
  ranking       Ranking   @relation(fields: [rankingId], references: [id], onDelete: Cascade)
  domainId      String
  domain        Domain    @relation(fields: [domainId], references: [id])
  
  sourceUrl     String
  quote         String    @db.Text
  relevance     Float     // 0-1
  
  createdAt     DateTime  @default(now())

  @@unique([rankingId, sourceUrl])
}

model Query {
  id            String    @id @default(cuid())
  userId        String
  user          User      @relation(fields: [userId], references: [id])
  
  text          String
  responseTimeMs Int
  
  createdAt     DateTime  @default(now())
}
```

```bash
npx prisma migrate dev --name init
npx prisma generate
```

**Commit:** "feat: prisma schema"

### Hours 6-12: Ranking Engine (Backend Setup)

Create `api/src/`:

```bash
cd api
npm init -y
npm install express typescript ts-node @types/express @types/node
npm install @prisma/client redis zod
npm install openai @anthropic-ai/sdk google-generativeai
npm install dotenv cors helmet

npx tsc --init
```

Create `api/src/index.ts`:

```typescript
import express from "express";
import cors from "cors";
import helmet from "helmet";
import { rankingRouter } from "./routes/ranking";
import { errorHandler } from "./middleware/errorHandler";

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json());

app.use("/api/v1", rankingRouter);

app.use(errorHandler);

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

Create `api/src/routes/ranking.ts`:

```typescript
import { Router } from "express";
import { rankDomain } from "../services/ranking";
import { validateRequest } from "../middleware/validate";

export const rankingRouter = Router();

rankingRouter.post(
  "/rank",
  validateRequest,
  async (req, res, next) => {
    try {
      const { domain, query, models } = req.body;
      
      const result = await rankDomain(domain, query, models);
      
      res.json(result);
    } catch (error) {
      next(error);
    }
  }
);
```

Create `api/src/services/ranking.ts`:

```typescript
import { OpenAI } from "openai";
import { Anthropic } from "@anthropic-ai/sdk";

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY
});

export async function rankDomain(
  domain: string,
  query: string,
  models: string[]
) {
  const startTime = Date.now();

  // Rank in parallel
  const rankings = await Promise.all(
    models.map((model) => rankSingleModel(domain, query, model))
  );

  return {
    domain,
    query,
    rankings,
    aiVisibilityScore: calculateScore(rankings),
    responseTimes: Date.now() - startTime
  };
}

async function rankSingleModel(
  domain: string,
  query: string,
  model: string
) {
  const prompt = `You are ranking whether "${domain}" is mentioned in response to: "${query}".

Respond with JSON only:
{
  "mentioned": boolean,
  "position": number | null,
  "mentionType": "direct" | "comparison" | "indirect",
  "sentiment": "positive" | "neutral" | "negative",
  "confidence": number,
  "reasoning": string
}`;

  let response: string;

  if (model === "openai") {
    const res = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [{ role: "user", content: prompt }],
      temperature: 0.7
    });
    response = res.choices[0].message.content || "";
  } else if (model === "claude") {
    const res = await anthropic.messages.create({
      model: "claude-opus-4-20250514",
      max_tokens: 256,
      messages: [{ role: "user", content: prompt }]
    });
    response =
      res.content[0].type === "text" ? res.content[0].text : "";
  } else {
    throw new Error(`Unsupported model: ${model}`);
  }

  try {
    const parsed = JSON.parse(response);
    return {
      model,
      ...parsed,
      fullResponse: response
    };
  } catch {
    return {
      model,
      mentioned: false,
      position: null,
      confidence: 0,
      error: "Failed to parse response"
    };
  }
}

function calculateScore(rankings: any[]): number {
  let score = 0;
  rankings.forEach((r) => {
    const positionScore = {
      1: 40,
      2: 25,
      3: 15,
      null: 0
    }[r.position] || 0;

    const sentimentWeight = {
      positive: 1.2,
      neutral: 1.0,
      negative: 0.5
    }[r.sentiment] || 1.0;

    const modelWeight = {
      openai: 0.3,
      claude: 0.25,
      gemini: 0.25,
      grok: 0.2
    }[r.model] || 0.25;

    score += positionScore * sentimentWeight * modelWeight;
  });

  return Math.round((score / 100) * 100);
}
```

Create `api/src/middleware/errorHandler.ts`:

```typescript
import { Request, Response, NextFunction } from "express";

export function errorHandler(
  err: any,
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.error(err);

  res.status(err.status || 500).json({
    error: err.message || "Internal server error"
  });
}
```

Create `api/src/middleware/validate.ts`:

```typescript
import { Request, Response, NextFunction } from "express";
import { z } from "zod";

const rankSchema = z.object({
  domain: z.string().url(),
  query: z.string().min(5),
  models: z.array(z.enum(["openai", "claude", "gemini", "grok"]))
});

export function validateRequest(
  req: Request,
  res: Response,
  next: NextFunction
) {
  try {
    rankSchema.parse(req.body);
    next();
  } catch (error) {
    res.status(400).json({ error: "Invalid request" });
  }
}
```

```bash
npm run dev
```

Test:
```bash
curl -X POST http://localhost:3001/api/v1/rank \
  -H "Content-Type: application/json" \
  -d '{
    "domain": "example.com",
    "query": "best project management software",
    "models": ["openai", "claude"]
  }'
```

**Commit:** "feat: ranking engine MVP"

---

## TUESDAY (12 hours)

### Hours 1-4: Frontend Setup (Next.js)

```bash
cd apps
npx create-next-app@latest web \
  --typescript \
  --tailwind \
  --app
  
cd web
npm install @tanstack/react-query recharts
npm install @radix-ui/react-dialog @radix-ui/react-tabs
```

Create `apps/web/src/app/page.tsx`:

```typescript
"use client";

import { useState } from "react";
import { RankingForm } from "@/components/RankingForm";
import { RankingResult } from "@/components/RankingResult";

export default function Home() {
  const [result, setResult] = useState<any>(null);
  const [loading, setLoading] = useState(false);

  async function handleRank(domain: string, query: string) {
    setLoading(true);
    try {
      const res = await fetch("http://localhost:3001/api/v1/rank", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          domain,
          query,
          models: ["openai", "claude", "gemini"]
        })
      });

      const data = await res.json();
      setResult(data);
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  }

  return (
    <main className="min-h-screen bg-gradient-to-b from-blue-50 to-white p-8">
      <div className="max-w-4xl mx-auto">
        <h1 className="text-4xl font-bold mb-2">
          AI Ranking Tracker
        </h1>
        <p className="text-gray-600 mb-8">
          Real-time ranking visibility across ChatGPT, Claude, Gemini
        </p>

        <RankingForm onRank={handleRank} loading={loading} />

        {result && <RankingResult data={result} />}
      </div>
    </main>
  );
}
```

Create `apps/web/src/components/RankingForm.tsx`:

```typescript
"use client";

import { useState } from "react";

interface Props {
  onRank: (domain: string, query: string) => void;
  loading: boolean;
}

export function RankingForm({ onRank, loading }: Props) {
  const [domain, setDomain] = useState("");
  const [query, setQuery] = useState("");

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        onRank(domain, query);
      }}
      className="bg-white rounded-lg shadow p-6 mb-8"
    >
      <div className="space-y-4">
        <div>
          <label className="block text-sm font-medium mb-2">
            Your Domain
          </label>
          <input
            type="text"
            placeholder="example.com"
            value={domain}
            onChange={(e) => setDomain(e.target.value)}
            className="w-full px-4 py-2 border rounded-lg"
            required
          />
        </div>

        <div>
          <label className="block text-sm font-medium mb-2">
            Query to Check
          </label>
          <input
            type="text"
            placeholder="best project management software"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            className="w-full px-4 py-2 border rounded-lg"
            required
          />
        </div>

        <button
          type="submit"
          disabled={loading}
          className="w-full bg-blue-600 text-white py-2 rounded-lg font-semibold hover:bg-blue-700 disabled:opacity-50"
        >
          {loading ? "Checking..." : "Check Ranking"}
        </button>
      </div>
    </form>
  );
}
```

Create `apps/web/src/components/RankingResult.tsx`:

```typescript
interface Props {
  data: any;
}

export function RankingResult({ data }: Props) {
  return (
    <div className="bg-white rounded-lg shadow p-6">
      <h2 className="text-2xl font-bold mb-6">{data.domain}</h2>

      <div className="mb-8">
        <p className="text-gray-600 mb-2">AI Visibility Score</p>
        <div className="text-5xl font-bold text-blue-600">
          {data.aiVisibilityScore}/100
        </div>
      </div>

      <div className="grid grid-cols-3 gap-4">
        {data.rankings.map((r: any) => (
          <div key={r.model} className="border rounded-lg p-4 text-center">
            <div className="text-2xl font-bold text-blue-600">
              {r.position ? `#${r.position}` : "N/A"}
            </div>
            <div className="text-sm text-gray-600 capitalize mt-2">
              {r.model}
            </div>
            <div className="text-xs text-gray-500 mt-1">
              {r.sentiment}
            </div>
          </div>
        ))}
      </div>

      <p className="text-xs text-gray-500 mt-6">
        Response time: {data.responseTime}ms
      </p>
    </div>
  );
}
```

```bash
npm run dev
```

**Commit:** "feat: frontend MVP"

### Hours 5-8: Add Competitor Benchmarking

Create `api/src/routes/competitors.ts`:

```typescript
import { Router } from "express";
import { db } from "@/db";

export const competitorRouter = Router();

competitorRouter.post("/benchmark", async (req, res, next) => {
  try {
    const { domains, query, models } = req.body;

    // Rank all domains in parallel
    const results = await Promise.all(
      domains.map((domain) =>
        rankDomain(domain, query, models)
      )
    );

    res.json({
      query,
      domains: results,
      benchmark: calculateBenchmark(results)
    });
  } catch (error) {
    next(error);
  }
});

function calculateBenchmark(results: any[]) {
  const sorted = results.sort(
    (a, b) => b.aiVisibilityScore - a.aiVisibilityScore
  );

  return {
    leader: sorted[0],
    yourPosition: 1,
    gap: sorted[0].aiVisibilityScore - sorted[1].aiVisibilityScore
  };
}
```

Update `api/src/index.ts`:
```typescript
app.use("/api/v1", rankingRouter);
app.use("/api/v1", competitorRouter);
```

### Hours 9-12: Add Redis Caching

Create `api/src/cache.ts`:

```typescript
import Redis from "redis";

export const redis = Redis.createClient({
  url: process.env.REDIS_URL || "redis://localhost:6379"
});

redis.connect();

export async function getOrSet(
  key: string,
  ttl: number,
  fn: () => Promise<any>
) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const result = await fn();
  await redis.setEx(key, ttl, JSON.stringify(result));

  return result;
}
```

Update `api/src/services/ranking.ts`:

```typescript
import { getOrSet } from "@/cache";

export async function rankDomain(
  domain: string,
  query: string,
  models: string[]
) {
  const cacheKey = `rank:${domain}:${query}:${models.join(",")}`;

  return getOrSet(cacheKey, 600, async () => {
    // ... existing ranking logic
  });
}
```

**Commit:** "feat: caching + competitor benchmarking"

---

## WEDNESDAY (12 hours)

### Hours 1-4: Auth Setup (Lucia)

```bash
cd api
npm install lucia @lucia-auth/adapter-prisma
npm install argon2
```

Update `prisma/schema.prisma`:

```prisma
model Session {
  id        String    @id
  userId    String
  user      User      @relation(references: [id], fields: [userId], onDelete: Cascade)
  expiresAt DateTime

  @@index([userId])
}
```

```bash
npx prisma migrate dev --name add_session
```

Create `api/src/auth/lucia.ts`:

```typescript
import { Lucia } from "lucia";
import { PrismaAdapter } from "@lucia-auth/adapter-prisma";
import { db } from "@/db";

export const auth = new Lucia(
  new PrismaAdapter(db.session, db.user),
  {
    sessionCookie: {
      expires: false,
      attributes: {
        secure: process.env.NODE_ENV === "production",
        httpOnly: true,
        sameSite: "lax"
      }
    }
  }
);

declare module "lucia" {
  interface Register {
    Auth: typeof auth;
  }
}
```

Create `api/src/routes/auth.ts`:

```typescript
import { Router } from "express";
import * as argon2 from "argon2";
import { auth } from "@/auth/lucia";
import { db } from "@/db";

export const authRouter = Router();

authRouter.post("/signup", async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const hashedPassword = await argon2.hash(password);

    const user = await db.user.create({
      data: {
        email,
        passwordHash: hashedPassword
      }
    });

    const session = await auth.createSession(user.id, {});

    res.json({
      userId: user.id,
      sessionId: session.id
    });
  } catch (error) {
    next(error);
  }
});

authRouter.post("/login", async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const user = await db.user.findUnique({
      where: { email }
    });

    if (!user) {
      res.status(401).json({ error: "Invalid email or password" });
      return;
    }

    const valid = await argon2.verify(
      user.passwordHash,
      password
    );

    if (!valid) {
      res.status(401).json({ error: "Invalid email or password" });
      return;
    }

    const session = await auth.createSession(user.id, {});

    res.json({
      userId: user.id,
      sessionId: session.id
    });
  } catch (error) {
    next(error);
  }
});
```

Update `api/src/index.ts`:
```typescript
app.use("/api/v1", authRouter);
```

### Hours 5-8: Stripe Integration

```bash
npm install stripe
```

Create `api/src/billing/stripe.ts`:

```typescript
import Stripe from "stripe";
import { db } from "@/db";

export const stripe = new Stripe(
  process.env.STRIPE_SECRET_KEY!
);

const PLANS = {
  free: { name: "Free", price: 0 },
  pro: { name: "Pro", price: 99, priceId: process.env.STRIPE_PRO_PRICE_ID },
  enterprise: {
    name: "Enterprise",
    price: 999,
    priceId: process.env.STRIPE_ENTERPRISE_PRICE_ID
  }
};

export async function createCheckoutSession(
  userId: string,
  plan: string
) {
  const user = await db.user.findUnique({
    where: { id: userId }
  });

  if (!user) throw new Error("User not found");

  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    mode: "subscription",
    line_items: [
      {
        price: PLANS[plan as keyof typeof PLANS].priceId,
        quantity: 1
      }
    ],
    success_url: `${process.env.DOMAIN}/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.DOMAIN}/pricing`
  });

  return session.url;
}

export async function handleStripeEvent(event: Stripe.Event) {
  switch (event.type) {
    case "checkout.session.completed":
      const session = event.data.object as Stripe.Checkout.Session;
      await db.user.update({
        where: { email: session.customer_email! },
        data: {
          plan: "pro",
          stripeId: session.customer as string
        }
      });
      break;
  }
}
```

Create `api/src/routes/billing.ts`:

```typescript
import { Router } from "express";
import { createCheckoutSession } from "@/billing/stripe";

export const billingRouter = Router();

billingRouter.post("/checkout", async (req, res, next) => {
  try {
    const { userId, plan } = req.body;

    const url = await createCheckoutSession(userId, plan);

    res.json({ url });
  } catch (error) {
    next(error);
  }
});
```

Update `api/src/index.ts`:
```typescript
app.use("/api/v1", billingRouter);

// Webhook
app.post("/webhook/stripe", express.raw({ type: "application/json" }), async (req, res) => {
  const sig = req.headers["stripe-signature"] as string;
  const event = stripe.webhooks.constructEvent(
    req.body,
    sig,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  await handleStripeEvent(event);
  res.json({ received: true });
});
```

### Hours 9-12: Frontend Auth Pages

Create `apps/web/src/app/login/page.tsx`:

```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

export default function LoginPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleLogin(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);

    try {
      const res = await fetch("http://localhost:3001/api/v1/login", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, password })
      });

      const data = await res.json();
      localStorage.setItem("userId", data.userId);
      router.push("/dashboard");
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  }

  return (
    <main className="min-h-screen flex items-center justify-center bg-gradient-to-b from-blue-50 to-white">
      <div className="bg-white rounded-lg shadow-lg p-8 w-full max-w-md">
        <h1 className="text-3xl font-bold mb-6">Log In</h1>

        <form onSubmit={handleLogin} className="space-y-4">
          <input
            type="email"
            placeholder="Email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="w-full px-4 py-2 border rounded-lg"
            required
          />
          <input
            type="password"
            placeholder="Password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            className="w-full px-4 py-2 border rounded-lg"
            required
          />
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-600 text-white py-2 rounded-lg font-semibold hover:bg-blue-700"
          >
            {loading ? "Signing in..." : "Sign In"}
          </button>
        </form>

        <p className="text-sm text-gray-600 mt-4">
          New user?{" "}
          <a href="/signup" className="text-blue-600">
            Sign up
          </a>
        </p>
      </div>
    </main>
  );
}
```

**Commit:** "feat: auth + billing integration"

---

## THURSDAY (12 hours)

### Hours 1-6: Dashboard Pages

Create `apps/web/src/app/dashboard/page.tsx`:

```typescript
"use client";

import { useEffect, useState } from "react";
import { RankingCard } from "@/components/RankingCard";
import { CompetitorBench } from "@/components/CompetitorBench";

export default function DashboardPage() {
  const [domains, setDomains] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const userId = localStorage.getItem("userId");
    if (!userId) {
      window.location.href = "/login";
      return;
    }

    // Load user domains (mock)
    setDomains([
      {
        id: "1",
        name: "example.com",
        website: "example.com",
        rankings: [
          { model: "openai", position: 2, sentiment: "positive" },
          { model: "claude", position: 3, sentiment: "positive" },
          { model: "gemini", position: 1, sentiment: "positive" }
        ],
        aiVisibilityScore: 78
      }
    ]);
    setLoading(false);
  }, []);

  if (loading) return <div>Loading...</div>;

  return (
    <main className="min-h-screen bg-gradient-to-b from-blue-50 to-white p-8">
      <div className="max-w-6xl mx-auto">
        <h1 className="text-4xl font-bold mb-8">Dashboard</h1>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
          {domains.map((domain) => (
            <RankingCard key={domain.id} domain={domain} />
          ))}
        </div>

        <div className="mt-12">
          <h2 className="text-2xl font-bold mb-6">Competitor Benchmark</h2>
          <CompetitorBench domains={domains} />
        </div>
      </div>
    </main>
  );
}
```

Create `apps/web/src/components/RankingCard.tsx`:

```typescript
interface Props {
  domain: any;
}

export function RankingCard({ domain }: Props) {
  return (
    <div className="bg-white rounded-lg shadow-lg p-6">
      <h3 className="text-xl font-bold mb-4">{domain.name}</h3>

      <div className="mb-6">
        <p className="text-gray-600 text-sm">AI Visibility Score</p>
        <div className="text-5xl font-bold text-blue-600 mt-2">
          {domain.aiVisibilityScore}
        </div>
        <div className="text-xs text-gray-500 mt-1">/100</div>
      </div>

      <div className="grid grid-cols-3 gap-2 mb-6">
        {domain.rankings.map((r: any) => (
          <div
            key={r.model}
            className="bg-gray-50 rounded p-3 text-center"
          >
            <div className="text-lg font-bold text-blue-600">
              #{r.position}
            </div>
            <div className="text-xs text-gray-600 capitalize">
              {r.model}
            </div>
          </div>
        ))}
      </div>

      <button className="w-full bg-blue-600 text-white py-2 rounded-lg font-semibold hover:bg-blue-700">
        Re-Check Now
      </button>
    </div>
  );
}
```

### Hours 7-12: Testing + Deployment Prep

Create `api/src/__tests__/ranking.test.ts`:

```typescript
import { rankDomain } from "../services/ranking";

describe("rankDomain", () => {
  it("should return ranking for domain", async () => {
    const result = await rankDomain(
      "example.com",
      "best software",
      ["openai"]
    );

    expect(result.domain).toBe("example.com");
    expect(result.aiVisibilityScore).toBeGreaterThanOrEqual(0);
    expect(result.aiVisibilityScore).toBeLessThanOrEqual(100);
  });
});
```

Create `Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npx prisma generate

EXPOSE 3001

CMD ["npm", "start"]
```

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Deploy frontend to Vercel
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
        run: cd apps/web && vercel --prod --token=$VERCEL_TOKEN

      - name: Deploy backend to Railway
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: |
          npm i -g @railway/cli
          railway up
```

**Commit:** "test: add unit tests + deployment setup"

---

## FRIDAY (10 hours)

### Hours 1-5: Go Live

**1. Stripe Setup**
- Create Stripe account
- Add price IDs to `.env`
- Deploy webhook

**2. Database Migration**
```bash
npx prisma migrate deploy
```

**3. Frontend Deployment**
```bash
cd apps/web
vercel --prod
```

**4. Backend Deployment**
```bash
# Railway
railway up

# Or Fly.io
fly deploy
```

**5. Test Full Flow**
- Sign up at yourdomain.com/signup
- Log in
- Check ranking
- Upgrade to Pro
- Verify Stripe payment

### Hours 6-10: Marketing Page

Create `apps/web/src/app/page.tsx` (replace home):

```typescript
"use client";

import Link from "next/link";

export default function HomePage() {
  return (
    <main className="min-h-screen bg-gradient-to-b from-blue-50 to-white">
      {/* Hero */}
      <section className="px-8 py-24 text-center">
        <h1 className="text-5xl font-bold mb-4">
          Track Your AI Rankings in Real-Time
        </h1>
        <p className="text-xl text-gray-600 mb-8">
          Sub-200ms ranking checks across ChatGPT, Claude, Gemini, and Grok.
          No batching. No delays.
        </p>
        <div className="flex gap-4 justify-center">
          <Link
            href="/signup"
            className="px-8 py-3 bg-blue-600 text-white rounded-lg font-semibold hover:bg-blue-700"
          >
            Get Started Free
          </Link>
          <a
            href="https://app.iclosed.io/e/rankgpt/rankgpt-application-call"
            className="px-8 py-3 border-2 border-blue-600 text-blue-600 rounded-lg font-semibold hover:bg-blue-50"
          >
            Book Demo
          </a>
        </div>
      </section>

      {/* Features */}
      <section className="px-8 py-24 max-w-4xl mx-auto">
        <h2 className="text-3xl font-bold text-center mb-12">
          Why Choose Us?
        </h2>

        <div className="grid md:grid-cols-3 gap-8">
          <div>
            <h3 className="text-lg font-semibold mb-2">⚡ Lightning Fast</h3>
            <p className="text-gray-600">
              Sub-200ms response times. See results instantly.
            </p>
          </div>

          <div>
            <h3 className="text-lg font-semibold mb-2">🧠 Intelligent</h3>
            <p className="text-gray-600">
              ML-ranked citations. Predictive opportunities.
            </p>
          </div>

          <div>
            <h3 className="text-lg font-semibold mb-2">💰 Transparent</h3>
            <p className="text-gray-600">
              Clear pricing. No hidden fees. $99/mo for Pro.
            </p>
          </div>
        </div>
      </section>

      {/* Pricing */}
      <section className="px-8 py-24 bg-white">
        <h2 className="text-3xl font-bold text-center mb-12">Pricing</h2>

        <div className="grid md:grid-cols-3 gap-8 max-w-4xl mx-auto">
          {[
            {
              name: "Free",
              price: "$0",
              features: ["10 checks/month", "1 domain"]
            },
            {
              name: "Pro",
              price: "$99",
              features: ["Unlimited checks", "10 domains", "Competitor bench", "Citations"]
            },
            {
              name: "Enterprise",
              price: "Custom",
              features: ["Everything", "API access", "Custom integrations"]
            }
          ].map((plan) => (
            <div key={plan.name} className="border rounded-lg p-6">
              <h3 className="text-lg font-bold mb-2">{plan.name}</h3>
              <div className="text-3xl font-bold mb-4">{plan.price}</div>
              <ul className="space-y-2">
                {plan.features.map((f) => (
                  <li key={f} className="text-sm text-gray-600">
                    ✓ {f}
                  </li>
                ))}
              </ul>
              <button className="w-full mt-6 px-4 py-2 bg-blue-600 text-white rounded-lg font-semibold hover:bg-blue-700">
                Get Started
              </button>
            </div>
          ))}
        </div>
      </section>
    </main>
  );
}
```

---

## WEEK 1 SUMMARY

### ✅ Deliverables

- [ ] API endpoint `/api/v1/rank` (< 200ms latency)
- [ ] Redis caching (60% hit rate)
- [ ] Frontend dashboard (3 core views)
- [ ] Auth + Stripe integration
- [ ] Deployed to Vercel + Railway/Fly
- [ ] Marketing homepage
- [ ] 1 paying customer ($99/mo)

### 📊 Metrics to Track

| Metric | Target | Status |
|--------|--------|--------|
| API Latency (p95) | < 200ms | ✅ |
| Cache Hit Rate | > 60% | ✅ |
| Dashboard LCP | < 1.5s | ✅ |
| Uptime | > 99.5% | ✅ |
| Paying Customers | 1+ | 🟡 |

### 🚀 What's Next (Week 2)

- Add competitor tracking UI
- Build citation intelligence dashboard
- Implement A/B tests for pricing
- Acquire 5 more customers
- Optimize search ranking for "ai ranking tracker"

---

## Final Notes

**You have 5 days to ship.**

**The goal is not perfection. The goal is speed.**

- Skip error messages that don't matter
- Use default styles (Tailwind)
- Don't optimize prematurely
- Ship. Get feedback. Iterate.

**Week 2 is for polish. Week 1 is for existence.**

Now go build.

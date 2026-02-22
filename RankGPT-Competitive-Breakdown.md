# BEAT RANKGPT: 30-STEP COMPETITIVE ENGINEERING BLUEPRINT

## What RankGPT Does (From Site Analysis)

**Core Features:**
- Multi-platform AI tracking (ChatGPT 92%, Claude 78%, Gemini 85%, Grok 65%)
- AI sentiment analysis
- Competitor benchmarking
- Citation tracking + building
- Content gap analysis
- Domain indexing for AI discovery
- Auto-content creation (mentioned "RankGPT Agent")
- Auto-citation building

**Their Tech Stack (Inferred):**
- Next.js frontend (App Router, TypeScript, Tailwind)
- Backend: Node or Python (likely)
- Postgres database
- Stripe billing integration
- iClosed.io for booking demo calls (third-party)

**Their Vulnerability:**
- Manual citation building (scalability bottleneck)
- Generic "Agent" - likely rule-based or simple prompt chains
- No visible real-time API response time optimization
- Tracking infrastructure seems centralized (not edge-deployed)
- No visible batch processing for large-scale queries

---

## YOUR COMPETITIVE ADVANTAGE PATH (4-8 Weeks, Solo)

You cannot build their entire product in 4-8 weeks. **You build ONE killer feature that solves their largest pain point.**

**Pick Your Wedge:** Real-time AI ranking API with sub-200ms response time + autonomous content optimization loop.

This means:
1. Faster tracking (edge-deployed, cached inference)
2. Better citation intelligence (ML-ranked vs. rule-based)
3. Real content insights (actual prompt simulation + scoring)

---

## PHASE 1: TACTICAL SETUP (Week 1)

### 1. Define Your Narrow Positioning Moat
**Goal:** Why you win on ONE dimension, not features.

❌ "We do what RankGPT does but better."
✅ "Real-time AI ranking with sub-200ms query response. No batching delays."

**Action:** Write this in one sentence and commit.

---

### 2. Set Performance Budget as Hard Constraint
- API latency: < 200ms (p95) for ranking queries
- Dashboard load: < 1.5s LCP
- Rank update latency: < 5min from prompt seed

**Why this matters:** RankGPT likely batches their checks. You respond live.

---

### 3. Choose Lean Tech Stack (Minimize Time to Value)

**Frontend:**
- Next.js 15 App Router (SSR for hero, ISR for results)
- TypeScript (mandatory)
- Tailwind CSS (design tokens only, no custom CSS)
- shadcn/ui for dashboard components (pre-built, accessible)

**Backend:**
- Node.js + Express (speed > Python here)
- Zod for validation (10 lines per schema)
- Lucia for auth (simpler than NextAuth)

**Database:**
- Postgres + Prisma ORM
- Redis for caching (real-time query results)
- Vector DB: Weaviate or Pinecone (for citation similarity matching)

**Inference:**
- Anthropic SDK for Claude prompting (you know this, fast iteration)
- OpenAI SDK for GPT-4 calls
- Google AI SDK for Gemini
- xAI SDK for Grok

**Deployment:**
- Vercel (frontend + edge functions for sub-200ms latency)
- Railway.app or Fly.io (backend microservice, simple PG + Redis)
- Cloudflare Workers (optional: edge-side caching layer)

---

### 4. Set Up Monorepo Structure (Solo Sanity)
```
rankgpt-killer/
  ├── apps/
  │   ├── web (Next.js frontend)
  │   └── api (Express backend)
  ├── packages/
  │   ├── shared-types (TypeScript interfaces)
  │   ├── prompts (prompt engineering library)
  │   └── db (Prisma schema)
  └── scripts/
      └── seed.ts (test data)
```

**Why:** Easier to share types, reuse prompts, scale.

---

### 5. Define Core Data Model
```
Domain
├── id (UUID)
├── name
├── website_url
├── competitors[] (relation)

Ranking
├── id
├── domain_id
├── model (openai, claude, gemini, grok)
├── query
├── position (1, 2, 3, etc.)
├── mention_type (direct, indirect, comparison)
├── sentiment (positive, neutral, negative)
├── citation_count
├── last_checked (timestamp)
├── response_time_ms
├── full_response (LLM output)

Citation
├── id
├── ranking_id
├── source_url
├── quote
├── relevance_score
├── date_found

User
├── id
├── email
├── subscription_tier
├── api_key

Query
├── id
├── user_id
├── text
├── executed_at
├── response_time_ms (track your latency)
```

---

### 6. Architect the Ranking Engine (Core Differentiator)

**RankGPT's likely approach:**
```
Query → OpenAI API → Parse response → Store
(Slow, batched)
```

**Your approach:**
```
Query
  ├→ Cache check (Redis)
  │  └→ Return if < 10min old (instant)
  │
  ├→ If cache miss:
  │   ├→ Route to fastest available model
  │   ├→ Run in parallel (GPT-4 + Claude + Gemini)
  │   ├→ Parse mention + sentiment + citation
  │   ├→ Score relevance (TF-IDF + BM25)
  │   ├→ Store + cache result
  │   └→ Return (200ms target)
  │
  └→ Background job (5min): Re-rank, update citations
```

**Key insight:** Cache aggressive for speed, background refresh for freshness.

---

### 7. Build Competitive Citation Intelligence (Your Moat #2)

RankGPT tracks citations. You predict them.

**Method:**
1. When AI returns a mention, extract: domain, confidence score, reasoning
2. Query vector DB: "What other sites mention [domain] in [industry]?"
3. ML-score: similarity to existing citations
4. Suggest new citation sources automatically

**Why:** Turns citation discovery from manual to predictive.

---

### 8. Architecture Diagram
```
                    ┌─────────────────┐
                    │  User Browser   │
                    │  (Next.js SSR)  │
                    └────────┬────────┘
                             │ HTTPS
                    ┌────────▼────────┐
                    │  Vercel Edge    │ ← Caching layer (200ms target)
                    │ (rate limit)    │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼──┐           ┌────▼──┐           ┌────▼──┐
   │ Redis │           │Postgres│          │Vector │
   │ (Hot) │           │(Cold)  │          │  DB   │
   └────▲──┘           └────┬──┘           └────┬──┘
        │                   │                   │
        └───────┬───────────┴───────────────────┘
                │
        ┌───────▼──────────┐
        │ Express Backend  │ ← Ranking engine
        │ + Inference      │
        │ (Railway/Fly)    │
        └───────┬──────────┘
                │
    ┌───────────┼───────────┬───────────┐
    │           │           │           │
 ┌──▼─┐  ┌──▼─┐  ┌──▼─┐  ┌──▼─┐
 │GPT4│  │ C. │  │Gem.│  │Grok│
 └────┘  └────┘  └────┘  └────┘
 (OpenAI APIs running in parallel, < 100ms each)
```

---

## PHASE 2: RANKING ENGINE (Weeks 1-2)

### 9. Build Ranking Query Handler
```typescript
// /api/v1/rank

POST /api/v1/rank
{
  "domain": "example.com",
  "query": "best project management tool",
  "models": ["openai", "claude", "gemini", "grok"],
  "use_cache": true
}

Response (target < 200ms):
{
  "query_id": "uuid",
  "domain": "example.com",
  "results": [
    {
      "model": "openai",
      "position": 2,
      "mention_type": "direct",
      "sentiment": "positive",
      "confidence": 0.92,
      "response_time_ms": 89,
      "full_response": "...",
      "citations": [
        {
          "source_url": "techreview.com",
          "quote": "...",
          "relevance": 0.88
        }
      ]
    },
    // ... other models
  ],
  "ai_visibility_score": 82,
  "competitive_position": "outperforming 67% of competitors",
  "overall_response_time_ms": 145
}
```

---

### 10. Implement Parallel Model Invocation
```python
# backend/src/services/ranking.ts

export async function rankDomain(
  domain: string,
  query: string,
  models: string[]
) {
  const queries = models.map((model) =>
    executeRankingQuery(domain, query, model)
  );

  // Race all LLM calls in parallel
  const results = await Promise.all(queries);

  // Parse, score, return
  return {
    query_id: generateId(),
    results: results.map(parseRankingResult),
    ai_visibility_score: calculateScore(results),
    overall_response_time_ms: Date.now() - startTime
  };
}
```

**Key:** Parallel, not sequential. 4 models in parallel = ~100ms total vs. 400ms sequential.

---

### 11. Smart Prompt Engineering (Extract + Score)
```typescript
// packages/prompts/ranking.prompts.ts

export const rankingPrompts = {
  extractor: `
You are analyzing whether [DOMAIN] was mentioned in response to: "[QUERY]"

Response: "[LLM_OUTPUT]"

Extract:
1. Is [DOMAIN] mentioned? (yes/no)
2. What position? (1st, 2nd, 3rd mention, or not present)
3. Mention type: (direct recommendation, comparison, indirect, not mentioned)
4. Sentiment: (positive, neutral, negative)
5. Confidence: (0-1, how confident are you?)

RESPOND ONLY IN JSON.
  `,

  citationExtractor: `
From this AI response about [DOMAIN]:
"[LLM_OUTPUT]"

Extract all citations/sources mentioned:
1. Source URL (infer if not explicit)
2. Exact quote
3. How it references [DOMAIN]

RESPOND ONLY IN JSON ARRAY.
  `
};
```

---

### 12. Cache Strategy (Redis)
```typescript
// backend/src/cache.ts

const CACHE_TTL = {
  ranking: 600, // 10 min (fresh enough)
  citation: 3600, // 1 hour
  competitor: 1800 // 30 min
};

async function getRanking(
  domain: string,
  query: string,
  model: string
): Promise<RankingResult> {
  const cacheKey = `rank:${domain}:${query}:${model}`;
  
  // Check Redis first
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Cache miss → invoke API
  const result = await executeRankingQuery(domain, query, model);

  // Store in Redis
  await redis.setex(cacheKey, CACHE_TTL.ranking, JSON.stringify(result));

  return result;
}
```

---

### 13. Background Refresh Job
```typescript
// backend/src/jobs/refreshRankings.ts

// Every 5 minutes, re-check top domains without waiting
export async function scheduleRankingRefresh() {
  setInterval(async () => {
    const activeDomains = await db.domain.findMany({
      where: { lastCheckedAt: { lt: fiveMinutesAgo() } }
    });

    for (const domain of activeDomains) {
      // Fire and forget (don't block user)
      queueRankingJob(domain.id).catch(console.error);
    }
  }, 300000); // 5 min
}
```

---

### 14. AI Visibility Score Calculation
```typescript
// backend/src/scoring.ts

export function calculateAIVisibilityScore(rankings: RankingResult[]): number {
  let score = 0;

  rankings.forEach((r) => {
    // Position weight (1st = 40pts, 2nd = 25pts, 3rd = 15pts)
    const positionScore = {
      1: 40,
      2: 25,
      3: 15,
      4: 5,
      null: 0 // not mentioned
    }[r.position] || 0;

    // Sentiment weight
    const sentimentWeight = {
      positive: 1.2,
      neutral: 1.0,
      negative: 0.5
    }[r.sentiment] || 1.0;

    // Model weight (GPT-4 = 30%, Claude = 25%, Gemini = 25%, Grok = 20%)
    const modelWeight = {
      openai: 0.3,
      claude: 0.25,
      gemini: 0.25,
      grok: 0.2
    }[r.model] || 0.25;

    score += positionScore * sentimentWeight * modelWeight;
  });

  // Normalize to 0-100
  return Math.round((score / 100) * 100);
}
```

---

### 15. Citation Intelligence Engine
```typescript
// backend/src/services/citations.ts

export async function predictCitations(
  domain: string,
  ranking: RankingResult
): Promise<PredictedCitation[]> {
  // Extract citations from LLM response
  const extractedCitations = await extractCitations(ranking.fullResponse, domain);

  // For each citation, find similar ones in vector DB
  const citationEmbeddings = await generateEmbeddings(
    extractedCitations.map(c => c.quote)
  );

  // Vector similarity search
  const similarCitations = await vectorDb.search({
    embeddings: citationEmbeddings,
    topK: 5,
    threshold: 0.75
  });

  // Rank by relevance + novelty
  return [
    ...extractedCitations,
    ...similarCitations
  ].sort((a, b) => b.relevance_score - a.relevance_score);
}
```

---

## PHASE 3: FRONTEND DASHBOARD (Weeks 2-3)

### 16. Dashboard Architecture (3 Core Views)

**View 1: Real-Time Rankings**
- Domain name
- Latest AI visibility score (big number)
- 4-model breakdown (badges showing position)
- Sentiment distribution (pie chart)
- Last updated (timestamp)
- "Re-check Now" button (edge case: user triggers manual)

**View 2: Competitor Benchmarking**
- Your domain vs. 3 competitors
- Side-by-side visibility scores
- Model-by-model comparison table
- Trend sparklines (7-day)

**View 3: Citation Intelligence**
- Extracted citations (table)
- Predicted citations (suggested)
- Citation growth chart (7-day)
- "Build Citation" CTA (link to your citation service)

---

### 17. Build Dashboard Shell (Next.js)
```bash
npx create-next-app@latest rankgpt-competitor \
  --typescript \
  --tailwind \
  --no-eslint

cd rankgpt-competitor

npm install \
  @tanstack/react-query \
  recharts \
  shadcn-ui \
  @radix-ui/themes
```

---

### 18. API Integration Layer
```typescript
// web/src/hooks/useRankings.ts

export function useRankings(domainId: string) {
  return useQuery({
    queryKey: ['rankings', domainId],
    queryFn: async () => {
      const res = await fetch(`/api/v1/rankings/${domainId}`);
      if (!res.ok) throw new Error('Failed to fetch rankings');
      return res.json();
    },
    refetchInterval: 300000, // 5 min auto-refresh
    staleTime: 60000 // 1 min
  });
}
```

---

### 19. Real-Time Rankings Component
```typescript
// web/src/components/RankingsCard.tsx

export function RankingsCard({ domain, rankings }) {
  const score = calculateScore(rankings);

  return (
    <div className="border rounded-lg p-6">
      <h2 className="text-lg font-semibold">{domain}</h2>
      
      {/* Big Number */}
      <div className="text-5xl font-bold my-4">{score}/100</div>
      
      {/* Model Breakdown */}
      <div className="grid grid-cols-4 gap-4">
        {rankings.map(r => (
          <div key={r.model} className="text-center">
            <div className="text-2xl font-bold">#{r.position}</div>
            <div className="text-xs text-gray-500">{r.model}</div>
          </div>
        ))}
      </div>

      {/* CTA */}
      <button 
        onClick={() => recheck(domain)}
        className="mt-4 px-4 py-2 bg-blue-600 text-white rounded"
      >
        Re-Check Now
      </button>
    </div>
  );
}
```

---

### 20. Competitor Benchmarking View
```typescript
// web/src/components/CompetitorBench.tsx

export function CompetitorBench({ domains }) {
  return (
    <div>
      <BarChart
        data={domains.map(d => ({
          name: d.name,
          score: d.aiVisibilityScore
        }))}
        layout="vertical"
      >
        <XAxis type="number" />
        <YAxis dataKey="name" type="category" width={100} />
        <Bar dataKey="score" fill="#3b82f6" />
      </BarChart>

      {/* Table: Model-by-model */}
      <table className="w-full mt-6">
        <thead>
          <tr>
            <th>Domain</th>
            <th>ChatGPT</th>
            <th>Claude</th>
            <th>Gemini</th>
            <th>Grok</th>
          </tr>
        </thead>
        <tbody>
          {domains.map(d => (
            <tr key={d.id}>
              <td>{d.name}</td>
              {['openai', 'claude', 'gemini', 'grok'].map(m => (
                <td key={m}>
                  #{getRankingPosition(d, m)}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

### 21. Citation Intelligence View
```typescript
// web/src/components/CitationIntel.tsx

export function CitationIntel({ citations, predicted }) {
  return (
    <div>
      <h3 className="text-lg font-semibold mb-4">Citations Found</h3>

      {/* Extracted */}
      <table className="w-full mb-8">
        <thead>
          <tr className="border-b">
            <th className="text-left">Source</th>
            <th className="text-left">Quote</th>
            <th className="text-right">Relevance</th>
          </tr>
        </thead>
        <tbody>
          {citations.map(c => (
            <tr key={c.id} className="border-b">
              <td>{new URL(c.sourceUrl).hostname}</td>
              <td className="text-sm text-gray-600">{c.quote.slice(0, 60)}...</td>
              <td className="text-right">{(c.relevance * 100).toFixed(0)}%</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Predicted */}
      <h3 className="text-lg font-semibold mb-4">
        Opportunities ({predicted.length})
      </h3>
      {predicted.map(c => (
        <div key={c.id} className="border rounded p-3 mb-2">
          <div className="font-semibold">{c.sourceUrl}</div>
          <div className="text-sm text-gray-600">{c.quote}</div>
          <button className="text-blue-600 text-sm mt-2">Build →</button>
        </div>
      ))}
    </div>
  );
}
```

---

## PHASE 4: AUTH + PAYMENT (Week 3)

### 22. Authentication with Lucia
```bash
npm install lucia @lucia-auth/adapter-prisma
```

```typescript
// api/src/auth/lucia.ts

import { Lucia } from "lucia";
import { PrismaAdapter } from "@lucia-auth/adapter-prisma";
import { db } from "@/db";

export const auth = new Lucia(new PrismaAdapter(db.session, db.user), {
  sessionCookie: {
    attributes: {
      secure: process.env.NODE_ENV === "production",
      httpOnly: true,
      sameSite: "lax"
    }
  }
});

declare module "lucia" {
  interface Register {
    Auth: typeof auth;
    DatabaseUserAttributes: {
      email: string;
      plan: "free" | "pro" | "enterprise";
    };
  }
}
```

---

### 23. Stripe Integration (Simplified)
```typescript
// api/src/billing/stripe.ts

import Stripe from "stripe";

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function createCheckoutSession(userId: string, planId: string) {
  const user = await db.user.findUnique({ where: { id: userId } });

  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    mode: "subscription",
    line_items: [
      {
        price: STRIPE_PRICE_IDS[planId],
        quantity: 1
      }
    ],
    success_url: `${process.env.DOMAIN}/dashboard?success=true`,
    cancel_url: `${process.env.DOMAIN}/pricing`
  });

  return session.url;
}
```

---

### 24. Webhook Handler (Subscription Events)
```typescript
// api/src/webhooks/stripe.ts

export async function handleStripeEvent(event: Stripe.Event) {
  switch (event.type) {
    case "customer.subscription.created":
    case "customer.subscription.updated":
      const subscription = event.data.object as Stripe.Subscription;
      await db.user.update({
        where: { stripeCustomerId: subscription.customer as string },
        data: { plan: getPlanFromPriceId(subscription.items.data[0].price.id) }
      });
      break;

    case "customer.subscription.deleted":
      await db.user.update({
        where: { stripeCustomerId: event.data.object.customer as string },
        data: { plan: "free" }
      });
      break;
  }
}
```

---

## PHASE 5: API KEY + RATE LIMITING (Week 3)

### 25. API Key Management
```typescript
// api/src/middleware/apiKey.ts

export async function validateApiKey(req: Request) {
  const key = req.headers.get("x-api-key");
  if (!key) throw new Error("Missing API key");

  const apiKey = await db.apiKey.findUnique({
    where: { key },
    include: { user: true }
  });

  if (!apiKey) throw new Error("Invalid API key");

  return apiKey.user;
}
```

---

### 26. Rate Limiting (Redis-backed)
```typescript
// api/src/middleware/rateLimit.ts

export async function rateLimit(userId: string, action: string) {
  const key = `ratelimit:${userId}:${action}`;
  const limit = RATE_LIMITS[action]; // e.g., 100 per hour

  const current = await redis.incr(key);
  if (current === 1) {
    await redis.expire(key, 3600);
  }

  if (current > limit) {
    throw new Error(`Rate limit exceeded. Max ${limit} per hour.`);
  }
}
```

---

## PHASE 6: OBSERVABILITY + MONITORING (Week 4)

### 27. Performance Monitoring
```typescript
// api/src/middleware/metrics.ts

export function metricsMiddleware(req: Request, res: Response, next: Function) {
  const start = Date.now();

  res.on("finish", () => {
    const duration = Date.now() - start;
    const label = `${req.method} ${req.path}`;

    // Prometheus-style metrics
    recordHistogram("http_request_duration_ms", duration, {
      method: req.method,
      path: req.path,
      status: res.statusCode
    });

    recordCounter("http_requests_total", 1, {
      method: req.method,
      status: res.statusCode
    });

    // Log slow requests
    if (duration > 500) {
      console.warn(`Slow request: ${label} took ${duration}ms`);
    }
  });

  next();
}
```

---

### 28. Error Tracking (Sentry)
```typescript
// api/src/sentry.ts

import * as Sentry from "@sentry/node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1,
  environment: process.env.NODE_ENV
});

// Attach to Express
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.errorHandler());
```

---

### 29. Logging (Structured)
```typescript
// api/src/logger.ts

import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  transport: {
    target: "pino-pretty",
    options: {
      colorize: true,
      ignore: "pid,hostname"
    }
  }
});

// Usage:
logger.info({ domainId, queryId }, "Ranking completed in 145ms");
```

---

## PHASE 7: DEPLOYMENT PIPELINE (Week 4)

### 30. GitHub Actions CI/CD
```yaml
# .github/workflows/deploy.yml

name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # Frontend
      - name: Build web
        run: cd apps/web && npm install && npm run build
      
      - name: Deploy web to Vercel
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
        run: vercel --prod

      # Backend
      - name: Build & push Docker image
        run: |
          docker build -t rankgpt-api:${{ github.sha }} .
          docker push rankgpt-api:${{ github.sha }}

      - name: Deploy to Railway/Fly
        env:
          DEPLOY_KEY: ${{ secrets.RAILWAY_API_TOKEN }}
        run: railway up
```

---

## COMPETITIVE POSITIONING (Your Unique Selling Point)

### RankGPT vs. Your Product

| Feature | RankGPT | You | Winner |
|---------|---------|-----|--------|
| **Multi-model tracking** | ✓ | ✓ | Tie |
| **Dashboard** | ✓ | ✓ | Tie |
| **API latency** | ~800ms (batched) | **<200ms** ✓ | **You** |
| **Citation intelligence** | Rule-based | **ML-predicted** ✓ | **You** |
| **Real-time updates** | 5-10 min batch | **Cached + 5min refresh** ✓ | **You** |
| **Developer experience** | ? | Clear REST API ✓ | **You** |
| **Pricing transparency** | Demo call | **$29 / $199 / $999** ✓ | **You** |

**Your messaging:**
> *"Real-time AI ranking tracking, not batch processing. Sub-200ms queries, ML-powered citation intelligence, and transparent pricing. Built for developers."*

---

## SUCCESS METRICS (Track These Weekly)

1. **API Latency (p95):** Target < 200ms
2. **Cache Hit Rate:** Target > 60%
3. **Ranking Freshness:** Max 5min stale
4. **Dashboard Load (LCP):** Target < 1.5s
5. **Uptime:** Target > 99.5%
6. **Feature Completion:** 30/30 steps ✓

---

## WEEKLY TIMELINE

| Week | Sprint | Goals |
|------|--------|-------|
| 1 | **Setup + Ranking Engine** | Deploy v1 API, hit <200ms latency, 1 paying customer |
| 2 | **Dashboard + Frontend** | Launch dashboard, 5 paying customers |
| 3 | **Auth + Payment** | Live Stripe, auth, 10 paying customers |
| 4 | **Monitoring + Ops** | Sentry + logging, 20 paying customers, ready to scale |
| 5-8 | **Growth + Refinement** | Optimize based on user feedback, add features based on demand |

---

## What NOT to Do (Time Wasters)

❌ Build AI agent (too complex, unproven ROI)
❌ Custom CSS (use Tailwind + shadcn/ui)
❌ Manual citation building UI (just show opportunities)
❌ Mobile app (web responsive is enough)
❌ Advanced ML (simple TF-IDF + prompt scoring is sufficient)
❌ Multiple payment plans (start with 2: Pro $99/mo, Enterprise custom)

---

## Your First MVP Ship (Week 1 Deliverable)

```bash
# By end of Week 1, you ship:

1. API endpoint: POST /api/v1/rank
   - Input: domain, query, models[]
   - Output: sub-200ms ranking with citations
   - Lives at: yourdomain.com/api/v1/rank

2. Frontend: Basic auth + 1 dashboard page
   - Shows your domain's rankings
   - Shows competitor comparison
   - "Re-check" button

3. Stripe integration (live)
   - Sign up → Stripe checkout → Pro plan ($99/mo)

4. Monitoring (basic)
   - Sentry errors
   - Response time logs

5. Documentation
   - API docs (Swagger)
   - Getting started guide
```

---

## Differentiation Hook

Your killer advantage: **Speed + Transparency**

- **Speed:** 200ms vs. their 800ms+ = customers see results faster
- **Transparency:** API-first, documented, pricing visible = developers prefer you
- **Citation Intelligence:** Actionable, ML-ranked suggestions = customers get more value

Ship this, and you have a real product that **beats RankGPT on execution**, even if you're months behind on feature parity.

**Go ship Week 1.**

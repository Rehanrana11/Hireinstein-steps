# QUICK REFERENCE: DECISION TREE + WHEN YOU'RE STUCK

## Use This When You Don't Know What to Do Next

---

## Q: "Should I do X feature?"

### Flowchart

```
Do I need this to get my first paying customer?
  ├─ YES → Do it now
  └─ NO → Is it <30 min of work?
      ├─ YES → Do it after MVP
      └─ NO → Skip it for 4+ weeks
```

### Examples

**Feature: Dark mode**
- Gets first customer? NO
- <30 min of work? NO
- **Decision: SKIP (Weeks 5+)**

**Feature: Email validation with Zod**
- Gets first customer? YES (prevents bad signups)
- <30 min of work? YES
- **Decision: DO NOW (Week 1)**

**Feature: Citation ranking by ML**
- Gets first customer? YES (your moat)
- <30 min of work? NO (1-2 hours)
- **Decision: DO NOW (Week 1)**

**Feature: Automated incident playbooks**
- Gets first customer? NO
- <30 min of work? NO
- **Decision: SKIP (After you have paying customers)**

---

## Q: "I'm on Day X and stuck. What do I do?"

### By Day of Week

#### MONDAY (Setup Day)
**If stuck on:** Setting up Prisma
- Read: `RankGPT-Competitive-Breakdown.md` → "PHASE 1 — RANKING ENGINE"
- Run: `npx prisma init` (it walks you through)
- If still stuck: Use ChatGPT with "I need a Postgres schema for..."

**If stuck on:** Environment variables
- Copy the `.env.local` template from `Week1-Execution-Checklist.md`
- Fill in your actual API keys
- Test with `echo $DATABASE_URL` in terminal

**If stuck on:** Monorepo structure
- Don't overthink it. Just do:
```
mkdir apps && mkdir packages
cd apps && npm create vite@latest web
cd ../.. && npm install
```

**Time checkpoint:** By end of Monday, you should have:
- [ ] GitHub repo initialized
- [ ] Monorepo structure created
- [ ] `.env.local` populated
- [ ] Prisma schema written
- [ ] Local database running (`npx prisma migrate dev`)

If you don't have all 4, **stop other work and finish these.**

---

#### TUESDAY (Ranking Engine Day)
**If stuck on:** Express server not starting
```bash
# Checklist:
npm install express typescript @types/express ts-node
npx tsc --init
touch api/src/index.ts
# Make sure PORT is defined: process.env.PORT || 3001
npm run dev
```

**If stuck on:** API call to OpenAI/Claude timing out
- Add timeout in SDK config:
```typescript
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  timeout: 30000 // 30 seconds
});
```

**If stuck on:** Parse JSON from LLM response
- LLMs can be inconsistent. Use this pattern:
```typescript
try {
  const parsed = JSON.parse(response);
  return parsed;
} catch {
  // Fallback: return default values
  return { mentioned: false, position: null };
}
```

**If stuck on:** Making API calls in parallel
- Use Promise.all():
```typescript
const results = await Promise.all([
  callOpenAI(domain, query),
  callClaude(domain, query),
  callGemini(domain, query)
]);
```

**Time checkpoint:** By end of Tuesday, you should have:
- [ ] Express server running on http://localhost:3001
- [ ] POST /api/v1/rank endpoint responding
- [ ] Actual ranking from 2+ models (OpenAI + Claude)
- [ ] Response time < 300ms (even if not cached)

If you don't have all 4, **don't move to frontend. Fix the API first.**

---

#### WEDNESDAY (Auth + Billing Day)
**If stuck on:** Lucia authentication
- Start fresh with the Lucia template:
```bash
npm create lucia-auth@latest
# Then integrate into your Express app
```

**If stuck on:** Password hashing
```typescript
import * as argon2 from "argon2";

const hashedPassword = await argon2.hash(plaintext);
const valid = await argon2.verify(hashedPassword, plaintext);
```

**If stuck on:** Stripe webhook signature verification
- Your STRIPE_WEBHOOK_SECRET must be the signing secret, not the API key
- Test webhook locally with: `stripe listen --forward-to localhost:3001/webhook/stripe`

**If stuck on:** Session management
- Don't build custom sessions. Use Lucia:
```typescript
const session = await auth.createSession(userId, {});
// Lucia handles httpOnly cookies automatically
```

**Time checkpoint:** By end of Wednesday, you should have:
- [ ] Sign up endpoint working
- [ ] Login endpoint working
- [ ] Session persists after login
- [ ] Stripe checkout creates a customer

If you don't have all 4, **test each endpoint individually before moving on.**

---

#### THURSDAY (Dashboard Day)
**If stuck on:** Next.js fetch from Express backend
```typescript
// In Next.js App Router component (use 'use client')
const res = await fetch("http://localhost:3001/api/v1/rank", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ domain, query, models })
});
```

**If stuck on:** CORS errors
- Add to Express:
```typescript
import cors from "cors";
app.use(cors({
  origin: process.env.NODE_ENV === "production" 
    ? "https://yourdomain.com" 
    : "http://localhost:3000"
}));
```

**If stuck on:** Tailwind not working
```bash
# Verify your tailwind.config.js has:
content: [
  "./src/**/*.{js,ts,jsx,tsx}",
]
# Then: npm run dev (watch mode)
```

**If stuck on:** React Query / data fetching
- Start simple without React Query:
```typescript
const [data, setData] = useState(null);
useEffect(() => {
  fetch(...).then(r => r.json()).then(setData);
}, []);
```
- Add React Query later if you want.

**Time checkpoint:** By end of Thursday, you should have:
- [ ] Dashboard page loads
- [ ] Can call API from dashboard
- [ ] Rankings display in UI
- [ ] Competitor comparison table renders

If you don't have all 4, **debug each piece separately.**

---

#### FRIDAY (Deployment Day)
**If stuck on:** Deploying to Vercel
```bash
npm i -g vercel
vercel login
vercel --prod
# Follow the prompts
```

**If stuck on:** Deploying to Railway/Fly
- Create account on Railway.app or Fly.io
- Connect your GitHub repo
- It auto-deploys on push

**If stuck on:** Environment variables not working in production
- Make sure secrets are set in Vercel/Railway dashboard
- Don't use .env.local in production. Use dashboard.

**If stuck on:** Database not accessible from production
- Railway/Fly gives you a DATABASE_URL. Copy it exactly.
- Run `npx prisma migrate deploy` after deployment.

**If stuck on:** CORS between frontend and backend in production
- Update origin in Express CORS config:
```typescript
const origin = process.env.NODE_ENV === "production"
  ? process.env.FRONTEND_URL // "https://yourdomain.vercel.app"
  : "http://localhost:3000";
app.use(cors({ origin }));
```

**Time checkpoint:** By end of Friday, you should have:
- [ ] Frontend deployed to Vercel
- [ ] Backend deployed to Railway/Fly
- [ ] Live sign-up working
- [ ] Live ranking check working
- [ ] Stripe live (not test mode)

---

## Decision Tree: "What do I prioritize?"

```
You're running out of time. Pick one:

1. "API is slow" 
   → Add Redis caching (1 hour)
   → Parallelize model calls (30 min)
   → Cache aggressively (15 min)

2. "Dashboard looks ugly"
   → Add one nice chart with Recharts (1 hour)
   → Color the ranking cards (15 min)
   → Done. Ship it.

3. "I don't have customers"
   → You have a free tier, right? Yes → Skip this
   → You need testimonials? → Ask 3 beta users
   → You need a sales page? → Use your homepage

4. "I'm worried about bugs"
   → Manually test critical paths (1 hour)
   → Don't write unit tests. Just test it yourself.
   → Ship it with bugs. Fix them next week.

5. "Should I add feature X?"
   → Read: Does it help get first customer? 
      ├─ YES → Do it (if <2 hours)
      └─ NO → Skip. Do after MVP.
```

---

## Decision Tree: "Is This Good Enough?"

```
Your question: "Is this [thing] production-ready?"

Answer checklist:

1. Does it work? 
   └─ YES → Go to 2
   └─ NO → Fix it

2. Can I charge money for it?
   └─ YES → Go to 3
   └─ NO → Make it more valuable

3. Would I be embarrassed if my mom saw it?
   └─ NO (it's fine) → Ship it
   └─ YES → Spend 1 hour making it look less bad
      └─ Then ship it anyway

4. Can users understand how to use it?
   └─ YES → Ship it
   └─ NO → Add 1 sentence of explanation
      └─ Then ship it
```

---

## Debugging by Symptom

### "My API returns 200 but the response is wrong"

1. **Check LLM prompt:** Is it clear what you're asking?
   ```typescript
   // Bad: "Rank this domain"
   // Good: "Is 'example.com' mentioned in response to 'best software'? Respond in JSON with: {mentioned: boolean, position: number | null}"
   ```

2. **Check JSON parsing:** Is the LLM returning valid JSON?
   ```typescript
   console.log("Raw response:", response);
   // If it's malformed, add error handling
   ```

3. **Check model:** Is the right model being called?
   ```typescript
   console.log("Using model:", model);
   ```

### "Caching isn't working"

1. Check Redis connection:
   ```bash
   redis-cli ping
   # Should return: PONG
   ```

2. Check cache key:
   ```typescript
   console.log("Cache key:", cacheKey);
   // Make sure it's consistent
   ```

3. Check TTL:
   ```typescript
   // Is it actually setting the TTL?
   console.log("Setting expire:", ttl);
   ```

### "Database won't connect"

1. Check DATABASE_URL:
   ```bash
   echo $DATABASE_URL
   # Should look like: postgresql://user:pass@host/db
   ```

2. Test connection:
   ```bash
   npx prisma db push
   # This will tell you if it can connect
   ```

3. Check Prisma client:
   ```typescript
   import { PrismaClient } from "@prisma/client";
   const db = new PrismaClient();
   db.$connect();
   ```

### "Stripe isn't charging"

1. Check webhook signature:
   ```typescript
   console.log("Webhook signature:", sig);
   // Should be the header from Stripe
   ```

2. Use test mode:
   ```bash
   # Use Stripe test keys: pk_test_... and sk_test_...
   # Test card: 4242 4242 4242 4242
   ```

3. Check stripe-cli tunnel:
   ```bash
   stripe listen --forward-to localhost:3001/webhook
   # Get the signing secret from the output
   ```

---

## Your "I'm Panicking" Checklist

**If you're 3 days in and nothing works:**

- [ ] Can you deploy? (Even if broken?)
  - YES → Deploy immediately. Fix in production.
  - NO → Stop everything. Fix deploy first.

- [ ] Can you sign up?
  - YES → Move forward
  - NO → Fix auth

- [ ] Can you call the API?
  - YES → Continue
  - NO → Debug API endpoint

- [ ] Can you see rankings?
  - YES → You're ahead of schedule
  - NO → Add basic output

**Do all 4 things work? You're on track. Ship it even if broken.**

---

## Time Management During Week 1

**Each day has a "hard stop" time.**

| Day | Hard Stop | What to Do |
|-----|-----------|-----------|
| Mon | By 11pm | Database + Prisma working, can run `npx prisma migrate dev` |
| Tue | By 11pm | API endpoint responds with ranking. Test with curl. |
| Wed | By 11pm | Can sign up. Can log in. Session persists. |
| Thu | By 11pm | Dashboard loads. Shows any ranking data (fake is OK). |
| Fri | By 5pm | Live on Vercel. Live on Railway. Can sign up and check ranking. |

**If you don't hit the hard stop:**
- Skip your current task
- Move to next day's work
- Finish current task on the next free hour

**Do NOT stay up late. You'll get diminishing returns and make mistakes.**

---

## When to Ask for Help

**Stuck for > 15 minutes on a single thing?**

→ **Search for the exact error + framework name**

Examples:
- "NextAuth session not persisting"
- "Prisma migration fails on Heroku"
- "Express CORS error with Vercel"

**Still stuck after 15 more minutes?**

→ **Generate a minimal reproducible example and ask in Discord/Reddit**

**Stuck for > 1 hour?**

→ **Abandon that approach. Use a different library or method.**

Examples:
- "Can't get authentication working with X?" → Switch to Y
- "Redis client is slow?" → Use in-memory cache for MVP
- "Stripe webhook not working?" → Poll instead of webhook

---

## MVP Cutoffs (When to Stop Polishing)

**Stop working on X when you have:**

| Feature | MVP Cutoff |
|---------|-----------|
| API latency | <200ms (cached) or <500ms (uncached) |
| Dashboard UI | Can see top 3 results. That's it. |
| Auth | Signup, login, logout. No password reset. |
| Stripe | Can charge money. That's it. No refunds UI. |
| Error handling | Shows *something* when broken. |
| Testing | Manually tested. No unit tests needed. |
| Documentation | README that says "yarn install && yarn dev" |

**Everything else is "nice to have".**

---

## Your Competitive Advantage During Execution

**RankGPT would take 2+ months to do your Week 1.**

Why?
- They over-engineer design systems
- They build features nobody needs yet
- They write tests for everything
- They plan for scale day one

**You move 3-4x faster because:**
- You use Tailwind defaults
- You build only what gets customers
- You test manually
- You scale when you have customers to scale for

**That's your real moat.** Not the code. The speed.

**Ship fast. Iterate based on feedback. Win.**

---

## Last-Minute Panic Fixes

**It's Friday 4pm and you're not done. What do you cut?**

**Priority order (top = keep, bottom = cut):**

1. ✅ **API endpoint** - This IS your product
2. ✅ **Auth** - Can't charge without it
3. ✅ **Stripe** - Can't get paid without it
4. ✅ **Deployment** - Can't launch without it
5. 🟡 **Dashboard** - Can be simple
6. 🟡 **Competitor comparison** - Launch without it
7. ❌ **Citation intelligence** - Cut for now
8. ❌ **Monitoring** - Cut for now
9. ❌ **SEO/analytics** - Cut for now

**If you only have auth + API + Stripe live: You can charge money. That's a business.**

---

## You've Got This

**Remember:**

- RankGPT took months to launch
- You're doing it in weeks
- Your code won't be perfect
- **Perfect code that nobody uses = worthless**
- Shipped MVP that customers love = priceless

**One more thing:** 

Every time you think "I should do this properly," ask:

> "Will this help me get my first paying customer?"

If NO → Don't do it.

If YES → Do it in the fastest, dumbest way possible.

Then ship.

**That's the game. Now go win it.**

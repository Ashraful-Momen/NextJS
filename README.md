# Next.js Complete Cheatsheet

---

## 1. App Router Structure

```
app/
 ├── layout.js          # Root layout (wraps all pages)
 ├── page.js            # Home page (/)
 ├── loading.js         # Loading UI
 ├── error.js           # Error boundary
 ├── not-found.js       # 404 page
 ├── template.js        # Re-renders on every navigation
 ├── default.js         # Fallback for parallel routes
 ├── globals.css
 ├── favicon.ico
```

---

## 2. Routing

```js
// Static Route
app/about/page.js          → /about

// Dynamic Route
app/users/[id]/page.js     → /users/123
export default function Page({ params }) {
  return <h1>{params.id}</h1>;
}

// Catch All Route
app/docs/[...slug]/page.js → /docs/a/b/c

// Optional Catch All
app/blog/[[...slug]]       → /blog  (also matches /blog/a/b)

// Route Group (doesn't affect URL)
app/(auth)/login/page.js   → /login
app/(auth)/register/page.js → /register

// Parallel Routes
app/dashboard/@modal/page.js
app/dashboard/layout.js → receives `modal` prop

// Intercepting Routes
app/dashboard/@modal/(.)photo/[id]/page.js  → intercepts /photo/[id]
```

### Generate Static Params

```js
export async function generateStaticParams() {
  const users = await db.user.findMany();
  return users.map(u => ({ id: String(u.id) }));
}
```

### Search Params

```js
// Server Component
export default function Page({ searchParams }) {
  console.log(searchParams.page); // ?page=2
}

// Client Component
"use client";
import { useSearchParams } from "next/navigation";
const params = useSearchParams();
```

---

## 3. Layouts & Templates

```js
// Root Layout (required)
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

// Nested Layout (persists across navigation)
export default function DashboardLayout({ children }) {
  return (
    <div className="flex">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}

// Template (re-renders every navigation, unlike layout)
// app/template.js
```

---

## 4. Navigation

```js
import Link from "next/link";
<Link href="/about">About</Link>
<Link href={`/users/${id}`}>User</Link>

// Programmatic (client only)
"use client";
import { useRouter, usePathname, useSearchParams } from "next/navigation";

const router = useRouter();
router.push("/dashboard");   // navigate
router.replace("/login");    // replace history
router.back();               // go back
router.refresh();            // refresh server data

const pathname = usePathname(); // "/dashboard"
const searchParams = useSearchParams(); // URLSearchParams

// Active link check
<Link className={pathname === "/dashboard" ? "active" : ""}>
```

---

## 5. Server vs Client Components

```js
// Server Component (default) — runs on server only
// ✅ Direct DB access, env vars, no JS sent to client
export default async function Page() {
  const users = await prisma.user.findMany();
  return <UserList users={users} />;
}

// Client Component — runs in browser
"use client";
import { useState } from "react";
export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

| Use Server | Use Client |
|-----------|-----------|
| DB queries | `useState`, `useEffect` |
| Secrets / env vars | `onClick`, `onChange` |
| SEO-critical content | Browser APIs |
| Large deps (no bundle cost) | `useRouter`, `useSearchParams` |

---

## 6. Data Fetching & Rendering

```js
// SSR — Server-side rendered (default for async components)
export default async function Page() {
  const data = await fetch("https://api.example.com/data");
  return <div>{data}</div>;
}

// Cache Control
await fetch(url, { cache: "no-store" });           // always fresh
await fetch(url, { next: { revalidate: 60 } });    // revalidate every 60s

// Route Segment Config
export const revalidate = 60;                 // ISR
export const dynamic = "force-dynamic";       // always SSR
export const dynamic = "force-static";        // static at build
export const dynamicParams = false;           // 404 for non-generated params

// Parallel Fetching
const [users, posts] = await Promise.all([
  prisma.user.findMany(),
  prisma.post.findMany(),
]);

// Streaming with Suspense
import { Suspense } from "react";
<Suspense fallback={<LoadingSkeleton />}>
  <AsyncDataTable />
</Suspense>
```

---

## 7. SWR (Client-side Data Fetching)

```js
import useSWR from "swr";

const fetcher = (url) => fetch(url).then(r => r.json());

function Users() {
  const { data, error, isLoading, mutate } = useSWR("/api/users", fetcher);

  if (error) return <div>Failed</div>;
  if (isLoading) return <div>Loading...</div>;

  return (
    <ul>
      {data.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}

// After mutation, revalidate
mutate();                          // re-fetch same key
mutate("/api/users");              // re-fetch specific key

// SWR Config (global)
<SWRConfig value={{ fetcher, revalidateOnFocus: false }}>
  <App />
</SWRConfig>
```

---

## 8. TanStack Query (Alternative to SWR)

```js
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

// Read
const { data } = useQuery({
  queryKey: ["users"],
  queryFn: () => fetch("/api/users").then(r => r.json()),
});

// Write
const mutation = useMutation({
  mutationFn: (newUser) => fetch("/api/users", { method: "POST", body: JSON.stringify(newUser) }),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ["users"] }),
});

// Infinite Scroll
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ["posts"],
  queryFn: ({ pageParam = 1 }) => fetch(`/api/posts?page=${pageParam}`),
  getNextPageParam: (lastPage) => lastPage.nextPage,
});
```

---

## 9. Prisma ORM

```bash
npm install prisma @prisma/client
npx prisma init
npx prisma migrate dev --name init
npx prisma generate
npx prisma studio          # GUI editor
npx prisma db push         # push schema without migration
```

### Schema

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  profile   Profile?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?          // soft delete
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  content  String?
  author   User   @relation(fields: [authorId], references: [id])
  authorId Int
  tags     Tag[]

  @@index([authorId])         // indexing
  @@map("posts")              // custom table name
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### Singleton Pattern

```js
// lib/prisma.js
import { PrismaClient } from "@prisma/client";
const globalForPrisma = globalThis;
const prisma = globalForPrisma.prisma || new PrismaClient();
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
export default prisma;
```

### CRUD Operations

```js
// Create
await prisma.user.create({ data: { email: "a@b.com", name: "Ash" } });

// Read
await prisma.user.findMany({ where: { role: "ADMIN" } });
await prisma.user.findUnique({ where: { email: "a@b.com" } });
await prisma.user.findFirst({ where: { name: { contains: "ash" } } });

// Update
await prisma.user.update({ where: { id: 1 }, data: { name: "New" } });
await prisma.user.updateMany({ where: { role: "USER" }, data: { role: "ADMIN" } });

// Delete
await prisma.user.delete({ where: { id: 1 } });
await prisma.user.deleteMany({ where: { role: "USER" } });

// Relations
await prisma.user.findMany({
  include: { posts: { include: { tags: true } } },
});

// Transactions
await prisma.$transaction([
  prisma.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } }),
  prisma.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } }),
]);

// Pagination — Offset
const users = await prisma.user.findMany({ skip: 10, take: 10 });

// Pagination — Cursor
const users = await prisma.user.findMany({
  take: 10,
  cursor: { id: lastId },
  orderBy: { id: "asc" },
});

// Aggregation
await prisma.user.count({ where: { role: "ADMIN" } });
await prisma.order.aggregate({ _sum: { amount: true } });

// Raw SQL
await prisma.$queryRaw`SELECT * FROM users WHERE id = ${id}`;
```

---

## 10. Route Handlers (API Routes)

```
app/api/users/route.js         → /api/users
app/api/users/[id]/route.js    → /api/users/:id
```

```js
// GET
export async function GET(request) {
  const { searchParams } = new URL(request.url);
  const page = searchParams.get("page") || "1";
  const users = await prisma.user.findMany({ skip: (+page - 1) * 10, take: 10 });
  return Response.json({ users, page: +page });
}

// POST
export async function POST(request) {
  const body = await request.json();
  const user = await prisma.user.create({ data: body });
  return Response.json(user, { status: 201 });
}

// PUT
export async function PUT(request, { params }) {
  const body = await request.json();
  const user = await prisma.user.update({ where: { id: +params.id }, data: body });
  return Response.json(user);
}

// DELETE
export async function DELETE(request, { params }) {
  await prisma.user.delete({ where: { id: +params.id } });
  return Response.json({ success: true });
}

// Streaming Response
export async function GET() {
  const stream = new ReadableStream({ start(controller) { ... } });
  return new Response(stream, { headers: { "Content-Type": "text/event-stream" } });
}
```

---

## 11. Forms & Validation

```js
// Basic Server Action Form
"use server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { z } from "zod";

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
});

export async function createUser(prevState, formData) {
  const validated = schema.safeParse({
    email: formData.get("email"),
    name: formData.get("name"),
  });

  if (!validated.success) {
    return { errors: validated.error.flatten().fieldErrors };
  }

  await prisma.user.create({ data: validated.data });
  revalidatePath("/users");
  redirect("/users");
}

// Client Component
"use client";
import { useFormState, useFormStatus } from "react-dom";

export default function CreateUserForm() {
  const [state, formAction] = useFormState(createUser, null);
  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <input name="name" required />
      <SubmitButton />
      {state?.errors && <p>{state.errors.email}</p>}
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "Saving..." : "Create"}</button>;
}
```

### React Hook Form + Zod

```js
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(6, "Min 6 chars"),
});

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data) => {
    await fetch("/api/auth/login", { method: "POST", body: JSON.stringify(data) });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}
      <input type="password" {...register("password")} />
      {errors.password && <span>{errors.password.message}</span>}
      <button type="submit">Login</button>
    </form>
  );
}
```

### File Upload

```js
// Client
const formData = new FormData();
formData.append("file", fileInput.files[0]);
await fetch("/api/upload", { method: "POST", body: formData });

// Server
export async function POST(request) {
  const formData = await request.formData();
  const file = formData.get("file");
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  // Save to S3, MinIO, Cloudinary, or local disk
}
```

---

## 12. Authentication

```js
// Auth.js (NextAuth v5) — app/api/auth/[...nextauth]/route.js
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import GitHub from "next-auth/providers/github";
import Google from "next-auth/providers/google";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Credentials({
      async authorize(credentials) {
        const user = await prisma.user.findUnique({ where: { email: credentials.email } });
        if (user && bcrypt.compareSync(credentials.password, user.password)) return user;
        return null;
      },
    }),
    GitHub, Google,
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.role = user.role;
      return token;
    },
    async session({ session, token }) {
      session.user.role = token.role;
      return session;
    },
  },
  pages: { signIn: "/login" },
});

export { handlers as GET, handlers as POST };
```

### JWT + HttpOnly Cookies (Manual)

```js
import { cookies } from "next/headers";
import jwt from "jsonwebtoken";
import bcrypt from "bcryptjs";

// Login
const token = jwt.sign({ userId: user.id, role: user.role }, process.env.JWT_SECRET, { expiresIn: "7d" });
cookies().set("token", token, {
  httpOnly: true, secure: true, sameSite: "strict", maxAge: 60 * 60 * 24 * 7, path: "/",
});

// Read cookie
const token = cookies().get("token")?.value;
const decoded = jwt.verify(token, process.env.JWT_SECRET);

// Delete cookie (logout)
cookies().delete("token");
```

---

## 13. Middleware

```js
// middleware.js (root or src/)
import { NextResponse } from "next/server";

export function middleware(request) {
  const token = request.cookies.get("token")?.value;
  const { pathname } = request.nextUrl;

  // Public routes
  if (pathname.startsWith("/login") || pathname.startsWith("/register")) {
    if (token) return NextResponse.redirect(new URL("/dashboard", request.url));
    return NextResponse.next();
  }

  // Protected routes
  if (!token) return NextResponse.redirect(new URL("/login", request.url));

  // Role-based check (decode JWT)
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  if (pathname.startsWith("/admin") && decoded.role !== "ADMIN") {
    return NextResponse.redirect(new URL("/unauthorized", request.url));
  }

  // Add custom header
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-user-id", decoded.userId);
  return NextResponse.next({ request: { headers: requestHeaders } });
}

export const config = {
  matcher: ["/dashboard/:path*", "/admin/:path*", "/login", "/register"],
};
```

---

## 14. Zustand (State Management)

```js
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";

// Basic Store
const useStore = create((set, get) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Persisted Store
const useAuthStore = create(
  persist(
    (set) => ({
      user: null,
      setUser: (user) => set({ user }),
      logout: () => set({ user: null }),
    }),
    { name: "auth-storage" }
  )
);

// Slice Pattern (large projects)
const useCartStore = create((...a) => ({
  ...createCartSlice(...a),
  ...createUISlice(...a),
}));
```

---

## 15. Optimistic UI & Transitions

```js
// useOptimistic — instant UI feedback
"use client";
import { useOptimistic } from "react";

function LikeButton({ postId, initialLikes }) {
  const [optimisticLikes, addOptimistic] = useOptimistic(
    initialLikes,
    (state, newLike) => state + (newLike ? 1 : -1)
  );

  async function handleLike() {
    addOptimistic(true);
    await fetch(`/api/posts/${postId}/like`, { method: "POST" });
  }

  return <button onClick={handleLike}>{optimisticLikes} Likes</button>;
}

// useTransition — non-blocking updates
import { useTransition } from "react";

function SearchResults() {
  const [isPending, startTransition] = useTransition();

  function handleSearch(query) {
    startTransition(async () => {
      const results = await fetchResults(query);
      setResults(results);
    });
  }

  return <input onChange={(e) => handleSearch(e.target.value)} disabled={isPending} />;
}
```

---

## 16. Caching

```js
// Request-level cache
fetch(url, { cache: "no-store" });             // no cache (always fresh)
fetch(url, { cache: "force-cache" });          // force cache
fetch(url, { next: { revalidate: 60 } });      // revalidate every 60s

// Cache Tags (on-demand invalidation)
fetch(url, { next: { tags: ["users"] } });

// In Server Action or Route Handler
import { revalidateTag, revalidatePath } from "next/cache";
revalidateTag("users");        // invalidate all fetches with tag "users"
revalidatePath("/users");      // invalidate entire path
revalidatePath("/users", "layout");  // invalidate layout and children

// Router Cache (client) — cleared on:
// router.refresh(), revalidatePath(), revalidateTag()

// Full Route Cache — controlled by:
// dynamic = "force-dynamic"  → disables
// cookies(), headers()       → opts out
// revalidate = 0 or false    → disables
```

---

## 17. Metadata & SEO

```js
// Static Metadata
export const metadata = {
  title: "My App",
  description: "Description here",
  keywords: ["next.js", "react"],
  openGraph: {
    title: "My App",
    description: "...",
    images: ["/og.png"],
  },
  twitter: { card: "summary_large_image" },
  robots: { index: true, follow: true },
};

// Dynamic Metadata
export async function generateMetadata({ params }) {
  const post = await prisma.post.findUnique({ where: { id: +params.id } });
  return { title: post.title, description: post.excerpt };
}

// Viewport
export const viewport = { themeColor: "#000", width: "device-width" };
export function generateViewport({ params }) { return { ... }; }

// Sitemap — app/sitemap.js
export default async function sitemap() {
  const posts = await prisma.post.findMany();
  return posts.map(p => ({ url: `https://example.com/posts/${p.id}`, lastModified: p.updatedAt }));
}

// Robots — app/robots.js
export default function robots() {
  return { rules: { userAgent: "*", allow: "/", disallow: "/api/" }, sitemap: "https://example.com/sitemap.xml" };
}
```

---

## 18. Image, Font & Link Optimization

```js
import Image from "next/image";
<Image
  src="/hero.jpg"              // or external URL
  alt="Hero"
  width={800}
  height={600}
  fill                         // fill parent container
  priority                     // preload above-the-fold images
  placeholder="blur"
  blurDataURL="..."
/>;

// External images config — next.config.js
images: { domains: ["res.cloudinary.com", "cdn.example.com"] }

// Fonts
import { Inter } from "next/font/google";
const inter = Inter({ subsets: ["latin"], display: "swap" });
<body className={inter.className}>

// Dynamic Import
import dynamic from "next/dynamic";
const Chart = dynamic(() => import("./Chart"), { loading: () => <p>Loading...</p>, ssr: false });
```

---

## 19. Error Handling

```js
// error.js — must be client component
"use client";
export default function Error({ error, reset }) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// loading.js
export default function Loading() {
  return <div className="animate-pulse">Loading...</div>;
}

// not-found.js
import Link from "next/link";
export default function NotFound() {
  return (
    <div>
      <h2>Not Found</h2>
      <Link href="/">Go Home</Link>
    </div>
  );
}

// global-error.js (catches root layout errors)
"use client";
export default function GlobalError({ error, reset }) {
  return <html><body><h2>App Error</h2><button onClick={reset}>Retry</button></body></html>;
}
```

---

## 20. Environment Variables

```bash
# .env.local (never commit)
DATABASE_URL="postgresql://user:pass@localhost:5432/mydb"
JWT_SECRET="super-secret"
NEXT_PUBLIC_API_URL="https://api.example.com"   # exposed to client
```

```js
// Server only
process.env.DATABASE_URL
process.env.JWT_SECRET

// Client (must start with NEXT_PUBLIC_)
process.env.NEXT_PUBLIC_API_URL
```

---

## 21. Security Checklist

```js
// Rate Limiting (use upstash/ratelimit or custom)
import { Ratelimit } from "@upstash/ratelimit";
const ratelimit = new Ratelimit({ redis, limiter: Ratelimit.slidingWindow(10, "10 s") });

// CORS
export async function OPTIONS(request) {
  return new Response(null, {
    headers: {
      "Access-Control-Allow-Origin": "https://example.com",
      "Access-Control-Allow-Methods": "GET, POST",
      "Access-Control-Allow-Headers": "Content-Type",
    },
  });
}

// Helmet equivalent — security headers in middleware
const response = NextResponse.next();
response.headers.set("X-Frame-Options", "DENY");
response.headers.set("X-Content-Type-Options", "nosniff");
response.headers.set("Referrer-Policy", "strict-origin-when-cross-origin");

// Input Sanitization
import DOMPurify from "isomorphic-dompurify";
const clean = DOMPurify.sanitize(userInput);

// CSRF — use SameSite cookies + verify Origin header
```

---

## 22. Docker

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

### next.config.js for Docker

```js
module.exports = {
  output: "standalone",   // required for Docker
  images: { domains: ["..."] },
};
```

---

## 23. Testing

```bash
# Unit & Integration
npm install -D vitest @testing-library/react @testing-library/jest-dom

# E2E
npm install -D playwright
npx playwright install
```

```js
// vitest.config.js
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
export default defineConfig({ plugins: [react()], test: { environment: "jsdom" } });

// Unit Test — __tests__/sum.test.js
import { expect, test } from "vitest";
test("adds numbers", () => { expect(1 + 2).toBe(3); });

// Component Test
import { render, screen } from "@testing-library/react";
test("renders greeting", () => {
  render(<Greeting name="Ash" />);
  expect(screen.getByText("Hello Ash")).toBeInTheDocument();
});

// E2E Test — e2e/auth.spec.js (Playwright)
import { test, expect } from "@playwright/test";
test("login flow", async ({ page }) => {
  await page.goto("/login");
  await page.fill('[name="email"]', "test@example.com");
  await page.fill('[name="password"]', "password123");
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL("/dashboard");
});
```

---

## 24. WebSocket / Socket.IO

```bash
npm install socket.io socket.io-client
```

```js
// Server — server.js (custom server or API route workaround)
import { Server } from "socket.io";
const io = new Server(httpServer, { cors: { origin: "*" } });
io.on("connection", (socket) => {
  console.log("User connected:", socket.id);
  socket.on("message", (msg) => { io.emit("message", msg); });
  socket.on("disconnect", () => { console.log("User disconnected"); });
});

// Client
import { io } from "socket.io-client";
const socket = io("http://localhost:3001");
socket.on("message", (msg) => { setMessages((prev) => [...prev, msg]); });
socket.emit("message", "Hello!");
```

---

## 25. Email

```js
// Nodemailer
import nodemailer from "nodemailer";
const transporter = nodemailer.createTransport({ host: "smtp.gmail.com", port: 465, auth: { user: process.env.EMAIL, pass: process.env.EMAIL_PASS } });
await transporter.sendMail({ from: process.env.EMAIL, to: "user@example.com", subject: "Welcome", html: "<h1>Hello!</h1>" });

// Resend
import Resend from "resend";
const resend = new Resend(process.env.RESEND_API_KEY);
await resend.emails.send({ from: "noreply@example.com", to: "user@example.com", subject: "Verify", html: "<p>Click to verify</p>" });
```

---

## 26. File Storage (S3 / MinIO)

```js
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: process.env.AWS_REGION });

// Upload
await s3.send(new PutObjectCommand({ Bucket: "my-bucket", Key: `uploads/${filename}`, Body: buffer, ContentType: file.type }));

// Presigned URL (temporary access)
const url = await getSignedUrl(s3, new GetObjectCommand({ Bucket: "my-bucket", Key: "uploads/file.pdf" }), { expiresIn: 3600 });
```

---

## 27. Background Jobs / Queue

```js
// BullMQ
import { Queue, Worker } from "bullmq";

const emailQueue = new Queue("emails", { connection: redis });
await emailQueue.add("welcome", { to: "user@example.com", template: "welcome" });

const worker = new Worker("emails", async (job) => {
  await sendEmail(job.data);
}, { connection: redis });
```

---

## 28. CI/CD — GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: npm ci
      - run: npx prisma generate
      - run: npm run build
      - run: npm test
      - name: Deploy
        run: docker build -t myapp . && docker push myregistry/myapp:latest
```

---

## 29. Monitoring & Logging

```js
// Pino (structured logging)
import pino from "pino";
const logger = pino();
logger.info({ userId: 123 }, "User logged in");
logger.error({ err }, "Database connection failed");

// Sentry (error tracking)
import * as Sentry from "@sentry/nextjs";
Sentry.init({ dsn: process.env.NEXT_PUBLIC_SENTRY_DSN, tracesSampleRate: 1.0 });
```

---

## 30. Project Folder Structure

```
src/
 ├── app/                    # Next.js App Router pages & layouts
 │   ├── (auth)/            # Route group
 │   ├── (dashboard)/
 │   ├── api/               # Route handlers
 │   ├── layout.js
 │   └── page.js
 ├── components/             # Reusable UI components
 │   ├── ui/                # Buttons, Cards, Inputs
 │   └── forms/             # Form components
 ├── lib/                    # Utilities, prisma client, helpers
 ├── services/               # Business logic layer
 ├── repositories/           # Data access layer
 ├── hooks/                  # Custom React hooks
 ├── store/                  # Zustand stores
 ├── validations/            # Zod schemas
 ├── types/                  # TypeScript types & interfaces
 ├── constants/              # App constants
 ├── configs/                # Config files
 ├── middleware.js           # Next.js middleware
 └── prisma/                 # Prisma schema & migrations
```

### Layered Architecture

```
Controller (Route Handler)
    ↓
Service (Business Logic)
    ↓
Repository (Data Access)
    ↓
Prisma (ORM)
    ↓
Database (PostgreSQL)
```

---

## Quick Reference: When to Use What

| Scenario | Solution |
|----------|----------|
| SEO page + DB data | Server Component (async) |
| Interactive widget | Client Component (`"use client"`) |
| Client data cache | SWR or TanStack Query |
| Cached + revalidated page | ISR (`revalidate = N`) |
| Protect routes | Middleware |
| Global state | Zustand |
| Form + validation | React Hook Form + Zod |
| Server mutation | Server Action + `revalidatePath` |
| Instant UI feedback | `useOptimistic` |
| File upload | FormData → S3/MinIO/Cloudinary |
| Auth | Auth.js or JWT + HttpOnly Cookie |
| RBAC | Middleware + JWT role check |
| Realtime | Socket.IO / Pusher |
| Background processing | BullMQ + Redis |
| Email | Resend / Nodemailer |
| Testing | Vitest (unit) + Playwright (E2E) |
| Deploy | Docker + GitHub Actions + NGINX |
| Monitoring | Sentry + Pino |
| Multi-language | next-intl |
| Multi-tenant | Path/Subdomain-based routing |

Here is the **cleanest, easiest, production-ready explanation of Prisma relationships** — **1:1**, **1:M**, and **M:M**, exactly like Laravel Eloquent relations.

Everything works with **MySQL + Prisma + Next.js**.

---

# ✅ **1️⃣ One-to-One (1:1)**

Example: **User → Profile**

### **Prisma Schema**

```prisma
model User {
  id       Int      @id @default(autoincrement())
  email    String   @unique
  profile  Profile?
}

model Profile {
  id      Int    @id @default(autoincrement())
  bio     String?
  userId  Int    @unique
  user    User   @relation(fields: [userId], references: [id])
}
```

### How it works:

* Each user has **one** profile.
* Each profile belongs to **one** user.
* `userId` is `@unique`, so only 1 profile per user.

### Create

```ts
const user = await prisma.user.create({
  data: {
    email: "test@gmail.com",
    profile: {
      create: {
        bio: "Hello world"
      }
    }
  }
});
```

### Query

```ts
const user = await prisma.user.findFirst({
  where: { id: 1 },
  include: { profile: true }
});
```

---

# ✅ **2️⃣ One-to-Many (1:M)**

Example: **User → Posts**

### **Prisma Schema**

```prisma
model User {
  id     Int     @id @default(autoincrement())
  email  String  @unique
  posts  Post[]
}

model Post {
  id      Int    @id @default(autoincrement())
  title   String
  userId  Int
  user    User   @relation(fields: [userId], references: [id])
}
```

### Create a post for a user

```ts
await prisma.post.create({
  data: {
    title: "My first post",
    user: { connect: { id: 1 } }
  }
});
```

### Query all user posts

```ts
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: { posts: true }
});
```

---

# ✅ **3️⃣ Many-to-Many (M:M)**

Example: **Post → Tags**

### **Prisma Schema** (automatic pivot table)

```prisma
model Post {
  id    Int    @id @default(autoincrement())
  title String
  tags  Tag[]  @relation("PostTags")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String
  posts Post[] @relation("PostTags")
}
```

🔹 Prisma will automatically create a pivot table internally
You don’t need to manually write it.

### Create tags for a post

```ts
await prisma.post.create({
  data: {
    title: "New Post",
    tags: {
      connectOrCreate: [
        {
          where: { name: "nextjs" },
          create: { name: "nextjs" }
        },
        {
          where: { name: "prisma" },
          create: { name: "prisma" }
        }
      ]
    }
  }
});
```

### Query post with tags

```ts
const post = await prisma.post.findFirst({
  where: { id: 1 },
  include: { tags: true }
});
```

---

# 🔥 **4️⃣ Many-to-Many With Pivot Table (Custom Pivot Columns)**

Example: **Users <-> Courses** with extra info like `progress`, `status`.

### Prisma Schema (manual pivot)

```prisma
model User {
  id       Int               @id @default(autoincrement())
  email    String            @unique
  courses  UserCourse[]
}

model Course {
  id       Int               @id @default(autoincrement())
  title    String
  users    UserCourse[]
}

model UserCourse {
  userId    Int
  courseId  Int
  progress  Int     @default(0)
  status    String  @default("pending")

  user      User    @relation(fields: [userId], references: [id])
  course    Course  @relation(fields: [courseId], references: [id])

  @@id([userId, courseId])
}
```

### Add a user to a course

```ts
await prisma.userCourse.create({
  data: {
    userId: 1,
    courseId: 4,
    progress: 20,
    status: "active"
  }
});
```

### Query user with all course data

```ts
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    courses: {
      include: {
        course: true
      }
    }
  }
});
```

---

# 🎯 Summary (Laravel Equivalent)

| Laravel           | Prisma                |
| ----------------- | --------------------- |
| `hasOne()`        | 1:1                   |
| `belongsTo()`     | reverse 1:1 or 1:M    |
| `hasMany()`       | 1:M                   |
| `belongsToMany()` | M:M                   |
| Pivot table       | Manual relation model |

---

# If you want:

I can provide ready-made working examples:

### ✅ Full Prisma + Next.js starter

### ✅ Admin panel (CRUD for relationship)

### ✅ ER Diagram for all relations

### ✅ Complete RBAC (roles, permissions)

### ✅ API + pages using these relations

Just tell me **“give me full Prisma starter project”**.

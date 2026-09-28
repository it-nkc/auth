```bash
npm install next-auth@beta
npm install bcryptjs
npm install -D @types/bcryptjs
```

_prisma/schema.prisma_

```tsx
model User {
  id           String   @id @default(cuid())
  name         String?
  email        String   @unique
  passwordHash String
  role         Role     @default(USER)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

enum Role {
  ADMIN
  STAFF
  USER
}
```

```bash
npx prisma migrate dev --name auth_user

npx prisma generate
```

ไฟล์ _.env_

```
AUTH_SECRET="ใส่-secret-ของคุณ"
```

สร้าง AUTH_SECRET ได้ด้วย

```bash
npx auth secret
```

_auth/ts_

```tsx
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";

import prisma from "@/app/lib/prisma";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Credentials({
      credentials: {
        email: {},
        password: {},
      },

      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        const user = await prisma.user.findUnique({
          where: {
            email: credentials.email as string,
          },
        });

        if (!user) {
          return null;
        }

        const passwordValid = await bcrypt.compare(
          credentials.password as string,
          user.passwordHash,
        );

        if (!passwordValid) {
          return null;
        }

        return {
          id: user.id,
          name: user.name,
          email: user.email,
          role: user.role,
        };
      },
    }),
  ],

  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
        token.role = user.role;
      }

      return token;
    },

    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.id as string;
        session.user.role = token.role as string;
      }

      return session;
    },
  },

  pages: {
    signIn: "/login",
  },
});
```

_app/api/auth/[...nextauth]/route.ts_

```tsx
import { handlers } from "@/app/auth";

export const { GET, POST } = handlers;
```

_app/admin/page.tsx_

```tsx
import { auth } from "@/app/auth";
import { redirect } from "next/navigation";

export default async function AdminPage() {
  const session = await auth();

  if (!session?.user) {
    redirect("/login");
  }

  if (session.user.role !== "ADMIN") {
    redirect("/unauthorized");
  }

  return (
    <div>
      <h1 className="text-2xl font-bold">Admin Dashboard</h1>

      <p>เฉพาะ ADMIN เท่านั้น</p>
    </div>
  );
}
```

_app/login/page.tsx_

```tsx
"use client";

import { signIn } from "next-auth/react";
import { useState } from "react";
import { useRouter } from "next/navigation";

export default function LoginPage() {
  const router = useRouter();

  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();

    setError("");

    const result = await signIn("credentials", {
      email,
      password,
      redirect: false,
    });

    if (result?.error) {
      setError("อีเมลหรือรหัสผ่านไม่ถูกต้อง");
      return;
    }

    router.push("/admin");
    router.refresh();
  }

  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-100">
      <div className="w-full max-w-md rounded-lg bg-white p-8 shadow">
        <h1 className="mb-6 text-2xl font-bold">Login</h1>

        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="mb-1 block">Email</label>

            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="w-full rounded border p-2"
              required
            />
          </div>

          <div>
            <label className="mb-1 block">Password</label>

            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full rounded border p-2"
              required
            />
          </div>

          {error && <p className="text-sm text-red-500">{error}</p>}

          <button
            type="submit"
            className="w-full rounded bg-blue-600 p-2 text-white"
          >
            Login
          </button>
        </form>
      </div>
    </div>
  );
}
```

_app/unauthorized/page.tsx_

```tsx
import Link from "next/link";

export default function UnauthorizedPage() {
  return (
    <main className="flex min-h-screen items-center justify-center bg-gray-100 px-4">
      <div className="w-full max-w-md rounded-xl bg-white p-8 text-center shadow-lg">
        <div className="mb-4 text-6xl">🚫</div>

        <h1 className="mb-2 text-3xl font-bold text-gray-800">
          ไม่มีสิทธิ์เข้าถึง
        </h1>

        <p className="mb-6 text-gray-600">
          คุณไม่มีสิทธิ์เข้าถึงหน้านี้
          กรุณาติดต่อผู้ดูแลระบบหากคิดว่าคุณควรมีสิทธิ์เข้าถึง
        </p>

        <div className="flex justify-center gap-3">
          <Link
            href="/dashboard"
            className="rounded-lg bg-blue-600 px-5 py-2.5 text-white hover:bg-blue-700"
          >
            กลับ Dashboard
          </Link>

          <Link
            href="/"
            className="rounded-lg border border-gray-300 px-5 py-2.5 text-gray-700 hover:bg-gray-100"
          >
            หน้าแรก
          </Link>
        </div>
      </div>
    </main>
  );
}
```

### สร้าง GitHub OAuth App

```
Settings
→ Developer settings
→ OAuth Apps
→ New OAuth App
```

#### กำหนดสำหรับ development

```
Application name:
Next.js RBAC Demo

Homepage URL:
http://localhost:3000

Authorization callback URL:
http://localhost:3000/api/auth/callback/github
```

ไฟล์ _.env_

```
AUTH_GITHUB_ID="xxxxxxxxxxxxxxxx"
AUTH_GITHUB_SECRET="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

แก้ _app/auth.ts_

```ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import GitHub from "next-auth/providers/github";
import bcrypt from "bcryptjs";

import prisma from "@/app/lib/prisma";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    // ==========================================
    // Login ด้วย Email / Password
    // ==========================================
    Credentials({
      credentials: {
        email: {},
        password: {},
      },

      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        const email = credentials.email as string;
        const password = credentials.password as string;

        const user = await prisma.user.findUnique({
          where: {
            email,
          },
        });

        if (!user) {
          return null;
        }

        const passwordValid = await bcrypt.compare(password, user.passwordHash);

        if (!passwordValid) {
          return null;
        }

        return {
          id: user.id,
          name: user.name,
          email: user.email,
          role: user.role,
        };
      },
    }),

    // ==========================================
    // Login ด้วย GitHub
    // ==========================================
    GitHub({
      clientId: process.env.AUTH_GITHUB_ID,
      clientSecret: process.env.AUTH_GITHUB_SECRET,
    }),
  ],

  callbacks: {
    // ==========================================
    // JWT
    // ==========================================
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;

        // Credentials มี role
        if (user.role) {
          token.role = user.role;
        }

        // GitHub user ไม่มี role จาก Credentials
        // ให้ USER เป็นค่าเริ่มต้น
        if (!token.role) {
          token.role = "USER";
        }
      }

      return token;
    },

    // ==========================================
    // Session
    // ==========================================
    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.id as string;

        session.user.role = (token.role as string) || "USER";
      }

      return session;
    },
  },

  pages: {
    signIn: "/login",
  },

  session: {
    strategy: "jwt",
  },
});
```

_app/login-github/page.tsx_

```tsx
import { redirect } from "next/navigation";
import { auth, signIn } from "@/app/auth";
import SignInGithub from "@/app/ui/github-auth";

export default async function LoginPage() {
  // const session = await auth()
  // if (session?.user) redirect('/')

  return (
    <main className="mx-auto max-w-md px-5 py-20">
      <section className="rounded-2xl border bg-white p-8 shadow-sm">
        <h1 className="text-3xl font-bold">Members Only</h1>
        <p className="mt-3 text-slate-600">เข้าสู่ระบบด้วย github</p>
        {/* <SignInGithub /> */}

        <form
          action={async () => {
            "use server";
            await signIn("github", { redirectTo: "/" });
          }}
        >
          <button className="mt-8 w-full rounded-lg bg-slate-950 px-5 py-3 font-medium text-white">
            Continue with GitHub
          </button>
        </form>
      </section>
    </main>
  );
}
```

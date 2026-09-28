## การ login ด้วย Username & password และ Github (Authentication)

> app/
> ├── login/
> │ └── page.tsx
> ├── admin/
> │ └── page.tsx
> ├── unauthorized/
> │ └── page.tsx
> ├── api/
> │ └── auth/
> │ └── [...nextauth]/
> │ └── route.ts
> ├── lib/
> │ └── prisma.ts
> └── layout.tsx

auth.ts

prisma/
└── schema.prisma

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

### สร้างหน้า sign up

_app/signup/actions.ts_

```ts
"use server";

import bcrypt from "bcryptjs";
import prisma from "@/app/lib/prisma";

export type SignUpState = {
  error?: string;
  success?: boolean;
};

export async function signUp(
  prevState: SignUpState,
  formData: FormData,
): Promise<SignUpState> {
  const name = formData.get("name")?.toString().trim();
  const email = formData.get("email")?.toString().trim().toLowerCase();
  const password = formData.get("password")?.toString();
  const confirmPassword = formData.get("confirmPassword")?.toString();

  // ตรวจสอบข้อมูล
  if (!name || !email || !password || !confirmPassword) {
    return {
      error: "กรุณากรอกข้อมูลให้ครบถ้วน",
    };
  }

  // ตรวจสอบ password
  if (password.length < 6) {
    return {
      error: "รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร",
    };
  }

  if (password !== confirmPassword) {
    return {
      error: "รหัสผ่านไม่ตรงกัน",
    };
  }

  // ตรวจสอบ email
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  if (!emailRegex.test(email)) {
    return {
      error: "รูปแบบ Email ไม่ถูกต้อง",
    };
  }

  // ตรวจสอบว่ามี User อยู่แล้วหรือไม่
  const existingUser = await prisma.user.findUnique({
    where: {
      email,
    },
  });

  if (existingUser) {
    return {
      error: "Email นี้ถูกใช้งานแล้ว",
    };
  }

  // Hash password
  const passwordHash = await bcrypt.hash(password, 12);

  // สร้าง User
  await prisma.user.create({
    data: {
      name,
      email,
      passwordHash,

      // สำคัญ:
      // ผู้สมัครใหม่จะเป็น USER เสมอ
      role: "ADMIN",
    },
  });

  return {
    success: true,
  };
}
```

_app/signup/page.tsx_

```tsx
"use client";

import Link from "next/link";
import { useActionState, useEffect } from "react";
import { useRouter } from "next/navigation";

import { signUp, type SignUpState } from "@/app/signup/actions";

const initialState: SignUpState = {};

export default function SignUpPage() {
  const router = useRouter();

  const [state, formAction, pending] = useActionState(signUp, initialState);

  useEffect(() => {
    if (state.success) {
      router.push("/login?registered=1");
    }
  }, [state.success, router]);

  return (
    <main className="flex min-h-screen items-center justify-center bg-gray-100 px-4">
      <div className="w-full max-w-md rounded-xl bg-white p-8 shadow-lg">
        <h1 className="mb-2 text-center text-3xl font-bold text-gray-800">
          Sign Up
        </h1>

        <p className="mb-6 text-center text-gray-500">สร้างบัญชีผู้ใช้งาน</p>

        <form action={formAction} className="space-y-4">
          {/* Name */}
          <div>
            <label
              htmlFor="name"
              className="mb-1 block text-sm font-medium text-gray-700"
            >
              ชื่อ
            </label>

            <input
              id="name"
              name="name"
              type="text"
              className="w-full rounded-lg border border-gray-300 px-3 py-2
                         focus:border-blue-500 focus:outline-none"
              placeholder="ชื่อของคุณ"
              required
            />
          </div>

          {/* Email */}
          <div>
            <label
              htmlFor="email"
              className="mb-1 block text-sm font-medium text-gray-700"
            >
              Email
            </label>

            <input
              id="email"
              name="email"
              type="email"
              className="w-full rounded-lg border border-gray-300 px-3 py-2
                         focus:border-blue-500 focus:outline-none"
              placeholder="you@example.com"
              required
            />
          </div>

          {/* Password */}
          <div>
            <label
              htmlFor="password"
              className="mb-1 block text-sm font-medium text-gray-700"
            >
              Password
            </label>

            <input
              id="password"
              name="password"
              type="password"
              className="w-full rounded-lg border border-gray-300 px-3 py-2
                         focus:border-blue-500 focus:outline-none"
              placeholder="อย่างน้อย 6 ตัวอักษร"
              minLength={6}
              required
            />
          </div>

          {/* Confirm Password */}
          <div>
            <label
              htmlFor="confirmPassword"
              className="mb-1 block text-sm font-medium text-gray-700"
            >
              Confirm Password
            </label>

            <input
              id="confirmPassword"
              name="confirmPassword"
              type="password"
              className="w-full rounded-lg border border-gray-300 px-3 py-2
                         focus:border-blue-500 focus:outline-none"
              placeholder="กรอกรหัสผ่านอีกครั้ง"
              minLength={6}
              required
            />
          </div>

          {/* Error */}
          {state.error && (
            <div className="rounded-lg bg-red-50 p-3 text-sm text-red-600">
              {state.error}
            </div>
          )}

          {/* Submit */}
          <button
            type="submit"
            disabled={pending}
            className="w-full rounded-lg bg-blue-600 px-4 py-2.5
                       font-medium text-white
                       hover:bg-blue-700
                       disabled:cursor-not-allowed
                       disabled:opacity-50"
          >
            {pending ? "กำลังสร้างบัญชี..." : "สร้างบัญชี"}
          </button>
        </form>

        {/* Login */}
        <div className="mt-6 text-center text-sm text-gray-600">
          มีบัญชีอยู่แล้ว?
          <Link
            href="/login"
            className="ml-1 font-medium text-blue-600 hover:underline"
          >
            Login
          </Link>
        </div>
      </div>
    </main>
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

แก้ไขไฟล์ _app/login/page.tsx_

```tsx
// * Github signin
const githubSignIn = async () => {
  await signIn("github", {
    callbackUrl: "/",
  });
};
```

```tsx

<p className="text-center my-3">-- OR --</p>
<div className="space-y-3">
    <button
        type="button"
        className="relative inline-flex w-full items-center justify-center rounded-md border border-gray-400 bg-white px-3.5 py-2.5 font-semibold text-gray-700 transition-all duration-200 hover:bg-gray-100 hover:text-black focus:bg-gray-100 focus:text-black focus:outline-none"
        onClick={githubSignIn}
    >
        <span className="mr-2 inline-block">
            <svg
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 50 50"
                width="30px"
                height="30px"
            >
                <path d="M17.791,46.836C18.502,46.53,19,45.823,19,45v-5.4c0-0.197,0.016-0.402,0.041-0.61C19.027,38.994,19.014,38.997,19,39 c0,0-3,0-3.6,0c-1.5,0-2.8-0.6-3.4-1.8c-0.7-1.3-1-3.5-2.8-4.7C8.9,32.3,9.1,32,9.7,32c0.6,0.1,1.9,0.9,2.7,2c0.9,1.1,1.8,2,3.4,2 c2.487,0,3.82-0.125,4.622-0.555C21.356,34.056,22.649,33,24,33v-0.025c-5.668-0.182-9.289-2.066-10.975-4.975 c-3.665,0.042-6.856,0.405-8.677,0.707c-0.058-0.327-0.108-0.656-0.151-0.987c1.797-0.296,4.843-0.647,8.345-0.714 c-0.112-0.276-0.209-0.559-0.291-0.849c-3.511-0.178-6.541-0.039-8.187,0.097c-0.02-0.332-0.047-0.663-0.051-0.999 c1.649-0.135,4.597-0.27,8.018-0.111c-0.079-0.5-0.13-1.011-0.13-1.543c0-1.7,0.6-3.5,1.7-5c-0.5-1.7-1.2-5.3,0.2-6.6 c2.7,0,4.6,1.3,5.5,2.1C21,13.4,22.9,13,25,13s4,0.4,5.6,1.1c0.9-0.8,2.8-2.1,5.5-2.1c1.5,1.4,0.7,5,0.2,6.6c1.1,1.5,1.7,3.2,1.6,5 c0,0.484-0.045,0.951-0.11,1.409c3.499-0.172,6.527-0.034,8.204,0.102c-0.002,0.337-0.033,0.666-0.051,0.999 c-1.671-0.138-4.775-0.28-8.359-0.089c-0.089,0.336-0.197,0.663-0.325,0.98c3.546,0.046,6.665,0.389,8.548,0.689 c-0.043,0.332-0.093,0.661-0.151,0.987c-1.912-0.306-5.171-0.664-8.879-0.682C35.112,30.873,31.557,32.75,26,32.969V33 c2.6,0,5,3.9,5,6.6V45c0,0.823,0.498,1.53,1.209,1.836C41.37,43.804,48,35.164,48,25C48,12.318,37.683,2,25,2S2,12.318,2,25 C2,35.164,8.63,43.804,17.791,46.836z" />
            </svg>
        </span>
        Sign in with Github
    </button>
</div>
```

_app/login-github/page.tsx_

```tsx
import { redirect } from "next/navigation";
import { auth, signIn } from "@/app/auth";

export default async function LoginPage() {
  // const session = await auth()
  // if (session?.user) redirect('/')

  return (
    <main className="mx-auto max-w-md px-5 py-20">
      <section className="rounded-2xl border bg-white p-8 shadow-sm">
        <h1 className="text-3xl font-bold">Members Only</h1>
        <p className="mt-3 text-slate-600">เข้าสู่ระบบด้วย github</p>

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

# Next.js 14 Full-Stack Application Template Guide

> **Purpose**: This comprehensive guide documents the technical architecture, setup process, and implementation patterns used in this Next.js 14 application. Use this as a blueprint for building similar full-stack applications with authentication, database integration, and modern UI patterns.

## 📋 Table of Contents

1. [Technical Stack Overview](#technical-stack-overview)
2. [Project Architecture](#project-architecture)
3. [Database Setup & Configuration](#database-setup--configuration)
4. [Authentication Implementation (Google OAuth)](#authentication-implementation-google-oauth)
5. [UI Design System](#ui-design-system)
6. [Application Structure](#application-structure)
7. [API Patterns & Best Practices](#api-patterns--best-practices)
8. [Step-by-Step Setup Guide](#step-by-step-setup-guide)
9. [Common Patterns & Code Examples](#common-patterns--code-examples)
10. [Deployment Configuration](#deployment-configuration)

---

## Technical Stack Overview

### Core Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| **Next.js** | 14.2.16 | React framework with App Router |
| **React** | 18 | UI library |
| **TypeScript** | 5+ | Type safety |
| **Prisma** | 6.16.1 | Database ORM |
| **PostgreSQL** | Latest | Primary database (via Neon) |
| **NextAuth.js** | 4.24.11 | Authentication |
| **Tailwind CSS** | 3.4.17 | Styling framework |
| **Radix UI** | Latest | Accessible UI components |
| **pnpm** | Latest | Package manager |

### Additional Dependencies

```json
{
  "@next-auth/prisma-adapter": "^1.0.7",
  "@prisma/client": "^6.16.1",
  "lucide-react": "^0.454.0",
  "next-themes": "latest",
  "date-fns": "^4.1.0",
  "katex": "^0.16.23",
  "recharts": "2.15.4"
}
```

### Development Tools

- **Node.js**: 18+ (LTS recommended)
- **pnpm**: Fast, disk-efficient package manager
- **Vercel**: Deployment platform (recommended)
- **Neon**: Serverless PostgreSQL database

---

## Project Architecture

### Monorepo Structure (Optional)

This application uses a monorepo structure with workspaces for better code organization:

```
project-root/
├── apps/
│   ├── admin/           # Admin dashboard (optional)
│   └── student/         # Main application
├── packages/
│   ├── auth/           # Shared authentication logic
│   ├── database/       # Prisma schema and client
│   ├── types/          # Shared TypeScript types
│   ├── ui/             # Shared UI components
│   └── utils/          # Shared utilities
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
├── package.json
└── turbo.json (optional)
```

### Single App Structure (Simpler Alternative)

For a simpler setup without monorepo:

```
project-root/
├── app/                    # Next.js App Router
│   ├── api/               # API routes
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts
│   │   └── [feature]/
│   │       └── route.ts
│   ├── (auth)/            # Auth route group
│   │   └── signin/
│   │       └── page.tsx
│   ├── (dashboard)/       # Protected route group
│   │   ├── layout.tsx     # Dashboard layout
│   │   └── dashboard/
│   │       └── page.tsx
│   ├── layout.tsx         # Root layout
│   ├── page.tsx          # Landing page
│   └── globals.css       # Global styles
├── components/
│   ├── ui/               # Base UI components
│   ├── landing/          # Landing page components
│   ├── dashboard/        # Dashboard components
│   └── providers.tsx     # Context providers
├── lib/
│   ├── auth.ts           # NextAuth configuration
│   ├── prisma.ts         # Prisma client singleton
│   └── utils.ts          # Helper functions
├── prisma/
│   └── schema.prisma     # Database schema
├── public/               # Static assets
├── types/
│   └── index.ts          # Type definitions
├── .env.local            # Environment variables
├── next.config.mjs       # Next.js configuration
├── tailwind.config.js    # Tailwind configuration
├── tsconfig.json         # TypeScript configuration
└── package.json
```

### Route Groups

Next.js 14 App Router uses route groups for logical organization:

- `(auth)`: Authentication pages (signin, signup, etc.)
- `(dashboard)`: Protected routes requiring authentication
- `(marketing)`: Public marketing pages (optional)

**Route groups do not affect the URL structure** - they're purely for organization.

---

## Database Setup & Configuration

### 1. Prisma Schema Structure

**Location**: `prisma/schema.prisma` (or `packages/database/prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// NextAuth.js Required Models
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  isAdmin       Boolean   @default(false)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  // Relations
  accounts      Account[]
  sessions      Session[]
  
  // Add your app-specific fields here
  // Example: progress, subscriptions, etc.
  
  @@index([email])
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime
  
  @@unique([identifier, token])
}

// Your application-specific models go here
// Example:
model Post {
  id        String   @id @default(cuid())
  title     String
  content   String
  published Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  authorId  String
  
  @@index([authorId])
}
```

### 2. Prisma Client Setup

**Location**: `lib/prisma.ts` (or `packages/database/src/client.ts`)

```typescript
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' 
      ? ['query', 'error', 'warn'] 
      : ['error'],
  })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

**Why this pattern?**
- Prevents multiple Prisma Client instances in development
- Enables query logging in development
- Production-ready singleton pattern

### 3. Database Setup Steps

```bash
# 1. Install dependencies
pnpm install

# 2. Create .env.local file with DATABASE_URL
# DATABASE_URL="postgresql://user:password@host:5432/dbname?sslmode=require"

# 3. Generate Prisma Client
pnpm prisma generate

# 4. Push schema to database (development)
pnpm prisma db push

# 5. Create migrations (production)
pnpm prisma migrate dev --name init

# 6. Open Prisma Studio to view data
pnpm prisma studio
```

### 4. Neon PostgreSQL Setup (Recommended)

1. **Create Account**: Visit [neon.tech](https://neon.tech)
2. **Create Project**: Choose a region close to your users
3. **Get Connection String**: Copy from Neon dashboard
4. **Format**: `postgresql://[user]:[password]@[host]/[dbname]?sslmode=require`
5. **Add to .env.local**: `DATABASE_URL="your-connection-string"`

**Benefits of Neon**:
- Serverless & autoscaling
- Built-in connection pooling
- Generous free tier
- Instant database branching
- Works perfectly with Vercel

---

## Authentication Implementation (Google OAuth)

### 1. Google Cloud Console Setup

**Step 1: Create OAuth 2.0 Credentials**

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create new project or select existing
3. Navigate to **APIs & Services** → **Credentials**
4. Click **Create Credentials** → **OAuth 2.0 Client ID**
5. Configure OAuth consent screen if prompted

**Step 2: Configure OAuth Client**

```
Application Type: Web Application
Name: Your App Name

Authorized JavaScript origins:
- http://localhost:3000 (development)
- http://localhost:3001 (if using port 3001)
- https://your-domain.vercel.app (production)

Authorized redirect URIs:
- http://localhost:3000/api/auth/callback/google
- http://localhost:3001/api/auth/callback/google
- https://your-domain.vercel.app/api/auth/callback/google
```

**Step 3: Get Credentials**
- Copy **Client ID**
- Copy **Client Secret**
- Add to `.env.local`

### 2. NextAuth.js Configuration

**Location**: `lib/auth.ts`

```typescript
import type { NextAuthOptions } from 'next-auth'
import GoogleProvider from 'next-auth/providers/google'
import { PrismaAdapter } from '@next-auth/prisma-adapter'
import { prisma } from '@/lib/prisma' // Adjust import path
import type { User, Session } from 'next-auth'

export const authOptions: NextAuthOptions = {
  adapter: PrismaAdapter(prisma),
  session: { 
    strategy: 'database',
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  
  // Use separate cookie names if running multiple apps on same domain
  cookies: {
    sessionToken: {
      name: process.env.NODE_ENV === 'production' 
        ? `__Secure-next-auth.session-token`
        : `next-auth.session-token`,
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: process.env.NODE_ENV === 'production',
      },
    },
  },
  
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  
  pages: {
    signIn: '/signin',      // Custom sign-in page
    error: '/signin',       // Error page redirects to signin
    // signOut: '/signout', // Optional custom sign-out page
    // newUser: '/welcome', // Optional new user onboarding
  },
  
  callbacks: {
    async signIn({ user, account, profile }) {
      // Add custom sign-in logic here
      // Example: restrict to certain domains
      // if (user.email?.endsWith('@yourcompany.com')) {
      //   return true
      // }
      // return false
      
      return true
    },
    
    async redirect({ url, baseUrl }) {
      // Redirect to dashboard after sign-in
      if (url.startsWith(baseUrl)) return url
      return `${baseUrl}/dashboard`
    },
    
    async session({ session, user }) {
      // Add custom fields to session
      if (session.user && user) {
        session.user.id = user.id
        
        // Fetch additional user data
        const dbUser = await prisma.user.findUnique({
          where: { id: user.id },
          select: { 
            isAdmin: true,
            // Add other fields you need in the session
          },
        })
        
        session.user.isAdmin = dbUser?.isAdmin || false
      }
      return session
    },
  },
  
  // Enable debug messages in development
  debug: process.env.NODE_ENV === 'development',
}
```

### 3. NextAuth API Route

**Location**: `app/api/auth/[...nextauth]/route.ts`

```typescript
import NextAuth from 'next-auth'
import { authOptions } from '@/lib/auth'

const handler = NextAuth(authOptions)

export { handler as GET, handler as POST }
```

### 4. Session Provider Setup

**Location**: `components/providers.tsx`

```typescript
'use client'

import { SessionProvider } from 'next-auth/react'

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <SessionProvider>
      {children}
    </SessionProvider>
  )
}
```

**Usage in Root Layout**: `app/layout.tsx`

```typescript
import { Providers } from '@/components/providers'
import './globals.css'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  )
}
```

### 5. TypeScript Session Type Extension

**Location**: `types/next-auth.d.ts`

```typescript
import 'next-auth'

declare module 'next-auth' {
  interface Session {
    user: {
      id: string
      name?: string | null
      email?: string | null
      image?: string | null
      isAdmin?: boolean
      // Add your custom fields here
    }
  }
  
  interface User {
    id: string
    isAdmin?: boolean
    // Add your custom fields here
  }
}
```

### 6. Sign-In Page

**Location**: `app/(auth)/signin/page.tsx`

```typescript
'use client'

import { signIn } from 'next-auth/react'
import { Button } from '@/components/ui/button'
import { Card } from '@/components/ui/card'
import { BookOpen } from 'lucide-react'

export default function SignInPage() {
  const handleSignIn = async () => {
    await signIn('google', { callbackUrl: '/dashboard' })
  }

  return (
    <div className="flex min-h-screen items-center justify-center p-6 bg-gradient-to-br from-purple-50 to-blue-50">
      <Card className="max-w-md w-full p-8">
        <div className="text-center mb-8">
          <div className="inline-flex items-center justify-center w-16 h-16 bg-gradient-to-r from-purple-600 to-blue-600 rounded-full mb-4">
            <BookOpen className="h-8 w-8 text-white" />
          </div>
          <h1 className="text-3xl font-bold mb-2">Welcome Back</h1>
          <p className="text-muted-foreground">
            Sign in to access your account
          </p>
        </div>

        <Button 
          onClick={handleSignIn}
          className="w-full bg-gradient-to-r from-purple-600 to-blue-600 hover:from-purple-700 hover:to-blue-700"
          size="lg"
        >
          <svg className="w-5 h-5 mr-2" viewBox="0 0 24 24">
            <path
              fill="currentColor"
              d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"
            />
            <path
              fill="currentColor"
              d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"
            />
            <path
              fill="currentColor"
              d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"
            />
            <path
              fill="currentColor"
              d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"
            />
          </svg>
          Sign in with Google
        </Button>
      </Card>
    </div>
  )
}
```

### 7. Protected Route Pattern

**Location**: `app/(dashboard)/layout.tsx`

```typescript
import { getServerSession } from 'next-auth'
import { redirect } from 'next/navigation'
import { authOptions } from '@/lib/auth'
import { Navigation } from '@/components/navigation'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const session = await getServerSession(authOptions)

  // Redirect to sign-in if not authenticated
  if (!session) {
    redirect('/signin')
  }

  return (
    <div className="min-h-screen bg-background">
      <Navigation user={session.user} />
      <main className="container mx-auto px-4 py-8">
        {children}
      </main>
    </div>
  )
}
```

### 8. Sign-Out Component

**Location**: `components/sign-out-button.tsx`

```typescript
'use client'

import { signOut } from 'next-auth/react'
import { Button } from '@/components/ui/button'
import { LogOut } from 'lucide-react'

export function SignOutButton() {
  const handleSignOut = async () => {
    await signOut({ callbackUrl: '/' })
  }

  return (
    <Button 
      onClick={handleSignOut}
      variant="outline"
      size="sm"
    >
      <LogOut className="h-4 w-4 mr-2" />
      Sign Out
    </Button>
  )
}
```

---

## UI Design System

### 1. Tailwind CSS Configuration

**Location**: `tailwind.config.js`

```javascript
const colors = require('tailwindcss/colors')

/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        // CSS variable-based colors for theming
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        popover: {
          DEFAULT: 'hsl(var(--popover))',
          foreground: 'hsl(var(--popover-foreground))',
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
        // Full color palettes for custom components
        purple: colors.purple,
        blue: colors.blue,
        green: colors.green,
        orange: colors.orange,
        yellow: colors.yellow,
        gray: colors.gray,
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
    },
  },
  plugins: [],
}
```

### 2. Global CSS Variables

**Location**: `app/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  /* Base shadcn/ui colors */
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --card: 0 0% 100%;
  --card-foreground: 222.2 84% 4.9%;
  --popover: 0 0% 100%;
  --popover-foreground: 222.2 84% 4.9%;
  --primary: 262.1 83.3% 57.8%;
  --primary-foreground: 210 40% 98%;
  --secondary: 210 40% 96%;
  --secondary-foreground: 222.2 84% 4.9%;
  --muted: 210 40% 96%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --accent: 210 40% 96%;
  --accent-foreground: 222.2 84% 4.9%;
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 210 40% 98%;
  --border: 214.3 31.8% 91.4%;
  --input: 214.3 31.8% 91.4%;
  --ring: 262.1 83.3% 57.8%;
  --radius: 0.5rem;

  /* Brand Colors */
  --brand: 262 83% 58%;
  --brand-foreground: 0 0% 100%;
  
  /* Semantic Colors */
  --success: 142 76% 36%;
  --success-foreground: 210 40% 98%;
  --warning: 38 92% 50%;
  --warning-foreground: 210 40% 98%;
  --info: 217 91% 60%;
  --info-foreground: 210 40% 98%;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  --card: 222.2 84% 4.9%;
  --card-foreground: 210 40% 98%;
  --popover: 222.2 84% 4.9%;
  --popover-foreground: 210 40% 98%;
  --primary: 262.1 83.3% 57.8%;
  --primary-foreground: 210 40% 98%;
  --secondary: 217.2 32.6% 17.5%;
  --secondary-foreground: 210 40% 98%;
  --muted: 217.2 32.6% 17.5%;
  --muted-foreground: 215 20.2% 65.1%;
  --accent: 217.2 32.6% 17.5%;
  --accent-foreground: 210 40% 98%;
  --destructive: 0 62.8% 30.6%;
  --destructive-foreground: 210 40% 98%;
  --border: 217.2 32.6% 17.5%;
  --input: 217.2 32.6% 17.5%;
  --ring: 262.1 83.3% 57.8%;
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### 3. Design Principles

**Color Usage**:
- **NO inline hex colors** - Always use CSS variables or Tailwind classes
- **Brand Colors**: Purple for primary actions, blue for secondary
- **Semantic Colors**: Green (success), Orange (warning), Red (destructive)
- **Neutral Colors**: Gray scale for text and backgrounds

**Spacing**:
- Use Tailwind spacing scale: `p-4`, `p-6`, `gap-4`, `gap-6`
- Consistent component padding: `p-6` for cards, `p-4` for smaller elements
- Page margins: `max-w-7xl mx-auto px-4` or `container mx-auto px-4`

**Border Radius**:
- Large components: `rounded-xl`
- Medium components: `rounded-lg`
- Small elements: `rounded-md`
- Buttons: `rounded-md` or `rounded-lg`

**Typography**:
- Headings: `font-bold` with `text-2xl`, `text-3xl`, `text-4xl`
- Body text: default font weight with `text-sm`, `text-base`
- Muted text: `text-muted-foreground`

**Gradients** (for special elements):
```css
bg-gradient-to-r from-purple-600 to-blue-600
bg-gradient-to-br from-purple-50 to-blue-50
```

### 4. Component Library Setup (Radix UI / shadcn/ui)

**Installation**:
```bash
# Option 1: Manual setup with Radix UI
pnpm add @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-select

# Option 2: Use shadcn/ui CLI
npx shadcn-ui@latest init
npx shadcn-ui@latest add button card dialog dropdown-menu
```

**Base Component Structure**: `components/ui/button.tsx`

```typescript
import * as React from 'react'
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = 'Button'

export { Button, buttonVariants }
```

---

## Application Structure

### 1. Landing Page

**Location**: `app/page.tsx`

```typescript
import { getServerSession } from 'next-auth'
import { redirect } from 'next/navigation'
import { authOptions } from '@/lib/auth'
import { Navigation } from '@/components/landing/navigation'
import { HeroSection } from '@/components/landing/hero-section'
import { FeaturesSection } from '@/components/landing/features-section'
import { CTASection } from '@/components/landing/cta-section'
import { Footer } from '@/components/landing/footer'

export default async function Home() {
  const session = await getServerSession(authOptions)

  // Redirect authenticated users to dashboard
  if (session) {
    redirect('/dashboard')
  }

  return (
    <div className="min-h-screen">
      <Navigation />
      <main className="pt-16">
        <HeroSection />
        <FeaturesSection />
        <CTASection />
      </main>
      <Footer />
    </div>
  )
}
```

### 2. Navigation Component

**Location**: `components/landing/navigation.tsx`

```typescript
'use client'

import { Button } from '@/components/ui/button'
import { Menu, X } from 'lucide-react'
import Link from 'next/link'
import { useState } from 'react'

export function Navigation() {
  const [isOpen, setIsOpen] = useState(false)

  const navLinks = [
    { name: 'Features', href: '#features' },
    { name: 'Pricing', href: '#pricing' },
    { name: 'About', href: '#about' },
  ]

  return (
    <nav className="fixed top-0 left-0 right-0 z-50 bg-white/90 backdrop-blur-md shadow-sm">
      <div className="max-w-7xl mx-auto px-6">
        <div className="flex items-center justify-between h-16">
          {/* Logo */}
          <Link href="/" className="flex items-center gap-2">
            <div className="w-10 h-10 bg-gradient-to-br from-purple-600 to-blue-600 rounded-lg flex items-center justify-center">
              <span className="text-white font-bold text-xl">A</span>
            </div>
            <span className="font-bold text-xl">Your App</span>
          </Link>

          {/* Desktop Navigation */}
          <div className="hidden md:flex items-center gap-8">
            {navLinks.map((link) => (
              <a
                key={link.name}
                href={link.href}
                className="text-gray-700 hover:text-purple-600 font-medium transition-colors"
              >
                {link.name}
              </a>
            ))}
          </div>

          {/* Desktop CTA */}
          <div className="hidden md:flex items-center gap-4">
            <Link href="/signin">
              <Button variant="outline">Sign In</Button>
            </Link>
            <Link href="/signin">
              <Button className="bg-gradient-to-r from-purple-600 to-blue-600">
                Get Started
              </Button>
            </Link>
          </div>

          {/* Mobile Menu Button */}
          <button
            className="md:hidden p-2"
            onClick={() => setIsOpen(!isOpen)}
          >
            {isOpen ? <X className="h-6 w-6" /> : <Menu className="h-6 w-6" />}
          </button>
        </div>

        {/* Mobile Navigation */}
        {isOpen && (
          <div className="md:hidden py-4 border-t">
            <div className="flex flex-col gap-4">
              {navLinks.map((link) => (
                <a
                  key={link.name}
                  href={link.href}
                  className="text-gray-700 hover:text-purple-600 font-medium py-2"
                  onClick={() => setIsOpen(false)}
                >
                  {link.name}
                </a>
              ))}
              <div className="flex flex-col gap-2 pt-4 border-t">
                <Link href="/signin" onClick={() => setIsOpen(false)}>
                  <Button variant="outline" className="w-full">Sign In</Button>
                </Link>
                <Link href="/signin" onClick={() => setIsOpen(false)}>
                  <Button className="w-full">Get Started</Button>
                </Link>
              </div>
            </div>
          </div>
        )}
      </div>
    </nav>
  )
}
```

### 3. Hero Section

**Location**: `components/landing/hero-section.tsx`

```typescript
import { Button } from '@/components/ui/button'
import { ArrowRight, Sparkles } from 'lucide-react'
import Link from 'next/link'

export function HeroSection() {
  return (
    <section className="relative overflow-hidden bg-gradient-to-br from-purple-50 via-white to-blue-50 pt-32 pb-24">
      <div className="max-w-7xl mx-auto px-6">
        <div className="text-center max-w-3xl mx-auto">
          {/* Badge */}
          <div className="inline-flex items-center gap-2 px-4 py-2 bg-purple-100 text-purple-700 rounded-full text-sm font-medium mb-6">
            <Sparkles className="h-4 w-4" />
            New features available now
          </div>

          {/* Heading */}
          <h1 className="text-5xl md:text-6xl font-bold mb-6 bg-gradient-to-r from-purple-600 to-blue-600 bg-clip-text text-transparent">
            Build Something Amazing
          </h1>

          {/* Subheading */}
          <p className="text-xl text-gray-600 mb-8">
            The modern way to create, collaborate, and grow. 
            Start your journey today with our powerful platform.
          </p>

          {/* CTA Buttons */}
          <div className="flex flex-col sm:flex-row items-center justify-center gap-4">
            <Link href="/signin">
              <Button size="lg" className="bg-gradient-to-r from-purple-600 to-blue-600 text-white">
                Get Started Free
                <ArrowRight className="ml-2 h-5 w-5" />
              </Button>
            </Link>
            <Button size="lg" variant="outline">
              Watch Demo
            </Button>
          </div>

          {/* Social Proof */}
          <p className="text-sm text-gray-500 mt-8">
            Join 10,000+ users already using our platform
          </p>
        </div>
      </div>
    </section>
  )
}
```

### 4. Dashboard Page

**Location**: `app/(dashboard)/dashboard/page.tsx`

```typescript
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'
import { Card } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { TrendingUp, Users, DollarSign, Activity } from 'lucide-react'
import Link from 'next/link'

export default async function DashboardPage() {
  const session = await getServerSession(authOptions)

  if (!session?.user) {
    return null
  }

  // Fetch your data here
  const stats = {
    totalUsers: 1234,
    revenue: 45678,
    activeProjects: 23,
    growth: 12.5,
  }

  return (
    <div className="max-w-7xl mx-auto">
      {/* Header */}
      <div className="mb-8">
        <h1 className="text-4xl font-bold mb-2">
          Welcome back, {session.user.name?.split(' ')[0]}! 👋
        </h1>
        <p className="text-muted-foreground text-lg">
          Here's what's happening with your account today.
        </p>
      </div>

      {/* Stats Grid */}
      <div className="grid md:grid-cols-4 gap-6 mb-8">
        <Card className="p-6">
          <div className="flex items-center justify-between mb-2">
            <span className="text-sm font-medium text-muted-foreground">Total Users</span>
            <Users className="h-4 w-4 text-purple-600" />
          </div>
          <p className="text-3xl font-bold">{stats.totalUsers.toLocaleString()}</p>
          <p className="text-xs text-green-600 mt-1">+{stats.growth}% from last month</p>
        </Card>

        <Card className="p-6">
          <div className="flex items-center justify-between mb-2">
            <span className="text-sm font-medium text-muted-foreground">Revenue</span>
            <DollarSign className="h-4 w-4 text-green-600" />
          </div>
          <p className="text-3xl font-bold">${stats.revenue.toLocaleString()}</p>
          <p className="text-xs text-green-600 mt-1">+{stats.growth}% from last month</p>
        </Card>

        <Card className="p-6">
          <div className="flex items-center justify-between mb-2">
            <span className="text-sm font-medium text-muted-foreground">Active Projects</span>
            <Activity className="h-4 w-4 text-blue-600" />
          </div>
          <p className="text-3xl font-bold">{stats.activeProjects}</p>
          <p className="text-xs text-muted-foreground mt-1">5 completed</p>
        </Card>

        <Card className="p-6">
          <div className="flex items-center justify-between mb-2">
            <span className="text-sm font-medium text-muted-foreground">Growth</span>
            <TrendingUp className="h-4 w-4 text-orange-600" />
          </div>
          <p className="text-3xl font-bold">{stats.growth}%</p>
          <p className="text-xs text-muted-foreground mt-1">vs last month</p>
        </Card>
      </div>

      {/* Quick Actions */}
      <div className="grid md:grid-cols-2 gap-6">
        <Card className="p-6">
          <h3 className="text-lg font-semibold mb-4">Quick Actions</h3>
          <div className="space-y-2">
            <Link href="/create">
              <Button className="w-full justify-start">
                Create New Project
              </Button>
            </Link>
            <Link href="/team">
              <Button variant="outline" className="w-full justify-start">
                Invite Team Members
              </Button>
            </Link>
          </div>
        </Card>

        <Card className="p-6">
          <h3 className="text-lg font-semibold mb-4">Recent Activity</h3>
          <div className="space-y-3">
            <div className="flex items-center gap-3">
              <div className="h-2 w-2 bg-green-600 rounded-full" />
              <p className="text-sm">New user signed up</p>
            </div>
            <div className="flex items-center gap-3">
              <div className="h-2 w-2 bg-blue-600 rounded-full" />
              <p className="text-sm">Project updated</p>
            </div>
          </div>
        </Card>
      </div>
    </div>
  )
}
```

---

## API Patterns & Best Practices

### 1. API Route Structure

**Location**: `app/api/[feature]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'
import { prisma } from '@/lib/prisma'
import { z } from 'zod' // Optional: for validation

// Request body validation schema (optional but recommended)
const requestSchema = z.object({
  title: z.string().min(1),
  content: z.string().min(1),
})

export async function GET(request: NextRequest) {
  try {
    // 1. Authenticate
    const session = await getServerSession(authOptions)
    
    if (!session?.user) {
      return NextResponse.json(
        { error: 'Unauthorized' }, 
        { status: 401 }
      )
    }

    // 2. Get query parameters
    const searchParams = request.nextUrl.searchParams
    const page = parseInt(searchParams.get('page') || '1')
    const limit = parseInt(searchParams.get('limit') || '10')

    // 3. Fetch data
    const items = await prisma.post.findMany({
      where: { authorId: session.user.id },
      take: limit,
      skip: (page - 1) * limit,
      orderBy: { createdAt: 'desc' },
    })

    // 4. Return response
    return NextResponse.json({ 
      items,
      page,
      totalPages: Math.ceil(items.length / limit)
    })
  } catch (error) {
    console.error('API Error:', error)
    return NextResponse.json(
      { error: 'Internal Server Error' }, 
      { status: 500 }
    )
  }
}

export async function POST(request: NextRequest) {
  try {
    // 1. Authenticate
    const session = await getServerSession(authOptions)
    
    if (!session?.user) {
      return NextResponse.json(
        { error: 'Unauthorized' }, 
        { status: 401 }
      )
    }

    // 2. Parse and validate request body
    const body = await request.json()
    const validatedData = requestSchema.parse(body)

    // 3. Create data
    const item = await prisma.post.create({
      data: {
        title: validatedData.title,
        content: validatedData.content,
        authorId: session.user.id,
      },
    })

    // 4. Return response
    return NextResponse.json({ item }, { status: 201 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation error', details: error.errors }, 
        { status: 400 }
      )
    }
    
    console.error('API Error:', error)
    return NextResponse.json(
      { error: 'Internal Server Error' }, 
      { status: 500 }
    )
  }
}

export async function PUT(request: NextRequest) {
  try {
    const session = await getServerSession(authOptions)
    
    if (!session?.user) {
      return NextResponse.json(
        { error: 'Unauthorized' }, 
        { status: 401 }
      )
    }

    const body = await request.json()
    const { id, ...updateData } = body

    // Check ownership
    const existing = await prisma.post.findUnique({
      where: { id },
      select: { authorId: true },
    })

    if (!existing || existing.authorId !== session.user.id) {
      return NextResponse.json(
        { error: 'Not found or unauthorized' }, 
        { status: 404 }
      )
    }

    // Update
    const updated = await prisma.post.update({
      where: { id },
      data: updateData,
    })

    return NextResponse.json({ item: updated })
  } catch (error) {
    console.error('API Error:', error)
    return NextResponse.json(
      { error: 'Internal Server Error' }, 
      { status: 500 }
    )
  }
}

export async function DELETE(request: NextRequest) {
  try {
    const session = await getServerSession(authOptions)
    
    if (!session?.user) {
      return NextResponse.json(
        { error: 'Unauthorized' }, 
        { status: 401 }
      )
    }

    const searchParams = request.nextUrl.searchParams
    const id = searchParams.get('id')

    if (!id) {
      return NextResponse.json(
        { error: 'Missing id parameter' }, 
        { status: 400 }
      )
    }

    // Check ownership
    const existing = await prisma.post.findUnique({
      where: { id },
      select: { authorId: true },
    })

    if (!existing || existing.authorId !== session.user.id) {
      return NextResponse.json(
        { error: 'Not found or unauthorized' }, 
        { status: 404 }
      )
    }

    // Delete
    await prisma.post.delete({
      where: { id },
    })

    return NextResponse.json({ success: true })
  } catch (error) {
    console.error('API Error:', error)
    return NextResponse.json(
      { error: 'Internal Server Error' }, 
      { status: 500 }
    )
  }
}
```

### 2. Client-Side API Calls

**Location**: `lib/api.ts`

```typescript
export class APIError extends Error {
  constructor(public status: number, message: string) {
    super(message)
    this.name = 'APIError'
  }
}

export async function fetchAPI<T>(
  url: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  })

  if (!response.ok) {
    const error = await response.json().catch(() => ({ error: 'Unknown error' }))
    throw new APIError(response.status, error.error || 'Request failed')
  }

  return response.json()
}

// Usage in components
export async function createPost(data: { title: string; content: string }) {
  return fetchAPI('/api/posts', {
    method: 'POST',
    body: JSON.stringify(data),
  })
}

export async function getPosts(page = 1, limit = 10) {
  return fetchAPI(`/api/posts?page=${page}&limit=${limit}`)
}
```

**Usage in Component**:

```typescript
'use client'

import { useState } from 'react'
import { createPost } from '@/lib/api'
import { Button } from '@/components/ui/button'

export function CreatePostForm() {
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setIsLoading(true)
    setError(null)

    const formData = new FormData(e.currentTarget)
    const data = {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
    }

    try {
      await createPost(data)
      // Success: redirect or show success message
    } catch (err) {
      setError(err instanceof Error ? err.message : 'An error occurred')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input 
        name="title" 
        placeholder="Title" 
        required 
        className="w-full px-4 py-2 border rounded-md"
      />
      <textarea 
        name="content" 
        placeholder="Content" 
        required 
        className="w-full px-4 py-2 border rounded-md"
      />
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <Button type="submit" disabled={isLoading}>
        {isLoading ? 'Creating...' : 'Create Post'}
      </Button>
    </form>
  )
}
```

---

## Step-by-Step Setup Guide

### Phase 1: Project Initialization

```bash
# 1. Create Next.js app
pnpx create-next-app@latest my-app --typescript --tailwind --app
cd my-app

# 2. Install core dependencies
pnpm add next-auth @next-auth/prisma-adapter @prisma/client
pnpm add -D prisma

# 3. Install UI dependencies
pnpm add lucide-react class-variance-authority clsx tailwind-merge

# 4. Initialize Prisma
pnpm prisma init
```

### Phase 2: Environment Setup

Create `.env.local`:

```env
# Database
DATABASE_URL="postgresql://user:password@host:5432/dbname?sslmode=require"

# NextAuth
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="generate-with-openssl-rand-base64-32"

# Google OAuth
GOOGLE_CLIENT_ID="your-google-client-id"
GOOGLE_CLIENT_SECRET="your-google-client-secret"
```

### Phase 3: Database Configuration

1. Update `prisma/schema.prisma` with User, Account, Session, VerificationToken models
2. Run `pnpm prisma generate`
3. Run `pnpm prisma db push`
4. Verify with `pnpm prisma studio`

### Phase 4: Authentication Setup

1. Create `lib/auth.ts` with NextAuth configuration
2. Create `lib/prisma.ts` with Prisma client
3. Create `app/api/auth/[...nextauth]/route.ts`
4. Create `components/providers.tsx`
5. Update `app/layout.tsx` with Providers
6. Create `types/next-auth.d.ts` for TypeScript

### Phase 5: UI Components

1. Create `components/ui/button.tsx`, `card.tsx`, etc.
2. Update `app/globals.css` with design tokens
3. Update `tailwind.config.js`

### Phase 6: Pages & Routes

1. Create landing page: `app/page.tsx`
2. Create sign-in page: `app/(auth)/signin/page.tsx`
3. Create dashboard layout: `app/(dashboard)/layout.tsx`
4. Create dashboard page: `app/(dashboard)/dashboard/page.tsx`

### Phase 7: API Routes

1. Create API routes following pattern above
2. Add data fetching logic
3. Test with tools like Postman or Thunder Client

### Phase 8: Testing & Deployment

1. Test locally: `pnpm dev`
2. Build: `pnpm build`
3. Deploy to Vercel
4. Set environment variables in Vercel
5. Test production deployment

---

## Common Patterns & Code Examples

### 1. Server Component Data Fetching

```typescript
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'
import { prisma } from '@/lib/prisma'

export default async function Page() {
  const session = await getServerSession(authOptions)
  
  const data = await prisma.post.findMany({
    where: { authorId: session?.user.id },
  })

  return <div>{/* Render data */}</div>
}
```

### 2. Client Component with State

```typescript
'use client'

import { useState, useEffect } from 'react'
import { fetchAPI } from '@/lib/api'

export function ClientComponent() {
  const [data, setData] = useState(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    fetchAPI('/api/data')
      .then(setData)
      .finally(() => setIsLoading(false))
  }, [])

  if (isLoading) return <div>Loading...</div>
  
  return <div>{/* Render data */}</div>
}
```

### 3. Form Handling with React Hook Form

```bash
pnpm add react-hook-form zod @hookform/resolvers
```

```typescript
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  title: z.string().min(1, 'Title is required'),
  content: z.string().min(10, 'Content must be at least 10 characters'),
})

type FormData = z.infer<typeof schema>

export function MyForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  })

  const onSubmit = async (data: FormData) => {
    await fetch('/api/posts', {
      method: 'POST',
      body: JSON.stringify(data),
    })
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('title')} />
      {errors.title && <span>{errors.title.message}</span>}
      
      <textarea {...register('content')} />
      {errors.content && <span>{errors.content.message}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        Submit
      </button>
    </form>
  )
}
```

### 4. Optimistic Updates

```typescript
'use client'

import { useOptimistic } from 'react'

export function TodoList({ initialTodos }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    initialTodos,
    (state, newTodo) => [...state, { ...newTodo, sending: true }]
  )

  async function createTodo(formData) {
    addOptimisticTodo({ id: Date.now(), text: formData.get('text') })
    await fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify({ text: formData.get('text') }),
    })
  }

  return (
    <form action={createTodo}>
      {optimisticTodos.map(todo => (
        <div key={todo.id} style={{ opacity: todo.sending ? 0.5 : 1 }}>
          {todo.text}
        </div>
      ))}
      <input name="text" />
      <button type="submit">Add</button>
    </form>
  )
}
```

### 5. Server Actions (Next.js 14)

```typescript
// app/actions.ts
'use server'

import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  const session = await getServerSession(authOptions)
  
  if (!session?.user) {
    throw new Error('Unauthorized')
  }

  await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
      authorId: session.user.id,
    },
  })

  revalidatePath('/dashboard')
}

// Component usage
'use client'

import { createPost } from './actions'

export function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit">Create</button>
    </form>
  )
}
```

---

## Deployment Configuration

### Vercel Deployment

**1. Connect Repository**
- Go to [vercel.com](https://vercel.com)
- Import your GitHub repository
- Vercel auto-detects Next.js

**2. Environment Variables**

Add to Vercel dashboard:
```env
DATABASE_URL=your-production-database-url
NEXTAUTH_URL=https://your-app.vercel.app
NEXTAUTH_SECRET=your-production-secret
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

**3. Build Settings**

Vercel default settings work for most cases:
- Build Command: `pnpm build` or `next build`
- Output Directory: `.next`
- Install Command: `pnpm install`

**4. Deploy**
- Push to `main` branch triggers automatic deployment
- Preview deployments for pull requests

### Custom Domain Setup

1. Add domain in Vercel dashboard
2. Update DNS records as instructed
3. Update `NEXTAUTH_URL` to your custom domain
4. Update Google OAuth redirect URIs

### Database Migrations

For production:
```bash
# Generate migration
pnpm prisma migrate dev --name init

# Apply in production (via Vercel)
# Add to package.json:
{
  "scripts": {
    "postinstall": "prisma generate",
    "vercel-build": "prisma migrate deploy && next build"
  }
}
```

---

## Appendix: Essential Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "prisma generate && next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    
    "db:generate": "prisma generate",
    "db:push": "prisma db push",
    "db:migrate": "prisma migrate dev",
    "db:studio": "prisma studio",
    "db:seed": "tsx prisma/seed.ts",
    
    "postinstall": "prisma generate"
  }
}
```

---

## Summary Checklist

### ✅ Setup Checklist

- [ ] Next.js 14 project created with TypeScript & Tailwind
- [ ] Database (Neon PostgreSQL) provisioned
- [ ] Prisma schema configured with NextAuth models
- [ ] Google OAuth credentials obtained
- [ ] Environment variables configured
- [ ] NextAuth.js setup complete
- [ ] Protected route layout created
- [ ] Landing page designed
- [ ] Dashboard created
- [ ] API routes implemented
- [ ] UI components library setup
- [ ] Deployment to Vercel configured

### 🚀 Production Readiness

- [ ] Database migrations run
- [ ] Environment variables set in Vercel
- [ ] Google OAuth redirect URIs updated for production
- [ ] Error handling implemented
- [ ] Loading states added
- [ ] SEO metadata configured
- [ ] Analytics integrated (optional)
- [ ] Monitoring setup (optional)

---

## Additional Resources

- **Next.js Documentation**: [nextjs.org/docs](https://nextjs.org/docs)
- **NextAuth.js Guide**: [next-auth.js.org](https://next-auth.js.org)
- **Prisma Docs**: [prisma.io/docs](https://prisma.io/docs)
- **Tailwind CSS**: [tailwindcss.com/docs](https://tailwindcss.com/docs)
- **Radix UI**: [radix-ui.com](https://radix-ui.com)
- **Vercel Docs**: [vercel.com/docs](https://vercel.com/docs)

---

**Last Updated**: October 2025  
**Version**: 1.0.0  
**Maintained By**: [Your Name/Team]

---

This guide provides a complete foundation for building modern, full-stack Next.js 14 applications with authentication, database integration, and production-ready patterns. Adapt and extend based on your specific requirements.


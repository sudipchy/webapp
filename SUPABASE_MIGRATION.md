# Supabase Migration Guide

This guide explains how to migrate QuicklyPeer from a local SQLite database to Supabase PostgreSQL while keeping your frontend running on localhost.

## Overview

After migration, your setup will be:
- **Frontend**: Running locally on `http://localhost:3000`
- **Database**: Supabase PostgreSQL (cloud-hosted)
- **Signaling Server**: Running locally on `http://localhost:3003`

## Prerequisites

1. A Supabase account (free tier is sufficient)
2. Node.js/Bun installed locally

## Step-by-Step Migration

### 1. Create a Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign in/sign up
2. Click "New Project"
3. Fill in the project details:
   - **Name**: quicklypeer (or your preferred name)
   - **Database Password**: Choose a strong password (save this!)
   - **Region**: Choose a region close to you
4. Wait for the project to be created (typically 1-2 minutes)

### 2. Get Your Database Connection String

1. In your Supabase project dashboard, go to **Settings** → **Database**
2. Scroll down to **Connection string**
3. Select **URI** tab
4. Copy the connection string, which looks like:
   ```
   postgresql://postgres:[YOUR-PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres
   ```
5. Replace `[YOUR-PASSWORD]` with your actual database password

### 3. Install New Dependencies

Since we added PostgreSQL support, install the new dependencies:

```bash
bun install
# or
npm install
```

### 4. Update Your Local `.env` File

1. Copy the example env file (if you haven't already):
   ```bash
   cp .env.example .env
   ```

2. Open `.env` in your editor

3. Replace the `DATABASE_URL` with your Supabase connection string:
   ```bash
   DATABASE_URL="postgresql://postgres:your-actual-password@db.your-project-ref.supabase.co:5432/postgres"
   ```

4. Keep the signaling server URL as is (for local development):
   ```bash
   NEXT_PUBLIC_SIGNALING_URL="http://localhost:3003"
   ```

### 5. Push the Database Schema to Supabase

Now sync your Prisma schema to Supabase:

```bash
bun run db:push
# or
npx prisma db push
```

This will create the following tables in your Supabase database:
- `TransferSession` - Manages transfer sessions
- `TransferFile` - Stores file metadata
- `Device` - Tracks connected devices

### 6. Configure Supabase CORS Settings

To allow your localhost frontend to connect to Supabase:

1. In Supabase dashboard, go to **Settings** → **API**
2. Under **Project URL**, copy your project URL (e.g., `https://your-project-ref.supabase.co`)
3. In Supabase dashboard, go to **Settings** → **Database**
4. Scroll to **Connection Pooling** (if using connection pooling)
5. Add `http://localhost:3000` to allowed origins if required

Note: Prisma connects directly to the database, so CORS is less critical for database access. If you later add Supabase Auth or Realtime, you'll need to configure CORS in the Supabase dashboard.

### 7. Test the Setup

1. Start the signaling server (in a separate terminal):
   ```bash
   cd mini-services/signaling-server
   bun run dev
   ```

2. Start the Next.js app (in your main terminal):
   ```bash
   bun run dev
   ```

3. Open `http://localhost:3000` in your browser

4. Try creating a transfer session to verify the database connection works

### 8. Verify the Migration

Check your Supabase dashboard:
1. Go to **Table Editor** in the left sidebar
2. You should see your three tables: `TransferSession`, `TransferFile`, `Device`
3. Try creating a transfer in your app - you should see data appear in Supabase

## Optional: Add Supabase Auth/Realtime (Future Enhancement)

The current migration only uses Supabase for the database. If you want to use Supabase for authentication or real-time features later:

### Enable Supabase Auth

1. In Supabase dashboard, go to **Authentication** → **Providers**
2. Enable email/password or social providers as needed
3. Add these to your `.env`:
   ```bash
   NEXT_PUBLIC_SUPABASE_URL="https://your-project-ref.supabase.co"
   NEXT_PUBLIC_SUPABASE_ANON_KEY="your-anon-key-from-dashboard"
   ```
4. Install the Supabase client:
   ```bash
   bun add @supabase/supabase-js
   ```

### Migrate Signaling to Supabase Realtime

To replace the local WebSocket signaling server with Supabase Realtime:
1. Enable Realtime in Supabase dashboard
2. Install Supabase Realtime client
3. Refactor `mini-services/signaling-server/index.ts` to use Supabase channels
4. Update `src/lib/signaling-client.ts` to connect to Supabase instead of localhost

This is more complex and not required for the basic migration.

## Troubleshooting

### "Connection refused" or "timeout" errors

- Verify your DATABASE_URL is correct
- Check that your Supabase project is active (not paused)
- Ensure your database password is correct
- Try the connection string in a tool like TablePlus or DBeaver to test connectivity

### Prisma migration fails

- Ensure you ran `bun install` after updating package.json
- Check that the DATABASE_URL is set in your `.env` file
- Try running `bun run db:push --force-reset` to reset the database (WARNING: deletes data)

### Frontend can't connect to database

- Prisma runs server-side, so the browser doesn't connect directly
- Check your server logs for Prisma errors
- Ensure the database credentials are correct in `.env`

### Signaling server issues

- Ensure the signaling server is running on port 3003
- Check that `NEXT_PUBLIC_SIGNALING_URL="http://localhost:3003"` is set in `.env`
- Restart the Next.js dev server after changing `.env` (NEXT_PUBLIC_* vars are read at startup)

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                   Local Development                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐         ┌──────────────┐              │
│  │  Next.js     │         │  Signaling   │              │
│  │  (localhost: │◄────────┤  Server      │              │
│  │  3000)       │  WS     │  (localhost: │              │
│  └──────┬───────┘         │  3003)       │              │
│         │                 └──────────────┘              │
│         │                                                │
│         │                                                │
│         │               Prisma Client                    │
│         │                      │                         │
│         │                      ▼                         │
│         │         ┌──────────────────────┐              │
│         └─────────┤  Supabase PostgreSQL │              │
│                   │  (Cloud-hosted DB)   │              │
│                   └──────────────────────┘              │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

## Benefits of This Setup

- **Frontend on localhost**: Fast development, hot reload, easy debugging
- **Database in cloud**: Persistent data, accessible from anywhere, production-ready
- **Signaling server local**: Simple setup, no extra infrastructure needed for dev
- **Easy to scale**: Move signaling server to cloud when ready for production

## Reverting to SQLite

If you need to revert to local SQLite:

1. Update `.env`:
   ```bash
   DATABASE_URL="file:./db/custom.db"
   ```

2. Update `prisma/schema.prisma`:
   ```prisma
   datasource db {
     provider = "sqlite"
     url      = env("DATABASE_URL")
   }
   ```

3. Run `bun run db:push` to recreate local database

4. Restart your dev server

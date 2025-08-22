```prisma
// schema.prisma (Postgres)

generator client {
  provider = "prisma-client-js"
  // If you want DESC index ordering directly in Prisma later:
  // previewFeatures = ["extendedIndexes"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum PollStatus {
  draft   // not visible / not votable
  open    // visible & votable
  closed  // visible, voting disabled
}

model Poll {
  id         String     @id @default(uuid()) @db.Uuid
  slug       String     @unique @db.Text       // URL handle
  question   String     @db.Text
  status     PollStatus
  category   String?    @db.Text

  // Server-managed timestamps (best practice: don't let clients set these)
  createdAt  DateTime   @default(now()) @db.Timestamptz(6)
  updatedAt  DateTime   @updatedAt      @db.Timestamptz(6)

  // Relations
  options    PollOption[]
  votes      Vote[]

  @@map("poll")
}

model PollOption {
  id        String  @id @default(uuid()) @db.Uuid
  pollId    String  @db.Uuid
  label     String  @db.Text             // No "position" column; sort in UI (alphabetical or whatever)

  poll      Poll    @relation(fields: [pollId], references: [id], onDelete: Cascade)
  votes     Vote[]

  // Nice to have: prevent duplicate labels within a poll
  @@unique([pollId, label])
  @@index([pollId])
  @@map("poll_option")
}

/**
 * Vote = append-only ledger (immutability keeps history + avoids UPDATE races).
 * Auth-only: userId is REQUIRED. No fingerprints.
 * Users can change opinion: we INSERT a new row; "current" = latest by votedAt for (pollId, userId).
 * Use idempotencyKey to de-dupe client retries on the same action.
 */
model Vote {
  id             String   @id @default(uuid()) @db.Uuid
  pollId         String   @db.Uuid
  optionId       String   @db.Uuid

  // Server time of the event
  votedAt        DateTime @default(now()) @db.Timestamptz(6)

  // Auth-only identity
  userId         String   @db.Uuid

  // Dedup on client retries (optional but recommended) or @@unique([userId, idempotencyKey])
  idempotencyKey String?  @unique @db.Text

  poll           Poll       @relation(fields: [pollId], references: [id], onDelete: Cascade)
  option         PollOption @relation(fields: [optionId], references: [id], onDelete: Restrict)

  // Helpful indexes
  @@index([pollId])
  @@index([optionId])
  @@index([pollId, userId]) // supports "latest per user" lookups

  @@map("vote")
}
```

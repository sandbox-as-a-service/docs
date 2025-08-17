```prisma
// Postgres setup
generator client {
  provider = "prisma-client-js"
  // If you want ordered indexes in Prisma later, enable previewFeatures = ["extendedIndexes"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

/// Poll status lifecycle.
/// - draft: not visible / not votable
/// - open: visible & votable
/// - closed: visible, voting disabled
enum PollStatus {
  draft
  open
  closed
}

/// Top-level poll entity.
/// URL-friendly lookup is via unique `slug`.
model Poll {
  id         String       @id @default(uuid()) @db.Uuid
  slug       String       @unique @db.Text
  question   String       @db.Text
  status     PollStatus
  category   String?      @db.Text
  createdAt  DateTime     @default(now()) @db.Timestamptz(6)
  updatedAt  DateTime     @updatedAt @db.Timestamptz(6)

  // Relations
  options    PollOption[]
  votes      Vote[]

  @@map("poll")
}

/// Poll options with stable display ordering via `position`.
/// (Index below speeds ordered reads; make it @@unique([pollId, position]) if
/// you want to *enforce* that each position is used only once per poll.)
model PollOption {
  id        String  @id @default(uuid()) @db.Uuid
  pollId    String  @db.Uuid
  label     String  @db.Text
  position  Int
  // Suggestion: keep labels immutable after opening the poll to avoid confusion.

  // Relations
  poll      Poll    @relation(fields: [pollId], references: [id], onDelete: Cascade)
  votes     Vote[]

  @@index([pollId, position]) // fast ordered render per poll
  @@map("poll_option")
}

/// Append-only vote *ledger*.
/// - Do not UPDATE or DELETE rows. A changed opinion is a *new* row.
/// - `idempotencyKey` prevents accidental duplicate inserts on client retries.
/// - Current results = "latest row per identity (userId OR voterFingerprint) for this poll".
model Vote {
  id               String   @id @default(uuid()) @db.Uuid

  // Required foreign keys
  pollId           String   @db.Uuid
  optionId         String   @db.Uuid

  // When this vote event happened (server time)
  votedAt          DateTime @default(now()) @db.Timestamptz(6)

  // Identity (choose one or both; both optional to keep MVP simple)
  userId           String?  @db.Uuid             // present if logged-in
  voterFingerprint String?  @db.Text             // fallback for anonymous

  // Idempotency for at-least-once delivery from clients
  idempotencyKey   String?  @unique @db.Text

  // Relations
  poll             Poll     @relation(fields: [pollId], references: [id], onDelete: Cascade)
  option           PollOption @relation(fields: [optionId], references: [id], onDelete: Restrict)

  // Basic counting/lookup indexes
  @@index([pollId])
  @@index([optionId])

  // NOTE: We also create (in raw SQL) a *generated* column `identity_key = COALESCE(user_id::text, voter_fingerprint)`
  // and an index (poll_id, identity_key, voted_at DESC, id DESC) to make "latest per identity" fast.
  // Prisma can't express generated columns or partial uniques yet—see migration SQL below.

  @@map("vote")
}
```

---
name: data-modeling
description: "Schema design, normalization levels, indexing strategy matched to access patterns. Use when designing database schemas, choosing between embed vs reference, or optimizing slow queries."
---

## Data Modeling

Design schemas that serve your access patterns efficiently. The right schema makes queries fast and simple. The wrong schema makes every query a workaround.

### Context

Domain to model: **$ARGUMENTS**

---

### Step 1: Identify Access Patterns First

Schema follows access patterns, not the other way around. List every query before writing a single table definition.

```
Access Pattern Analysis Template:
Query                                  Frequency    Latency Target
----------------------------------------------------------------------
Get user by email (login)              Very High    < 10ms
Get all posts by user, newest first    High         < 50ms
Get post with author info              High         < 50ms
Get comments for a post, paginated     Medium       < 100ms
Search posts by keyword                Medium       < 200ms
Get user follower count                High         < 10ms
Get users a user follows (feed)        High         < 50ms
```

This list drives every schema decision. If a query isn't on the list, it's not shaping the schema.

---

### Step 2: Relational (PostgreSQL / MySQL)

**Normalization levels in practice:**

- **1NF**: Atomic values, no repeating groups. Enforce with proper column types.
- **2NF**: No partial dependencies on composite key. Each column depends on the whole key.
- **3NF**: No transitive dependencies. Each column depends only on the primary key.
- **Denormalization**: Intentionally break 3NF when join cost exceeds duplication cost. Always document why.

```sql
-- Well-normalized blog schema

CREATE TABLE users (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  email       VARCHAR(255) NOT NULL UNIQUE,
  display_name VARCHAR(50) NOT NULL,
  password_hash CHAR(60)  NOT NULL,  -- bcrypt output is always 60 chars
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE posts (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  author_id   UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title       VARCHAR(255) NOT NULL,
  body        TEXT        NOT NULL,
  status      VARCHAR(20) NOT NULL DEFAULT 'draft'
                          CHECK (status IN ('draft', 'published', 'archived')),
  published_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id     UUID        NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  author_id   UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  body        TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tags: many-to-many requires junction table
CREATE TABLE tags (
  id   UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50)  NOT NULL UNIQUE
);

CREATE TABLE post_tags (
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  UUID NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);
```

---

### Step 3: Indexing Strategy

Indexes are fast reads at the cost of slower writes and storage. Create indexes for columns that appear in WHERE, JOIN ON, and ORDER BY.

```sql
-- Primary key indexes are automatic

-- Index for login query (email lookup)
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Index for "posts by author, newest first"
CREATE INDEX idx_posts_author_created ON posts(author_id, created_at DESC);

-- Index for "published posts only" (partial index — smaller and faster)
CREATE INDEX idx_posts_published ON posts(published_at DESC)
  WHERE status = 'published';

-- Index for comments by post (paginated)
CREATE INDEX idx_comments_post_created ON comments(post_id, created_at ASC);

-- Full-text search index
CREATE INDEX idx_posts_fts ON posts USING GIN(
  to_tsvector('english', title || ' ' || body)
);
```

**Index selection rules:**
- B-tree (default): equality and range queries, sorting
- GIN: full-text search, JSONB operators, array contains
- GiST: geographic data, range types
- BRIN: time-series data, naturally ordered (cheap, but less precise)
- Hash: equality only, no range — rarely better than B-tree in PostgreSQL

**Composite index column order:** put equality predicates first, range predicates last.
```sql
-- Query: WHERE author_id = $1 AND status = 'published' ORDER BY created_at DESC
-- Good: (author_id, status, created_at DESC) — filters by both equality first
-- Bad:  (created_at, author_id) — sorts before filtering
```

---

### Step 4: MongoDB / Document Store

Decide between embedding and referencing based on access patterns and cardinality.

**Embed when:**
- Data is always accessed together (read together = store together)
- One-to-few relationship (< ~16 documents in the embedded array)
- Embedded data doesn't grow unboundedly
- No need to query embedded documents independently

**Reference when:**
- Data is queried independently
- One-to-many or many-to-many with large cardinality
- Embedded array would grow unboundedly
- Multiple parents reference the same child

```javascript
// Example: Blog (Mongoose)

// EMBED: post with its tags (few, always accessed with post, not queried standalone)
const postSchema = new mongoose.Schema({
  authorId:  { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  title:     { type: String, required: true, maxlength: 255 },
  body:      { type: String, required: true },
  status:    { type: String, enum: ['draft', 'published', 'archived'], default: 'draft' },
  tags:      [{ type: String, maxlength: 50 }],  // embedded — tags are few and simple
  publishedAt: { type: Date },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

// Compound index for "posts by author, newest first"
postSchema.index({ authorId: 1, createdAt: -1 });

// Partial index for published posts
postSchema.index({ publishedAt: -1 }, { partialFilterExpression: { status: 'published' } });

// Text search index
postSchema.index({ title: 'text', body: 'text' });


// REFERENCE: comments (potentially many, queried independently)
const commentSchema = new mongoose.Schema({
  postId:   { type: mongoose.Schema.Types.ObjectId, ref: 'Post', required: true, index: true },
  authorId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  body:     { type: String, required: true, maxlength: 2000 },
  createdAt:{ type: Date, default: Date.now }
});

commentSchema.index({ postId: 1, createdAt: 1 });  // paginate comments for a post
```

**Extended Reference Pattern** (denormalize a subset of frequently-needed fields):
```javascript
// Instead of joining to get author's displayName every time:
const postSchema = new mongoose.Schema({
  author: {
    _id:         mongoose.Schema.Types.ObjectId,  // for real-time refresh if needed
    displayName: String,                           // denormalized — accept stale risk
    avatarUrl:   String
  },
  // ...rest of schema
});
// Trade-off: stale data if user updates displayName. Acceptable for non-critical display.
// Mitigation: background job to sync, or invalidate on update.
```

---

### Step 5: Common Pitfalls

**N+1 queries:** Loading N posts then issuing N queries to get each author.
```javascript
// Bad: N+1
const posts = await Post.find({ status: 'published' }).limit(20);
for (const post of posts) {
  post.author = await User.findById(post.authorId);  // N extra queries!
}

// Good: batch with populate (Mongoose) or JOIN (SQL)
const posts = await Post.find({ status: 'published' })
  .limit(20)
  .populate('authorId', 'displayName avatarUrl');  // single $in query
```

**Missing updated_at:** Always add `updatedAt` timestamps — invaluable for debugging, cache invalidation, and change data capture.

**Storing passwords in plain text:** Use bcrypt with work factor 12. Never store plaintext, never log passwords.

**Unbounded arrays (MongoDB):** An array that grows without limit will hit the 16MB BSON document limit. Use references instead of embedding when array size is unbounded.

**Missing soft delete:** Use `deletedAt TIMESTAMPTZ` instead of hard delete for auditable entities. Filter with `WHERE deleted_at IS NULL`.

---

### Output Format

```
## Data Model: [Domain]

### Access Patterns
| Query | Frequency | Latency Target |

### Schema

#### [Entity 1]
[Fields, types, constraints, indexes]

#### [Entity 2]
...

### Relationships
[ERD or relationship descriptions]

### Index Strategy
| Index | Columns | Type | Justification |

### Embed vs Reference Decisions (if document store)
[For each relationship: decision and rationale]

### Tradeoffs
[3-5 explicit tradeoffs: normalization vs performance, consistency vs flexibility, etc.]
```

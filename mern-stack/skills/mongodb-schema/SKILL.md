---
name: mongodb-schema
description: "Schema design with Mongoose: embedded vs referenced documents, validation rules, compound indexes, TTL indexes, sparse indexes. Use when designing MongoDB schemas for a Node.js application."
---

## MongoDB Schema Design (Mongoose)

Mongoose adds structure to MongoDB's flexibility. Use it to enforce schema, add validation, and create a clean data access layer.

### Context

Domain to model: **$ARGUMENTS**

---

### Mongoose Model Structure

```javascript
// models/User.model.js
import mongoose from 'mongoose';

const userSchema = new mongoose.Schema(
  {
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true,
      match: [/^[^\s@]+@[^\s@]+\.[^\s@]+$/, 'Invalid email format'],
      maxlength: [255, 'Email cannot exceed 255 characters']
    },
    passwordHash: {
      type: String,
      required: true,
      select: false  // never returned in queries unless explicitly requested
    },
    displayName: {
      type: String,
      required: [true, 'Display name is required'],
      trim: true,
      maxlength: [50, 'Display name cannot exceed 50 characters']
    },
    role: {
      type: String,
      enum: {
        values: ['user', 'moderator', 'admin'],
        message: 'Role must be one of: user, moderator, admin'
      },
      default: 'user'
    },
    emailVerified: {
      type: Boolean,
      default: false
    },
    deletedAt: {
      type: Date,
      default: null  // soft delete
    }
  },
  {
    timestamps: true,  // adds createdAt and updatedAt automatically
    toJSON: {
      virtuals: true,
      transform: (doc, ret) => {
        delete ret.passwordHash;  // never expose in JSON
        delete ret.__v;
        return ret;
      }
    }
  }
);

// Indexes
userSchema.index({ email: 1 }, { unique: true });  // login lookup
userSchema.index({ deletedAt: 1 }, { sparse: true });  // sparse: only indexes non-null

// Instance method
userSchema.methods.isPasswordMatch = async function (plainPassword) {
  return bcrypt.compare(plainPassword, this.passwordHash);
};

// Static method
userSchema.statics.findActiveByEmail = function (email) {
  return this.findOne({ email, deletedAt: null });
};

// Virtual (computed field, not stored)
userSchema.virtual('profileUrl').get(function () {
  return `/users/${this._id}`;
});

export const User = mongoose.model('User', userSchema);
```

---

### Embed vs Reference Decision

```javascript
// Rule: embed when data is ALWAYS accessed together and is ONE-TO-FEW
// Rule: reference when data is queried INDEPENDENTLY or is ONE-TO-MANY/MANY

// EMBED: Post with embedded tags (always fetched together, bounded, simple)
const postSchema = new mongoose.Schema({
  title: { type: String, required: true },
  body:  { type: String, required: true },
  tags:  [{ type: String, maxlength: 50 }],  // embed: few, simple, always needed
  author: {
    _id:         { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    displayName: String,   // denormalized: stale risk accepted for read performance
    avatarUrl:   String
  }
});

// REFERENCE: Comments on a post (potentially many, queried independently)
const commentSchema = new mongoose.Schema({
  postId:   { type: mongoose.Schema.Types.ObjectId, ref: 'Post', required: true },
  authorId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  body:     { type: String, required: true, maxlength: 2000 },
}, { timestamps: true });

commentSchema.index({ postId: 1, createdAt: 1 });  // paginate comments per post

// Mixed: Post with embedded category (1:few, stable) but referenced reviews (1:many)
const productSchema = new mongoose.Schema({
  name:     String,
  category: {            // embed: category is small and stable
    _id:  mongoose.Schema.Types.ObjectId,
    name: String,
    slug: String
  },
  // reviews: NOT embedded — could be thousands; use Review model with productId
});
```

---

### Advanced Indexes

```javascript
// 1. Compound index for common query pattern
// Query: posts by author, sorted by date
postSchema.index({ authorId: 1, createdAt: -1 });

// 2. Partial index: only index active documents
// More efficient than indexing all documents when most are deleted
userSchema.index(
  { email: 1 },
  {
    unique: true,
    partialFilterExpression: { deletedAt: null }  // only active users
  }
);

// 3. TTL index: automatically delete expired documents
// Use for: sessions, verification tokens, temporary data
const verificationTokenSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, required: true },
  token:  { type: String, required: true },
  createdAt: { type: Date, default: Date.now }
});

verificationTokenSchema.index(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // documents deleted 1 hour after createdAt
);

// 4. Text index for full-text search
postSchema.index(
  { title: 'text', body: 'text' },
  { weights: { title: 2, body: 1 } }  // title matches ranked higher
);

// Query with text index:
Post.find({ $text: { $search: 'javascript async' } }, { score: { $meta: 'textScore' } })
  .sort({ score: { $meta: 'textScore' } });

// 5. Sparse index: only indexes documents where the field exists
userSchema.index({ googleId: 1 }, { sparse: true, unique: true });
// googleId might not exist for all users (only OAuth users)
```

---

### Schema Validation Best Practices

```javascript
// Mongoose validators vs. Joi/Zod:
// Mongoose validators: last line of defense at DB layer
// Joi/Zod: validate at API layer BEFORE hitting DB
// Both: Joi/Zod for API input, Mongoose for data integrity

// Custom validator example
const priceSchema = {
  type: Number,
  required: true,
  validate: {
    validator: (v) => v >= 0,
    message: 'Price cannot be negative'
  }
};

// Conditional validation with pre-save hook
userSchema.pre('save', function (next) {
  if (this.isModified('passwordHash') && this.passwordHash.length < 60) {
    next(new Error('Password must be hashed before saving'));
  } else {
    next();
  }
});

// Mongoose query middleware (runs before any find query)
// Automatically filter out soft-deleted documents
userSchema.pre(/^find/, function () {
  this.find({ deletedAt: { $eq: null } });
  // Now User.find({}) automatically excludes soft-deleted users
  // To include deleted: User.find({ deletedAt: { $ne: null } }).setOptions({ skipMiddleware: true })
});
```

---

### Common Mongoose Pitfalls

```javascript
// 1. Missing .lean() on read-only queries (huge performance impact)
// Without .lean(): returns full Mongoose document with methods (~5-10x memory overhead)
// With .lean(): returns plain JS object (fast, memory efficient)

const user = await User.findById(id).lean();  // use for reads
const user = await User.findById(id);          // use only if you need to save() it

// 2. N+1 queries — use populate wisely
// BAD: N+1
const posts = await Post.find({ status: 'published' });
for (const post of posts) {
  post.fullAuthor = await User.findById(post.author._id);  // N queries!
}

// GOOD: populate (single $in query)
const posts = await Post.find({ status: 'published' })
  .populate('authorId', 'displayName avatarUrl email')  // fields to include
  .lean();

// 3. Missing await on mongoose operations
const user = User.create({ email: 'a@b.com' });  // returns Promise, not user!
const user = await User.create({ email: 'a@b.com' });  // correct

// 4. Using _id vs id
// _id: MongoDB ObjectId (stored in DB)
// id: string representation (virtual added by Mongoose toJSON)
// In queries: always use _id
// In JSON responses: use id (cleaner, no underscore)
```

---

### Output Format

```
## MongoDB Schema: [Domain]

### Models

#### [Model Name]
```js
[Full Mongoose schema with validators, indexes, methods]
```

### Embed vs Reference Decisions
| Relationship | Decision | Rationale |

### Index Strategy
| Index | Fields | Type | Purpose |

### Tradeoffs
[3-5 explicit tradeoffs]
```

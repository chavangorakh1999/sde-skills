---
description: "Design a MongoDB schema with Mongoose: embed vs reference, indexes, validation, and migration plan"
argument-hint: "[entity or domain to model, e.g. 'e-commerce orders with line items']"
---

## MongoDB Schema Design

Design a production-ready Mongoose schema for the given domain.

### Phase 1 — Entity Analysis

Map the domain:
- Identify all entities and their attributes
- Determine cardinality (1:1, 1:N, M:N)
- Classify access patterns — what queries will run most often?
- Identify hotspot documents (frequently updated fields)

### Phase 2 — Embed vs Reference Decision

Apply the rules:
- **Embed when**: data is accessed together, child rarely queried alone, bounded size (<= ~16MB), 1:few relationship
- **Reference when**: data accessed independently, shared across entities, unbounded arrays, M:N relationship
- Consider: "Is the child entity meaningful outside the parent?"

Produce a schema diagram showing embed/reference choices with rationale.

### Phase 3 — Mongoose Schema Code

Write the complete schema:
- All fields with types, required, default, enum
- Custom validators where schema-level validation adds value
- Timestamps: `{ timestamps: true }`
- Virtual fields for computed properties
- Pre-save hooks (e.g., capitalize, slugify)
- Schema methods and statics

```javascript
// Example structure expected:
const schema = new Schema({
  // fields...
}, {
  timestamps: true,
  toJSON: { virtuals: true, transform: /* remove __v, _id */ },
});
```

### Phase 4 — Index Strategy

Design indexes for the access patterns:
- Single field indexes for equality filters
- Compound indexes for sort + filter combinations (order matters: equality → range → sort)
- Sparse indexes for optional unique fields
- TTL indexes for expiring documents (sessions, tokens)
- Text indexes for search (or Atlas Search for production)

Provide the index definitions and explain why each is needed.

### Phase 5 — Repository Implementation

Write the repository layer:
- `findById`, `findOne`, `list` (with cursor pagination), `create`, `update`, `softDelete`
- `.lean()` on read queries for performance
- Error mapping: CastError → NotFoundError, code 11000 → ConflictError
- Projection: exclude sensitive fields (`-password`) by default

### Phase 6 — Migration Plan

If evolving an existing schema:
- Identify breaking vs additive changes
- Write migration script using Mongoose's `.updateMany()`
- Plan for zero-downtime: add optional field first, backfill async, make required in v2
- Recommend migrate-mongo or custom scripts for tracking applied migrations

### Output

1. **Entity relationship diagram** (text-based)
2. **Embed/reference decisions** with rationale table
3. **Complete Mongoose model code** with all fields, virtuals, hooks
4. **Index definitions** with access pattern mapping
5. **Repository layer** with all CRUD methods
6. **Migration plan** if applicable

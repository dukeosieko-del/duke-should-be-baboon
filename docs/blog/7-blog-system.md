# 7. Blog & community system

**Ground truth:** `src/lib/blog/{data,queries,notifications,types}.ts`,
`src/app/api/blog/**`, `src/app/api/admin/blog/**`,
`src/components/blog/**`, `supabase/migrations/20250101000031_blog_community_tables.sql`.

## Architecture
- **Static fallback** (`src/lib/blog/data.ts`): 10 hardcoded posts + 6 categories
  + 7 tags, used when the DB is empty (commit `ef9b295`, "fallback to static blog
  posts when database returns empty results").
- **DB source** (`src/lib/blog/queries.ts`): `blog_posts`, `blog_categories`,
  `blog_tags`, `blog_post_tags`, `blog_comments`, `blog_ratings`, `blog_reports`.
- **Authoring:** `/blog/write`, `/blog/drafts`, `/blog/my-articles` (auth-protected
  via middleware `/blog/write`, `/blog/drafts`, `/blog/my-articles`, `/blog/ && /edit`).

## Public read API
| Route | Purpose |
|---|---|
| `GET /api/blog/posts` | list (published), paginated, filterable by category/tag/search |
| `GET /api/blog/posts/[slug]` | single post + author + category + tags + rating stats |
| `GET /api/blog/posts/[slug]/comments` | nested comments (approved only) |
| `POST /api/blog/posts/[slug]/comments` | post a comment (auth) |
| `POST /api/blog/posts/[slug]/ratings` | rate 1-5 (auth, one per user) |
| `POST /api/blog/comments/[id]/report` | report a comment |
| `GET /api/blog/categories` | category list |
| `GET /api/blog/tags` | tag list |

## Admin moderation API (`requireAdmin`)
| Route | Purpose |
|---|---|
| `GET /api/admin/blog/posts` | list all (incl. drafts) |
| `GET/POST /api/admin/blog/posts/[id]` | read / update (status, visibility, featured, rejection_reason) |
| `GET /api/admin/blog/comments` | list all |
| `GET/PATCH /api/admin/blog/comments/[id]` | read / moderate (approve/reject/spam) |
| `POST /api/admin/blog/comments/[id]/reply` | reply → Brevo `getContactReplyHtml` email |
| `GET /api/admin/blog/ratings` | rating list |
| `GET /api/admin/blog/reports` | reported comments |
| `GET /api/admin/blog/authors` | author list |
| `GET/POST /api/admin/blog/categories` | category CRUD |
| `GET/POST /api/admin/blog/tags` | tag CRUD |
| `GET /api/admin/blog/notifications` | admin notification feed |

## Comments
- Nested via `nestComments()` (parent_id → replies tree).
- Statuses: `pending | approved | rejected | spam`.
- Reports stored in `blog_reports`.

## Ratings
- `blog_ratings(post_id, user_id, rating)` — one rating per user per post.
- `getPostRatingStats(post_id)` → `{ average, count }`.
- `getUserRating(postId, userId)` for the current user's rating.

## Notifications
- `increment_blog_view` RPC bumps `view_count`.
- `createNotification` / `notifyUser` / `notifyAdmins` for blog events.

## Walkthrough integration
`src/lib/walkthrough/**` drives an onboarding tour via `data-walkthrough`
attributes on key components (`WalkthroughRenderer`, `Spotlight`, `StepPopover`).
Journeys: `all`, `blog`, `customer`, `guest-order`, `notifications`, `orientation`.

## Rationale
- **Static fallback keeps the blog renderable without a DB.** The fallback is
  passive — when the DB has posts, `getLatestPosts()` returns DB rows and the
  static data is unused.
- **Comments are the only user-generated content on the site** (besides contact
  messages), so they have a full moderation pipeline: report → admin review →
  approve/reject/spam + reply-by-email.
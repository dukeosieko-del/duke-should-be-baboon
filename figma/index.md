# Figma Tree Index

This directory holds references to the Figma design trees that describe the
customer-facing UI surfaces. Each entry maps a Figma frame to the code route
that implements it, so an agent can jump from design → code surgically.

| Figma tree | Frame | Code route / component | Status |
|---|---|---|---|
| Onboarding | Sign-in modal | `src/components/AuthModal.tsx` → `/auth/sign-in` | implemented |
| Onboarding | Sign-up modal | `src/components/AuthModal.tsx` → `/auth/sign-in` (register tab) | implemented |
| Onboarding | Google OAuth button | `src/components/auth/GoogleAuthButton.tsx` | implemented |
| Catalog | Service catalogue | `src/components/ServiceCatalog.tsx` + `ServiceCatalogClient.tsx` | implemented (dynamic import wrapper) |
| Catalog | Platform listing | `src/app/services/[platform]/page.tsx` | implemented |
| Catalog | Others fallback | `src/app/services/others` (via `categorizeServices()`) | implemented |
| Checkout | Order form | `src/components/OrderForm.tsx` | implemented |
| Checkout | Guest checkout toggle | `src/app/api/orders/anonymous/route.ts` | implemented |
| Payments | M-Pesa modal | `src/components/MpesaModal.tsx` | implemented |
| Payments | Deposit success | `/deposit/success` | implemented |
| Wallet | Top-up flow | `/pay` → MpesaModal | implemented |
| Orders | Order tracking | `/orders/track` (public) + `/orders/all` (auth) | implemented |
| Admin | Services CRUD | `/admin/services` → `src/app/api/admin/services/route.ts` | implemented |
| Admin | Orders | `/admin/orders` | implemented |
| Admin | Ledger | `/admin/ledger` | implemented |
| Admin | Integrations | `/admin/integrations` | implemented |
| Blog | Article list | `/blog` | implemented |
| Blog | Article detail | `/blog/[slug]` | implemented |
| Blog | Comments + ratings | `src/components/blog/BlogComments.tsx`, `BlogRating.tsx` | implemented |
| Blog | Author dashboard | `/blog/write`, `/blog/drafts`, `/blog/my-articles` | implemented |
| Walkthrough | Engine overlay | `src/components/walkthrough/*` + `src/lib/walkthrough/*` | implemented |

> **Rationale:** Figma trees are referenced, not vendored, because the binary
> `.fig` files cannot live in a markdown repo. The mapping above is the
> authoritative index. If a Figma file is missing a frame, add the row here and
> flag the gap — that is the signal a route is unimplemented.
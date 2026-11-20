# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Use React 19 components and React Router to present each capability, Bootstrap for layout, and the shared `src/api/client.ts` helper for every API request. Keep loading, empty, success, and accessible error states in the presentation components.
- Logic tier decision: Use Express 5 routes under `/api/*` and keep validation and business behavior in explicit service functions so routes remain thin. Return deliberate 400/404 responses for invalid IDs, missing records, and unknown users instead of allowing Mongoose cast errors to become 500 responses.
- Data tier decision: Use Mongoose models for all four entities, MongoDB on port 27017, required schema validation, and an idempotent seed script that preserves unrelated records.
- Activity tracking approach: Store valid `user` and `team` references with type, minutes, computed points, and an occurrence timestamp. Validate the submitted user and team IDs, confirm both records exist, and expose create/list routes through the API contract.
- Team management approach: Require team names and mascots in Mongoose and reject POST requests without a name. Default members to an empty array, accept existing member IDs, and preserve submitted names, mascots, and members in API responses.
- Leaderboard approach: Aggregate activity points by team, sort descending, and attach a 1-based rank to each `{ teamName, totalPoints, rank }` result.
- Workout suggestions approach: Find the seeded user by email, derive the first recommendation from the user's fitness goal or recent activity pattern, and return a reason that explains the match. Return a deliberate 4xx response for unknown emails.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — Failures mean Mongoose fields, required validation, model exports, or idempotent seeding are incomplete.
- API command: `npm run test:api --prefix octofit-tracker/backend` — Failures mean routes or service behavior for activities, teams, the leaderboard, or workout suggestions do not satisfy the API contract.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — Failures mean React components do not load data through the shared client or do not present accessible loading, empty, success, and error states.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — Failures mean Express, Mongoose, or TypeScript types do not compile and must be corrected before tier completion.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — Failures mean React, Router, or Vite TypeScript/code compilation is incomplete and must be corrected before final verification.

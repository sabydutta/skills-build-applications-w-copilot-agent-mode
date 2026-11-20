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

- **Presentation tier:** Implement React 19 route components under `octofit-tracker/frontend/src/pages` and use React Router for `/activities`, `/teams`, `/leaderboard`, and `/workouts`. Use Bootstrap for layout and `src/api/client.ts` for every API request. Each capability must keep loading, empty, success, and accessible error states separate and explicit.
- **Logic tier:** Implement Express 5 routes under `/api/*` and keep validation and business behavior in explicit service functions so routes remain thin. Validate ObjectIds before querying, return deliberate `400`/`404` responses for invalid or missing references and unknown users, and never expose Mongoose cast errors as `500` responses.
- **Data tier:** Implement Mongoose models for `User`, `Team`, `Activity`, and `Workout`. Use MongoDB on port `27017`, required schema fields and references, and an idempotent seed script that preserves unrelated records and existing members outside the seeded fixtures.
- **Activity tracking:** Store valid `user` and `team` references, `type`, `minutes`, computed `points`, and `occurredAt`. Validate the submitted IDs, confirm both records exist, and expose create/list routes through the API contract.
- **Team management:** Require team names and mascots in Mongoose and reject POST requests without a name. Default members to an empty array, accept existing member IDs, and preserve submitted names, mascots, and members in API responses.
- **Leaderboard:** Aggregate activity points by team, sort descending, and attach a 1-based rank to each `{ teamName, totalPoints, rank }` result.
- **Workout suggestions:** Find the seeded user by email, derive the first recommendation from the user's fitness goal or recent activity pattern, and include a reason that explains the match. Return a deliberate 4xx response for unknown emails.
- **Smallest safe implementation order:** Complete and verify data models and idempotent seeding before logic-tier services and routes. Verify API behavior next, then add presentation components and states. Run the focused contract suites after each tier and finish with both builds.

## Test plan

- **Data tier:** `npm run test:data --prefix octofit-tracker/backend`
  - Failures mean Mongoose fields, required validation, model exports, or idempotent seeding are incomplete.
  - Pass criterion: all 5 data contract tests pass, including preservation of unrelated data and repeatable seed counts.
- **API tier:** `npm run test:api --prefix octofit-tracker/backend`
  - Failures mean routes or service behavior for activities, teams, the leaderboard, or workout suggestions do not satisfy the API contract.
  - Pass criterion: all 4 API contract tests pass with deliberate 4xx responses for invalid and unknown references.
- **Frontend tier:** `npm test --prefix octofit-tracker/frontend`
  - Failures mean React components do not load data through the shared client or do not present accessible loading, empty, success, and error states.
  - Pass criterion: all presentation contract tests pass for the four capability routes.
- **Backend build:** `npm run build --prefix octofit-tracker/backend`
  - Failures mean Express, Mongoose, or TypeScript types do not compile and must be corrected before tier completion.
- **Frontend build:** `npm run build --prefix octofit-tracker/frontend`
  - Failures mean React, Router, or Vite TypeScript/code compilation is incomplete and must be corrected before final verification.

### Execution record

The following results were observed on 2026-11-20:

- Data contract: 4 failed and 1 passed. Required fields and validation are missing across the four Mongoose entities; the seed idempotency test passes.
- API contract: 4 failed. Activity/team POST routes are not implemented, leaderboard aggregation is missing, and workout suggestions lack an explanatory reason.
- Frontend contract: 15 failed. Capability routes do not yet provide the required loading, data, empty, error, and workout-request behavior.
- Backend build: passed.
- Frontend build: passed.

GitHub Actions workflow `Validate Exercise` is active, but its latest run was skipped because the workflow's maintainer-only `if` condition requires the repository to be a template. The workflow is therefore not evidence of application contract results.

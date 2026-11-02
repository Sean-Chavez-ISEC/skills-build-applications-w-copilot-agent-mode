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

- **Presentation tier**: Keep route components focused on rendering and user interaction. Fetch through the shared API client; show an accessible loading status, meaningful empty state, and accessible error alert for each capability. Do not duplicate scoring or recommendation rules in React.
- **Logic tier**: Keep HTTP parsing and status codes in the Express routes and put validation and domain behavior in `octofitService`. Reject invalid input and missing references with deliberate 4xx responses before persistence, rather than allowing Mongoose cast errors to become 500s.
- **Data tier**: Use Mongoose schemas to require each contract field and represent user/team links as ObjectId references. Keep the existing upsert-based seed idempotent and preserve records that are not demo fixtures.
- **Activity tracking**: Accept `userId`, `teamId`, `type`, and positive `minutes`; validate both IDs and confirm both records exist. Persist the references, activity details, `occurredAt`, and computed `points`. Use one point per minute, consistent with the seeded examples.
- **Team management**: Require a non-empty name and mascot, default omitted members to an empty array, and store submitted member IDs as user references. List the stored name, mascot, and member IDs.
- **Leaderboard**: Sum activity points by team, sort totals descending, then assign 1-based ranks in the returned order.
- **Workout suggestions**: Find the user by email or return 404. Prefer workouts matching the user's fitness goal; if that does not identify a focus, use their most recent activity. Put the best match first and include a reason tied to the goal or activity.

Implement in dependency order: first add schema validation and keep the data/seed contract green; next complete and test activity and team service behavior; then add leaderboard aggregation and personalized workout selection; finally connect the four presentation routes to the API client and implement their loading, empty, and error states. Re-run the affected contract suite after each step, then run all suites and both builds.

## Test plan

- `npm run test:data --prefix octofit-tracker/backend`: verifies required model fields, references/defaults, and idempotent non-destructive seeding. Current baseline: 4 of 5 tests fail on missing required User, Team, Activity, and Workout fields; the seed test passes.
- `npm run test:api --prefix octofit-tracker/backend`: verifies HTTP validation, persistence/listing, leaderboard totals/ranks, and user-specific workout recommendations. Current baseline: all 4 tests fail; activity and team creation return 501 instead of validating input, the leaderboard is empty, and Mona's first workout does not match her endurance goal/activity. Later assertions in those API tests are not reached after the first failure.
- `npm test --prefix octofit-tracker/frontend`: verifies API-backed rendering, the user-specific workout request, and loading/empty/error states for all four routes. Current baseline: all 15 tests fail because the route components still render “Implementation pending.”
- `npm run build --prefix octofit-tracker/backend`: TypeScript check. Current baseline passes.
- `npm run build --prefix octofit-tracker/frontend`: TypeScript and production Vite build. Current baseline passes.

Run the relevant tier's contract command while implementing, then run all five commands above before considering the work complete. A failure in a contract suite identifies a behavior that still violates that tier's contract; a build failure identifies a compile-time or bundling issue rather than a runtime contract failure.

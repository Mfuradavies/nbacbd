# NBA Dashboard: Final Implementation Plan

## 1. Product scope

Build a desktop-first, public NBA dashboard that combines reliable game data,
live scores, player and team statistics, standings, and officially sourced
YouTube highlights.

Version 1 includes:

- Current-season NBA schedule, scores, game details, box scores, and play-by-play.
- Live scoreboard and live game pages.
- Player directories, profiles, game logs, career totals, shot charts, and comparisons.
- Team directories and team profiles.
- Standings, league leaders, and transparent advanced metrics.
- YouTube highlight discovery, matching, embedding, and administrative review.
- Email/password accounts with one favorite team and personalized ordering.
- Global search across players, teams, and games.
- Administration, ingestion monitoring, data verification, and quality reports.
- Legal, attribution, methodology, freshness, and unofficial-status pages.

Version 1 does not include:

- Other sports or leagues.
- Betting, odds, fantasy, predictions, notifications, or social features.
- Hosting or downloading video files; videos are embedded only.
- Mobile-specific layouts. Mobile is a post-launch phase.

Visual design is intentionally deferred. Accessibility, loading states, empty
states, error states, and responsive foundations are implementation
requirements, not visual-design decisions.

## 2. Technical decisions

- **Backend:** Python, Flask, SQLAlchemy, and database migrations.
- **Database:** PostgreSQL in local development and production. Do not design
  separate SQLite behavior.
- **Web:** Server-rendered Jinja templates with JavaScript for live updates,
  search, charts, and interactive tables(Javascript animation libralies if needed).
- **Worker:** A separate long-running Python worker for all external API access
  and scheduled jobs.
- **Cache:** Redis in production, accessed through a cache interface. Database
  fallback is required when Redis is unavailable.
- **Live transport:** Server-Sent Events from the web service to scoreboard and
  game-page clients.
- **NBA source:** Official NBA endpoints through adapters. `nba_api` may be used
  behind those adapters, but routes must never call external APIs directly.
- **Video source:** YouTube RSS for discovery and the YouTube Data API for
  metadata. Do not scrape YouTube HTML.
- **Testing:** pytest, recorded ingestion fixtures, metric formula tests,
  route tests, authentication tests, and cache-key tests.
- **Deployment:** Docker images, paid production web and worker services,
  managed PostgreSQL, managed Redis, HTTPS, and a custom domain.

## 3. Architecture and data flow

```text
NBA endpoints ───────┐
                     ├─> Worker adapters ─> normalizers ─> PostgreSQL
YouTube RSS/API ─────┘                                  └─> Redis/cache

Browser ──> Flask web service ──> PostgreSQL/Redis
     └──── SSE connection <────── live snapshots
```

The web service reads only local database and cache data. It must not make
request-time calls to NBA or YouTube. The worker owns ingestion, normalization,
aggregation, verification, cache warming, and retries.

Every ingestion job is idempotent and records an `ingestion_runs` row containing
job name, start time, finish time, status, rows written, and an error summary.
Use stable source IDs and unique constraints for upserts.

## 4. Ingestion jobs

| Job | Source | Frequency | Main output |
|---|---|---:|---|
| Reference sync | NBA static data | Daily and on demand | Teams, players, seasons |
| Schedule sync | NBA schedule endpoints | Daily and after postponements | Games and statuses |
| Live scoreboard poll | NBA live scoreboard | Every 20 seconds while games are live | Scores, periods, clocks |
| Live game detail poll | NBA live box score/play-by-play | Every 20 seconds per live game | Live stats and new events |
| Game finalization | NBA box score/play-by-play | On final status; retry until complete | Final stats, events, status |
| Player stat rollup | Internal | After finalization | Season and career aggregates |
| Standings rebuild | Internal plus NBA standings | After finalization and every 5 minutes on game days | Standings |
| Advanced metric compute | Internal | After finalization | Per-game and per-season metrics |
| YouTube feed poll | YouTube RSS | Every 10 minutes | New video records |
| Video metadata fetch | YouTube Data API | After discovery | Metadata and embeddability |
| Video matching | Internal | After metadata fetch | Candidate matches and confidence |
| Data verification | Internal/source comparison | Hourly and after finalization | Mismatches and quality flags |
| Cache warm | Internal | After finalization/standings rebuild | Hot pages and summaries |

Polling windows are schedule-driven: begin about five minutes before tip-off,
continue while the game is live, and stop shortly after finalization. Polling
uses backoff on failures and writes only changed live values where practical.

## 5. Data model

### Reference and game data

- `seasons`: label, dates, current flag.
- `teams`: source ID, names, abbreviation, conference, division, colors, logo.
- `players`: source ID, bio, current team, active state, headshot.
- `games`: source ID, season, UTC tip-off, teams, status, period, clock, score,
  arena, attendance, last sync.
- `line_scores`: game/team rows with period points and overtime count.
- `player_game_stats`: minutes, scoring, rebounds, assists, steals, blocks,
  turnovers, fouls, shooting, plus/minus, and starter status.
- `team_game_stats`: team counting stats and available pace/possession data.
- `play_events`: sequence, period, clock, participants, description, score,
  and optional shot coordinates.
- `shots`: player/team, time, coordinates, result, type, and zone.

### Aggregates

- `player_season_stats`: totals, averages, shooting percentages, and traded-team
  splits.
- `player_career_stats`: career totals and averages.
- `team_season_stats`: wins, losses, points for/against, pace, and ratings.
- `standings`: ranks, record, streak, last ten, home/road record, games behind,
  and whether the row is computed or source-backed.
- `advanced_metrics`: entity, season, metric name, value, formula version.
- `metric_definitions`: display name, formula, inputs, and version for the
  methodology page.

### Highlights

- `videos`: YouTube ID, channel, metadata, publish time, duration,
  embeddability, thumbnail, and status.
- `video_matches`: video/game association, confidence, method, reviewer, and
  review time.
- `highlight_channels`: YouTube channel ID, name, enabled state, optional team,
  and notes.

### Accounts and administration

- `users`: unique email, password hash, display name, role/status, timestamps.
- `favorite_teams`: one row per user and team.
- Server-side session/revocation data.
- Password reset tokens that are single-use and expire.
- `ingestion_runs`, verification results, search index, and cache metadata as
  needed by operations.

Use UTC timestamps and foreign keys throughout. Add unique constraints and
indexes for source IDs, game dates, player/team lookups, search fields, and
common page queries.

## 6. Caching and freshness

Use four layers:

1. Versioned browser assets and long-lived image caching.
2. HTTP response caching for public, non-personal pages.
3. Redis query/fragment caching behind a cache interface.
4. PostgreSQL summary tables or materialized views for expensive aggregates.

Freshness targets:

| Data | Target |
|---|---:|
| Live scoreboard/game pages | 20 seconds, plus SSE |
| Final game pages | Until a correction |
| Player/team season pages | 1 hour or finalization |
| Standings | 5 minutes on game days, 1 hour otherwise |
| Highlight feed | 10 minutes |
| Schedule | 1 hour |

Invalidate or version keys after finalization, corrections, schedule syncs,
standings rebuilds, and video matching. Personalization must not create one
cache entry per user: use shared per-team data and apply user-specific ordering
at response time.

If Redis fails, use PostgreSQL and show freshness timestamps. Never present
delayed live data as current.

## 7. Live behavior and reliability

- SSE is limited to the scoreboard and game pages.
- Clients reconnect and fetch the newest snapshot after disconnects.
- A source delay displays a visible delayed-data state and last-update time.
- Finalization triggers verification, standings rebuild, stat rollups, metric
  computation, and cache warming.
- Public pages keep serving the last successful data when ingestion fails.
- Alerts fire after two consecutive job failures or more than two minutes of
  live-data delay during a game.
- Final scores and key totals are checked against the source and surfaced in
  the admin quality report.

## 8. Page inventory

### Public pages

1. **Home:** favorite-team block, hero highlight, live/final/upcoming games,
   leaders, and standings summary.
2. **Scoreboard:** date navigation, game cards, statuses, highlight indicators,
   and live updates.
3. **Game page:** header, highlight embed, line score, comparisons, box scores,
   play-by-play, win-probability chart, shot chart, and live state.
4. **Highlights:** matched videos with team/date/channel filters.
5. **Schedule:** season calendar/list, date/team filters, and localized times.
6. **Standings:** East/West and division views with freshness timestamp.
7. **Stats leaders:** category and minimum-games filters.
8. **Players directory:** search, team/position/status filters, sorting.
9. **Player profile:** bio, season/career stats, game log, shots, metrics,
   percentiles, splits, and related highlights.
10. **Player comparison:** two to four players, shared season, metrics, and URL.
11. **Teams directory:** conference/division grouping and records.
12. **Team page:** record, roster, stats, schedule, leaders, and highlights.
13. **Search results:** grouped players, teams, and games; favorite team first.
14. **Methodology:** formulas, inputs, versions, and plain-language definitions.
15. **About/data sources:** attribution, source terms, freshness, contact, and
    unofficial disclaimer.
16. **Legal:** terms, privacy, video/rights notice, and cookie notice if needed.
17. **Errors:** 404, 500, maintenance, and no-data states.

### Account pages

- Sign up, log in, log out, password reset, and account settings.
- Favorite-team selection and deletion of account data.
- My Team view using the normal team page data, with personalized ordering.

### Admin pages

- Dashboard with review, ingestion, verification, and missing-data counts.
- Video review queue and manual match tool.
- Channel management.
- Ingestion monitor with rerun and per-game refresh actions.
- Data verification/quality report.
- User administration without password access.
- Settings for current season, intervals, and feature flags.

## 9. Accounts and personalization

Use Argon2 or bcrypt, secure HTTP-only session cookies, CSRF protection,
rate limits on login/reset/search, and role checks for all admin actions.

For users with a favorite team:

- Home shows that team's live game, latest result, next game, and highlights
  first.
- Scoreboard lists its game first.
- Schedule preselects it.
- Standings highlights it.
- Highlights and search prioritize it.
- Navigation exposes a stable shortcut to its team page.

Users without a favorite team receive the normal global ordering.

## 10. Security, privacy, compliance, and observability

- Keep all secrets in environment variables.
- Store only email, password hash, display name, favorite team, account status,
  and required timestamps.
- Support account deletion and remove associated user data.
- Record data-source attribution and takedown requests.
- Embed videos only; do not download or re-host them.
- Publish the unofficial NBA disclaimer and applicable legal pages.
- Use structured web and worker logs, error tracking, uptime monitoring, and
  alerts described in the live reliability section.

## 11. Milestones

### Milestone 0: Foundation (3–5 days)

Repository, Docker Compose, PostgreSQL, migrations, configuration, logging,
worker/web skeleton, and endpoint fixtures for schedule, final box score,
play-by-play, live scoreboard, and standings.

### Milestone 1: Core data and past games (1–2 weeks)

Reference/schedule sync, final-game ingestion, line scores, box scores,
verification, idempotent upserts, and initial Scoreboard, Game, Standings, and
Teams pages.

### Milestone 2: Live pipeline (1–2 weeks)

Schedule-driven polling, changed-only live writes, SSE, live scoreboard/game
pages, live standings behavior, and cache layers.

### Milestone 3: Players, teams, and metrics (1–2 weeks)

Season/career rollups, team stats, advanced metrics, methodology page, player
directory/profile/comparison, leaders, and complete team page.

### Milestone 4: Highlights (1 week)

RSS discovery, API metadata, confidence matching, embeds, home/feed/game
surfaces, review queue, manual matching, and channel management.

### Milestone 5: Accounts and personalization (1–2 weeks)

Authentication, reset flow, settings, favorite team behavior, user
administration, and authentication tests/security review.

### Milestone 6: Search, admin, and hardening (1 week)

Search index and fuzzy matching, remaining admin tools, quality reports,
performance tuning, cache testing, and critical-path coverage.

### Milestone 7: Production launch (1 week)

Legal and attribution review, paid deployment, managed services, HTTPS/domain,
monitoring, browser QA, and one full game-day verification.

Estimated desktop v1 effort: approximately 9–12 focused weeks. Mobile follows
as a separate responsive-design and performance phase.

## 12. Definition of done

- All public, account, and admin pages are deployed with real current-season data.
- Live scoreboard and game pages operate through one full game day.
- Every final game has a verified score and either a matched highlight or a
  recorded reason for no match.
- Ingestion failures do not break public pages and are visible to admins.
- Every advanced metric is tested and documented with a formula version.
- Authentication, authorization, CSRF, rate limits, reset, and deletion flows
  are tested.
- Legal pages, attribution, rights notice, and unofficial disclaimer are live.
- The application runs on paid production services with HTTPS, monitoring,
  backups, and documented environment configuration.
- Desktop QA passes on the latest two versions of Chrome, Safari, and Firefox.

## 13. Risks and mitigations

| Risk | Mitigation |
|---|---|
| NBA endpoint changes or blocking | Adapter layer, fixtures, retries, alerts, and timestamped fallback data |
| Source rate limits | Shared fetches, game-window polling, caching, backoff, and changed-only writes |
| Delayed live source data | Explicit delayed state, timestamps, retry/backoff, and alerting |
| Incorrect standings or metrics | Compare with source, formula tests, versioned methodology, and quality flags |
| Inconsistent highlight titles | Time/team matching, confidence thresholds, and manual review |
| Video takedowns or rights issues | Embedding only, attribution, takedown process, and no unlicensed hosting |
| Account/security incident | Strong hashing, secure cookies, CSRF, rate limits, minimal data, and deletion |
| Stale personalized cache | Shared per-team caches, short TTLs, versioned keys, and response-time ordering |
| Scope creep | Milestone gates and a post-launch backlog for excluded features |

## 14. Decisions to confirm before implementation

1. Final product name and domain.
2. Whether launch contains only the current season or selected historical seasons.
3. Initial YouTube channels and whether team channels are included.
4. Exact advanced metrics for the first release; prefer transparent metrics over
   an unexplained composite rating.
5. Email provider for password reset.
6. Production hosting provider and managed PostgreSQL/Redis sizing.

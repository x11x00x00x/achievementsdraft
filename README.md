---
name: Achievements and Gamerscore System
overview: "Design for a 50-achievement gamerscore system using existing Insignia Stats data: play time, leaderboards, events, logins, and site activity. Defines scoring model, unlock rules, and data sources for each achievement."
todos: []
isProject: false
---

# Achievements and Gamerscore System Design

## Data sources (from codebase)


| Source                  | Table / API                                          | Per-user?                   | Key fields                                            |
| ----------------------- | ---------------------------------------------------- | --------------------------- | ----------------------------------------------------- |
| **Play time**           | `play_time_tracking`                                 | Yes (username)              | `total_minutes`, `by_game_json` (game name → minutes) |
| **Leaderboards**        | `leaderboard_entries`, `leaderboards`, `games`       | By `player_name` (gamertag) | `rank`, `player_name`, `leaderboard_id`, `game_id`    |
| **Leaderboard history** | `leaderboard_changes`                                | By `player_name`            | `change_type`, `new_rank`, `previous_rank`, `game_id` |
| **Events**              | `events`, `event_registrations`                      | Yes (username)              | `created_by`, `event_id`, `username`, `status`        |
| **Event payouts**       | `event_payouts`                                      | Yes (username)              | Winner per event                                      |
| **Logins**              | `login_log`                                          | Yes (username)              | `username`, `logged_in_at`                            |
| **Site roles**          | `admin_users`, `game_managers`, `game_manager_games` | Yes (username)              | Games managed, admin                                  |
| **Games catalog**       | `games`                                              | No                          | Total games on Insignia (for “every game” type)       |


**Important:** Play time is keyed by **username** (Insignia Stats account). Leaderboard data is keyed by **player_name** (gamertag). To award leaderboard-based achievements to a logged-in user, the site needs an optional **linked gamertag** (e.g. in user profile or `play_time_tracking`). Achievements that use leaderboards are defined below as “per gamertag”; implementation can either require linking or show them only after link.

---

## How the scoring will work

- **Gamerscore:** Each achievement has a fixed point value (e.g. 5–100 points). Total gamerscore = sum of points for all unlocked achievements. No partial or decay; once unlocked, points are permanent.
- **Rarity:** Optionally store unlock count per achievement (e.g. “12% of users have this”) for display; not required for scoring.
- **Unlock rule:** Each achievement has a single, deterministic condition. When the condition is first satisfied (and recorded in DB), the achievement is unlocked and points are added. No “level” or XP; only binary unlock + gamerscore.

---

## How achievements will work

1. **Storage:** New table e.g. `user_achievements (username, achievement_id, unlocked_at)` and `achievements (id, key, name, description, points, data_source)`.
2. **Evaluation:**
  - **On-demand:** When viewing profile/dashboard, compute which achievement conditions are met from current data (play time, leaderboards, events, etc.), then insert into `user_achievements` only for newly satisfied achievements (idempotent).  
  - **Background (optional):** Cron or job that runs periodically, recomputes conditions for active users, and inserts new unlocks.  
  - **Real-time (optional):** After login, after play-time sync, or after leaderboard sync, run evaluator for affected users/gamertags.
3. **Linking:** For leaderboard-based achievements, if the site has a “linked gamertag” per user, use it to resolve `player_name` → `username` and write unlocks to `user_achievements` by username. If no link, leaderboard achievements can still be computed per gamertag and shown on a “player” page or after link.

---

## 50 achievements (with points and data source)

**Play time (by user – `play_time_tracking`)**


| #   | Name                   | Description                              | Points | Condition (data source)                                                                                              |
| --- | ---------------------- | ---------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------- |
| 1   | **Insignia Tourist**   | Play at least 1 game on Insignia         | 5      | `Object.keys(by_game_json).length >= 1` (count games with > 0 min)                                                   |
| 2   | **Variety Hour**       | Play 5 different games                   | 10     | Distinct games in `by_game_json` >= 5                                                                                |
| 3   | **Renaissance Player** | Play 10 different games                  | 15     | Distinct games >= 10                                                                                                 |
| 4   | **Completionist**      | Play every game on Insignia              | 100    | Distinct games in `by_game_json` >= total games in `games` table (with optional minimum minutes per game, e.g. 1)    |
| 5   | **First Steps**        | Log 1 hour total play time               | 5      | `total_minutes >= 60`                                                                                                |
| 6   | **Dedicated**          | Log 10 hours total play time             | 15     | `total_minutes >= 600`                                                                                               |
| 7   | **Veteran**            | Log 50 hours total play time             | 25     | `total_minutes >= 3000`                                                                                              |
| 8   | **Marathon**           | Log 100 hours total play time            | 50     | `total_minutes >= 6000`                                                                                              |
| 9   | **One Game Wonder**    | Spend 10+ hours in a single game         | 20     | `max(Object.values(by_game_json)) >= 600`                                                                            |
| 10  | **Mainstay**           | Spend 50+ hours in a single game         | 35     | `max(Object.values(by_game_json)) >= 3000`                                                                           |
| 11  | **Night Owl**          | Be online (tracked) in 10 different days | 10     | Count distinct days from `last_check_ts` / sync history; requires day-level log or infer from `updated_at` over time |
| 12  | **Week Warrior**       | Be online on 7 different days            | 10     | Same as above, 7 days                                                                                                |


**Leaderboards (by `player_name` / linked gamertag)**


| #   | Name                   | Description                                         | Points | Condition                                                                                                 |
| --- | ---------------------- | --------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------- |
| 13  | **On the Board**       | Appear on any leaderboard                           | 10     | At least one row in `leaderboard_entries` for this `player_name`                                          |
| 14  | **Top 100**            | Reach top 100 on any leaderboard                    | 15     | `MIN(rank) <= 100` in `leaderboard_entries` for player                                                    |
| 15  | **Top 10**             | Reach top 10 on any leaderboard                     | 25     | `MIN(rank) <= 10`                                                                                         |
| 16  | **Number One**         | Get rank 1 on any leaderboard                       | 50     | Any `rank === 1` in `leaderboard_entries`                                                                 |
| 17  | **Multi-Board**        | Appear on 5 different leaderboards                  | 15     | Count distinct `leaderboard_id` for player >= 5                                                           |
| 18  | **Multi-Board Master** | Appear on 20 different leaderboards                 | 30     | Distinct leaderboards >= 20                                                                               |
| 19  | **Multi-Game**         | Appear on a leaderboard in 3 different games        | 20     | Count distinct `game_id` (via leaderboards) >= 3                                                          |
| 20  | **All-Rounder**        | Appear on a leaderboard in 10 different games       | 40     | Distinct games >= 10                                                                                      |
| 21  | **Champion**           | Hold rank 1 in 3 different leaderboards             | 75     | Count leaderboards where player has rank 1 >= 3                                                           |
| 22  | **Improver**           | Improve your rank on any leaderboard (from changes) | 15     | At least one `leaderboard_changes` row for player with `new_rank < previous_rank` (or equivalent)         |
| 23  | **Dethroned**          | Have held rank 1 and been overtaken (from changes)  | 10     | `leaderboard_changes` with `previous_rank === 1` and `change_type` / new_rank indicating lost first place |


**Events (by username)**


| #   | Name                 | Description                     | Points | Condition                                                                              |
| --- | -------------------- | ------------------------------- | ------ | -------------------------------------------------------------------------------------- |
| 24  | **Event Goer**       | Register for 1 event            | 10     | At least one `event_registrations` row (any status or `status = 'paid'` as you prefer) |
| 25  | **Regular**          | Register for 5 events           | 20     | Count `event_registrations` >= 5                                                       |
| 26  | **Community Pillar** | Register for 20 events          | 40     | Count >= 20                                                                            |
| 27  | **Host**             | Create 1 event                  | 15     | At least one `events` row with `created_by = username`                                 |
| 28  | **Organizer**        | Create 5 events                 | 35     | Count events created >= 5                                                              |
| 29  | **Winner**           | Win an event (receive a payout) | 50     | At least one `event_payouts` row for username                                          |
| 30  | **Paid Competitor**  | Register for a paid event       | 15     | At least one registration where event has `is_paid_event = 1` (join `events`)          |


**Login / site activity (by username)**


| #   | Name              | Description                                         | Points | Condition                                                     |
| --- | ----------------- | --------------------------------------------------- | ------ | ------------------------------------------------------------- |
| 31  | **First Login**   | Log in to Insignia Stats once                       | 5      | At least one `login_log` row for username                     |
| 32  | **Returning**     | Log in 10 times                                     | 10     | Count `login_log` for username >= 10                          |
| 33  | **Loyal**         | Log in 50 times                                     | 20     | Count >= 50                                                   |
| 34  | **Early Adopter** | Have an account that logged in before a cutoff date | 25     | `MIN(logged_in_at) < cutoff` (e.g. first 6 months of feature) |


**Roles / management (by username)**


| #   | Name                   | Description                        | Points | Condition                                                                |
| --- | ---------------------- | ---------------------------------- | ------ | ------------------------------------------------------------------------ |
| 35  | **Game Manager**       | Be a manager for at least one game | 20     | At least one row in `game_managers` or `game_manager_games` for username |
| 36  | **Multi-Game Manager** | Manage 5+ games                    | 35     | Count games managed >= 5                                                 |
| 37  | **Admin**              | Be a site admin                    | 30     | Present in `admin_users`                                                 |


**Combination / special**


| #   | Name                       | Description                                                       | Points   | Condition                                                                                 |
| --- | -------------------------- | ----------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------- |
| 38  | **Play + Compete**         | Have play time and at least one leaderboard entry (same identity) | 25       | Play time user with linked gamertag that has leaderboard_entries; or heuristic if no link |
| 39  | **Event Regular + Player** | Register for 3+ events and 10+ hours play time                    | 30       | Events count >= 3 and `total_minutes >= 600`                                              |
| 40  | **Triple Threat**          | Play 5+ games, appear on a leaderboard, and register for an event | 40       | Play 5 games + leaderboard presence + 1 event (requires link for leaderboard)             |
| 41  | **Century**                | Reach 100 total gamerscore                                        | 0 (meta) | Sum of unlocked achievement points >= 100; display-only or special badge                  |
| 42  | **Half Thousand**          | Reach 500 total gamerscore                                        | 0 (meta) | Total points >= 500                                                                       |
| 43  | **Thousand Club**          | Reach 1000 total gamerscore                                       | 0 (meta) | Total points >= 1000                                                                      |


**Play time – game-specific (optional; by user)**


| #   | Name               | Description                  | Points | Condition                                                    |
| --- | ------------------ | ---------------------------- | ------ | ------------------------------------------------------------ |
| 44  | **Forza Fan**      | 5+ hours in Forza Motorsport | 15     | `by_game_json['Forza Motorsport'] >= 300` (or match by name) |
| 45  | **Halo 2 Regular** | 5+ hours in Halo 2           | 15     | Same for Halo 2 (exact name from `games` / Insignia)         |
| 46  | **PSO Explorer**   | 5+ hours in PSO              | 15     | Same for PSO title                                           |


**Community / aggregate (optional; not per-user gamerscore)**


| #   | Name               | Description                                             | Points | Condition                                                                                                                                                                                                  |
| --- | ------------------ | ------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 47  | **Peak Hour**      | Be online (tracked) when Insignia had 500+ users online | 20     | At time of user’s session, `insignia_online_snapshots` or `insignia_daily_online_minmax` had value >= 500; requires storing “user was online at timestamp” or comparing play_time session to snapshot time |
| 48  | **Friend Tracker** | Have 5+ friends with tracked play time                  | 10     | Count rows in `play_time_friends` for `owner_username` >= 5                                                                                                                                                |
| 49  | **Wallet Linked**  | Link a Bitcoin wallet for paid events                   | 10     | Row in `user_bitcoin_wallets` for username                                                                                                                                                                 |
| 50  | **Paid Winner**    | Win a paid event (payout from paid event)               | 60     | `event_payouts` for user where event has `is_paid_event = 1`                                                                                                                                               |


---

## Implementation notes

- **“Play every game” (Completionist):** Compare count of games in `by_game_json` (with optional minimum minutes, e.g. 1) to `SELECT COUNT(*) FROM games`. Game names in `by_game_json` come from Insignia profile; normalize (trim, case) and match to `games.name` (or maintain a mapping) so variants don’t break the count.
- **Leaderboard ↔ user link:** Add optional `linked_gamertag` (or similar) to user profile; evaluator uses it to attribute leaderboard achievements to a username and write to `user_achievements`.
- **Idempotency:** When evaluating, only insert into `user_achievements` if no row exists for (username, achievement_id); use DB unique constraint.
- **Rarity:** Optional table `achievement_unlock_counts (achievement_id, unlock_count)` updated when new unlocks are written; or count `user_achievements` per achievement_id for “X% of users” display.

This gives 50 achievements, a clear scoring model (fixed points per achievement, total = sum), and a concrete unlock flow using only existing datapoints and one optional link (gamertag) for leaderboard achievements.

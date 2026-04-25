# Haines Harvest — Session Summary

## 2026-04-25 — Wired the PWA to live Supabase data

### What changed
Replaced the hardcoded `recipes`, `weekData` (4-week rotation), `heroSubs`, and `shoppingData` blocks in `index.html` with live reads from Supabase.

- **Recipes** → `hh_recipes`. A small tag→metadata map (`TAG_META` / `TAG_PRIORITY`) derives emoji, category, color, and tool from `tags`. `hh_recipes.notes` is used for the description. The per-recipe `kids:` line and per-recipe `veggie:` side were dropped (the side dish now comes from the day's `hh_meal_rotation_template.side_dish` or `hh_meal_plan.notes`).
- **Today / This Week** → `hh_meal_plan` for the current week, falling back to `hh_meal_rotation_template` when no plan row exists for a date. The Week 1–4 selector on "This Week" was removed; the page now shows the current ISO Mon–Sun.
- **4-Week Plan view** → still shows 4 weeks, fed entirely from `hh_meal_rotation_template`.
- **Grocery List** → derived from the next 7 days of meal-plan rows (today through today+6), joined to `hh_recipes`, ingredients flattened and grouped by `store` field (`costco / aldi / festival_foods / any / leftover`). v1 lists every line, no quantity summing across recipes. Week 1–4 selector was removed.
- **Recipe 19 (Balsamic Flank Steak)** marinade fix: `1.33 cup` → `1/3 cup` for balsamic vinegar, soy sauce, dijon mustard. Garlic and other ingredients left as-is.

### Plan-row date convention (load-bearing)
`hh_meal_plan.week_of` is treated as an anchor date; each row's actual date = the first day on/after `week_of` whose name matches `day_of_week`. Example: `week_of=2026-04-25, day_of_week=Monday` → 2026-04-27. This is what `planRowDate(row)` implements. If the convention changes (e.g., `week_of` becomes "the Monday of the planned week"), update that one helper.

### Supabase config
Embedded directly in `index.html`:
- URL: `https://optzbdbavpnxstpxrpbh.supabase.co`
- Publishable key: `sb_publishable_...` (apikey-only header, no Authorization)
- Existing RLS on `hh_recipes`, `hh_meal_plan`, `hh_meal_rotation_template` is `ALL` for `anon`+`authenticated` (broader than the food_log SELECT-only pattern, but Cam chose to leave it).

### Status
- Live data wired end-to-end; smoke-tested via JSC against real Supabase responses.
- File serves locally without errors; hasn't been opened in a real browser yet — verify on iPhone Home Screen PWA after deploy.
- 24 recipes, 8 plan rows (week_of=2026-04-25 covers Apr 25–May 1; week_of=2026-05-02 covers May 2), 28 template rows.

### Next / open
- Push to `main` and verify on GitHub Pages + iPhone PWA. No deploy has happened yet.
- "This Week" page uses ISO Mon–Sun. Today (Sat Apr 25) shows 5 empty days because plan and template both start Apr 27. Self-corrects after Apr 27, but if you want immediate continuity flip `renderCalendar` to use `today + 6` like the grocery list does.
- Smash Burgers emoji shows 🍗 because `air_fryer` wins in `TAG_PRIORITY` over `kid_favorite`. Tweak the priority list or add a `burger` tag if it bothers you.
- Recipe 19 still has `servings=11` in `hh_recipes`. The marinade now reads sensibly (1/3 cup), but the recipe modal will say "serves 11". Consider normalizing servings to 6 family-wide later.
- Steward agent (rotation auto-population, pantry subtraction, unit math, skip-meal carry-over against the new DB layer) is explicitly deferred.

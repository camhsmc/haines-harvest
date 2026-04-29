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
- Smash Burgers emoji shows 🍗 because `air_fryer` wins in `TAG_PRIORITY` over `kid_favorite`. Tweak the priority list or add a `burger` tag if it bothers you.
- Recipe 19 still has `servings=11` in `hh_recipes`. The marinade now reads sensibly (1/3 cup), but the recipe modal will say "serves 11". Consider normalizing servings to 6 family-wide later.
- Steward agent (rotation auto-population, pantry subtraction, unit math, skip-meal carry-over against the new DB layer) is explicitly deferred.

---

## 2026-04-25 (later same day) — Grocery aggregation, step fix, rolling 7-day view

### What changed
- **Grocery list aggregation.** Raw ingredient lines now group by canonical name and sum within volume/weight classes. A `NAME_ALIASES` table merges variants (`frozen garlic cubes or fresh minced cloves` / `frozen garlic cube or minced clove` / `garlic` → `garlic`; same pattern for shredded cheese, green onions, lemon, cucumber, arugula). Per-meal annotations dropped — list shows totals only. 78 raw → 62 grouped on the current week.
- **Pack-size rounding.** New `PACK_SIZES` table maps known ingredients + units to typical containers (e.g. `sour cream` cup → `16 oz tub`, `garlic` clove → `1 head` per 10, `olive oil` cup → `25 oz bottle` per 3). `findPackSize` ceils the summed total to the next whole pack. Unknown ingredients fall through to summed amount in their natural display unit. Search `PACK_SIZES` in `index.html` to extend.
- **Unit normalization.** `UNIT_SINGULAR` map collapses plurals (`cloves`→`clove`, `cans`→`can`, `lbs`→`lb`, etc.) so the same ingredient written either way aggregates. `findPackSize` also normalizes the rule's unit at lookup time, so future pack rules can be written either way.
- **Recipe modal step fix.** `hh_recipes.steps` is stored as `[{order, instruction}, …]` JSONB objects, not strings. Modal was rendering `[object Object]`. Adapter now unwraps `.instruction` (with `.text` and plain-string fallbacks).
- **Rolling 7-day view.** "This Week" replaced with "Next 7 Days" — today through today+6, matching the grocery range. Theme label (Pasta Night / Taco Night / etc.) removed from day cards since the rotation isn't being followed strictly.
- **Data fix.** Recipe 22 (Grilled Veggie Rainbow Bowl) had `unit: "cups dry"` on jasmine rice; corrected to `cups`.

### Status
- All changes deployed via GitHub Pages (`main` is `6981474`). Cam confirmed step rendering and 7-day view in the live PWA.
- Pack-size lookup covers ~30 staples; less-common ingredients (`prosciutto`, `toasted walnuts`, `blue cheese crumbles`, `crumbled feta`, `shredded lettuce`, `zucchini`, `sweet potato`, `avocado`) still show raw summed amounts. Cam's plan: extend `PACK_SIZES` opportunistically when something looks weird in the live list.

### Next / open
- (Carryover) Smash Burgers emoji shows 🍗.
- (Carryover) Recipe 19 servings still 11; consider normalizing.
- (Carryover) Steward agent deferred.
- Pack-size table will need iterative refinement as new recipes / weeks come into the plan. Each rule is one line in `PACK_SIZES`.

---

## 2026-04-28 — Cerebro polish pass: emoji, marinade, meal_date, sides, aisles, grams, theme drop, skip-push

Cam came back after the trip. Pulled latest Cerebro asks via Notes 169 (audit), 170 (future improvements), 173 (breakfast/lunch — deferred). Carryovers in this CLAUDE.md were stale; Cerebro had the real list.

### What changed
- **Smash Burgers emoji** — added `burger` tag to recipe 15 + a `TAG_META` entry that wins over `air_fryer` in `TAG_PRIORITY`. Now displays 🍔. (`a72337f`)
- **Recipe 19 marinade right-sized** for the 2.75 lb flank: dijon `1/3 cup → 2 tbsp`, honey `3.5 tbsp → 1 tbsp`, garlic `11 → 3 cloves`. Balsamic + soy stay at `1/3 cup`. The marinade only touches the Saturday bulk cook; cooked steak feeds Mon and Wed without re-marinating, so amounts scale with meat weight, not meal count. (`a72337f`)
- **`meal_date` generated column on `hh_meal_plan`** — `STORED` from `week_of + day_of_week`, plus an index. JS `planRowDate(row)` prefers `meal_date` if present and falls back to the legacy compute. The May 2 Smash Burgers row was moved back to its real `week_of=2026-05-02` (the prior workaround had collapsed it onto Apr 25, masking the date-window bug). (`85d27af`)
- **Stale "15 dinners" copy** — `renderRecipes` now sets the page subhead from `RECIPE_LIST.length`. (`85d27af`)
- **`side` column on `hh_meal_plan`** — `buildMeal` now reads `row.side || row.side_dish || ''` and no longer falls back to `row.notes` (which holds prep context, not sides). Backfilled per Cam's heuristic ("skip the side when the main already has carb + veg/fruit built in"): only Apr 25 Balsamic Flank Steak got "Corn on the cob + green beans"; all 7 other dinners qualified for no-side. (`4da389c`)
- **Aisle grouping inside each store on the grocery list** — new `AISLES_ORDER` + `AISLE_RULES` regex table. Each store card now renders Produce / Meat & Seafood / Dairy & Eggs / Bakery & Bread / Frozen / Canned & Jarred / Pantry & Dry Goods / Condiments, Sauces & Oils / Spices / Other. First-match-wins; reserved-word ordering matters (e.g. "broth" Pantry rule sits before "chicken" Meat rule). (`5df91c5`)
- **Grams in parentheses for sauce/marinade ingredients** — `GRAMS_PER_UNIT` lookup; `formatIngredient` appends `(80g)` after the existing line when the ingredient name and unit (cup/tbsp/tsp/clove) match a rule. Then expanded to: universal `oz→g` (28.35) and `lb→g` (454) fallback for any ingredient by weight; rules for grains (jasmine/white/brown/day-old rice, panko, pasta family), nuts (walnuts, almonds, pecans, pine nuts, cashews), cheeses (parmesan, shredded mozzarella/cheese/cheddar, feta, blue cheese, ricotta, cottage), and fresh herbs (basil, cilantro, parsley, thyme, oregano, mint, dill, chives, scallions/green onions). Skipped intentionally: produce by volume (cherry tomatoes, cucumber, brussels sprouts) and count-based items (zucchini, onion, avocado) per Cam's "loose by nature, no scale needed" rule. (`003c07b`, `863c58f`)
- **Theme labels removed everywhere** — Cam isn't following Mon=Pasta / Tue=Taco / etc. strictly anymore (Apr 27 pita meal showed "Pasta Night" because Monday). Dropped from Today and the 7-Day calendar; deleted the `themes` const, `.today-theme` / `.theme-label` CSS, and refreshed the 4-Week Rotation page copy. (`b597799`)
- **Skip pushes meal to next open slot** — tap "Skip This Meal" now PATCHes the plan row's `(week_of, day_of_week)` to the first calendar date with no other plan row (90-day search window). Cached metadata in `localStorage.hh_skips[<date>]`: `movedRowId`, `movedTo`, `movedRecipeId`, `movedRecipeTitle`, `originalWeekOf`, `originalDayOfWeek`. The Today card stays on the original date showing the moved recipe grayed out with `↳ Moved to Sun May 3`. Undo Skip restores the row via PATCH back to the cached original. Falls back gracefully if the day has no DB row (template-fallback): just marks the day skipped locally. (`d420ff6`)

### Forward-compat notes (Cam asked explicitly)
- Auto-applies to all future rows: `meal_date`, `side` field, recipe display from live data, recipe-count subhead, skip-push (any future plan row), Smash Burgers via the `burger` tag.
- Lookups (auto-apply if name matches an existing pattern, otherwise fall through):
  - `AISLE_RULES` — new ingredient class → "Other".
  - `PACK_SIZES` — no rule → raw summed amount.
  - `GRAMS_PER_UNIT` — no rule → no `(Xg)`. Universal `oz/lb→g` always works.
  - `NAME_ALIASES` — variant spelling not matched → treated as a separate ingredient.
  - `TAG_META` / `TAG_PRIORITY` — new tag → falls to `🍽️ Dinner`.

### Status
- All changes deployed (`main` is `863c58f`). Cam reviewed live on phone and confirmed flank steak modal grams + the rainbow bowl modal grams.
- Recipe 23 still has `2.25 cloves` garlic (decimal cloves from earlier scaling); not addressed this session.

### Next / open
- (Carryover) Recipe 19 `servings=11` — modal still says "serves 11"; consider normalizing.
- (Carryover) Steward agent deferred.
- (New, from Cerebro Note 173) Breakfast and lunch in the grocery list — Cam wants to think it through first before building.
- (New, from Cerebro Note 170) Sauces in grams as primary unit (full migration vs current parenthetical) — the parenthetical approach may be sufficient; revisit if Cam still wants the data-side migration.
- Recipe 23's `2.25 cloves` garlic decimal artifact.
- Pack-size, aisle, and gram lookups will keep needing one-line additions when new ingredient classes show up. The prosciutto/toasted-walnut/blue-cheese-crumbles fall-through items from last session are mostly still uncovered.

---

## 2026-04-29 — Steward agent + Sat–Fri shopping week + servings sweep

Big build day. Three bundles in two commits.

### Bundle 1 (`7b916fe`) — Sat–Fri grocery week, Serves chip, shop-day badge
- **Servings normalized to 6** in `hh_recipes` for all 24 recipes (10 needed updating). The CLAUDE.md "Recipe 19 says Serves 11" note was stale — `servings` was never read by the JS until this session. Adapter now passes `r.servings` to the recipe modal, which renders a `🍽 Serves 6` chip alongside time/tool/Family Fave.
- **Grocery week toggle** — replaced rolling 7-day with **Sat–Fri shop weeks** anchored to Cam's Saturday shop trip. New state `groceryWeekOffset` + `startOfShopWeek(date)` helper. ◀/▶ buttons, "This week" reset, label like `Apr 25 – May 1`. Per-week persistence in `localStorage.hh_grocery_checks` keyed by Monday's date (now Saturday's date in the new model). Hidden bug fixed in passing: under the old ISO Mon–Sun anchoring, Saturday meals were dropping out of the week's list until Sunday.
- **Saturday "🛒 Grocery Shopping Day" badge** on the Next 7 Days view's Saturday card.

### Bundle 2 (this commit) — Steward agent (Layer 1 + 2)
**Vision (per Cam):** the Steward auto-plans a 4-week-forward dinner schedule from `hh_recipes`. Page-load JS, idempotent. Never touches the current shop week (already shopped). New recipes added via Claude sessions auto-slot into the next 2 weeks. Skipping a meal cascades into the following week.

**Schema additions to `hh_meal_plan`:**
- `carryover boolean default false` — meals pushed forward by skip
- `edited_by_user boolean default false` — protects against bumping by `scheduleNewRecipe`

**Steward function (`runSteward`, page-load, idempotent):**
- Window: next shop week's Saturday through 4 shop weeks later (28 days)
- Active recipes only, **excludes tags `sauce`, `condiment`, `dressing`, `salad_topper`, `bulk_protein_topper`** (caught when steward initially scheduled vinaigrette/romesco/pickled-onions as standalone dinners). Pool = 21 of 24 active recipes.
- Picks oldest-last-seen first, **7-day cooldown** so the same recipe doesn't repeat too soon (looks at all of `PLAN_ROWS`, not just window)
- Inserts with `edited_by_user=false`, `carryover=false`

**4-Week Plan view rewritten** to read from `hh_meal_plan` (was reading from `hh_meal_rotation_template`, which is now effectively dead). Shows current shop week + next 3, dates in headers, clickable rows open recipe modal, `⏭` icon next to carryover meals.

**Skip flow rewritten** as Postgres RPCs for atomicity:
- `hh_skip_meal(p_skip_date)` — DELETE original, shift week N+1 onwards forward by 1 day (cross-week ripple), INSERT carryover at next Saturday with `carryover=true`, `edited_by_user=true`. The cascade tail's recipe gets dropped from the schedule (effectively replaced when the steward refills the now-empty last day on next page load).
- `hh_unskip_meal(p_skip_date, p_recipe_id)` — reverse cascade. Only works for the most recent skip; older skips' carryovers have shifted forward by subsequent skips.
- Day-of-week ↔ date math is encoded in CASE expressions; `meal_date` generated column auto-recomputes.

**Grocery list filter** — `groceryListByStore` now skips any meal where `meal.carryover === true` so already-bought ingredients don't double-up next week.

**`hh_schedule_new_recipe(p_recipe_id)` RPC** — entry point for future Claude sessions. After `INSERT INTO hh_recipes`, this finds the earliest steward-placed slot (`edited_by_user=false AND carryover=false`) in the next 2 shop weeks and swaps the recipe in, marking `edited_by_user=true`. If all next-2-week slots are user-protected, returns `queued=true` and the steward picks it up via rotation later.

### Forward-compat notes
- The `hh_meal_rotation_template` table is no longer the source of truth. `mealForDate` still falls back to it for dates outside the steward's 28-day window (rare; happens if Cam doesn't open the app for a long time). Could be deleted; left in place as a safety net.
- Bootstrap May 2 (Smash Burgers) and May 3 (Taco Bar) plan rows are `edited_by_user=false`. A future `scheduleNewRecipe` call could displace them. Cam confirmed he doesn't care; if this becomes a problem we'll add a "lock this row" UI.

### Verification — 14 + 12 assertions PASS via headless-Chrome QA
- Steward: 28-row window fill, 7-day cooldown respected across plan, no duplicate dates, current week untouched
- Cascade: Apr 29 skip → May 2 carryover, cross-week shift to May 3..May 29, recipe at May 29 displaced, idempotent re-run, no carryover ingredients in next-week grocery list, ⏭ rendered in 4-Week view
- Pool filter: sauces/condiments/dressings excluded from steward picks; recipe 18 (chopped salad) correctly retained as a main

### Status
- All changes deployed to GitHub Pages (commit on `main`)
- Real-world skip cascade not yet exercised on phone

### Next / open
- (v1.1) **Vacation mode** — `hh_vacations(start_date, end_date)` table; plan rows in range get deleted, grocery list skips those dates, shop-day badge hides, steward respects ranges. Was queued during this session (Cam asked about long family vacations).
- (Carryover) Recipe 19 `servings` reframing — modal now correctly says "Serves 6" but the bulk-cook semantics are masked. Could add `bulk_cook` flag if it becomes confusing.
- (Carryover) Cerebro Note 173 (breakfast/lunch in grocery list) and Note 170 (sauces in grams primary).
- (Carryover) Pack-size / aisle / gram lookup tables — add one-liners as new ingredients show up.
- Skip undo only works for the most recent skip. If Cam wants multi-skip undo, we'd need a skip log table.

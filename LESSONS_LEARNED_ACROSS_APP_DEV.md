# Lessons Learned Across App Development

Universal lessons that apply to all iOS app projects. Update this file
whenever a significant lesson is learned. Copy to all relevant repos and
push to GitHub when updated.

---

## Xcode as Primary Build Method (Not EAS)

EAS free tier monthly build quota is easily exhausted during iterative
debugging. Use Xcode local archive for all production builds.

- EAS is reserved for exceptional cases only
- Always check `eas build:list --platform ios --limit 5` before triggering
  any EAS build
- Cancel queued EAS builds immediately via expo.dev if plans change

---

## Version & Build Number Workflow

Update BOTH files before every archive. Use Terminal — do not rely on the
Xcode GUI as it may not save correctly.

**Step 1 — Update Info.plist (hardcoded values required):**
```bash
sed -i '' 's/<string>OLD_VERSION<\/string>/<string>NEW_VERSION<\/string>/' \
  ios/AppName/Info.plist
sed -i '' 's|<string>OLD_BUILD</string>|<string>NEW_BUILD</string>|' \
  ios/AppName/Info.plist
```

⚠️ Do NOT use `$(MARKETING_VERSION)` or `$(CURRENT_PROJECT_VERSION)` variables
in Info.plist — this causes CompileAssetCatalogVariant failures at archive time.
Always use hardcoded values.

**Step 2 — Update project.pbxproj:**
```bash
sed -i '' 's/MARKETING_VERSION = OLD;/MARKETING_VERSION = NEW;/g' \
  ios/AppName.xcodeproj/project.pbxproj
sed -i '' 's/CURRENT_PROJECT_VERSION = OLD;/CURRENT_PROJECT_VERSION = NEW;/g' \
  ios/AppName.xcodeproj/project.pbxproj
```

**Step 3 — Update app.json version to match.**

**Step 4 — Verify in Xcode General tab** that Version and Build show correctly.

**Step 5 — Commit all three files before archiving.**

### Build Number Convention
- Build numbers increment globally — never reset between versions
- Check App Store Connect → TestFlight → iOS for full build history

---

## Archive Steps (Every Release)

1. **Product → Clean Build Folder** (Shift + Cmd + K)
2. **Product → Archive**
3. In Organizer → **Distribute App** → select **App Store Connect** → **Distribute**
   - Use App Store Connect (not TestFlight Internal Only) — gives you both
     TestFlight testing AND App Store submission from the same build
4. Install via TestFlight on a physical device and test before submitting for review
5. In App Store Connect, attach build to submission manually — easy to miss

---

## Signing

- Use **manual signing** on the **Release tab** in Signing & Capabilities
- Automatic signing fails for distribution builds
- Always verify signing on the Release tab specifically — Debug tab settings
  do not affect archive builds

---

## Xcode Quirks with Expo/React Native

- `expo-updates` must be disabled — it conflicts with local Xcode builds.
  Purge `EXUpdates` from Pods after disabling.
- Use `DevSettings.reload()` not `Updates.reloadAsync()` in DeviceErrorBoundary
  files — expo-updates is removed.
- Remove any Metro cache-wiping blocks from `metro.config.js` — they break
  production builds in Xcode's sandbox.
- `babel-plugin-module-resolver` is required for `@/` path aliases to resolve
  in Xcode's sandbox. Metro does not read `tsconfig.json` paths.
- Always run `pod install` from the `ios/` directory after adding or removing
  native dependencies.
- `appVersionSource: remote` in eas.json is irrelevant for Xcode builds —
  ignore it.

---

## Splash Screen Fix (expo-splash-screen)

`SplashScreen.hideAsync()` returns `undefined` on the first call in release
builds on physical iOS devices. The splash screen stays frozen until
`hideAsync()` actually succeeds. This is a known `expo-splash-screen` bug.

### Fix — use this pattern in _layout.jsx for every app:
```js
const hideSplash = async () => {
  let attempts = 0;
  while (attempts < 10) {
    const result = await SplashScreen.hideAsync();
    if (result !== undefined) break;
    await new Promise(r => setTimeout(r, 100));
    attempts++;
  }
};

useEffect(() => {
  const timeout = setTimeout(hideSplash, 1500);
  return () => clearTimeout(timeout);
}, []);

useEffect(() => {
  if (isReady && disclaimerChecked) {
    hideSplash();
  }
}, [isReady, disclaimerChecked]);
```

- Call `hideSplash()` instead of `SplashScreen.hideAsync()` everywhere
- The 1.5s timeout is a failsafe — normal flow hides splash as soon as
  isReady and disclaimerChecked are both true
- Remove the render gate (`if (!isReady) return null`) — it causes permanent
  blank screens if async init hangs
- Apply this fix proactively to every new app before first App Store submission

---

## Data Isolation Verification Checklist

When fixing any bug related to data isolation (e.g. devices sharing history
or alerts), always verify in this order:

1. **Backend** filters data by device ID (`WHERE device_id = ?` in every
   relevant query)
2. **Frontend** sends device ID on every request (no unguarded API calls)
3. **End-to-end test** with two real physical devices — confirm data does
   not bleed across

Sending `deviceId` on the frontend means nothing if the backend ignores it.
Always verify the backend first.

---

## Two Machine Setup

- **Mac Studio** (username: `cg`) — primary dev machine
- **MacBook Air** (username: `carltongrizzle`) — secondary machine
- Paths may differ between machines — always confirm before running commands
- Not all repos are cloned on both machines — verify with `ls ~/Projects/Apps/`
  before starting work on a machine

---

## Deployment

- Railway was explored and ruled out as a deployment platform — do not suggest it
- Vercel or alternative platforms to be reconsidered when ready to deploy

---

## Grep Coverage

Always search all directories except `node_modules` when looking for code.
The `__create/` directory has caused repeated failures when excluded from
searches.
```bash
grep -rn "search_term" ./src --include="*.js" --include="*.jsx" \
  --include="*.ts" --include="*.tsx"
```

---

## Pre-Submission Checklist (Every App)

Before submitting any app to the App Store for the first time:
- [ ] Apply `hideSplash()` retry fix to `_layout.jsx`
- [ ] Verify backend filters all user data by device ID
- [ ] Test on a physical device via TestFlight (release build)
- [ ] Test cold launch (fresh install, not warm restart)
- [ ] Verify version and build numbers are correct in Xcode General tab
- [ ] Attach build to submission manually in App Store Connect
- [ ] Audit package.json — remove @anythingai/app and expo-updates before building
- [ ] Verify all UI controls are fully functional (non-functional toggles trigger 2.3.1 rejection)

---

## Dependency Audit — Remove Platform-Specific Packages First

When migrating away from a platform (e.g., Anything.ai), the very first step
is auditing `package.json` and removing all platform-specific packages before
doing anything else.

In SFF's case, `@anythingai/app` (v0.1.96) remained as a dependency long after
migration began. It brought in a duplicate `expo-router` which caused a
`ViewManagerAdapter` duplicate registration crash that froze the splash screen
across all production builds. This took months to identify because the root
cause was hidden inside `node_modules`, not in source files.

SCF had the identical dependency (`@anythingai/app: 0.1.96`) and would have
hit the same freeze.

Always run before building any app:
  cat package.json | grep -E "@anythingai|expo-updates|expo-dev-client"
  find node_modules -name "expo-router" -type d -maxdepth 4

- @anythingai/app must be fully removed — not just unused
- expo-updates must be fully removed (not just disabled) — leaving it in
  contaminates the Pods project with EXUpdates
- expo-dev-client as a direct dependency causes Release builds to launch
  as dev client builds, masking real production behavior

---

## Non-Functional Features Trigger App Store Rejection

App Store Guideline 2.3.1 covers misleading apps. If UI controls (e.g., filter
toggles) are visible but have no effect on app behavior, Apple will reject the
submission. All UI controls must be fully wired before submission.

For SCF specifically: the four cosmetics filter toggles (showSyntheticFragrances,
showParabens, showPFAS, showSulfates) must be connected to backend ingredient
matching logic before first submission.

---

## LESSONS_LEARNED Lives in SimplyFacts/dev-notes (GitHub)

This file is the single source of truth. It lives at:
  https://github.com/SimplyFacts/dev-notes

On any machine:
  cd ~/Projects/Apps/dev-notes && git pull   # start of session
  ... make additions (append only, never overwrite) ...
  git add . && git commit -m "lessons: <topic>" && git push

Never store this file only locally. Always push after updating.

---

## Partial Migrations Are Dangerous

When migrating a feature (e.g., from backend to local AsyncStorage), ALL files
touching that feature must be updated in the same pass. A partial migration
leaves the app in an inconsistent state that may not be immediately visible.

SFF example: useScanHistory.js and alertsStore.js were migrated to local
AsyncStorage, but settingsActions.js was not updated — it still calls
/api/scan-history and /api/alerts for clear operations. This means "Clear
History" and "Clear Alerts" silently fail for SFF users on the live app.

Pending fix: Update SFF settingsActions.js to use local AsyncStorage directly:
- handleClearScanHistory: call AsyncStorage.removeItem(HISTORY_KEY) directly
- handleClearAlerts: call useAlertsStore.getState().clearAlerts() directly
No backend calls needed for either operation.

Always audit ALL files touching a migrated feature before shipping.

---

## SCF Pending Work (Pre-Submission)

### Alert Matching Architecture — Needs Refactor
SCF has two independent alert matching systems that both need fixing:

1. matchAlerts() in alertMatching.js — used for AlertsSection banner at top
   of product screen. Problems:
   - Does not use preset variations array from alertPresets.js
   - "Fragrance / Parfum" alert will not match "fragrance" in ingredients text
   - "BHA (Butylated Hydroxyanisole)" alert will not match "BHA" or
     "butylated hydroxyanisole" in ingredients text
   - Same problem for all multi-word or slash-separated preset names
   - CATEGORY_MAPPING is food-focused (dairy, gluten, peanuts) — not cosmetics

2. hasAlert() in ingredientCategoryDetection.js — used for highlighting
   individual ingredients in Full Ingredients List. Same variations problem.
   ALERT_CATEGORY_MAPPING is also food-focused.

### Food Logic That Must Be Replaced With Cosmetics Equivalents
These files contain SFF food logic that was never updated for SCF:
- ingredientCategories.js — 425 lines of food additive categories (E-numbers,
  sweeteners, dairy, gluten, etc.) — replace with cosmetics categories
  (parabens, PFAS, sulfates, synthetic fragrances, endocrine disruptors,
  formaldehyde releasers, sunscreen chemicals, contaminants, animal-derived)
- additiveCategories.js — large file with food E-numbers and sweetener
  detection — evaluate what if anything is needed for cosmetics
- CATEGORY_MAPPING in alertMatching.js — food allergen mapping, replace with
  cosmetics preset category mapping
- ALERT_CATEGORY_MAPPING in ingredientCategories.js — food categories, replace

### Planned Fix Architecture
1. Add resolveAlertVariations(alertName) to alertPresets.js — returns
   variations array for any preset name, falls back to [alertName] for
   custom alerts
2. Pass detectedIngredients into matchAlerts() — use pre-computed detection
   results for the 6 auto-detected categories (syntheticFragrances, parabens,
   pfas, sulfates, artificialColors, artificialIngredients)
3. Update hasAlert() in ingredientCategoryDetection.js to also use
   resolveAlertVariations()
4. Replace ingredientCategories.js food data with cosmetics-relevant categories
5. Evaluate and slim down or replace additiveCategories.js

### Other Pending SCF Items
- All Clear badge: show when no ingredient categories detected, with explainer
  that appears on first view and on tap
- ArtificialIngredientsSection: confirm return null when empty (consistent
  with other sections)
- App Store copy audit — Guidelines 2.3.1 and 1.4.1
- Version/build numbers set correctly in Xcode before first archive
- Xcode archive and TestFlight build
- TestFlight test on physical device
- SFF settingsActions.js still hits backend for clear operations — fix before
  next SFF release

---

## Forking Is Fast But Carries Hidden Debt

Forking an existing app to create a related app is tempting because it provides
a head start on shared infrastructure. But every inherited file needs to be
explicitly audited against the new app's domain — not assumed relevant just
because it compiled without errors.

SCF was forked from SFF. The cost of that fork:
- Inherited Anything.ai scaffolding caused the same splash freeze that took
  months to debug in SFF — same root cause, same fix needed
- Food-specific logic (additiveCategories.js 680 lines, ingredientCategories.js
  425 lines, alertMatching.js CATEGORY_MAPPING) was never updated for cosmetics
  and went unnoticed until real user testing
- Two independent alert matching systems both had the same fundamental flaw
- Dead code (deviceId.js, useHandleStreamResponse.js, usePreventBack.js, auth
  utilities) had to be found and removed manually

The better approach: start from a clean scaffold, then deliberately port only
what is genuinely relevant to the new app's domain. This forces a conscious
decision about every file rather than inheriting everything and discovering
problems through testing.

Rule: when forking, create an explicit audit checklist of every non-trivial
file in the source app and mark each as: keep as-is / port and adapt /
replace entirely / delete. Do this before writing any new code.

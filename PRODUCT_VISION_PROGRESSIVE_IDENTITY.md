# Progressive Identity: Product Vision & Strategy

> **Core Philosophy:** Earn the right to ask for sign-in by proving value first.

## Executive Summary

HolisticMe adopts a "Progressive Identity" architecture where users get full functionality without signing in. We use anonymous device UUIDs for rate limiting and Adapty anonymous IDs for purchases. Sign-in becomes an opt-in upgrade for cross-device sync, not a gate to features.

This document captures the strategic rationale and implementation roadmap for this approach.

---

## Why Anonymous-First?

### The Problem with Sign-In Gates

Traditional apps gate features behind sign-in, creating friction at the worst possible moment—before the user has experienced any value. This leads to:

- **High drop-off rates** at sign-in screens
- **Fake/throwaway accounts** from users who just want to try the app
- **Trust erosion** ("Why do they need my email just to scan food?")
- **Support burden** from password resets and account issues

### The Stoic App Precedent

Stoic (a successful wellness app) proves this model works:
- No sign-in required for any features
- iCloud sync happens automatically
- Purchases tied to Apple ID, not app accounts
- Result: Higher retention, better reviews, less support burden

### Our Approach

We adopt the best of both worlds:
- **Anonymous by default** - Full functionality on Day 1
- **Account as upgrade** - Position sign-in as unlocking cross-device sync
- **Protection framing** - "Protect your data" not "Unlock features"
- **Future-ready** - Data model designed for clean migration to accounts

---

## Phase 1: Anonymous Architecture (v1.0)

**Status:** Implementing now

### Technical Implementation

| Component | Approach |
|-----------|----------|
| Rate Limiting | Device UUID via `flutter_secure_storage` |
| Purchases | Adapty anonymous customer ID |
| Local Storage | Hive for user data, SQLite for symptoms |
| AI Features | Device UUID in `X-Device-UUID` header |
| Reports | Cached locally in Hive |

### Key Files

```
lib/core/services/device_id_service.dart    # Device UUID management
lib/core/config/app_config.dart             # Feature flags
docs/BACKEND_ANONYMOUS_AUTH_SPEC.md         # Go backend spec
```

### Feature Flags

```dart
// lib/core/config/app_config.dart
static const bool requireSignIn = false;      // Anonymous mode enabled
static const bool showSignInPrompts = false;  // Hide sign-in UI in v1
```

### What Ships

- All AI features work without sign-in
- Purchases work via Adapty anonymous ID
- Data stored locally (survives app updates, but not uninstall on Android)
- No sign-in prompts anywhere in the app
- Sign-in screen accessible but not promoted

### Success Metrics

- App Store rating > 4.5
- Day 7 retention > 40%
- Conversion to paid > 5%
- Support tickets related to auth: ~0

---

## Rate Limiting & Abuse Prevention

### Free Tier Quotas

| Feature | Limit | Reset |
|---------|-------|-------|
| Food Scans | 3/day | Daily |
| GingerBee Chat | 5/day | Daily |
| Tongue Analysis | 1/month | Monthly |
| Meal Plans | 1 total (3-day max) | Never |

### Paid Tier Quotas

| Feature | Premium | Premium Plus |
|---------|---------|--------------|
| Food Scans | 30/day | 100/day |
| GingerBee Chat | 50/day | 200/day |
| Tongue Analysis | 2/month | 4/month |
| Meal Plans | 5/month | 36/month (soft cap) |

### Abuse Prevention Strategy

**V1 Approach: Keep It Simple**

1. **Device UUID via `flutter_secure_storage`**
   - Survives app uninstall on iOS (Keychain persistence)
   - Mostly survives on Android (EncryptedSharedPreferences)
   - Stops casual abuse (reinstall to reset quotas)
   - ~30 minutes to implement ✅

2. **Don't over-engineer**
   - No device fingerprinting
   - No phone verification
   - No CAPTCHA
   - No IP blocking

3. **Monitor, then react**
   - Watch OpenAI dashboard for anomalies
   - If same IP has 50+ device IDs → investigate manually
   - Only add friction if abuse becomes material

**Why This Works:**

- Most users won't bother reinstalling to get 3 more scans
- iOS Keychain survives uninstall → primary abuse vector blocked
- Android abuse is harder (requires app data wipe or factory reset)
- Revenue comes from engaged users who convert to paid, not from blocking free users

**When to Add More Protection (Phase 2+):**

Only if we see material abuse:
- OpenAI costs spike unexpectedly
- Same IP with many device UUIDs
- Bot-like usage patterns

Then consider:
- IP-based soft rate limiting (warn, don't block)
- Device attestation (Play Integrity / App Attest)
- Account requirement for heavy users

**What NOT to Do:**

- Don't require phone verification (kills conversion)
- Don't add CAPTCHA (terrible UX)
- Don't fingerprint devices (privacy violation, App Store risk)
- Don't block VPNs (punishes legitimate users)

---

## Phase 2: Optional Account Layer (v1.5-2.0)

**Status:** Design now, implement after PMF validation

### User Journey

```
[User has been using app for 2 weeks]
[Has 14 daily check-ins, symptom logs, meal plans]

App shows gentle prompt:
┌─────────────────────────────────────────┐
│  🐝 Protect Your Wellness Journey       │
│                                         │
│  You've logged 14 check-ins and         │
│  tracked 8 symptoms this month.         │
│                                         │
│  Create an account to:                  │
│  • Sync across all your devices         │
│  • Restore your data if you switch      │
│    phones                               │
│  • Never lose your wellness history     │
│                                         │
│  [Create Account]        [Not Now]      │
└─────────────────────────────────────────┘
```

### Prompt Triggers (Soft, Never Blocking)

| Trigger | Location | Message Focus |
|---------|----------|---------------|
| 7+ days active | Home screen banner | "Protect your progress" |
| 10+ check-ins | Profile section | "Sync across devices" |
| After purchase | Post-purchase screen | "Secure your subscription" |
| Monthly report | Report screen | "Access reports anywhere" |

### Technical Requirements

#### Data Migration Path

When user creates account, we need to:

1. **Link Adapty customer** - Transfer anonymous purchases to account
2. **Upload local data** - Sync Hive/SQLite to cloud
3. **Preserve device UUID** - For continuity during transition
4. **Handle conflicts** - If user had old account with different data

#### Data Model Considerations (Design Now)

```dart
// Every local record needs a UUID that can become a server ID
class DailyCheckIn {
  final String id;           // UUID, becomes server ID
  final String? odId;  // Device UUID until account created
  final String? userId;      // Firebase UID after account created
  final DateTime createdAt;
  final DateTime? syncedAt;  // null = never synced
  // ... other fields
}
```

#### Migration Service (Stub Now)

```dart
// lib/core/services/account_migration_service.dart
class AccountMigrationService {
  /// Migrate all local data to cloud account
  /// Called when user signs up/signs in for first time
  Future<MigrationResult> migrateToAccount(String firebaseUid) async {
    // 1. Link Adapty anonymous ID to Firebase UID
    // 2. Upload local check-ins, symptoms, profiles
    // 3. Upload local meal plans, scan history
    // 4. Mark local data as synced
    // 5. Enable cloud sync going forward
  }
}
```

### What Ships in v1.5-2.0

- "Create Account" option in Profile (not prominent)
- Gentle prompts after value thresholds (dismissible, 7-day cooldown)
- Full data migration on account creation
- Cross-device sync for account holders
- Subscription portability via Adapty customer linking

---

## Phase 3: Expand Use Cases (v2.0+)

**Status:** Future roadmap

### Practitioner Portal

```
┌─────────────────────────────────────────┐
│  TCM Practitioner Dashboard             │
│                                         │
│  Your Patients:                         │
│  • Sarah M. - Last check-in: Today      │
│  • John D. - Needs follow-up            │
│  • Lisa K. - New symptom reported       │
│                                         │
│  [View Patient] [Send Recommendation]   │
└─────────────────────────────────────────┘
```

**Requirements:**
- User account required (to link with practitioner)
- HIPAA-compliant data sharing
- Practitioner verification flow
- Revenue: B2B subscription model

### Family Sharing

```
┌─────────────────────────────────────────┐
│  Zhang Family Wellness                  │
│                                         │
│  👤 Mom (You) - 🌿 Balanced             │
│  👤 Dad - 🔥 Yang Deficient             │
│  👤 Emma - 💧 Yin Deficient             │
│                                         │
│  Shared meal plans | Individual logs    │
└─────────────────────────────────────────┘
```

**Requirements:**
- Family manager account
- Invite system for family members
- Shared meal planning
- Individual health tracking (private by default)
- Revenue: Family plan pricing tier

### B2B Wellness Programs

```
Corporate Wellness Dashboard
├── Employee engagement metrics
├── Anonymous aggregate health trends
├── TCM workshop scheduling
└── ROI reporting for HR
```

**Requirements:**
- Enterprise SSO integration
- Admin dashboard
- Anonymized aggregate reporting
- Custom branding
- Revenue: Enterprise contracts

---

## Design Principles

### 1. Never Block Value

Every feature works without sign-in. Sign-in unlocks convenience (sync), not capability.

```
❌ "Sign in to use AI meal planning"
✅ "Sign in to sync your meal plans across devices"
```

### 2. Protection Framing, Not Unlock Framing

Position accounts as protecting what users have built, not unlocking what they're missing.

```
❌ "Create account to unlock premium features"
✅ "Protect your 30 days of wellness data"
```

### 3. Respect Dismissal

When users say "Not now," respect it. Don't nag.

- 7-day cooldown after dismissing prompt
- Maximum 1 prompt per session
- Never show prompt during active task (scanning, logging)

### 4. Data Portability

Users own their data. Make export easy, even without account.

```dart
// Always available, no account required
Future<File> exportAllData() async {
  // Export to JSON/CSV
  // Include all check-ins, symptoms, meal plans
  // User can import to new device or share with practitioner
}
```

### 5. Graceful Degradation

If cloud services fail, app should still work perfectly in offline mode.

```dart
// Every cloud operation has local fallback
try {
  await cloudService.syncData();
} catch (e) {
  // Continue with local data
  // Queue sync for later
  // Never block user
}
```

---

## Implementation Checklist

### Phase 1 (v1.0) - Current Sprint

- [x] Device UUID service (`flutter_secure_storage`)
- [x] Feature flags (`requireSignIn`, `showSignInPrompts`)
- [x] Backend services use device UUID
- [x] Sign-in prompts disabled
- [x] Backend spec for Go developer
- [ ] QA testing of anonymous flow
- [ ] App Store submission

### Phase 1.5 (Pre-v2.0) - Design Work

- [ ] Data model audit - ensure all records have migration-ready UUIDs
- [ ] Design account creation flow (Apple/Google sign-in)
- [ ] Design data migration UX (progress indicator, error handling)
- [ ] Design prompt triggers and copy
- [ ] Adapty customer linking documentation

### Phase 2 (v1.5-2.0) - Implementation

- [ ] Account creation flow
- [ ] Guest engagement tracking
- [ ] Sign-in prompt banner (dismissible)
- [ ] Data migration service
- [ ] Adapty customer linking
- [ ] Cross-device sync
- [ ] Cloud backup/restore

### Phase 3 (v2.0+) - Future

- [ ] Practitioner portal design
- [ ] Family sharing design
- [ ] B2B requirements gathering
- [ ] HIPAA compliance audit

---

## FAQ

### Why not just use iCloud sync like Stoic?

iCloud sync is great but:
- Android users get no sync at all
- No path to practitioner features
- No aggregate analytics for product improvement
- Harder to implement cross-platform

Our approach: Start with local-only, add optional cloud accounts. Best of both worlds.

### What if users lose their phone before creating account?

Their data is lost. This is the trade-off for not requiring sign-in. We mitigate by:
- Prompting after they have valuable data ("Protect your 30 check-ins")
- Making account creation frictionless (Apple/Google sign-in)
- Offering data export at any time

### How do we handle subscription restoration without accounts?

Adapty handles this via App Store/Play Store account linking. User's subscription follows their Apple ID or Google account, not our app account.

### What about GDPR/privacy without accounts?

Actually easier! Less PII to manage. Device UUID is not personally identifiable. We only collect what's on-device. Clear privacy story.

---

## References

- [Backend Anonymous Auth Spec](./BACKEND_ANONYMOUS_AUTH_SPEC.md)
- [Meal Plan Architecture](../lib/features/meal_planning/MEAL_PLAN_ARCHITECTURE_V2.md)
- [Adapty Documentation](https://docs.adapty.io/)
- [flutter_secure_storage](https://pub.dev/packages/flutter_secure_storage)

---

*Last Updated: January 2026*
*Authors: Product & Engineering Team*

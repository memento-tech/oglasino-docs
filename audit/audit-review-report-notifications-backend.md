# Audit — REVIEW + REPORT notification producers (deep-link path standardization)

**Type:** read-only audit. No code changed.
**Date:** 2026-06-02
**Scope:** ONLY the review notifications (admin review service) and the report-resolve
notification (admin report facade). Favorite/follow/ban/message deliberately not audited.
**Ground truth:** the code as it stands on `dev`. No prior specs/audits were used as source.

Files in scope:
- `src/main/java/com/memento/tech/oglasino/admin/service/impl/DefaultAdminReviewService.java`
- `src/main/java/com/memento/tech/oglasino/admin/facade/impl/DefaultAdminReportFacade.java`
- `src/main/java/com/memento/tech/oglasino/notifications/service/impl/DefaultFirebasePushService.java`
- `src/main/java/com/memento/tech/oglasino/notifications/enums/NotificationCategoryId.java`
- `src/main/java/com/memento/tech/oglasino/notifications/enums/NotificationType.java`
- `src/main/java/com/memento/tech/oglasino/notifications/service/NotificationConstants.java`

---

## 0. Producer inventory (brief-vs-reality confirmation)

The brief named three producer methods in `DefaultAdminReviewService`
(`sendReviewerSuccessNotification`, `sendTargetUserSuccessNotification`, `sendFailNotification`)
and asked whether an approve path also exists. **All three exist, and the approve path is
exactly those first two.** No other review-notification producer exists. The full mapping:

| Review action | Trigger method (file:line) | Notification producer(s) | Recipient |
|---|---|---|---|
| **Approve** | `approveReview(long reviewId)` `DefaultAdminReviewService.java:47` → `sendSuccessNotification(review):78` | `sendReviewerSuccessNotification` + `sendTargetUserSuccessNotification` | reviewer **and** the user being reviewed (two notifications) |
| **Disapprove** | `disapproveReview(ApproveDisapproveRequestDTO):58` | `sendFailNotification` | reviewer only |

`sendSuccessNotification` (`:78`) is a private dispatcher, not itself a producer — it just
calls the two success producers. So **the approve action emits TWO notifications, the
disapprove action emits ONE.** There is no fourth producer.

Repo-wide confirmation that these four navigate literals are the only review/report ones:
`grep -rn "/dashboard/reviews\|/dashboard/reports" src/main/java` returns exactly the four
lines reported in §3 below and nothing else. The only other `sendAsyncNotification` callers
(`DefaultAdminProductFacade` = ban/unban, `DefaultUserFacade`, `DefaultFavoriteProductFacade`,
`DefaultNotificationsService`) are out of scope per the brief.

---

## 1. REVIEW NOTIFICATIONS — every producer (verbatim)

### 1a. `sendReviewerSuccessNotification(Review review)` — `DefaultAdminReviewService.java:83`

- **Triggered by:** review **approve** (`approveReview` → `sendSuccessNotification`).
- **Recipient:** the **reviewer** (the person who wrote the review).
  - `reviewerNotification.setUserId(review.getReviewer().getId());` — `:95`
  - Sourced server-side from the persisted `Review` entity's `reviewer` association, not from
    client input.
- **notificationCategoryId:** `NotificationCategoryId.NAVIGATION` — `:97`
- **notificationType:** `NotificationType.SUCCESS` — `:96`
- **data map (verbatim, `:86–93`):**
  ```java
  Map.of(
      "navigate",
      "/dashboard/reviews/given?reviewId=" + review.getId(),
      "label",
      translationService.getBackendTranslation(
          "notif.review.see.my.button", preferredLanguage));
  ```
  - `navigate` literal: **`"/dashboard/reviews/given?reviewId=" + review.getId()`** (`:89`)
  - `label` key: **`notif.review.see.my.button`**
- **Title key:** `notif.review.check.success.title` (`:101–102`)
- **Description key:** `notif.review.check.success.description` (`:104–105`) — no `.formatted(...)`.
- **Language:** `review.getReviewer().getPreferredLanguage().getCode()` (`:84`)

### 1b. `sendTargetUserSuccessNotification(Review review)` — `DefaultAdminReviewService.java:110`

- **Triggered by:** review **approve** (`approveReview` → `sendSuccessNotification`).
- **Recipient:** **the user being reviewed** (the review's target user).
  - `targetUserNotification.setUserId(review.getTargetUser().getId());` — `:122`
  - Sourced server-side from `review.getTargetUser()`.
- **notificationCategoryId:** `NotificationCategoryId.NAVIGATION` — `:124`
- **notificationType:** `NotificationType.NORMAL` — `:123`
- **data map (verbatim, `:113–119`):**
  ```java
  Map.of(
      "navigate",
      "/dashboard/reviews/received?reviewId=" + review.getId(),
      "label",
      translationService.getBackendTranslation(
          "notif.review.see.new.button", preferredLanguage));
  ```
  - `navigate` literal: **`"/dashboard/reviews/received?reviewId=" + review.getId()`** (`:116`)
  - `label` key: **`notif.review.see.new.button`**
- **Title key:** `notif.review.new.title` (`:128`)
- **Description key:** `notif.review.new.description` (`:131`), rendered with
  `.formatted(review.getReviewer().getDisplayName())` (`:132`).
- **Language:** `review.getTargetUser().getPreferredLanguage().getCode()` (`:111`)

### 1c. `sendFailNotification(Review review, String reason)` — `DefaultAdminReviewService.java:137`

- **Triggered by:** review **disapprove** (`disapproveReview` → `sendFailNotification`,
  `:75`). `reason` is `approveDisapproveRequest.getReason()`.
- **Recipient:** the **reviewer**.
  - `reviewNotification.setUserId(review.getReviewer().getId());` — `:162`
  - Sourced server-side from `review.getReviewer()`.
- **notificationCategoryId:** `NotificationCategoryId.NAVIGATION` — `:164`
- **notificationType:** `NotificationType.WARNING` — `:163`
- **data map (verbatim, `:153–159`):**
  ```java
  Map.of(
      "navigate",
      "/dashboard/reviews/given?reviewId=" + review.getId(),
      "label",
      translationService.getBackendTranslation(
          "notif.review.see.my.button", reviewerPreferredLanguage));
  ```
  - `navigate` literal: **`"/dashboard/reviews/given?reviewId=" + review.getId()`** (`:156`)
  - `label` key: **`notif.review.see.my.button`** (same key as 1a)
- **Title key:** `notif.review.check.fail.title` (`:168–169`)
- **Description key:** `notif.review.check.fail.description` (`:172`), rendered with
  `.formatted(productName, reason)` (`:173`). `productName` is resolved server-side from the
  reviewed product's translation in the reviewer's preferred language, falling back to the
  literal `"PRODUCT NAME"` (`:140–151`).
- **Language:** `review.getReviewer().getPreferredLanguage().getCode()` (`:138`)

---

## 2. REPORT-RESOLVE NOTIFICATION — `DefaultAdminReportFacade.java`

- **Method:** `resolveReport(ReportResolveRequestDTO):84` → `sendReportNotification(...):101`.
- **Recipient:** the **reporter** (the user who filed the report). Confirmed.
  - `reportNotification.setUserId(report.getReporter().getId());` — `:106`
  - Sourced server-side from the persisted `Report` entity loaded by id (`reportRepository
    .findById(reportResolveRequest.getReportId())`, `:86`), via `report.getReporter()`.
- **Conditional?** YES — gated on an opt-in flag. `resolveReport` returns early before
  sending if `!reportResolveRequest.isNotifyUser()` (`:94–96`). The report is still marked
  resolved/finished and saved regardless; only the notification is skipped.
- **notificationCategoryId:** `NotificationCategoryId.NAVIGATION` — `:108`
- **notificationType:** `NotificationType.NORMAL` — `:107`
- **data map (verbatim, `:109`):**
  ```java
  reportNotification.setData(Map.of("navigate", "/dashboard/reports?reportId=" + report.getId()));
  ```
  - `navigate` literal: **`"/dashboard/reports?reportId=" + report.getId()`** (`:109`)
  - **No `label` key** in the report data map (unlike the three review producers).
- **Title / Description:** NOT translation keys. They are admin-supplied free text from the
  request:
  - `reportNotification.setTitle(reportResolveRequest.getTitle());` — `:104`
  - `reportNotification.setDescription(reportResolveRequest.getDescription());` — `:105`
  - (The same title/description are also concatenated into `report.setResolution(title + " | "
    + description)` at `:88–89`.) So the report notification uses **no `notif.*` translation
    keys at all** — title, description, and there is no button label.

---

## 3. NAVIGATE STRINGS THAT NEED CHANGING (`/dashboard/*` literals, review + report only)

| # | File:line | Producer | Exact literal |
|---|---|---|---|
| 1 | `DefaultAdminReviewService.java:89` | `sendReviewerSuccessNotification` (approve → reviewer) | `"/dashboard/reviews/given?reviewId=" + review.getId()` |
| 2 | `DefaultAdminReviewService.java:116` | `sendTargetUserSuccessNotification` (approve → target user) | `"/dashboard/reviews/received?reviewId=" + review.getId()` |
| 3 | `DefaultAdminReviewService.java:156` | `sendFailNotification` (disapprove → reviewer) | `"/dashboard/reviews/given?reviewId=" + review.getId()` |
| 4 | `DefaultAdminReportFacade.java:109` | `sendReportNotification` (resolve → reporter) | `"/dashboard/reports?reportId=" + report.getId()` |

These four are the **complete** set of `/dashboard/*` literals emitted by the review + report
producers. (Ban/unban's `/dashboard/products?productId=` lives in `DefaultAdminProductFacade`
and was explicitly excluded — not re-audited here.)

Note the literals are string-concatenations with the id appended; the query-param name differs
(`reviewId` for reviews, `reportId` for reports). The base path `/dashboard/...` is the part
the owner-route flattening to `/owner/*` presumably needs to standardize.

---

## 4. NPE-SAFETY CONSTRAINT — categoryId must stay non-null

**Confirmed: `DefaultFirebasePushService` still dereferences `getNotificationCategoryId()` such
that a null categoryId WILL NPE.** Relevant line (verbatim, `DefaultFirebasePushService.java:58`):

```java
dataMap.put(CATEGORY_ID_PARAM_NAME, notification.getNotificationCategoryId().toString());
```

`getNotificationCategoryId()` returns `null` if unset, and `.toString()` on null throws
`NullPointerException`. This is inside the `try` at `:51`, but the `catch` blocks (`:90`
`FirebaseMessagingException`, `:105` generic `Exception`) only log — the push silently fails to
send for that token. So a null categoryId does not crash the request, but it **does drop the
web push entirely** for every Firebase (web) token.

The immediately preceding line (`:57`) has the same hazard on type:
```java
dataMap.put(TYPE_PARAM_NAME, notification.getNotificationType().toString());
```
A null `notificationType` would NPE the same way.

`CATEGORY_ID_PARAM_NAME` = `"categoryId"`, `TYPE_PARAM_NAME` = `"type"`
(`NotificationConstants.java:6–7`).

**Implication for the path-standardization brief:** categoryId (and type) must remain non-null
on every review/report notification regardless of any other change. The B1 session's choice to
set the disapprove notification's categoryId to `NAVIGATION` remains load-bearing for the
Firebase web-push path. All four producers in scope currently set `NAVIGATION` + a non-null
type, so all are safe today; **do not remove or null the categoryId** on any of them. Changing
the categoryId *value* (e.g. to a different non-null enum constant) is safe; removing it is not.

For reference, the available enum constants:
- `NotificationCategoryId` (`NotificationCategoryId.java`): `MESSAGE`, `SAVED_PRODUCT`,
  `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`, `NAVIGATION`, `INFO`.
- `NotificationType` (`NotificationType.java`): `NORMAL`, `SUCCESS`, `DANGER`, `WARNING`.

---

## 5. Tab-specificity per review producer

The review navigate strings ARE tab-specific — they distinguish the `given` vs `received`
review subpaths:

| Producer | Recipient | Subpath today |
|---|---|---|
| `sendReviewerSuccessNotification` (approve) | reviewer | **`/dashboard/reviews/given`** |
| `sendTargetUserSuccessNotification` (approve) | target user (reviewed) | **`/dashboard/reviews/received`** |
| `sendFailNotification` (disapprove) | reviewer | **`/dashboard/reviews/given`** |

Logic: the **reviewer** (author) is always pointed at the **`given`** tab (their own reviews,
both on approve-success and disapprove-fail); the **reviewed user** is pointed at the
**`received`** tab (reviews about them). The report-resolve notification points at a flat
**`/dashboard/reports`** with no tab/subpath, just a `?reportId=` query param.

---

## Summary of literals for the standardization brief

| Recipient | Action | Current literal (file:line) |
|---|---|---|
| reviewer | approve-success | `/dashboard/reviews/given?reviewId=<id>` (`DefaultAdminReviewService.java:89`) |
| reviewed user | approve-success | `/dashboard/reviews/received?reviewId=<id>` (`:116`) |
| reviewer | disapprove-fail | `/dashboard/reviews/given?reviewId=<id>` (`:156`) |
| reporter | report resolved (opt-in `notifyUser`) | `/dashboard/reports?reportId=<id>` (`DefaultAdminReportFacade.java:109`) |

**Hard constraint carried into any edit:** keep `notificationCategoryId` (and
`notificationType`) non-null on all four — `DefaultFirebasePushService.java:58` (and `:57`)
NPE on null, silently dropping the web push.

# Push notifications without a relay service

**Ticket:** [#5 — 05 Research: push notifications without a relay service](https://github.com/PhillipChaffee/goose/issues/5) (map issue #1)
**Question:** How does "agent needs approval / turn finished" reach a phone whose app is backgrounded, under the standing constraint **no account, no relay, no service in the path**? The app connects to the user's own `goose serve` host over their Tailscale tailnet (iOS + Android).

---

## TL;DR — Recommendation set

1. **Android: direct tailnet socket, background-safe today.** The app holds a persistent WebSocket/SSE connection to `goose serve` over the tailnet. While a turn is running, run a **dedicated-type foreground service** with a slim persistent notification ("Agent running — tap to review"), and request a **battery-optimization exemption** (documented acceptable use case for non-FCM messaging). Raise local notifications for approval-needed / turn-finished. No third-party service, account, or relay anywhere in the path.
2. **iOS: hard OS constraint — foreground-only until the constraint is relaxed.** iOS permits **no background notification path without APNs**. Apple's own architecture makes every remote notification flow through the hosted Apple Push Notification service, and every iOS-side alternative (background pushes, Live Activities, VoIP push, notification service extensions) is APNs-dependent or App-Store-risky. Self-hosted ntfy's own docs confirm this and require forwarding to ntfy.sh → APNs for instant iOS delivery. Ship iOS v1 foreground-only and say so plainly in-product.
3. **iOS mitigations inside the constraint:** on turn start, schedule a **local notification** at an estimated completion time (system delivers scheduled local notifications even when the app is later suspended); immediately post a local notification when the user backgrounds the app mid-turn; keep the agent state in an app badge/Live Activity *only* while the app runs.
4. **If the constraint is ever relaxed for iOS:** the only real path is APNs (Apple Developer Program, push capability, an APNs auth key, and a provider server reachable from the public internet — a hosted relay in the path by definition). UnifiedPush has no iOS distributor. WebPush is a non-option because the browser/OS vendor, not the app, selects the push service.
5. **Android fallback if direct sockets prove unreliable on aggressive OEMs:** self-hosted ntfy or a UnifiedPush distributor app on the tailnet. Still no account or off-tailnet relay, but it is an extra moving part implementing the same foreground-service mechanism as (1) — prefer (1) first.

---

## (a) APNs and FCM: what hosted infrastructure do they truly require?

### APNs (iOS) — a hosted relay is unavoidable and self-hosting is impossible

Apple's server-setup documentation defines remote notifications as exactly four components: **"your company's server, known as the *provider server*; Apple Push Notification service (APNs); the user's device; your app."** The provider server sends every notification **to APNs**, which "handles the delivery of the notification to the user's device," and "Apple manages everything in between" — including "an accredited, encrypted, and persistent IP connection to the user's device."
([Setting up a remote notification server](https://developer.apple.com/documentation/usernotifications/setting-up-a-remote-notification-server))

Concretely, APNs requires:

- An **Apple Developer Program** account: the push capability adds the `aps-environment` entitlement, and you must "enable the push notification service for the App ID assigned to your project" in your developer account. ([Registering your app with APNs](https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns))
- A **provider server with an APNs authentication key** (`.p8`), Key ID, and Team ID issued from that developer account, speaking HTTP/2+TLS to Apple's endpoints. "You request this key from your developer account on developer.apple.com." ([Establishing a token-based connection to APNs](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns))
- Every app registers for a device token per bundle ID and forwards it to the provider server. ([Registering your app with APNs](https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns))

There is **no** Apple-sanctioned way to self-host the "APNs side": Apple owns and operates the device-side connection. A self-run provider server does not remove the relay — Apple's servers are still in the path, and the provider must be reachable from the public internet (not just the tailnet) to reach `api.push.apple.com`. So on iOS, "no relay in the path" and "background notification" are mutually exclusive.

### FCM (Android) — Google-hosted transport, Firebase project required

FCM's architecture is "two main components for sending and receiving: a trusted environment such as Cloud Functions for Firebase **or an app server** on which to build, target, and send messages" plus the client app receiving via "the corresponding platform-specific transport service" — i.e., Google's servers. Setup requires creating a Firebase project and adding the FCM SDK. ([Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging))

FCM is therefore not a way to honor the constraint: it puts Google's hosted service in the path and requires a Firebase project (an account). The ntfy project states it flatly: FCM is "the *only* Google approved way to send push messages to Android devices" for apps relying on Play-grade background delivery. ([ntfy: from your phone](https://docs.ntfy.sh/subscribe/phone/#instant-delivery)) But Android does not *require* FCM — a sideloaded app can hold its own connection (see (d)).

## (b) Self-hosted push over the tailnet

### Self-hosted ntfy

- **Android: fully workable relay-free.** The ntfy Android app achieves instant delivery "even when your phone is in doze mode … achieved with a foreground service" (permanent notification). It uses FCM "only for the main host ntfy.sh, and only in the Google Play flavor — it won't use Firebase for any self-hosted servers, and not at all in the F-Droid flavor." Without instant delivery, "messages may arrive with a significant delay (sometimes many minutes, or even hours later)." ([ntfy: from your phone](https://docs.ntfy.sh/subscribe/phone/#instant-delivery)) The app also supports custom headers (e.g., Tailscale Funnel auth) and trusting self-signed/mTLS certificates — both useful on a tailnet-only deployment. ([Advanced settings](https://docs.ntfy.sh/subscribe/phone/#advanced-settings))
- **iOS: self-hosted ntfy alone cannot do instant background delivery.** ntfy's docs state: "Unlike Android, iOS heavily restricts background processing, which sadly makes it impossible to implement instant push notifications without a central server." Self-hosted servers that want instant iOS delivery must set `upstream-base-url: "https://ntfy.sh"`, which forwards a `poll_request` (message ID only) to ntfy.sh, which relays via Firebase → APNs. Without upstream forwarding, "delivery can take hours … 20-30 minutes" when the phone is in use. ([ntfy: iOS instant notifications](https://docs.ntfy.sh/config/#ios-instant-notifications)) That upstream hop is a public hosted relay — it violates the constraint.

### UnifiedPush

UnifiedPush's architecture is: application server → **push server** → **push distributor app** → end-user app. The distributor (e.g., the ntfy app, NextPush) maintains the connection to the push server and forwards messages to registered apps; you can point it at "your own selfhosted ntfy server." ([UnifiedPush definitions](https://unifiedpush.org/developers/spec/definitions/), [ntfy as distributor](https://docs.ntfy.sh/subscribe/phone/#unifiedpush)) The specification documents only **Android** and **Linux (D-Bus)** distributor bindings — **there is no iOS distributor**, precisely because iOS cannot keep a third-party app's connection alive in the background. On Android, the distributor is just another foreground-service client; for a single-app product like goose, adding a distributor app is redundant machinery around the same foreground-service mechanism.

### WebPush (RFC 8030) with a self-hosted server

The WebPush protocol itself is a standard HTTP API — anyone *can* host a compliant push service. But the client does not get to choose it: "A **push service** is a web service controlled by your user's browser vendor … **Each browser uses whatever push service it wants. You as a website developer have no control over that.**" (Chrome's example endpoint is `fcm.googleapis.com`.) ([web.dev: Push notifications overview](https://web.dev/articles/push-notifications-overview)) So:

- Browser/PWA clients: always terminate on the vendor's hosted service (FCM for Chrome, Mozilla autopush for Firefox, Apple's web push for Safari). A self-hosted push service never enters the path. Not constraint-compatible.
- A **native** Android app could speak RFC 8030 against a self-hosted push service, but that is functionally identical to holding your own socket, with extra standardization overhead. It buys nothing over recommendation (1).
- On iOS there is no API for a native app to register a custom push service at all — remote delivery is APNs-only.

## (c) iOS-specific background limits

iOS gives background runtime only through grant-based, system-scheduled mechanisms ([Choosing background strategies for your app](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)):

| Mechanism | What Apple grants | Why it can't deliver "agent needs approval" |
|---|---|---|
| `beginBackgroundTask` after backgrounding | "a limited amount of time"; "The system terminates your app if you fail to call `endBackgroundTask`" | Seconds-to-minutes grace; the socket dies with the process when it suspends |
| `BGAppRefreshTask` / `BGProcessingTask` | "The system decides the best time to launch"; app refresh gets "up to 30 seconds" | Timing is the system's choice (charging, usage patterns); not event-driven |
| **Background (silent) push**, `content-available: 1` | Wakes app ~30 s; **"If you send background pushes more frequently than three times per hour, the system imposes rate limitations"** | Requires APNs (it *is* an APNs delivery), so not relay-free; rate-limited |
| Notification Service Extension | Wake on a received push | Runs only after an APNs delivery; still APNs |
| **VoIP push (PushKit)** | Wakes app out of suspension on APNs delivery | iOS 13+ SDK **requires CallKit**: "If you are unable to support CallKit in your app, **you cannot use PushKit** to handle push notifications" ([Responding to VoIP notifications from PushKit](https://developer.apple.com/documentation/pushkit/responding-to-voip-notifications-from-pushkit)) — abusing it for agent alerts presents fake call UI and is grounds for removal of push privileges |
| **Live Activities** | Lock Screen/Dynamic Island updates | Remote start/update/end all require **APNs** payloads (`apns-push-type: liveactivity`) from a provider server ([Starting and updating Live Activities with ActivityKit push notifications](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications)); local updates only flow while the app is running |
| Scheduled **local** notifications | System fires them at trigger time even if the app is suspended | Trigger times must be scheduled **by the app while it runs**; `goose serve` cannot inject them, so they only work as estimates (usable mitigation, not a delivery channel) |

**Plain statement of the hard constraint:** iOS suspends background apps and kills their sockets; the only OS-mediated way to wake a suspended app on iOS is APNs, which is Apple-hosted, requires an Apple Developer Program identity, and cannot be self-hosted. Therefore **background push on iOS without a relay in the path is impossible** — not merely impractical. (`BGContinuedProcessingTask` extends foreground work into the background for specific entitlement cases; it is not a notification channel and not generally available to arbitrary apps.)

## (d) Android's latitude

Android has no mandatory push provider for non-Play distribution:

- **Foreground services** are the sanctioned way to keep work alive: "Foreground services let you asynchronously perform operations that are noticeable to the user" and "show a status bar notification." They require declaring a foreground-service type, and there are restrictions on *starting* them from the background — which is fine here, since the user starts the turn in-app. ([Foreground services overview](https://developer.android.com/develop/background-work/services/foreground-services))
- **Doze** "suspends network access" for idle apps with periodic maintenance windows — but a "process currently in the foreground, either as an activity **or foreground service**" is not idle. ([Optimize for Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby))
- **Battery-optimization exemption**: the user-configurable exemption list lets an app "use the network and hold partial wake locks during Doze." Apps may use `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` where the use case is acceptable — the documented acceptable case is: "**can't use FCM because of technical dependency on another messaging service** or Doze and App Standby break the core function of the app." goose serve over the tailnet is exactly that technical dependency. Note the caveat: "**Google Play policies prohibit apps from requesting direct exemption** from Power Management features … unless the core function of the app is adversely affected" — if the app were ever Play-distributed, the in-app prompt must be justified accordingly. ([Doze exemptions](https://developer.android.com/training/monitoring-device-state/doze-standby))
- FCM's own advantage over a direct socket is only Doze-friendliness ("FCM is optimized to work with Doze"); a foreground service + exemption achieves the same continuity without Google in the path.

## (e) Foreground-only v1 and its UX cost

- **Android:** effectively zero cost if recommendation (1) is implemented — backgrounded app, notification arrives, user taps to approve.
- **iOS:** the cost is real for a long-running agent. Turn lengths of "many minutes" mean the user must either keep the app foregrounded (screen on, app active) through the turn, or accept that completion/approval events are only visible on next open. Estimated local notifications (scheduled at turn start) soften this: an over-estimate notification ("your agent may be waiting for approval") arrives even while the app is suspended. It is a reminder with the wrong semantics for variable-length turns — early finish = stale notification, late finish = missed — so it must be framed as an estimate in-product, not as a completion signal.

## Comparison

| Option | Works in background | Relay/account in path | What must be installed/running | iOS | Android |
|---|---|---|---|---|---|
| Direct socket (app ⇄ `goose serve` over tailnet) + foreground service (Android) | Yes (Android) | None — user's own host only | goose app; FGS + battery exemption (Android) | n/a (socket dies when suspended) | ✅ Recommended v1 |
| Foreground-only | Foreground only | None | goose app | ✅ Only legal option under the constraint | Unnecessary given (1) |
| Self-hosted ntfy (tailnet) | Android yes; iOS **not instant** | None on Android; **iOS requires ntfy.sh → FCM/APNs upstream** | ntfy server + ntfy app as client or distributor | ❌ violates constraint for instant delivery | ✅ viable alternative |
| UnifiedPush distributor (self-hosted push server) | Yes via distributor app | None if distributor + server are self-hosted/tailnet | distributor app (ntfy/NextPush) + server | ❌ no iOS distributor exists | viable, but redundant for a single app |
| WebPush with self-hosted push service | Yes in theory | Self-hosted server — but **UA chooses the push service**; browsers hardcode vendor endpoints | compliant push service + browser support | ❌ (Safari → Apple's service) | ❌ pointless vs. direct socket |
| APNs (Apple-hosted) | Yes | **Apple's relay in path**; Apple Developer Program, `.p8` key, public provider server | Xcode push capability; provider server | the only background path | — |
| FCM (Google-hosted) | Yes | Google's relay in path; Firebase project | FCM SDK; Play services on device | — | the "standard" but constraint-violating path |

## Hard constraints that force a trade

1. **iOS:** background notification ⇒ APNs ⇒ hosted Apple relay + Apple Developer Program + a public-internet provider server. **No legal alternative exists.** The constraint set (no account, no relay) and iOS background notifications cannot both hold; iOS v1 must be foreground-only, or the constraint must be consciously relaxed for iOS only.
2. **Android:** full background delivery is achievable with zero third-party components (foreground service + battery-optimization exemption + direct tailnet socket). The exemption request is policy-gated on Play; sideloaded/F-Droid-style distribution is unaffected.
3. **WebPush/UnifiedPush do not rescue iOS.** WebPush endpoints are vendor-chosen; UnifiedPush has no iOS distributor because the same OS constraint binds any distributor app.

## Recommended shape for goose mobile v1

- **Both platforms:** app connects directly to `goose serve` over the tailnet; `serve` emits approval-needed / turn-finished events over that socket; local notifications raised from the app.
- **Android:** dedicate a foreground-service type to the agent-session service; show a persistent low-priority notification while a turn runs (tap → review); prompt for battery-optimization exemption with the non-FCM technical-dependency justification; reconnect with backoff; serve queues events missed while disconnected.
- **iOS:** foreground-only delivery; at turn start schedule a local "check on your agent" notification at a generous estimated completion time (user-adjustable); when the user backgrounds the app mid-turn, post an explicit "notifications pause while backgrounded" local notification so the limitation is visible, not surprising. Document the APNs trade as the known path to full background delivery if/when the no-relay constraint is revisited.
- **Deferred/optional:** self-hosted ntfy or UnifiedPush distributor on the tailnet as an Android robustness fallback for aggressive OEM battery managers.

## Sources

Apple (primary):

- Setting up a remote notification server — https://developer.apple.com/documentation/usernotifications/setting-up-a-remote-notification-server
- Registering your app with APNs — https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns
- Establishing a token-based connection to APNs — https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns
- Choosing background strategies for your app — https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app
- Background Tasks framework (BGTaskScheduler / BGAppRefreshTask) — https://developer.apple.com/documentation/backgroundtasks
- Responding to VoIP notifications from PushKit (CallKit requirement) — https://developer.apple.com/documentation/pushkit/responding-to-voip-notifications-from-pushkit
- Starting and updating Live Activities with ActivityKit push notifications — https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications

Android (primary):

- Foreground services overview — https://developer.android.com/develop/background-work/services/foreground-services
- Optimize for Doze and App Standby (exemptions, FCM guidance) — https://developer.android.com/training/monitoring-device-state/doze-standby
- Firebase Cloud Messaging (architecture) — https://firebase.google.com/docs/cloud-messaging

ntfy / UnifiedPush / WebPush (primary):

- ntfy server configuration: iOS instant notifications (`upstream-base-url`) — https://docs.ntfy.sh/config/#ios-instant-notifications
- ntfy phone subscription: instant delivery, foreground service, FCM usage, UnifiedPush distributor — https://docs.ntfy.sh/subscribe/phone/
- UnifiedPush specification: definitions (push server / distributor model) — https://unifiedpush.org/developers/spec/definitions/
- web.dev: Push notifications overview (push service chosen by browser vendor) — https://web.dev/articles/push-notifications-overview
- WebPush protocol (RFC 8030) — https://datatracker.ietf.org/doc/html/rfc8030
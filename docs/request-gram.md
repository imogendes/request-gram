# Request GRAM

In a private chat, a user asks the other person for an amount of GRAM. The request is the normal
Gram transfer card in a **request** state. The payer opens it, checks who is asking (with their
@username), and pays with one tap plus Face ID, Touch ID or the device passcode. The same card then
turns **paid**, and a grey service line confirms the payment. No second transfer card is posted.

Everything is behind `gram_requests_enabled` (app config, default off) or Debug Settings ▸
**Request GRAM** (`ExperimentalUISettings.requestGram`). Every limit is enforced by the server; the
client only mirrors what it can see, to hide or explain what the server would refuse.

## Status

Not built with `Make.py`. This machine has only the Command Line Tools (no Xcode 26.2) and no access
to the codesigning repository, so the full build and the `ios_unit_test` targets could not run.

What was checked:

| Part | Check |
|---|---|
| `swift_scheme.tl` | Every new constructor id recomputed with the TL CRC32 rule, validated against existing entries; no collisions |
| TelegramApi + TelegramCore | `generate-swiftpm.sh`, then `swift build --package-path submodules/TelegramCore` (macOS, as Telegram-Mac builds it): compiles |
| TelegramCore tests | `GramRequestTests`, `GramRequestPrivacyTests` typechecked against the built modules (stand-in XCTest module) |
| WalletContext tests | `WalletGramRequests.swift` + `WalletGramRequestTests` typechecked in isolation against TelegramCore |
| UI modules | `swiftc -parse` only (they need the iOS SDK) |
| `PostboxImportCheck.py` | passes |

Run before merging:

```sh
python3 build-system/Make/Make.py --overrideXcodeVersion --cacheDir ~/telegram-bazel-cache build --continueOnError --configurationPath build-system/appstore-configuration.json --gitCodesigningRepository git@gitlab.com:peter-iakovlev/fastlanematch.git --gitCodesigningType development --gitCodesigningUseCurrent --buildNumber=1 --configuration=debug_sim_arm64
```

and `Make.py test --target //submodules/TelegramCore:TelegramCoreTests` and
`--target //submodules/WalletContext:WalletCardStickersTests`.

## Flow

1. **📎 → Money.** `WalletSendScreen` shows a **Send | Request** switch under the recipient card when
   Request is offered: the feature is on and the peer is a regular user (not a bot, a service or
   deleted account, or the account itself).
2. **Request tab.** The title becomes *Request Money from*. The keypad, animated amount, fiat pill and
   comment are the Send screen's own; balance, fee and deposit are hidden, and nothing is signed. The
   line above the button reads *imogen can send it right from this chat.* and *4 of 5 requests left
   today*, or why a request cannot be sent (with *Set Username* when a username is missing). The
   button reads *Request 250 Grams*. Switching to Request asks `wallet.getGramRequestEligibility`.
   When the server accepts the request, `requestGram` delivers the new message's id
   (`gramRequestCreatedMessageId`: the `updateMessageID` carrying the call's `random_id`), and the amount's
   gem spins and flies out of the sheet into the new card, exactly as a sent transfer's does
   (`ChatWalletTransferAnimation`, the flight keyed by `walletGramRequestFlightId` instead of a wallet
   operation). On landing the card spins and bumps, and the blue `request` ribbon fades in where a transfer
   would show its sending clock.
3. **Card.** `messageActionGramRequest` renders with `ChatMessageTransferBubbleContentNode`: same card,
   gem, amount and comment, the requester's name and address, a blue `request` ribbon (`0x42b0ff`), no
   Pay button. Service line: *You requested 250 Grams ($312.50) from imogen* / *Caesar requested 250
   Grams ($312.50)* (the transfer card's own line: Semibold 13 for the amount, and for the other person's name
   only on the receiving side). On the payer's side an unread pending request plays the received transfer's
   arrival: the card rises, the gem lands with star bursts, and the blue ribbon grows out of the corner
   (`playsArrivalAnimation`, the same `WalletTransferArrivalAnimation` timing).
4. **Details sheet.** Tapping the card opens `WalletGramRequestScreen` (built from the transaction
   sheet's parts): amount, fiat, comment bubble, and a table with the requester (avatar, name,
   @username), the address and the date. Warnings above the button: *Username changed in the last 7
   days*, *You haven't messaged this person before*, *Not in your contacts*. Button: *Send 250 Grams*.
5. **Pay.** (a) fetch the server's record (`wallet.getGramRequest`) and the requester's wallet
   (`wallet.getUserAddresses`), and refuse unless the request is pending and the record, the message
   and the wallet agree (`walletGramRequestPaymentCheck`); (b) Face ID, Touch ID or the device passcode
   (`LAContext.deviceOwnerAuthentication`), on every payment; (c) the regular transfer pipeline
   (`beginWalletFlow` → `prepareTransfer` → `submitTransfer`) to the **record's** address, with
   `gramRequestId`. If the wallet cannot sign on this device, the sheet restores signing as the Send
   screen does, or opens recovery-phrase entry.
6. **Paid.** The server flips the request (`updateGramRequestStatus`); the same card morphs its ribbon
   to green `paid` (`0x00cf00`) with the glint, the card spin and the completion haptics
   (`animateCompletion(fromClock: false)`), and drops the comment. `messageActionGramRequestPaid`
   renders as the service line alone: *You sent Caesar 250 Grams ($312.50)* / *imogen sent you 250
   Grams ($312.50)*.
7. **Other states.** The card's context menu has *Cancel Request* (requester) or *Decline* (payer),
   each confirmed. Declined, cancelled and expired cards are greyed. Expiry is also shown locally once
   `expires_at` passes.
8. **First-time safety sheet.** The first received request opened shows the tips
   (`WalletInfoScreen`, mode `.gramRequestSafety`) with *Turn Off Requests*, which opens the privacy
   setting.

## Privacy setting

*Privacy and Security ▸ Request GRAM* ("Who can request GRAM from me"): Everybody · **Premium users**
(default) · My contacts · Nobody, with Always Allow / Never Allow exceptions. It is
`UpdateSelectiveAccountPrivacySettingsType.gramRequests` / `inputPrivacyKeyGramRequests`, shown by
`SelectivePrivacySettingsController` kind `.gramRequests`. *Premium users* is stored as Nobody with
Premium allowed; a server that has stored nothing for the key returns no rules, which reads as the
default (`SelectivePrivacySettings.gramRequests(apiRules:peers:)`).

The key is fetched only with `requestAccountPrivacySettings(includeGramRequests: true)`, which only
the Privacy screen passes while the feature is on, and in its own request: `combineLatest` over the
other keys fails as a whole, so a server without the new key would otherwise empty the Privacy
screen. Saving it does not use `retryRequest` for the same reason.

## Proposed TL (draft; the server owns the final shape)

```
messageActionGramRequest#2e15c9ee flags:# amount:long requester_address:string request_id:long expires_at:int comment:flags.0?string status:GramRequestStatus = MessageAction;
messageActionGramRequestPaid#12489390 amount:long transaction_id:string request_msg_id:int = MessageAction;
gramRequestStatusPending#f0a42b42 = GramRequestStatus;
gramRequestStatusPaid#6f1238c2 transaction_id:string paid_at:int = GramRequestStatus;
gramRequestStatusDeclined#93c88df6 = GramRequestStatus;
gramRequestStatusCancelled#eeaf210f = GramRequestStatus;
gramRequestStatusExpired#3a0d5cd3 = GramRequestStatus;
updateGramRequestStatus#82d2eb56 peer:Peer msg_id:int status:GramRequestStatus = Update;
inputPrivacyKeyGramRequests#12a2591c = InputPrivacyKey;
privacyKeyGramRequests#4b0c2358 = PrivacyKey;
wallet.gramRequestEligibility#eaaba49c flags:# allowed:flags.0?true reason:flags.1?string remaining_today:int = wallet.GramRequestEligibility;
wallet.gramRequest#1a42257 flags:# username_recently_changed:flags.1?true no_outgoing_messages:flags.2?true request_id:long amount:long requester_address:string expires_at:int status:GramRequestStatus comment:flags.0?string = wallet.GramRequest;
---functions---
wallet.requestGram#27ae9ba3 flags:# peer:InputUser amount:long comment:flags.0?string random_id:long = Updates;
wallet.cancelGramRequest#a241ef7f peer:InputUser msg_id:int = Updates;
wallet.declineGramRequest#5037b3c6 peer:InputUser msg_id:int = Updates;
wallet.getGramRequestEligibility#d911a5d8 peer:InputUser = wallet.GramRequestEligibility;
wallet.getGramRequest#ed750a10 peer:InputUser msg_id:int = wallet.GramRequest;
wallet.payGramRequest#998b1c1c flags:# data_normal:bytes data_gasless:flags.0?bytes user_id:InputUser random_id:long request_id:long = Updates;
```

Differences from the brief's draft, and why:

- **`wallet.payGramRequest` instead of a `request_id` flag on `wallet.sendTransfer`.** A TL id is the
  CRC32 of the definition, so adding a field changes `sendTransfer`'s id. The client would then send an
  id today's server does not know for **every** transfer. The sibling method takes the same signed
  payload and is only used for request payments.
- **`messageActionGramRequestPaid`** for the payment's service line. Adding a field to
  `messageActionGramTransfer` would change its id the same way, and the client would stop parsing the
  transfers the server sends today.
- **`wallet.getGramRequest`**: the server record a payment is built from. Its two `?true` flags carry
  the warnings only the server can know (username history, whether the payer ever wrote). `?true`
  fields do not change an id.
- **`flags:#` on `wallet.requestGram`**: the draft's `comment:flags.0?string` needs it.

Errors the client maps (`GramRequestIneligibilityReason`, `GramRequestError`):
`GRAM_REQUESTS_DISABLED`, `GRAM_REQUEST_USERNAME_REQUIRED`, `GRAM_REQUEST_ACCOUNT_TOO_NEW`,
`GRAM_REQUEST_DAILY_LIMIT`, `GRAM_REQUEST_PENDING_EXISTS`, `GRAM_REQUEST_PRIVACY`,
`GRAM_REQUEST_NO_CONVERSATION`, `GRAM_REQUEST_PAID_MESSAGES`, `GRAM_REQUEST_RESTRICTED`,
`GRAM_REQUEST_AMOUNT_INVALID`; and for one request `GRAM_REQUEST_NOT_FOUND`,
`GRAM_REQUEST_NOT_PENDING`, `GRAM_REQUEST_EXPIRED`, plus `GRAM_REQUEST_MISMATCH` on payment.

## What the server must enforce

All eligibility rules from the brief (conversation with the payer's participation or contact, public
username, account age, spam/scam/fake, daily limit, one pending request per chat, TTL, the payer's
privacy setting, paid messages unless exempt), and:

- bind `requester_address` to the requester's wallet at creation; reject edits and forwards of request
  messages (the client already excludes service messages from both);
- on `payGramRequest`, accept only a transfer to the record's address for the record's amount, once
  (`request_id`, and `random_id` for retries); otherwise reject without broadcasting;
- post `messageActionGramRequestPaid` and `updateGramRequestStatus` to both sides, not a transfer card.

## Client invariants

- **A request payment never creates a local pending transfer card.** `prepareTransferSubmission` skips
  `createPendingTransferMessage` when `gramRequestId` is set; the request card turns paid instead.
- **`PendingTransfer.gramRequestId` is persisted** (`decodeIfPresent`) and copied by every rebuild of a
  pending transfer, so a resumed submission still pays the request.
- **The destination comes from the server record, never from the message.**
- **Request cards never set `isIncomingTransfer`** in the card node: that flag also means "received
  money" (wallet state, the default blue ribbon, the debug toggle). The arrival is gated by its own flag,
  `playsArrivalAnimation` (a received transfer, or a received request that is still pending), and the
  sender-side flight by `isAwaitingTransferFlight`, which a pending request card also accepts. A settled
  request (paid, declined, cancelled, expired) is always drawn at rest.
- **Status updates fold through `TelegramGramRequestStatus.applying(_:)`**: nothing reopens a terminal
  request, a payment is never hidden by a later expiry, decline or cancel.

## Safety tips (in-app sheet and help page)

How to stay safe with Request GRAM

- Always check the @username on the request before you send. Scammers copy names and profile photos,
  but they can't copy a username.
- Only pay people you know. Look back through your chat: is this really them, and did you expect this
  request?
- Ignore requests you didn't expect, even small ones. You can decline them, and nothing is ever sent
  without your Face ID or passkey.
- Holding a lot of GRAM? Turn on paid messages (Stars) so strangers can't request from you, or set
  "Who can request GRAM from me" to My Contacts.
- Telegram will never send you a GRAM request.

## Tests

- `TelegramCore/Tests/GramRequestTests.swift`: the status reducer and local expiry, applying an update
  to stored media, Postbox round trips, Api mapping, the server error mapping.
- `TelegramCore/Tests/GramRequestPrivacyTests.swift`: the privacy key, the four options' encode/decode
  round trip (with exceptions), the Premium default, app-config limits.
- `WalletContext/Tests/WalletGramRequestTests.swift` (in `WalletCardStickersTests`): eligibility reason →
  notice, the client mirror, payer warnings, the payment check.

## Known gaps

- A card laid out before its expiry turns grey on its next layout, not at the second it expires.
- The Composer harness stub (`HarnessSharedAccountContext+Generated.swift`) got the two new members by
  hand. Regenerating it also adds `makeShareScreen`, which master already lacks.

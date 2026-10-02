# CHANGELOG — what was added to the original README's version

The starting point was a working voice-call mod: take the phone, see who is online, call
someone, they get an incoming-call screen, they accept, you both talk over any distance, and
either side hangs up from the phone GUI.

This file lists everything added on top of that, in the order it was done. `README.md`
remains the full documentation.

---

## 1. Calling UI

### Call screens that are not the phone

| Before | After |
|---|---|
| Only the phone GUI existed | `IncomingCallScreen`, `OutgoingCallScreen`, `InCallScreen` |
| The caller saw nothing while ringing | `OutgoingCallScreen` shows the target's name and number with a cancel button |
| Hanging up required opening the phone | HANG UP on the call screens; the phone is no longer required |
| Screen | not styled | All three screens share the phone panel style |

`IncomingCallPacket` and `OutgoingCallPacket` now carry the caller's phone number, so both
screens can display it.

Closing a ringing screen cancels that call, which is handled in `removed()` rather than
`onClose()` so that a programmatic close does not send a duplicate reject.

### Live call duration

`InCallScreen` now shows `M:SS`. A `StringWidget` renders its text once at `init()`, so a
dedicated widget is rewritten every `tick()`; previously the counter was frozen at 0.

### Per-player do not disturb

`isCallable` used to mean only "alive and not a spectator". It now also reads the account's
`dnd` flag, so a player can opt out of incoming calls entirely. Turning it on while the
phone is ringing cancels that call.

The flag lives in the account, so it survives a relog — it was originally in-memory only
and reset on disconnect.

---

## 2. Player list

| Before | After |
|---|---|
| Players without a phone were hidden entirely | Shown as **No phone** with no call button |
| Calling a phone-less player failed silently | The server replies `no_phone` and the caller sees the reason |
| List refreshed every 60 ticks while idle | Refreshed when the server sends a new list |

`PlayerListPacket.Contact` gained a `hasPhone` field. The server-side check in `canCall` is
unchanged and still authoritative, so a stale list or a modified client cannot get through.

---

## 3. Accounts, numbers, contacts, SMS, history

A whole persistence layer was added. Before this the mod had **no** stored data at all.

### New package `data/`

| File | Role |
|---|---|
| `PhoneData` | `SavedData` of the world; holds every account and the number counter |
| `PhoneDataStorage` | resolves it from the **overworld**, so the Nether and End see the same data |
| `PhoneAccount` | one player: number, dnd, ringtone, silent, contacts, messages, history, blocked |
| `PhoneService` | the server-side rules for contacts, SMS, history, ringtones, blocking |
| `PhoneLimits` | server-enforced limits |
| `PhoneRateLimiter` | in-memory throttles, never written to disk |
| `PhoneContact`, `PhoneMessage`, `CallHistoryEntry` | the stored records |
| `CallDirection`, `CallResult` | enums for history |
| `Ringtones` | the ringtone allow-list |

### Phone numbers

Every player gets a number on first use, generated sequentially from `1000001` and never
changed, so contacts and messages keep working across sessions. Numbers are normalised
(`555-0002` and `5550002` address the same player) and displayed as `100-0001`.

The UUID of the Minecraft profile stays the internal identity; the number is the
user-facing address. This is what makes offline SMS and saved contacts possible.

### Saved contacts

A contact is keyed by number, not UUID, so it keeps working while the owner is offline and
after a rename. The picker only offers numbers the server already knows — the address book
is not free text, so nobody can save a contact that does not exist.

### SMS

* Stored for **both** sides, so a message survives the recipient being offline.
* Delivered live via `SmsReceivedPacket` when the recipient is online; nothing is queued in
  memory, an offline recipient simply finds it in their inbox on the next login.
* Server-enforced limits: 256 characters, 10 per minute, 200 stored per player.
* The sender number is **always** taken from the account of `context.player()`, never from
  the packet, so a forged sender is impossible.
* A blocked sender's message is silently dropped and the sender is told nothing, so they
  cannot learn they were blocked.

### Call history

Written in `CallManager.endCall`, one entry per side: `OUTGOING` for the caller, `INCOMING`
for the callee, with a result of answered / missed / rejected / hang up / no phone / busy /
offline / do not disturb. A call that was never accepted is a miss, not a hang up — `Call`
tracks `everConnected` for that. 100 entries kept per player.

A refused call is still recorded on the caller's side, so dialling someone who is offline
leaves a trace.

### Blocking

Per-account block list. A blocked caller is refused exactly like any other invalid call, and
their SMS is dropped — both responses are indistinguishable from ordinary failure on purpose.

---

## 4. GUI rewrite

The single `PhoneScreen` became a tabbed app:

| Screen | Contents |
|---|---|
| `PhoneHomeScreen` | your number, unread count, the tab buttons |
| `DialScreen` | online players with CALL buttons |
| `ContactsScreen` | saved contacts, delete and SMS |
| `AddContactScreen` | contact picker |
| `MessagesScreen` | conversation list with unread badges |
| `ConversationScreen` | one thread, wrapped messages, text box |
| `HistoryScreen` | finished calls |
| `SettingsScreen` | ringtone, do not disturb, sounds |

`PhoneScreen` and the unused `ContactRowWidget` were deleted. `PhonePanelScreen` was added
as the shared base: panel drawing, colours, scrolling, auto-refresh.

The old layout mixed the player list into the home screen, so a CALL button was visible even
when no tab was meant to show one. The list now lives only in the Dial tab.

---

## 5. Bugs found and fixed after the rewrite

These were all reported from actual play, not theoretical.

### Screens did not refresh when data arrived

Every screen built its widgets once at `init()`. Data that arrived later was stored but never
drawn, so messages, call history and settings appeared to update only after closing and
reopening the screen.

Fixed with a revision counter in `PhoneClientState` that is bumped by every server payload;
`PhonePanelScreen.tick()` compares it and calls `rebuildWidgets()`.

### SMS did not arrive while you were already in that conversation

The server sent the conversation refresh only to the sender. Now both sides get their
conversation and both inboxes are re-sent on every successful send.

### The unread badge did not clear

The badge lives in the inbox payload, but the server only re-sent the profile after marking
a conversation read. The inbox is now re-sent too.

### Long messages were cut off

Messages were truncated at 46 characters with an ellipsis. They are now wrapped over several
lines on word boundaries, with a hard cut only for a single word longer than the line width.
The tail of a conversation is shown so the newest message is always visible.

### Scrollable lists

Every list used a hard-coded `Math.min(size, 8)`. On a server with 40 players the dial list
showed the first 8 and the other 32 were unreachable — you could not call them and there was
no indication the list was longer.

`PhonePanelScreen.ScrollingRows` adds mouse-wheel scrolling with a drawn scrollbar. Widgets
are built for the visible window only, so the cost stays flat regardless of list length. The
scroll offset lives in the screen, so an auto-refresh does not reset it to the top. Applied
to the dial, contacts, messages, history and contact-picker lists.

### Input field lost focus and draft text

An auto-refresh rebuilt the widgets and stole focus mid-typing, dropping characters. Focus is
now set only on the first build, and a draft message is kept in a static field.

### Direction markers replaced with colour

Incoming/outgoing were shown as `>` and `<`. These were replaced by colour and alignment:
incoming text is amber and left-aligned, outgoing is teal and right-aligned, muted grey for
metadata. Shared constants live in `PhonePanelScreen`. The `You:` prefix was added briefly
and then removed as redundant.

---

## 6. Ringtones and sounds

### Four selectable ringtones

| Before | After |
|---|---|
| Only three vanilla sounds, not selectable | Four registered `SoundEvent`s, chosen per player |
| — | Classic, Digital, Soft, Vibrate |

`Ringtones` is an allow-list. Only ids on that list are accepted from a packet, so a client
cannot ask the server to play an arbitrary sound path. A missing or unreadable file only
logs a warning and falls back to the vanilla bell.

Ringtones are played through `PlayPhoneSoundPacket` so the client resolves the id against its
own registry; the server decides *when*, the client decides *what is legal*.

### The `.ogg` finding

Minecraft's `sounds.json` has no separate path field: it always appends `.ogg` to the name,
and its only decoder is JOrbis. A WAV file therefore cannot be referenced at all — the log
shows `Unable to load sound phone:ringtone_classic.wav does not exist` no matter how the
entry is written. The resource genuinely has to be Ogg Vorbis.

The four melodies are synthesised, not sampled:

```bash
node tools/generate-ringtones.js   # writes .wav
node tools/fetch-ogg-encoder.js    # one-off, unpacks the encoder without npm
node tools/wav-to-ogg.js           # encodes to .ogg and deletes the .wav
```

`docs/ringtones.md` explains how to replace them or add your own.

### Call-state sounds

`ANVIL_LAND` on call end, `AMETHYST_BLOCK_RESONATE` on SMS received. A silent-mode toggle
mutes all phone sounds.

---

## 7. Tooling

| File | Purpose |
|---|---|
| `tools/generate-ringtones.js` | synthesises the four melodies as WAV |
| `tools/fetch-ogg-encoder.js` | unpacks `vorbis-encoder-js` + `prelude-ls` (npm is not installed here) |
| `tools/wav-to-ogg.js` | WAV → Ogg Vorbis |
| `tools/dump-phone-data.js` | unzips and pretty-prints `phone_data.dat` |
| `docs/ringtones.md` | how to change the melodies |
| `.gitignore` | excludes `tools/.ogg-cache/` and the run/build dirs |

---

## 8. Data storage

Everything persistent is in **one file inside the world save**:

```
<world>/dimensions/minecraft/overworld/data/phone/phone_data.dat
```

gzip-compressed NBT, one entry per account. Deleting the world deletes every number,
conversation and call record; copying the save copies all of it.

Not stored: active calls, voice groups and the rate limiter. Those are in-memory only, so an
unfinished call does not survive a restart and the message throttle resets.

---

## 9. Client / server split

`voicechat` moved from `depends` to `suggests` in `fabric.mod.json`. `VoiceChatManager.api()`
already returned `null` when the mod was absent, so the phone now loads and runs without it —
voice simply does not work. This made the mod testable in a dev environment without the
second mod installed.

---

## 10. Networking added

Sixteen new payloads. Requests and responses are deliberately separate rather than one large
`PhoneStatePacket`, so each screen fetches only what it needs.

**C2S** — `RequestPhoneProfilePacket`, `RequestContactsPacket`, `AddContactPacket`,
`RemoveContactPacket`, `RenameContactPacket`, `RequestSmsInboxPacket`,
`RequestSmsConversationPacket`, `SendSmsPacket`, `MarkConversationReadPacket`,
`RequestCallHistoryPacket`, `SetRingtonePacket`, `ToggleBlockPacket`, `SetSilentModePacket`,
plus `DoNotDisturbPacket`.

**S2C** — `PhoneProfilePacket`, `ContactsPacket`, `SmsInboxPacket`, `SmsConversationPacket`,
`SmsReceivedPacket`, `SmsResultPacket`, `CallHistoryPacket`, `PlayPhoneSoundPacket`,
`OutgoingCallPacket`, `DoNotDisturbStatusPacket`.

The same server-authority rule holds: there is no packet that sets state. Every handler
resolves the acting player from `context.player()` and delegates the decision to
`CallManager` or `PhoneService`.

---

## Not implemented

* multiple phone lines, or accounts separate from the player profile;
* per-country number formats — numbers are generated as `100-0001` sequentially;
* changing or transferring an issued number;
* group calls, video, contact sync between servers;
* reading SMS with the GUI closed beyond the one-line notice.

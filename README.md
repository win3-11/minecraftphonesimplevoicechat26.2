# Phone — Minecraft 26.2 Fabric mod

In-game phone item with voice calls over any distance, built on Simple Voice Chat.

The mod is complete end to end: take the phone, see who is online, call someone, they get
an incoming-call screen, they accept, you both talk over voice regardless of distance, and
either side hangs up from the phone GUI.

## How it works (start here)

### The call flow

```text
Player right-clicks the phone
        ↓
PhoneScreen opens (client only)
        ↓ sends PlayerListRequestPacket
Server collects callable players → replies PlayerListPacket
        ↓
Player presses CALL → CallRequestPacket(targetUuid)
        ↓
CallManager.canCall(...) — all checks server-side
        ↓ creates Call(RINGING)
Server sends IncomingCallPacket to the callee
        ↓
Callee presses ACCEPT → AcceptCallPacket
        ↓
CallManager.acceptCall(...) — re-validates
        ↓
VoiceChatManager.connectPlayers(...) → CallState.CONNECTED
        ↓
Server sends CallStartedPacket to both sides
        ↓
Either side opens the phone and presses HANG UP → HangUpPacket
        ↓
CallManager.endCall(...) → VoiceChatManager.disconnectPlayers(...)
        ↓ CallState.ENDED, Call removed from the map
Server sends CallEndedPacket to both sides
```

### Server-authoritative by design

The client never decides anything. It may only *ask* for an action:

| Client → Server | Meaning | Server decides |
|---|---|---|
| `PlayerListRequestPacket` | "send me the player list" | which players are callable |
| `CallRequestPacket` | "I want to call this UUID" | whether the call may be created |
| `AcceptCallPacket` | "I accept" | whether it is still RINGING |
| `RejectCallPacket` | "I reject" | whether it is still RINGING |
| `HangUpPacket` | "end my call" | whose call it is |

There is deliberately **no packet that sets a call state**. A client cannot send
`state = CONNECTED`; it sends `AcceptCallPacket` and the server validates before flipping
the state. The acting player is always resolved from the network connection
(`context.player()`), never from packet data, so a forged UUID cannot impersonate anyone.

### Validation performed by `CallManager.canCall`

1. `caller` exists (came from the connection).
2. `target` resolves to a real, online player on this server.
3. `caller != target` — no calling yourself.
4. `target` is online (`PlayerList.getPlayer(uuid) != null`).
5. **`caller` owns a phone.**
6. **`target` owns a phone** — someone without a phone cannot be reached.
7. `caller` is not already in a call.
8. `target` is not already in a call.
9. `target` is callable (alive, not a spectator).

If any check fails, no call is created and the client is not told why.

## Owning a phone

A player must carry a `phone:phone` item to use the phone at all. The check lives in
`item/PhoneOwnership` and runs entirely on the server:

```java
player.getInventory().contains(PhoneOwnership::isPhone);
```

The **whole** inventory is searched (hotbar + main + offhand), not only the selected slot,
so putting the phone in a chest does not lose your number.

Where the rule is enforced:

| Situation | Without a phone |
|---|---|
| Opening the phone GUI | impossible — you need the item in hand to use it |
| The list in the GUI | players without a phone are **hidden** from the list |
| Someone calls you | the call is refused; the caller sees "That player has no phone" |
| You accept a call | if you dropped the phone while it rang, the call ends with reason `no_phone` |
| A call is connected | already established; the rule is not re-checked mid-call, and losing the item mid-call does not cut the call |

The last row is a deliberate choice: the phone is needed to *start* a call, not to keep
one alive. If you would rather have the call drop when the item is dropped, that is a
small addition to `endCall`/a tick check — say so and I will add it.

The player list is filtered by this server-side too, so a phone-less player is simply not
callable and cannot be spammed with calls.

### Call states

`CallState` is server-only:

| State | Meaning |
|---|---|
| `RINGING` | call placed, callee has not accepted yet |
| `CONNECTED` | callee accepted, voice connection active |
| `ENDED` | finished; the `Call` is removed from `CallManager` |

`Call` holds `caller`, `target`, `state` and the Simple Voice Chat `voiceGroupId`. It is
deliberately **not** tied to any screen — the GUI is a view, the call is server state.

### Edge cases that are handled

* calling yourself;
* the target leaving between opening the GUI and pressing CALL;
* the caller leaving mid-call;
* either side already busy in another call;
* two players calling the same person at once (the second is rejected as "busy");
* accepting a call that was already cancelled;
* rejecting a call that already ended;
* pressing HANG UP repeatedly (idempotent — the second call finds nothing to do);
* forged UUIDs and packets sent too frequently (every packet re-validates).

### Disconnect handling

`ServerPlayConnectionEvents.DISCONNECT` calls `CallManager.handleDisconnect`:

* **RINGING** → the call is cancelled;
* **CONNECTED** → the voice group is torn down, the other player is notified, the call is
  removed.

`ServerLifecycleEvents.SERVER_STOPPED` clears all calls and voice groups.

## Verified versions

| Component | Version | Source |
|---|---|---|
| Minecraft | 26.2 | fabricmc meta / official `fabric-example-mod` 26.2 branch |
| Fabric Loader | 0.19.5 | meta.fabricmc.net |
| Fabric API | 0.161.0+26.2 | Modrinth |
| Fabric Loom | 1.18-SNAPSHOT (resolved 1.18.2) | maven.fabricmc.net |
| Gradle | 9.8.0 | services.gradle.org |
| Java | 25 (Temurin 25.0.4.1 installed) | Mojang version manifest (`java-runtime-epsilon`, major 25) |
| Simple Voice Chat | fabric-2.6.24+26.2 (`voicechat` mod id) | Modrinth |

### Important 26.x findings (differ from older tutorials)

* **Minecraft 26.x ships unobfuscated.** There are *no* Yarn mappings for 26.x
  (`maven.fabricmc.net/net/fabricmc/yarn` still ends at `1.21.11+build.6`) and Mojang's
  version JSON for 26.2/26.3 no longer publishes a `client_mappings`/`server_mappings`
  download, so `loom.officialMojangMappings()` fails. The official
  `fabric-example-mod` branch `26.2` therefore has **no `mappings` line at all**.
  Loom plugin id there is `net.fabricmc.fabric-loom`.
* `ResourceLocation` → **`net.minecraft.resources.Identifier`** (renamed back),
  factory `Identifier.fromNamespaceAndPath(...)`.
* Client class `MinecraftClient` → **`net.minecraft.client.Minecraft`**;
  `setScreen` → **`setScreenAndShow`**.
* `GuiGraphics` → **`GuiGraphicsExtractor`**; `render(...)` → **`extractRenderState(...)`**;
  `drawString` → **`text(font, component, x, y, color)`**.
* `Button.builder(...).bounds(...)` is unchanged.
* `Registry.register(Registries.ITEM, ...)` no longer compiles — `Registries.ITEM` is a
  `ResourceKey`; use **`BuiltInRegistries.ITEM`**.
* `GameProfile.getName()` → **`GameProfile.name()`** (record accessor, authlib 9.x).
* Fabric networking API 6.3.x: `PayloadTypeRegistry.playC2S()/playS2C()` →
  **`serverboundPlay()` / `clientboundPlay()`**.
* **`ServerPlayNetworking` has no `sendIfPresent`** — use `send(player, payload)`.
  `CallManager` therefore passes the already-resolved `ServerPlayer` objects to its
  `CallNotifier`, so it never needs a global server reference in the networking layer.
* **Fabric allows only one `ServerPlayNetworking` global receiver per payload id**:
  `Duplicate packet registration` throws on the second call.
* **`UseItemCallback` runs on the SERVER thread** — even in singleplayer (the log shows
  `handleUseItem` → `ServerGamePacketListenerImpl`). Opening a GUI from it throws
  `IllegalStateException: Rendersystem called from wrong thread`. Every
  `setScreenAndShow(...)` must be bounced onto the render thread:
  `if (!Minecraft.getInstance().isSameThread()) { Minecraft.getInstance().execute(...); return; }`.
  `Minecraft` extends `BlockableEventLoop`, which provides both `isSameThread()` and
  `execute(Runnable)`.
* **Register BOTH payload types from the common initializer.** On a physical client the
  client also has to decode the serverbound packet, and Fabric allows only one
  registration per packet id — a second call throws
  `IllegalArgumentException: Packet type Type[id=...] is already registered!`.
  So `PhoneNetworking` registers `serverboundPlay()` + `clientboundPlay()` together in
  `PhoneMod#onInitialize`, and the client entrypoint registers no types at all.
* `Item#use` signature: `InteractionResult use(Level, Player, InteractionHand)`.
* **`Screen#isPauseScreen()` returns `true` by default** (it is called from the
  `Minecraft` constructor), so any custom GUI pauses a singleplayer game. Override it to
  `false` for in-game UIs like the phone.
* **`Item.Properties.setId(ResourceKey<Item>)` is mandatory since 26.2.** Constructing an
  `Item` without it throws `NullPointerException: Item id not set`
  (`Item$Properties#itemIdOrThrow` called from `Item#effectiveDescriptionId`).
  Registration therefore needs a `ResourceKey`:
  `Registry.register(BuiltInRegistries.ITEM, ResourceKey.create(Registries.ITEM, id), new PhoneItem(new Item.Properties().setId(key)))`.
* Item translation keys are `item.<namespace>.<path>` (e.g. `item.phone.phone`).
* **26.2 added a mandatory `assets/<ns>/items/<item>.json` layer.** An item model is no
  longer found by convention — the client resolves item → model through this new directory.
  Verified by unpacking `minecraft-client.jar`, where 1538 files live in `assets/minecraft/items/`
  alongside 1272 in `assets/minecraft/models/item/`. Format:

  ```json
  { "model": { "type": "minecraft:model", "model": "phone:item/phone" } }
  ```

  **Missing this file gives the black-and-magenta "missing texture" cube, not an error in
  the log.** This is the single biggest 26.2 gotcha: a correct
  `models/item/<item>.json` + a valid PNG is still not enough. Older 1.21 tutorials never
  mention this directory, which is exactly why it is easy to miss.
* `minecraft:item/generated` still exists and is still the correct parent for a flat
  sprite model, so item models themselves are unchanged.

## Simple Voice Chat API

Verified with `javap` against `voicechat-fabric-2.6.24+26.2.jar` — not guessed, and the
enum values below were re-checked after a mistake was made earlier:

```
de.maxhenkel.voicechat.plugins.impl.VoicechatServerApiImpl.instance() : VoicechatServerApi

VoicechatServerApi:
    Group.Builder groupBuilder();   // setId / setName / setPassword / setPersistent / setHidden / setType
    boolean removeGroup(UUID id);
    Group  getGroup(UUID id);
    VoicechatConnection getConnectionOf(UUID playerId);

Group:
    UUID getId(); String getName(); boolean hasPassword();
    boolean isPersistent(); boolean isHidden(); Group.Type getType();

Group.Type:                       // <- these are the only three constants that exist
    NORMAL
    OPEN
    ISOLATED

VoicechatConnection:
    Group getGroup(); boolean isInGroup(); void setGroup(Group); boolean isConnected();
```

There is **no `FORCE_ALL`**. An earlier version of this file claimed there was; it does
not exist in the API and would not compile.

### Why `Group.Type.OPEN`

A call creates one group per `Call`, then puts both players into it:

```java
api.groupBuilder()
        .setId(groupId)                  // random UUID
        .setName("phone-call-" + groupId)
        .setPassword("phone-internal")
        .setPersistent(false)            // removed on hang up, not saved to disk
        .setHidden(true)                 // not listed in the player's group UI
        .setType(Group.Type.OPEN)        // <- open, not isolated
        .build();

api.getConnectionOf(callerUuid).setGroup(group);
api.getConnectionOf(calleeUuid).setGroup(group);
```

`OPEN` gives exactly the phone behaviour that is wanted:

* the two participants hear each other at **any distance** (group chat, no distance
  attenuation for members);
* players standing nearby can also hear a member speaking, like ordinary proximity voice;
* the group is still hidden and password-protected, so nobody joins someone else's call by
  accident.

`ISOLATED` would mute the participants from the surrounding world; `NORMAL` would apply
distance attenuation. `OPEN` is the right choice for a phone.

On hang up: `setGroup(null)` for both players, then `api.removeGroup(groupId)`.

All of this lives in `voice/VoiceChatManager` — `CallManager` knows nothing about Simple
Voice Chat beyond asking it to connect or disconnect two players.

One caveat: the public `Group` interface has no usable "is anybody still in this group"
check (`getPlayers()` returns `Player` handles, not UUIDs), so membership is tracked inside
`VoiceChatManager` to know when a group can be deleted.

## Sounds

All three sounds are vanilla Minecraft sounds, chosen so that nothing has to be registered
and no `sounds.json` is needed. They live in one file, `phone/PhoneSounds`:

| Event | Sound | Who hears it |
|---|---|---|
| Incoming call (RINGING) | `SoundEvents.BELL_BLOCK` | only the player being called |
| Call accepted (CONNECTED) | `SoundEvents.EXPERIENCE_ORB_PICKUP` | both players |
| Call ended (any reason) | `SoundEvents.BREWING_STAND_BREW` | both players |

They are played with `ServerPlayer.playSound(SoundEvent, float, float)`, which is a
**per-player** call — so a ringing phone is heard by the callee wherever they are standing,
not only nearby. That is the whole point of a phone.

To use custom sounds later, register a `SoundEvent`, add a `sounds.json`, and change only
`PhoneSounds`.

### 26.2 sound API gotcha

Two things that cost time here, both verified with `javap`:

* `Player.playSound` has **no** overload taking a `SoundSource`. The old
  `playSound(sound, SoundSource.PLAYERS, volume, pitch)` does not compile. Use the
  three-argument `playSound(sound, volume, pitch)`.
* **Not every `SoundEvents` constant is a `SoundEvent`.** Some are
  `Holder<SoundEvent>` — e.g. `SoundEvents.NOTE_BLOCK_BELL` is a `Holder`, while
  `SoundEvents.BELL_BLOCK` is a plain `SoundEvent`. Passing a `Holder` where a `SoundEvent`
  is required fails to compile. When picking a sound, check its declared type.

## Project layout

```
build.gradle
gradle.properties
settings.gradle
libs/voicechat-api.jar              Simple Voice Chat 2.6.24+26.2 (compileOnly)
src/main/java/com/example/phone/
    PhoneMod.java                    common entrypoint + disconnect hook
    PhoneIds.java                    Identifier constants
    item/PhoneItems.java             item registry
    item/PhoneItem.java              item behaviour only
    item/PhoneOwnership.java         "does this player own a phone" check
    phone/PhoneSounds.java           all sound effects
    call/CallState.java              RINGING / CONNECTED / ENDED
    call/Call.java                   caller, target, state, voice group id
    call/CallManager.java            the only place where call state is decided
    voice/VoiceChatManager.java      the only place that talks to Simple Voice Chat
    network/PhoneNetworking.java     payload types + all receivers + notifications
    network/PlayerListPacket.java    S2C
    network/PlayerListRequestPacket.java  C2S
    network/CallRequestPacket.java   C2S  "call this player"
    network/AcceptCallPacket.java    C2S  "I accept"
    network/RejectCallPacket.java    C2S  "I reject"
    network/HangUpPacket.java        C2S  "end my call"
    network/IncomingCallPacket.java  S2C  somebody is calling
    network/CallStartedPacket.java   S2C  call is CONNECTED
    network/CallEndedPacket.java     S2C  call is over (both parties always)
src/client/java/com/example/phone/screen/
    PhoneModClient.java              client entrypoint, GUI opening, receivers, call state
    PhoneScreen.java                 phone GUI (dial list, or compact in-call panel)
    IncomingCallScreen.java          ACCEPT / REJECT
    InCallScreen.java                in-call screen (closed by default; call shows in the phone)
    ContactRowWidget.java            legacy row widget, no longer used by PhoneScreen
src/main/resources/
    fabric.mod.json
    assets/phone/items/phone.json           <- 26.2 item -> model link (do NOT forget)
    assets/phone/models/item/phone.json      <- model, parent minecraft:item/generated
    assets/phone/textures/item/phone.png    <- 16x16 sprite
    assets/phone/lang/en_us.json
    assets/phone/lang/ru_ru.json
```

`loom.splitEnvironmentSourceSets()` is used, so all client-only code lives in `src/client`
and the common/server code in `src/main` — this is what keeps `PhoneItem` and
`PhoneNetworking` free of client references and safe on a dedicated server.

## How the phone GUI works

The phone has two layouts, chosen by call state:

**Idle** — dial list:

```text
ТЕЛЕФОН
──────────────
Кому хотите позвонить?

Alex        [ЗВОНОК]
Steve       [ЗВОНОК]

Других игроков нет

[ЗАКРЫТЬ]
```

**In call** — compact panel, the player list and the prompt are hidden:

```text
ТЕЛЕФОН
──────────────
Идёт разговор

Собеседник: Alex   2:35

[СБРОСИТЬ]
[ЗАКРЫТЬ]
```

Notes on the implementation:

* All text is drawn with `StringWidget` and all buttons with `Button` — the vanilla widget
  pipeline. An earlier version drew text manually in `extractRenderState(...)` and rows via a
  custom `AbstractWidget`; on 26.2 the text simply did not appear, which is why the GUI looked
  like it had only buttons.
* Accepting a call does **not** open an overlay. The player stays in the game and opens the
  phone to see who they are talking to, for how long, and to hang up.
* The elapsed time is counted on the client from the moment `CallStartedPacket` arrives, and
  is displayed as `M:SS`.
* The player list is not refreshed while a call is active (no point, and it saves traffic).

## Localisation

All user-facing text goes through `Component.translatable("phone.gui.…")` and lives in
`assets/phone/lang/en_us.json` and `ru_ru.json`. Player names are the one exception — they
come from the server and are inserted as `Component.literal(...)`.

The phone item's own name uses `item.phone.phone`.

## How the player list works

1. Client opens `PhoneScreen` (from `UseItemCallback` when the held item is `PhoneItem`).
2. `PhoneScreen` sends `PlayerListRequestPacket` once on open, and then every 60 ticks while
   idle.
3. The server runs `collectCallablePlayers(viewer)`, which walks
   `viewer.level().getServer().getPlayerList().getPlayers()` and skips:
   * the viewer themselves,
   * players already in a call (busy — hidden, as chosen for the MVP),
   * players who are dead or spectating,
   * players who do not have a phone.

   Offline players are never in that list, so they can never appear.
4. The server replies with `PlayerListPacket` containing `List<Contact(uuid, name)>`.
5. The client renders exactly what it received — it performs no availability checks of its
   own, and it does not re-request inside `init()` (that would loop, because
   `rebuildWidgets()` calls `init()`).

## Build

Gradle is vendored in `.tools/gradle-9.8.0` (the wrapper jar could not be generated
because the `wrapper` task itself triggers Minecraft setup).

```powershell
.\.tools\gradle-9.8.0\bin\gradle.bat build
```

Output: `build/libs/phone-0.1.0.jar`.

## To test

Give yourself the item:

```
/give @s phone:phone
```

Expected log output for a successful call between two players:

```
[phone] call started: Hera23 -> Bob
[phone] call accepted, now CONNECTED
[phone] voice group <uuid> created for 2 participants
```

and on hang up or disconnect:

```
[phone] player disconnected during CONNECTED
[phone] cleaned up voice group <uuid> after disconnect
```

If you see `voice connection failed, call stays UI-only`, Simple Voice Chat was loaded
but had no connection for one of the players — usually means voice chat itself is not
connected yet (check the voice chat screen / microphone first).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Black-and-magenta cube in hand | missing `assets/phone/items/phone.json` | add it (26.2 requires this file; no error is logged) |
| GUI shows only buttons, no text | text drawn manually in `extractRenderState` | use `StringWidget` / `Button` |
| `Rendersystem called from wrong thread` on opening the phone | `UseItemCallback` runs on the server thread | bounce to `Minecraft.getInstance().execute(...)` |
| `Packet type ... is already registered!` | payload types registered in both entrypoints | register only in `PhoneMod#onInitialize` |
| `NullPointerException: Item id not set` | `Item.Properties` without `setId` | pass `ResourceKey.create(Registries.ITEM, id)` |
| `Duplicate packet registration` | two global receivers for one payload id | one receiver per id |
| Singleplayer pauses when the phone opens | `isPauseScreen()` defaults to `true` | override to `false` |
| No voice at all | Simple Voice Chat mod not installed / not connected | install the mod, check the microphone |
| A player is missing from the list | they have no phone, are busy, dead, or spectating | this is intended |
| `Holder<SoundEvent> cannot be converted to SoundEvent` | picked a `SoundEvents` constant that is a `Holder` | use a plain `SoundEvent` constant, e.g. `BELL_BLOCK` not `NOTE_BLOCK_BELL` |
| Changing the .jar changes nothing | stale assets in the client cache | fully close Minecraft, then restart |

## Current state / not implemented

Deliberately out of scope for the MVP — do not assume they exist:

* SMS / text messages, contacts, call history, phone numbers, accounts;
* ringtone / sound effects beyond the three vanilla ones;
* a reject/hang-up button outside the phone GUI (hanging up requires opening the phone);
* any GUI on the caller's side while ringing (the caller sees nothing until the callee
  accepts, then must open the phone to see the call);
* a "player has no phone" message in the caller's list — such players are simply hidden,
  and the message only appears if a call was attempted anyway;
* per-player "do not disturb" setting — `isCallable` currently just means alive, not a
  spectator, and owning a phone.
# Relay

Relay is a Blueprint API for Voices of the Void mods. It gives BP mods features
that normally require native C++:

- observe, change, or cancel Blueprint function calls;
- register shop items, keybinds, calendar events, and dreams (WIP);
- connect game instances over UDP and exchange typed messages;
- replicate mod actors and selected vanilla classes without editing them.

Relay began as a small function watcher for an older mod. It now provides
function hooks, runtime game integration, and a multiplayer transport/replication layer.

Relay calls a function hook a **Watch**. A Watch targets one class and function
and returns an object with bindable events. A Pre Watch can change supported
input parameters; a cancelable Watch can stop the call.

Networking is now the largest part of Relay. It provides low-latency UDP
transport, typed messages, and a replication model inspired mainly by Unreal
Engine 4.27, with ideas from Source, Doom 3, and Tribes.

Relay's goal is to let Blueprint authors create networked mods without relying
on WebSockets over TCP or locking multiplayer into one closed ecosystem. Relay
is designed to coexist with UE4SS and other Relay mods, allowing mods to
communicate, share services, and build on one another.

**Relay is not a co-op mod. It is a networking backend that other mods can use
to build co-op or other multiplayer modes. Installing Relay by itself adds no
host/join flow or replication rules.**
<p align="center" width="100%">
  <a href="https://blueprintue.com/blueprint/g3s09x9c/">Relay README Blueprint</a>
</p>

<p align="center" width="100%">
<a href="https://discord.gg/Bq7HCMRfjk"><img width="5%" src="https://www.dropbox.com/scl/fi/96uoyd529gq617880m0cu/636e0a6a49cf127bf92de1e2_icon_clyde_blurple_RGB.png?rlkey=343xgtya1h3r53bblx8lns473&st=ih0q2alh&dl=1"></a>
</p>

## Public test notice

This public test is intended for mod authors. It includes example assets, but it
is not a finished multiplayer conversion of VotV.

Before testing multiplayer:

- back up every save involved;
- use the same Relay DLL, `Relay.pak`, gameplay mod, and rule profile on every
  machine;
- keep the UE4SS log from both machines when reporting a problem.

Relay does not provide a public lobby or a ready-to-play multiplayer mode. The
release includes `RelayExampleUI` in cooked and editable form. Its basic
Host/Join UI and rules are an integration example, not a complete gameplay mod.

To try it, rename `RelayExampleUI.pak.example` to `RelayExampleUI.pak` inside:

`%AppData%\r2modmanPlus-local\VotV\profiles\<profile>\shimloader\pak\Moddy-Relay`

Hosting asks for permission the first time: press Host, allow it in the
Windows prompt, then press Host again. `allow_listen=1` in `relay_net.ini`
does the same thing without a prompt.

Raw, uncooked assets for mod authors are in the release's `editor` folder and
on GitHub.

<details>
<summary>Click here to see how Relay protects network traffic</summary>

Relay still uses UDP, but Relay gameplay and voice payloads are not sent as
plain text. It uses the official libsodium library and the same security
handshake for direct, punched, and relayed connections:

- the host signs the handshake with a persistent Ed25519 identity;
- each connection gets fresh X25519 session keys. Relay encrypts and
  authenticates every later datagram with XChaCha20-Poly1305, and refuses
  tampered, replayed, or incorrectly tagged packets;
- a connect or friend code pins the host identity. Relay remembers identities
  for later direct connections and refuses an unexpected replacement;
- an invite password uses Argon2id and challenge-bound proofs. Relay never
  sends the plaintext password;
- a stateless source cookie is checked before expensive handshake work.
  Per-source connection rates, per-source and host-wide pending-peer limits,
  an ENet duplicate-peer cap, and handshake deadlines limit resource use before
  authentication;
- packet sizes, message lengths, queues, and reliable backlogs have hard
  limits. Decoders reject truncated, oversized, malformed, or trailing data
  before it reaches Blueprint calls;
- encryption does not grant gameplay authority. The host identifies the peer
  behind each connection, validates replicated actions, and denies remote
  authority leases unless that peer proved a non-empty invite or the host
  explicitly enabled `allow_untrusted_leases=1`;
- relay and rendezvous services do not receive session keys and cannot
  impersonate a pinned host. They can route, drop, or misdirect traffic, but
  misdirection ends in a refused handshake;
- Relay creates the host identity seed with a protected Windows ACL. Only the
  owner, SYSTEM, and Administrators receive access.

</details>

## Install

A player needs both parts of the same Relay build. Install through r2modman, or
place them manually:

1. RelayCpp at `Mods/Moddy-Relay/dlls/main.dll`.
2. `Relay.pak` at `Content/Paks/LogicMods/Relay.pak` or r2modman's shimmed pak
   path.

A mod using Relay should declare Relay as a dependency. Authors work against the
assets under `/Game/Mods/Relay/`.

`Relay_Version` returns `0` when the DLL is absent. The current Blueprint API
returns `3`. `Relay_NetVersion` returns `0` when networking is unavailable and
`19` for this build. Check both before enabling Relay-dependent UI or gameplay.

The DLL and pak form one API contract. If one is stale, nodes, properties, or
struct fields may be missing. Replace both before debugging the gameplay mod.

## Player quick start

### Host

1. Start hosting in a gameplay mod. The first try gets refused, and a
   Windows prompt asks if this PC can host. 
2. Pick **Allow hosting**, then press Host again. Relay saves this choice as
   `allow_listen=1` in `<profile>/shimloader/cfg/relay_net.ini` (it creates the
   file if needed) and won't ask again. **Allow once** works the same way but
   only until you close the game. Set `allow_listen=0` to turn it off again.
3. Open the session's UDP port on the host's firewall. Hosting over the
   Internet also needs that port forwarded on the router. If the mod uses LAN
   discovery, also open UDP `Port + 1` — e.g., session port `7777` needs
   `7778` open for LAN search. LAN discovery isn't used for Internet games.
4. Share the friend code (or a reachable `host:port`) and the password, if any.

Editing the INI is optional. `allow_listen=1` in the ini to skip the prompt. Relay owns the prompt, while gameplay mods sets the port, password, peer limit,
bind/public addresses, and authority rules via Blueprint nodes.

A shared password lets a peer connect. Without a password, a peer can still
join but can't request temporary authority over lease-backed objects. Setting
`AllowPasswordlessAuthority=true` removes that limit for everyone; only use
this for a trusted private session.

### Join

Joining requires no network INI and never opens a listening socket.

1. Install the same Relay and gameplay-mod versions as the host.
2. Paste the host's friend code into the mod's Address field. A friend code is
   accepted directly by `Relay_NetConnect`. For a direct connection, enter the
   host's reachable hostname/IP and session port instead.
3. Enter the same password, if the host used one.
4. Wait for the gameplay mod to load or fetch the host's save. A connected
   socket is not enough: replication stays closed until both machines report
   the same world.

A friend code pins the host's persistent identity on the first connection. It
is not a password and contains no invite secret.

The gameplay mod owns this UI, save handling, and travel.
`RelayExampleUI` shows the basic Blueprint node flow.

<details>
<summary>Click here for explaination about direct and rendezvous friend codes</summary>

A direct friend code contains a reachable host address, port, and persistent
host identity. Relay cannot infer a public address when the host listens on all
interfaces. The mod should pass a public hostname/IP through
`Relay_NetHost.PublicAddress` or `Relay_NetFriendCode.PublicAddress`.

An operator may set `rendezvous_address=host:port` in `relay_net.ini`. After the
host registers successfully, `Relay_NetFriendCode` prefers the rendezvous code.
The client only needs that code; it does not need a rendezvous INI setting. This
path is experimental and requires a compatible service supplied by the mod
distributor or server operator.

`RelayCpp-Services.zip` is an optional operator download, not part of the player
installation. Run its rendezvous and relay executables together when fallback
transport is required. The included operator guide covers deployment and
privacy.

</details>

## Mod author: session flow

### Host

```text
Relay_NetHost(
    Port=7777,
    BindAddress="",
    PublicAddress="",
    MaxPeers=8,
    Channels=8,
    Password="",
    AllowPasswordlessAuthority=false)
    -> Net, Reason
```

- `BindAddress`: interface to listen on. Blank means all interfaces and is the
  normal choice. `127.0.0.1` is useful only for two instances on one machine.
- `PublicAddress`: hostname/IP encoded into the direct friend code. It does not
  affect the listening socket.
- `MaxPeers`: requested session limit, capped by `relay_net.ini`.
- `Channels`: Relay needs eight. Request more only if your mod deliberately uses
  additional ENet channels.
- `Password`: invite secret; never stored in the INI or sent as plaintext.
- `AllowPasswordlessAuthority`: lets passwordless peers request leases. Leave it
  off for public or discoverable sessions.
- `Reason`: synchronous host refusal, such as missing listen permission, an
  invalid option, or a port that could not bind.

`Relay_NetHost` returns null when it fails right away. Always display `Reason`
instead of a custom message. If Windows prompts for hosting permission on the
first attempt, approve it and retry.

### Connect

```text
Relay_NetConnect(Address, Port=7777, Channels=8, Password="") -> Net
```

`Address` accepts a hostname, an IP address, or a complete friend code. A
connect is asynchronous: a returned handle means the attempt started, not that
the handshake succeeded. Bind the handle's `OnConnect`, `OnDisconnect`, and
`OnError` dispatchers.

### Kick a peer

```text
Relay_NetKickPeer(Peer) -> bool
```

This node is host-only. Pass a live `obj_relayPeer` from the current session's
`Peers` array. `true` means Relay queued an orderly disconnect. It returns
`false` when the caller is not the host, the peer is null, stale, or from
another session, or no transport is active. Normal disconnect events still
fire. This is a disconnect, not a ban; the peer may connect again.

### Recover the process session after a map load

Host and client sessions are process-scoped. They survive map travel and the
destruction of the ModActor that started the connection.

Use:

```text
Relay_NetSession -> Net, IsRunning, IsHost, LocalPeerId
```

Call it from the new world's ModActor to recover the current session handle. Do
not call Host or Connect again just to recover that handle. The host's
`LocalPeerId` is `0`; a client's is `-1` until it receives Welcome.

Repeating Host or Connect with the exact settings for the same live endpoint
returns that endpoint instead of opening another one. `Relay_NetSession` is the
direct way to read the session that already exists.

### Get a code for UI

```text
Relay_NetFriendCode(PublicAddress="") -> Code, Reason
```

- With a configured rendezvous service and a blank input, it returns the
  rendezvous code.
- Otherwise it returns the direct code created by `Relay_NetHost`.
- Supplying `PublicAddress` creates a direct code for that address.
- It refuses wildcard addresses and explains the missing prerequisite in
  `Reason`.

### Session events and state

Bind these on `obj_relayNet`:

| Dispatcher | Meaning |
|---|---|
| `OnConnect` | This machine completed session setup. A host receives it on the frame after Host returns, allowing Blueprint time to bind. |
| `OnDisconnect` | This machine's session ended or was refused. |
| `OnPeerConnected` / `OnPeerDisconnected` | Another peer joined or left. |
| `OnPeerStatus` | Another peer paused or resumed its world. |
| `OnWorldStatusChanged` | World identity or replication readiness changed. |
| `OnMessage` / `OnBytes` | An application payload arrived. |
| `OnServerFound` | One LAN discovery response arrived. |
| `OnNetLease` | A lease grant, denial, release, or expiry was applied. |
| `OnError` | An asynchronous network error occurred. |

The handle also maintains `IsRunning`, `IsHost`, `LocalPeerId`, `Peers`, and
basic byte/ping statistics.

`OnConnect` means the transport is ready. It does not mean actors may replicate.
Gate multiplayer gameplay on `OnWorldStatusChanged.ReplicationOpen`, or query:

```text
Relay_NetWorldStatus
    -> LocalKnown, HostKnown, IdentityMatches, ReplicationOpen
    -> LocalMap, LocalSaveId, HostMap, HostSaveId, Reason
```

Show `Reason` when the fence is closed. Do not hide a save mismatch behind a
generic "connecting" message.

### World/save identity

The world fence always compares the map. A gameplay mod that knows the save
should also provide its stable identity:

- `Relay_NetSetWorldIdentity(SaveId)` when the mod already has the value.
- `Relay_NetSetWorldIdentitySource(Holder, PropertyName)` when Relay should
  re-read a property.

Both sources are cleared on map load; set them from each world's ModActor. For
VotV, the relevant game-instance property is commonly `slotName`.

`Relay_NetHostWorld` returns the host's announced map/save. Use
`Relay_NetPublishSave` and `Relay_NetRequestSave` only as the transfer plane;
the gameplay mod still owns save consent, naming, loading, and travel.

## Built-in VotV adapters

<details>
<summary>Click here to read about VotV systems Relay handles automatically</summary>

Relay handles these VotV-specific systems in native C++. Gameplay mods should
use these adapters instead of building a second multiplayer path:

- **Physics grab:** Relay watches VotV's local grab input and sends authenticated
  begin, movement, rotation, release, and throw input to the host. The host runs
  the accepted light/heavy grab against the correct player and body, then normal
  `$Move` replication sends the result. This also covers swinger behavior and
  registered non-prop physics targets such as the ATV. Do not add a second grab
  RPC or request a target lease just for grabbing. Optional observation is
  available through `OnGrabChanged`, `Relay_NetGrabState`, and
  `Relay_NetGrabbers`. This adapter is still public-test behavior and may need
  fixes.
- **Hook prop and cables:** Relay handles `prop_hook_C` launch, `hook_C` flight,
  the first and second attachment, dynamic or static endpoints, disconnect
  cleanup, and recovery into inventory. The host owns the cable topology; each
  endpoint's existing authority decides where its physics response runs. Do not
  replicate `hook_C` as an ordinary spawned actor or add a second hook RPC.
  Cable-length scroll input is not implemented yet.
- **Hold, drop, and item identity:** VotV often destroys one prop actor and
  creates another when an item is held, stored, equipped, or dropped. Relay
  hooks those transitions and keeps the same logical item identity across the
  short actor gap. Do not manually mint a new NetId for each replacement.
- **Per-player inventory and equipment:** The host keeps a separate inventory
  and equipment partition for each authenticated player. Relay pages container
  rows to the client UI and routes supported container transfers, world pickup,
  hold, drop, place, equip, and battery actions through the host. The client's
  inventory rows are a mirror, not the canonical copy. Do not replicate raw
  `GObjStack`, equipment, or hold arrays with a second Blueprint system.
- **Player save partition and profile:** A returning player is bound to the same
  host/world profile even when their session PeerId changes. The profile covers
  the supported vitals, inventory, equipment, held item, effects, and player
  transform. When loading the host's save on a client, use the included
  `VotV_LoadSaveAndTravel` flow so the native partition reset/overlay runs
  before travel. The gameplay mod still owns save consent, slot naming, and
  when to load or travel. Whole-profile restore remains public-test behavior.

</details>

These adapters do not make ordinary VotV actors replicable by themselves. Mod
authors must still declare the same rules on every peer, include one per-peer
player presentation, use `Transform` or `TransformAttach` for physical actors,
and wait for `ReplicationOpen`. An explicit `Player` role is optional when
exactly one per-peer VotV player presentation exists; register the role to
remove ambiguity or to use that player as a routed-function identity.

## Replication model

For every actor, answer four separate questions:

| Question | Contract |
|---|---|
| How is it discovered? | `comp_relayNet` on a mod actor, or `Relay_NetBindClass` for a class you do not own. |
| How is the same object identified elsewhere? | Stable `NetKey`, per-peer key, minted runtime id, or explicit `Relay_NetBind`. |
| Who may write canonical state? | Declared Authority plus any current lease. |
| Does the other machine already have it? | Presence: bind an existing actor, spawn one, or bind-then-spawn. |

### Mod-owned actor

Add `comp_relayNet` and declare its `Fields`, `Events`, authority, identity,
presence, and optional remote class. Relay registers the actor when the
component starts.

### Vanilla or third-party class

Build a `struct_relayNetRuleDef` and call:

```text
Relay_NetBindClass(Def, Owner=self) -> Rule
```

The Rule handle exposes `OnActorBound` and `OnActorUnbound`. Use those events
instead of rescanning the world. `Relay_NetRuleBoundActors` is the current
snapshot for UI or late initialization.

Large rule sets can live in `shimloader/cfg/relay_rules.json`; set
`rules_profile=relay_rules.json` in the public INI. The file is read only when
that key names it, and its rows outrank Blueprint rows for the same class. Every peer must use the same
rule contract.

When a ModActor already has the full JSON document as a string, call
`Relay_NetLoadRulesJson(RulesJson)`. It accepts the same `relay.rules.v1` schema
as the file profile and adds the rules to the same pending rule set. Invalid
JSON or an invalid top-level document returns `false` and installs nothing. A
valid document returns `true`; malformed rows inside it are logged and skipped.
Loading the same semantic document again in one world is a successful no-op.
Rules are cleared on map travel, so each new ModActor must still declare them
again for the new world.

JSON defaults and Blueprint rules may coexist. Precedence applies to the
**whole class rule**, regardless of load order:

| Same-class declarations | Effective rule |
|---|---|
| Identical | Shared; no duplicate replication |
| JSON + differing `Relay_NetBindClass` | Blueprint replaces JSON |
| Different, same priority | Existing rule stays; incoming rule conflicts |

A Blueprint override replaces the entire JSON rule; it does not inherit omitted
fields, routed functions, or quiesce settings. This applies to JSON loaded from
a file or a ModActor string. Load the same effective rules on every peer.

Overrides are informational and counted separately from invalid rules and
conflicts. Conflict diagnostics show both sources and their differences. Calls
from one ModActor can conflict just like calls from different mods. Blueprint
sources identify the caller object/function and, when available, its bytecode
offset; file sources identify the JSON path.

With `rules_dump=1`, `relay_rules_dump.json` includes the effective rules,
their `declaration_sources`, and `decisions` for overrides and conflicts. Each
decision records the rejected declaration and its differences; `source` shows
the winning priority when holders are mixed. Dump metadata is informational:
reloading a dump creates JSON baselines, and editing metadata does not create
another holder or grant Blueprint priority.

### Classes that must stay local

Some actors are *subjective*. They're spawned from local per-player state, so
every machine already makes its own. VotV's hunger hallucinations are a good
example: the gamemode checks **this** machine's food level and spawns a
`prop_burgerHallucinate_C` next to **this** player. If you replicate one, the
other player sees a prop that shouldn't exist for them, and it wastes an
ObjectId.

A class rule also matches subclasses, so a broad rule (like one on `prop_C`)
will catch these too. Exclude them next to your `Relay_NetBindClass` call, in
either order:

```text
Relay_NetExcludeClass(Class, bIncludeChildren=true) -> Success
```

Or do the same thing in the public INI, with no asset work needed:

```text
exclude_classes=/Game/objects/prop_burgerHallucinate.prop_burgerHallucinate_C
exclude_class_trees=/Game/objects/eyer
```

`exclude_classes` names one class. `exclude_class_trees` names a class and all
its subclasses. Both accept a full object path or just the package name
(`/Game/objects/eyer` matches `eyer_C`), and can be comma- or
semicolon-separated. They're case-insensitive.

A few things to know: exclusion only overrides rules inherited from a **base**
class - a rule written for the excluded class itself still binds it. That's
also how you opt a class back in, since there's no separate un-exclude call.
Rules reset on map travel, so re-declare exclusions after each trip. An
excluded actor never registers, announces, or gets a NetId. And exclusion is a
local-only decision: if one peer excludes a class, that peer just won't see it
- it has no effect on other peers.

### Fields

A field name can be a reflected property such as `Health`, an object-relative
path such as `Component.Value`, or one of Relay's predefined pseudo-fields.
Pseudo-field names begin with `$`, are case-sensitive, and do not require a
matching Blueprint property.

<details>
<summary>Built-in replication pseudo-fields</summary>

| Pseudo-field | Value |
|---|---|
| `$Location`, `$Rotation`, `$Scale` | The actor's world transform, read and applied through engine actor functions. |
| `$Velocity` | The actor's current velocity. Prefer `$Move` when replicating movement. |
| `$Move` | World location/rotation, physics state, and linear/angular velocity. |
| `$Attach` | Parent actor, component/socket, and relative transform. A null parent means detach. |
| `$ControlRotation` | The source pawn controller's aim rotation. |
| `$CameraLocation`, `$CameraRotation` | The source player's camera pose. |

The `Transform` preset adds `$Move`. `TransformAttach` adds `$Attach` and
`$Move`, so held or attached actors follow the same parent remotely. Read
remote controller/camera values with `Relay_NetControlRotation`,
`Relay_NetCameraLocation`, and `Relay_NetCameraRotation`; Relay does not write
them into a proxy controller.

</details>

Each field row also controls when and how often Relay sends it, wire precision,
optional smoothing, and an optional zero-argument OnRep callback.

Field order is part of the wire schema. Append new fields. Do not insert,
remove, or reorder rows between compatible builds.

Whole reflected structs and arrays are supported when both sides have the same
shape. Actor/object pointers are not portable across processes; send a stable
id/key and resolve it locally.

### Identity

- Use a stable `NetKey` when both machines already create the same logical
  object.
- Use `bPerPeer` for exactly one object per peer, such as a remote-player proxy
  or name badge.
- Leave the key blank for a runtime actor spawned on one authority; Relay mints
  an id and the spawn record carries it.
- Use `Relay_NetRegisterActor` when you assign a key at runtime.
- Use `Relay_NetBind` only when an id is already exchanged out of band.

Do not give independently spawned objects the same key while declaring each
side as Owner authority. Both machines will believe they own one object and
Relay will reject the conflict.

### Presence and auto-spawn

Auto-spawn is on by default for eligible mod actors. A remote class must be
known locally and allowed by the receiver's own component/rule contract.
Derived world identities are never spawned as duplicates. Rate, total, queue,
and per-frame work limits are enforced internally.

Use `RemotePresence=BindExisting` for actors the world already creates,
`SpawnProxy` when only a presentation proxy should exist remotely, and
`BindOrSpawn` when a save actor may or may not exist on the receiver.

### Authority and leases

Use `Relay_NetIsAuthority(Actor)` before writing replicated state.
`Relay_NetAuthorityFor` answers who owns it and whether the current ownership is
a lease.

For an interaction that may wait on the host, prefer:

```text
Relay_NetRequestAuthorityAsync(Actor) -> Request
Bind Request.OnGranted / Request.OnDenied
```

Always release a lease when the interaction ends:

```text
Relay_NetReleaseAuthority(Actor)
```

The older `Relay_NetRequestAuthority` returns only whether a request was sent;
it is not a grant result.

### Events and routed functions

A replicated field is durable state. An event is a one-time action.

Give every declared Event or RoutedFunction one clear direction: client to
host/authority, authority to one peer, or authority to all peers. Replicate
durable values such as health, position, or an open/closed flag as fields.
Then use OnRep to update local effects or UI when a value changes. Do not send
durable state only as an event: late joiners and resync cannot recover its
current value.

### Arbitrary messages

For chat or mod-specific data not owned by an actor:

```text
Relay_NewParams -> AddString/AddInt/... -> Net.Broadcast
Net.OnMessage -> GetString/GetInt/...
```

Channels `0` through `7` are allocated to Relay planes. If a mod deliberately
uses a higher channel, request enough Channels on Host and Connect on every
peer. Prefer Relay's typed actor/event surfaces for gameplay.

### Diagnostics

Call `Relay_Debug` after WorldReady. It returns stable `RLY-*` issue strings and
error/warning/info counts. Also useful:

- `Relay_NetMarkDirty(Actor, PropertyName)` forces the next authoritative field
  flush; blank marks all fields.
- `Relay_NetRequestResync(Actor)` asks for bounded targeted recovery of one
  remote-owned actor. This is diagnostics/recovery, not normal gameplay.
- `Relay_NetIdFor`, `Relay_NetFindByKey`, `Relay_NetPeer`, and
  `Relay_NetRuleBoundActors` expose current state without log scraping.

## Watch and integration API

### Observe a Blueprint call

1. Call `Relay_Watch(Class, Function, Phase=Post, Owner=self, bAllowTick=false)`.
2. Bind the returned Sub's `OnFired` dispatcher.
3. Read `Target`, `Function`, `Phase`, and `Params` in the handler.

`Params` supports Bool, Int, Float, String, Name, Byte, Vector, Rotator, and
Object getters. `Has` and `ListParams` are safer than guessing a pin name.
`Caller` and `CallerFunction` identify the Blueprint frame that made the call.

A Pre watch may change supported input parameters with `Set*`. A Post watch is
observation only.

Use `Relay_WatchCancelable` and call `Params.Cancel()` to suppress a function.
Cancellation rewrites Blueprint bytecode. It is allowed for shipped game
Blueprints by default and can damage gameplay or saves if used carelessly.
`relay_gates.ini` supplies the operator kill switches.

<details>
<summary>Other Watch and local integration nodes</summary>

| Node | Use |
|---|---|
| `Relay_WatchInstance` | One object; releases when that object dies. |
| `Relay_WatchPath` | Class known only by content path at runtime. |
| `Relay_SubscribeTag` / `Relay_Emit` | Broadcast between mods in one process. `WorldReady` is Relay's built-in ready tag. |
| `Relay_Publish` / `Relay_Find` / `Relay_Unpublish` | Typed service object shared between mods in one process. |

</details>

Pass `Owner=self` for subscriptions and rule handles so they die with the
world. Host/connect sessions are the exception and intentionally have no Owner
pin.

<details>
<summary>WIP game-content registration</summary>

Call once from the ModActor; repeated calls update or share the same
registration.

| Node | Effect |
|---|---|
| `Relay_RegisterShopItem` | Adds a real store row. |
| `Relay_RegisterKeybind` | Adds a controls-menu action and returns press/release events. |
| `Relay_RegisterEvent` | Adds a calendar event and returns its trigger dispatcher. |
| `Relay_RegisterDream` | Adds a weighted `dreamBase_C` subclass to the sleep roll. |

Registrations are cached under `shimloader/cfg`. Removing Relay while custom
keybind rows remain can make VotV reset all keybinds to defaults on its next
cleanup pass; warn players on the mod page.

These registrations are local only. Register them on every machine that needs
them; Relay does not replicate them.

</details>

## Voice API

Voice is separate from session hosting. Copy `relay_voice.example.ini` to
`relay_voice.ini` and set `allow_capture=1` before opening a microphone. With
`0`, Relay can list devices but refuses to open capture hardware. Listing
devices and changing the preferred device do not open the microphone.

The public nodes are `Relay_VoiceListDevices`, `Relay_VoiceOpen`,
`Relay_VoiceGet`, `Relay_VoiceBindSink`, `Relay_VoiceMonitorLocal`,
`Relay_VoiceSetPreferredDevice`, and `Relay_VoiceGetPreferredDevice`.
`Relay_VoiceVersion` returns `0` when unavailable.

## Settings

All persistent settings and identity files live under
`<profile>/shimloader/cfg/`. Package updates should not replace this directory.

<details>
<summary>Configuration file options</summary>

### `relay_net.ini`

The public example intentionally contains only normal operator/mod-author
choices:

```ini
log_votv_hints=0
allow_listen=0
max_peers=8
server_name=
rendezvous_address=
punch_only=0
rules_profile=relay_rules.json
exclude_classes=
exclude_class_trees=
```

`allow_listen` is the hosting opt-in. Relay auto-writes `allow_listen=1` on permission prompts.

Set `log_votv_hints=1` only while correlating VotV popup text with UE4SS logs.
It is `0` by default, so Relay does not install the hint tap.

`exclude_classes` / `exclude_class_trees` keep subjective actors out of
inherited class rules - see "Classes that must stay local".

### `relay_gates.ini`

Optional cancellation controls:

```ini
enabled=1
dump_prologue=0
allow_game_functions=1
```

Set `enabled=0` to rule gating out of a bad session. Set
`allow_game_functions=0` to restrict cancellation to `/Game/Mods/` classes.
`dump_prologue=1` is diagnostic log noise, not a normal setting.

</details>

## Troubleshooting

| Symptom | Action |
|---|---|
| Host returns null | Display the Host node's `Reason`. |
| Connect handle returns but nothing happens | Bind `OnConnect`, `OnDisconnect`, and `OnError`. Connect is asynchronous. Check the host firewall/forwarding and address. |
| Connected but no actors sync | Read `Relay_NetWorldStatus.Reason`. Load the same map/save and declare the same rules. |
| Friend code is empty | Read `Relay_NetFriendCode.Reason`. Supply a reachable PublicAddress or configure rendezvous. |
| Peer can see but cannot hold/drive objects | Use a non-empty password on both nodes, or deliberately enable `AllowPasswordlessAuthority` for a private session. |
| Node or property is missing | Replace both the DLL and `Relay.pak`; one is stale. |
| Schema/rule refusal | Every peer needs the same gameplay pak and field/rule order. Run `Relay_Debug`. |
| `OWNERSHIP CONFLICT` / repeated rejected writes | Fix NetKey/Authority. Independently spawned objects are claiming one identity. |
| Spawn refused | The receiving machine has no matching class/component rule, presence mode, or remote class. |
| Client reads NetId `0` immediately | Wait for Welcome/registration; blank and per-peer identities need the assigned peer id. |
| Host identity changed | Restore the host identity file, or deliberately remove the corresponding known-host record on clients before reconnecting. |

When reporting a multiplayer bug, include both UE4SS logs, exact DLL/pak/mod
versions, host/connect options, map/save names, and whether direct or rendezvous
transport was used.

## Node reference

<details>
<summary>Blueprint node tables</summary>

### Core

| Node | Result |
|---|---|
| `Relay_Version` | Core API version. |
| `Relay_Watch`, `Relay_WatchPath`, `Relay_WatchInstance`, `Relay_WatchCancelable` | Subscription handle. |
| `Relay_SubscribeTag` | Subscription handle. |
| `Relay_Emit` | Broadcasts a local tag payload. |
| `Relay_Publish`, `Relay_Find`, `Relay_Unpublish` | Local service registry. |
| `Relay_NewParams` | Typed parameter bag. |
| `Relay_Debug` | Stable diagnostic issue list and counts. |

### Network session

| Node | Result |
|---|---|
| `Relay_NetVersion` | Network API/wire version, or `0`. |
| `Relay_NetHost` | Host session handle plus synchronous `Reason`. |
| `Relay_NetConnect` | Asynchronous client session handle. |
| `Relay_NetSession` | Current process session and role/id state. |
| `Relay_NetFriendCode` | Shareable direct/rendezvous code plus `Reason`. |
| `Relay_NetDiscover` | Temporary LAN scanner handle. |
| `Relay_NetKickPeer` | Host-only disconnect of a live peer; returns whether the disconnect was queued. |
| `Relay_NetWorldStatus`, `Relay_NetHostWorld` | World-fence state. |
| `Relay_NetSetWorldIdentity`, `Relay_NetSetWorldIdentitySource` | Local save identity source. |
| `Relay_NetPublishSave`, `Relay_NetRequestSave` | Explicit save-transfer surface. |
| `Relay_NetSetPaused` | Explicit pause state, combined with engine detection. |

### Replication and authority

| Node | Result |
|---|---|
| `Relay_NetBindClass` | Class-rule handle. |
| `Relay_NetExcludeClass` | Keep a subjective class out of rules inherited from its base. |
| `Relay_NetLoadRulesJson` | Add `relay.rules.v1` rules from a Blueprint string; returns whether the document is valid. |
| `Relay_NetRegisterActor`, `Relay_NetUnregisterActor`, `Relay_NetBind` | Explicit actor registration/binding. |
| `Relay_NetIdFor`, `Relay_NetFindByKey`, `Relay_NetRuleBoundActors` | Current object queries. |
| `Relay_NetIsAuthority`, `Relay_NetAuthorityFor` | Current authority queries. |
| `Relay_NetRequestAuthorityAsync` | Per-request lease handle; recommended. |
| `Relay_NetRequestAuthority`, `Relay_NetReleaseAuthority` | Low-level sent/release operations. |
| `Relay_NetPeer`, `Relay_NetLocalPeerId`, `Relay_NetIsHost`, `Relay_NetIsRemote` | Peer/session identity queries. |
| `Relay_NetRegisterPeerRole`, `Relay_NetUnregisterPeerRole`, `Relay_NetPeerRole` | Typed peer-to-game-object role registry. |
| `Relay_NetMarkDirty`, `Relay_NetRequestResync` | Forced flush and bounded recovery. |
| `Relay_NetControlRotation`, `Relay_NetCameraLocation`, `Relay_NetCameraRotation` | Replicated player presentation queries. |
| `Relay_NetGrabState`, `Relay_NetGrabbers` | VotV grab-state queries. |

`obj_relaySub` provides `Stop`, `IsActive`, `FireCount`, and `OnFired`.
`obj_relayNetRule` provides `Stop`, `RuleId`, `IsActive`, `OnActorBound`, and
`OnActorUnbound`. `obj_relayNet` provides send/broadcast/disconnect/close,
session/peer dispatchers, the current `Peers` array, and basic statistics.

</details>

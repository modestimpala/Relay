# RelayCpp self-hosted services

Experimental infrastructure for operators who want to provide rendezvous and
relay fallback. Players install only RelayCpp; they do not run these programs.

## Requirements

- Windows x64 host with stable public IPv4 connectivity.
- Inbound UDP 47777 for rendezvous and UDP 47778 for relay fallback.
- Enough bandwidth for every relayed session. Direct sessions do not consume
  relay bandwidth; relayed gameplay and save transfers do.

Keep `RelayRendezvousd.exe`, `RelayRelayd.exe`, and `libsodium.dll` together.
Do not publish or back up `relay_ticket.key` to shared storage.

## Start

Run both processes from the bundle directory under the same service account.
Start rendezvous in its own supervisor or terminal first; it creates the shared
key with a current-user ACL. Relay refuses to start without that key.

```text
RelayRendezvousd.exe --bind 0.0.0.0 --port 47777 ^
  --relay PUBLIC_HOST:47778 --ticket-key-file relay_ticket.key
RelayRelayd.exe --bind 0.0.0.0 --port 47778 ^
  --ticket-key-file relay_ticket.key
```

Set hosts' `rendezvous_address=PUBLIC_HOST:47777`. Joining players need only the
friend code. Keep `--log-addresses` off; it is a short-term debugging option,
not a production default.

## Supervision and restarts

Run each executable under a service supervisor with automatic restart on
failure. Set its working directory to the bundle directory and start
rendezvous before relay. A restart safely drops memory-only registrations,
tickets, circuits, and queues. Hosts re-register and clients reconnect. Keep
`relay_ticket.key` across ordinary restarts so both processes continue sharing
the same key.

## Compatibility

Deploy both executables, `libsodium.dll`, and the player DLL from the same
RelayCpp release. Do not mix service binaries from different bundles. Each
executable's `--help` output reports its service protocol version. A service
update drops active sessions, so update both processes together during a
maintenance window.

## Capacity and bandwidth

`RelayRendezvousd.exe --help` documents `--max-hosts`.
`RelayRelayd.exe --help` documents `--max-links` and `--max-circuits`. Start
below the machine and uplink limits, then raise them only after observing CPU,
memory, packet loss, and outbound transfer. The relay has bounded internal
queues but no operator-configured bandwidth quota; enforce an external traffic
budget if the host or provider requires one.

The minute counters are the normal health signal. Persistent refusals, queue
overflow, or backpressure mean the configured population exceeds available
capacity or bandwidth.

## Privacy

Rendezvous learns participant endpoints and when they communicate. Relay also
sees circuit labels, packet sizes, timing, and byte counts. It does not receive
gameplay plaintext or session keys. Publish this metadata exposure and your
retention policy to users. Default logs omit addresses; protect any diagnostic
logs created with `--log-addresses` and delete them when the incident is closed.

## Updates and rollback

1. Verify `SHA256SUMS.txt` before deployment.
2. Keep the previous bundle and the existing `relay_ticket.key`.
3. Stop relay, then rendezvous.
4. Replace both executables and `libsodium.dll` as one unit.
5. Start rendezvous, confirm it is listening, then start relay.
6. Confirm both minute-counter lines are healthy before announcing service.

If the update fails, stop both processes, restore the previous bundle without
replacing `relay_ticket.key`, and restart them in the same order.

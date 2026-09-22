# 0.9.0

## Voices of the Void gameplay / Default Example Rules

- Store purchases now share one balance and are priced by the host. Pending orders
  and drone deliveries survive a reconnect.
- A client's thrown item now flies the arc they aimed, instead of looking to
  everyone else like they dropped it.
- Interaction presses on the power panel, drone, and drone console now run on
  the host, so a press no longer doubles up or does nothing at all.
- Random events and weather (rain, fog, lightning) are now rolled by the host, so
  every player gets the same ones, including someone who joins midway through.
- Event triggers now fire for whichever player walks into them, not only the host.
- Light switches, doors, cord sockets, alarm lamps, the garage, the drone door, and
  plants now read the same on every machine, including right after a join.
- Plant growth is simulated by the host alone, instead of each game growing its own
  copy.

## Feel and performance

- Your own drops, throws, grabs, and inventory actions now happen immediately as a
  client, instead of hanging until the host answers.
- Picking up an item takes one network round trip instead of two.
- Spinning quickly while grabbing no longer drops the held item.
- Faster host frame rate in large worlds.
- Fixed the repeated quarter-second freezes under Proton/Wine.

## Setup and rules

- The first time you host, Relay asks permission in a Windows prompt instead of
  making you edit an ini file.
- A rules file is loaded only when `relay_net.ini` names it, so a leftover
  `relay_rules.json` can no longer outrank the rules a mod declares.
- Rule errors are clearer, and each one names the file or Blueprint it came from.
- Expanded the example rules to cover more Voices of the Void systems.

## Fixes

- Fixed swapped capture and apply values, and a missing points field, in the
  example spawn rule.
- Fixed the player child proxy class in the example rules.

# 0.8.1
- Added host rule assignment backfill functionality (for when rules are declared mid-game/mid-world).
- Added `Relay_NetExcludeClass` functionality to prevent subjective actors from being bound by inherited class rules.
- Introduced `exclude_classes` and `exclude_class_trees` settings in `relay_net.ini` to specify classes that should not replicate across peers.

# 0.8.0 

Initial release
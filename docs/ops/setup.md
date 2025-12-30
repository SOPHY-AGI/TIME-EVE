🟦🟦 

Locked. **Canonical = int64 Unix seconds.** 

Rooms are **strictly pre-created / programmatically inserted** (no auto-spawn on teleport). 

That’s a strong, upgrade-safe foundation.

## Canonical room labeling (simple + unambiguous)

Use **room.key as the canonical label** and store the integer too:

* `room.key = "UTC:<unix_seconds>"`  *(canonical, human + machine readable)*
* `room.db.utc = <unix_seconds>`     *(int64 identity)*
* `room.aliases += ["1970-01-01T00:00:00Z"]` *(optional human alias)*

This lets you `goto UTC:1123200` exactly, every time, using built-in movement/search.

## Bootstrap plan (built-in, no framework mods)

1. **Create 1200 anchor rooms** at the first second of each month (1970–2069).

2. **Link exits**:

   * each anchor gets `next` → next anchor
   * each anchor gets `prev` → previous anchor

3. **Strict rule**: teleport only works to existing rooms (anchors/events).
   New UTC rooms are created only by your controlled process.

## One-time population using `evennia shell` (no game code changes)

From `/opt/evennia/TIMELINE` with venv active:

```bash
evennia shell
```

Then paste this in the shell:

```python
from datetime import datetime, timezone
from evennia.utils.create import create_object, create_exit
from evennia.utils.search import search_object

ROOM_TC = "typeclasses.rooms.Room"   # default game room typeclass
EXIT_TC = "typeclasses.exits.Exit"   # default game exit typeclass

def iso_z(dt):
    return dt.strftime("%Y-%m-%dT%H:%M:%SZ")

# Generate month-start UTC instants: 1970-01-01 .. 2069-12-01 (inclusive) = 1200 months
anchors = []
for year in range(1970, 2070):
    for month in range(1, 13):
        dt = datetime(year, month, 1, 0, 0, 0, tzinfo=timezone.utc)
        utc = int(dt.timestamp())
        key = f"UTC:{utc}"
        iso = iso_z(dt)
        anchors.append((utc, key, iso))

# Create rooms (idempotent: skip if exists)
rooms = []
for utc, key, iso in anchors:
    existing = search_object(key, exact=True)
    if existing:
        room = existing[0]
    else:
        room = create_object(ROOM_TC, key=key)
        room.db.utc = utc
        room.aliases.add(iso)   # optional human-friendly alias
    rooms.append(room)

# Sort by utc to be safe
rooms.sort(key=lambda r: r.db.utc)

# Helper to remove old next/prev exits (if rerun)
def delete_exit(room, name):
    ex = [o for o in room.contents if o.__class__.__name__.lower().endswith("exit") and o.key == name]
    for e in ex:
        e.delete()

# Link prev/next exits
for i, room in enumerate(rooms):
    delete_exit(room, "prev")
    delete_exit(room, "next")
    if i > 0:
        create_exit(EXIT_TC, key="prev", location=room, destination=rooms[i-1])
    if i < len(rooms) - 1:
        create_exit(EXIT_TC, key="next", location=room, destination=rooms[i+1])

print(f"Anchors created/verified: {len(rooms)}")
print("First:", rooms[0].key, rooms[0].db.utc, rooms[0].aliases.all())
print("Last: ", rooms[-1].key, rooms[-1].db.utc, rooms[-1].aliases.all())
```

Exit the shell:

```python
exit()
```

## Verify in-game using built-ins

In the web client (as admin):

* `goto UTC:0`  *(1970-01-01T00:00:00Z)*
* `next`
* `prev`
* `goto 2045-01-01T00:00:00Z` *(works if you kept ISO aliases)*

## Strict insertion (later, still upgrade-safe)

When your process adds an event at exact UTC:

* it creates a new room `UTC:<t>`
* splices it between the correct neighbors by rewiring only:

  * predecessor’s `next`
  * successor’s `prev`
  * new room’s `prev/next`

No auto-creation on user teleport.

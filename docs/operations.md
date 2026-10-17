# Running Veyra on a station

What an administrator sets up and looks after once Veyra inspects real production: storage and retention, backups,
sign-in and users, and what to do when something goes wrong. Each section says where it is in the studio and what
the runtime does on its own.

## Where the data is

Everything Veyra stores is in one data folder: `%LOCALAPPDATA%\Veyra` on Windows, `~/.local/share/Veyra` on Linux,
`~/Library/Application Support/Veyra` on macOS (`VEYRA_HOME` overrides it).

| Folder or file | What it holds | Removed by |
|---|---|---|
| `veyra.sqlite3` | inspections, reviews, holds, the model registry, releases' records, datasets' rows, settings, system events, the audit log | nothing (records can expire by a rule; the audit log never does) |
| `images/YYYY/MM/DD/` | the original image of each inspection the station's image policy keeps | the retention rules for OK, NG and indeterminate/error images |
| `models/`, `releases/`, `datasets/` | registered models, released workflows, frozen dataset versions | never automatically |
| `training/` | checkpoints and logs of training jobs | the *Training artefacts* rule |
| `exports/` | Excel workbooks and evidence packages written there | the *Exported reports* rule |
| `logs/` | `studio-`, `runtime-` and `trainer-YYYYMMDD.log`, `crash.log` | the *Logs* rule (90 days by default) |
| `thumbnails/`, `cache/` | rebuildable thumbnails and previews | the *Caches* rule (30 days by default) |
| `backups/` | database backups with their checksums | only the backup rule |
| `weights/` | pretrained weights downloaded by the AI environment, the OCR models | never automatically |
| `ai-env/` | the AI environment, when Veyra created it | never automatically |

## Storage and retention

**Storage → Usage** shows the data volume with each category's share, how fast evidence grows (measured over the last
30 days of production, with the assumption stated), and when the free space would reach the warning level at that
rate. Image sizes come from the database; other folders are walked.

**Storage → Retention** sets how long each category is kept. A new installation keeps every image and record until an
administrator decides otherwise. Typical site rules:

| Category | Example | Why |
|---|---|---|
| OK-part images | 30 days | rarely looked at after the shift; the record stays |
| NG-part images | 1 year | evidence for complaints and model improvement |
| Indeterminate and error images | 90 days | needed to fix the inspection, then less so |
| Inspection records | 5 years (or forever) | traceability of what was shipped; the policy refuses to keep records shorter than images |

What is **always protected**, whatever the rules: inspections under a quality hold, anything newer than the minimum
age (24 hours by default), rejected parts nobody has reviewed yet (switchable), and the audit log, models, datasets,
releases and backups.

**Automatic cleanup** applies the rules once a day at the chosen hour. **React to low disk space** starts below the
cleanup level: expired data first; then, oldest first and only from the categories ticked as removable early (caches
and OK-part images by default), until the target free space. NG-part images newer than the protection period are never
removed early. If the allowed categories cannot free enough, an error event says so and what to do; protected data is
not sacrificed.

**Clean up…** removes data by hand: by the policy, or by category, date range and camera. It always shows a preview
first (how many items, how much space, what is protected and why) and asks for a confirmation. Removing an image keeps
the inspection record, which then says when and by which cleanup the image was removed. Every cleanup, scheduled or by
hand, is in the cleanup history and the audit log.

**Check consistency** compares the database with the files: records whose image is missing (deleted outside Veyra)
can be marked as such; image files no record refers to are reported, never removed.

## Station I/O

**Stations → Camera assignments → Station I/O…** chooses where results go: the simulated module while setting up
(the pre-start checklist warns that no reject gate is driven), or a PLC or remote I/O module over **Modbus TCP**:
address, port and unit, each output of the workflow's *Output signals* tool (`OUT1` good, `OUT2` reject, `OUT3` error
by default) mapped to a coil address, and inputs (a part sensor, a door switch) to discrete-input addresses. **Test the
connection** connects and reads the inputs without writing any output. A failed write is reported and counted; the next
one reconnects. Modbus TCP is implemented from the protocol specification and tested against an independent server
implementation (pymodbus), not yet against a physical PLC; Modbus RTU, OPC UA and vendor protocols are not implemented.

## The AI environment and tool models

Ultralytics and Anomalib train in an AI environment of their own (see [models.md](models.md)): **Settings → AI
environment** shows which Python is used and what it can do, and sets one up. It needs about 2 GB (CPU build of
PyTorch) or 5 GB (CUDA build) and can live on another disk (point *Choose Python…* at it). Pretrained weights are
downloaded once into the data folder's `weights/` and recorded with their SHA-256 in every model trained from them.
The OCR models of the *Read text* tool (about 14 MB) are installed from **Settings → Text reading (OCR) models**; they
are downloaded from their pinned revision and refused if their checksum differs.

## Quality holds

In **Inspections**, *Review and holds → Place on hold…* protects one inspection; **Hold all shown…** holds every
inspection that matches the calendar and filters (a batch, a lot, a day of one camera). A hold needs a reason and is
released the same way. Holds and releases are kept as a history.

## Backups and restore

**Storage → Backups**: *Back up now* writes a consistent copy of the database while production runs, checks its
integrity and records its SHA-256; *Verify* recomputes both. The retention page sets a daily backup and how many to
keep (only this rule removes backups, and never the last one). Images are not in a backup: they are covered by the
retention policy, and a restored database shows images that are gone as missing.

*Restore at next start…* verifies the backup and schedules it. At the next start, before the database is opened, the
current database is moved into the backups folder (so a restore can be undone) and the backup copied in place. The
outcome is a system event and an audit entry.

## Users, roles and sign-in

**Settings → Users and roles** adds users with a role (operator, quality engineer, vision engineer, AI engineer,
administrator) and sets passwords. **Require sign-in** makes every user sign in with their password; turning it on
asks for your own password, so you cannot lock yourself out. *Sign out after … min idle* signs an idle studio out
(production keeps running). Five wrong passwords lock an account for five minutes.

If the last administrator's password is lost, reset it on the station itself:

```bash
Veyra.exe users list
Veyra.exe users reset-password NAME      # asks for the new password twice
Veyra.exe users sign-in-off               # if no administrator can sign in at all
```

Anyone who can run this can read the data folder anyway, so it does
not weaken a local installation; every use is in the audit log.

## When something goes wrong

**System → Health** checks the runtime, model inference (which execution providers ONNX Runtime has), the database
(integrity), disk space, retention, backups, a pending restore, the station and its cameras, missing evidence and open
events. Each problem says what is wrong and opens the page where it is fixed.

**System → Events** lists what the runtime noticed by itself: a camera that stopped (the other cameras keep
inspecting), the station faulting, low disk space, a cleanup or backup that failed, an interrupted cleanup completed at
start-up. A condition that lasts is one open event, not one per check; it closes when the condition clears or when
someone marks it resolved.

**System → Audit log** lists who did what: reviews, holds, approvals, activations, configuration, users, sign-ins,
retention changes, cleanups, exports.

For support, **Save a diagnostic bundle…** writes a zip with the health checks, versions, open events, the station
configuration, the storage policy, the cleanup history and today's logs. It contains no images and no inspection
records. **Open logs folder** shows the log files themselves.

| Symptom | Where to look | Usual cause |
|---|---|---|
| A camera tile says *Fault* | the tile, System → Events | cable, power, another program using the camera, network settings for GigE |
| *Dropped frames* rise | Stations | inspection slower than the camera; the station keeps the newest frame |
| *Results could not be stored in time* | Overview, Storage | disk too slow or full |
| Disk warnings | Storage → Usage | retention rules keep more than the disk holds; see the growth forecast |
| Sign-in refuses a user | Settings → Users | locked after wrong passwords (wait five minutes) or removed |

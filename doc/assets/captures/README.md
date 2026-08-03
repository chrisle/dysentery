# CDJ-2000NXS hardware captures, July 2026

Thirteen packet captures taken from a rig of **two CDJ-2000nexus players on
firmware 1.44**, recorded 2026-07-29. Every byte here came off the wire between
real Pioneer hardware. Nothing in this directory was generated, replayed or
synthesised.

The captures were made while working on an independent Python implementation of
the protocol, and each one was taken to answer a specific question. The `NOTES.md`
in each directory is the lab record written at the time — hardware state, what
was pressed on the decks, and what the capture turned out to show.

## The rig

| | Deck A | Deck B |
|---|---|---|
| model | CDJ-2000nexus | CDJ-2000nexus |
| firmware | 1.44 | 1.44 |
| MAC | `74:5e:1c:56:67:ac` | `74:5e:1c:56:ca:54` |
| IP | `169.254.103.172` | `169.254.202.84` |
| device number | 1, set manually | 2, set manually (AUTO in S13) |
| name field | `CDJ-2000nexus` | `CDJ-2000nexus` |

The `NOTES.md` files are reproduced as written during the session. Where one
still has `ip=?` or `firmware=?` in its *Hardware state* block, the rig above is
what applied — those placeholders were simply never filled in on the night, and
the values are unchanged across the whole set.

Both decks report the **identical** 20-byte name, so the name does not identify a
device — group by MAC. No DHCP server was present; everything self-assigned in
`169.254/16`. A Mac was attached to the same switch as the capture host, at
`169.254.99.100` (`5e:e9:1e:d9:2f:01` on the bridge, `a0:ce:c8:e2:26:de` on the
dongle).

## Read this before drawing conclusions: the tap matters

Five of these captures were taken on a **BSD bridge interface** (`bridge1`) and
eight on the **member interfaces** (`pktap,en12,en9`). The difference is not
cosmetic, and it produced one wrong finding before it was understood:

> A BSD bridge **floods** broadcast frames, so keep-alives, hellos and claim
> packets all reach the bridge's BPF tap and the capture looks healthy — both
> decks visible, packets round-tripping byte-exactly. Learned **unicast** is
> forwarded member-to-member directly and never reaches that tap.
>
> Status (UDP 50002), the media query/response, and all dbserver TCP are unicast.
> On a `bridge1` capture they are **absent because of the instrument**, not
> because the decks were silent.

Same two decks, same network, same activity, only the tap differs:

| Capture | Tap | udp/50002 | tcp |
|---|---|---|---|
| S04 | `bridge1` | **0** | 0 |
| S05 | `pktap,en12,en9` | **1440** | 1672 |

So: **treat any absence in a `bridge1` capture as unmeasured.** The `Tap` column
below tells you which is which. The rule of thumb that came out of it — verify a
tap with *unicast*, not broadcast: a LINK browse must produce TCP on 12523 and
1051, and if it doesn't, the tap is lying.

The `pktap` captures have the opposite quirk: because they record both member
interfaces at once, every **broadcast** frame is written twice, a fraction of a
millisecond apart. Keep-alives and the claim packets on UDP 50000 are therefore
duplicated, and counting them naively gives double the real number and an
apparent cadence of half the real one. Unicast frames appear once. The `bridge1`
captures have a single interface and are free of this.

## The captures

| Session | Tap | Pkts | Length | What it shows |
|---|---|---|---|---|
| [S01-cold-boot-a](S01-cold-boot-a/) | `bridge1` | 75 | 71 s | Deck A cold-boots alone: 3× Hello, 3× ClaimMac, 3× ClaimIp, 3× ClaimNumber, then keep-alives |
| [S1b-cold-boot-b-alone](S1b-cold-boot-b-alone/) | `bridge1` | 62 | 45 s | Deck B cold-boots alone — the controlled twin of S01 |
| [S02-deck-b-joins](S02-deck-b-joins/) | `bridge1` | 163 | 154 s | Deck B joins a network deck A already owns; **one** ClaimNumber, not three |
| [S2c-deck-a-joins](S2c-deck-a-joins/) | `bridge1` | 65 | 58 s | The mirror of S02, run as an explicit prediction — and confirmed |
| [S04-media-insert](S04-media-insert/) | `bridge1` | 2 597 | 843 s | USB insert/eject; a full NFS pull of `export.pdb` off a live deck |
| [S4b-media-insert](S4b-media-insert/) | `pktap` | 1 910 | 149 s | S04 re-run on a tap that can actually see unicast |
| [S05-link-browse](S05-link-browse/) | `pktap` | 3 400 | 144 s | Deck A browses deck B's USB over LINK: dbserver end to end |
| [S06-load-and-play](S06-load-and-play/) | `pktap` | 5 805 | 180 s | Load and play from the other deck's USB; play/pause/cue/hot-cue, beats on 50001 |
| [S13-format-ground-truth](S13-format-ground-truth/) | `pktap` | 53 368 | 232 s | MP3 / AAC / WAV / AIFF loaded over LINK, with INFO opened on each. Deck B in **AUTO** numbering |
| [S15a-sd-alone](S15a-sd-alone/) | `pktap` | 5 925 | 118 s | Deck A holds an **SD card only**; deck B browses and loads from it |
| [S15b-sd-and-usb](S15b-sd-and-usb/) | `pktap` | 3 287 | 78 s | Deck A holds SD **and** USB; both browsed, one track loaded from each, then both ejected |
| [S16a-settings-over-link](S16a-settings-over-link/) | `pktap` | 605 | 44 s | `UTILITY → LOAD SETTINGS` pulled across the link |
| [S20-browse-ground-truth](S20-browse-ground-truth/) | `pktap` | 9 930 | 449 s | The big browse: every root category, full drill-downs, search, and all twelve sorts |

`cmd.txt` in each directory is the exact `tcpdump` invocation used.

## What these captures change, and what they confirm

The first group are corrections and additions to the current Packet Analysis,
each one made as a doc edit in the same pull request. The second group is
independent confirmation of things already documented — worth having from pinned
hardware, but not news. The `NOTES.md` files carry the detail and the reasoning.

### Corrections and additions

- **Keep-alive byte `0x25` is not a mixer-versus-CDJ discriminator.**
  `startup.adoc` gives it as `01` on a CDJ and `02` on a mixer. It is really
  "was I first on this network?", latched at boot and held for the session: each
  of these decks came up `02` alone (S01, S1b) and `01` joining (S02, S2c), so it
  is not a property of the model. In S02 the incumbent held `02` while its peer
  count went 1 → 2.
- **The final-stage claim repeat count does not follow the numbering mode.**
  `startup.adoc` says a player set to a specific number sends one packet and an
  auto-assign player sends three. Booting alone, a *manually* numbered deck sends
  **three** (S01, S1b) — as does the auto deck in S13. Booting onto an occupied
  network it sends **one** (S02, S2c, and the manual deck in S13). What governs
  it is whether a peer cuts the series short.
- **The "assignment finished" packet is not mixer-specific.** The doc describes
  type `05` as something a mixer sends from a channel-specific port. In S13 an
  ordinary CDJ already settled on the network sends the identical 38-byte packet,
  carrying its own device number and `CDJ-2000nexus` in the name field, on a plain
  unmanaged switch with no mixer present — 2 ms after the booting deck's single
  final-stage packet, which is exactly the short-circuit above. Being unicast, it
  is invisible on the `bridge1` captures, which is why S02 and S2c show the effect
  without the cause.
- **Keep-alive cadence is 2.0026 s** on these decks. The doc gives 1.5 s for the
  mixer and no figure for a CDJ. The settled deck in S02 sends 78 consecutive
  keep-alives with every gap between 2.002 and 2.003 s; only the first gap after
  startup is shorter.
- **Unsolicited media broadcasts were never seen.** `media.adoc` says standalone
  CDJs periodically broadcast type `06` media packets so that passive observers
  can learn about mounted media. Across the whole set every type `06` on port
  50002 — six of them — was a unicast reply to a type `05` query in the same
  millisecond, with as many replies as queries and none broadcast. S4b and S15b
  cover insertion and ejection with a passive observer present and a tap that sees
  unicast, and produced none.
- **An unnamed volume is not an empty one**, so occupancy has to key on the track
  count rather than the name. The volume label is big-endian UTF-16, which the
  doc does not currently state.

### Confirmations of things already documented

- The name field really is `CDJ-2000nexus` with that exact casing — previously
  inferred, since no published capture contained a literal CDJ-2000 name field.
  Both decks report the identical name, so it does not identify a device.
- Status packets are **284 bytes** (`0x11c`) on firmware 1.44, which matches the
  documented "newer firmware / Nexus 2" row. Worth noting only because it is a
  plain nexus: **packet length does not tell you the generation**, and a parser
  keying behaviour off length will mis-classify these decks.
- Byte `0x31` of the second-stage claim is `01` for auto-assign and `02` for a
  specific number, exactly as documented — S13 has one deck of each.
- Slot `02` is SD and slot `03` is USB.
- Status bytes `0x6f`/`0x73` behave exactly as `vcdj.adoc` describes, including
  the `02` and `03` intermediate values while media is being unmounted. S15b
  ejects SD first and then USB, and the two bytes move in that order.
- Status is **unicast to announced peers only**. In S4b all 1 503 status packets
  went deck-to-deck — 754 one way, 749 the other — and **not one** was addressed
  to the Mac, which had an address on the network the whole time but had never
  announced itself. Nothing here is broadcast.
- The **media query** `0x05` and its `0x06` response appear deck-to-deck in five
  of these captures (S4b, S13, S15a, S15b, S16a). The response is 192 bytes and
  carries the responding device number, the slot, a UTF-16**BE** volume label,
  and track and playlist counts:

  | Capture | Responder | Slot | Label | Tracks | Playlists |
  |---|---|---|---|---|---|
  | S15a | 1 | 2 (SD) | `Sam CDJ1000mk3` | 113 | 11 |
  | S15b | 1 | 3 (USB) | `SAM2` | 692 | 35 |
  | S13 | 1 | 3 (USB) | *(empty)* | 651 | 1 |
  | S4b | 2 | 3 (USB) | *(empty)* | 611 | 35 |

  Two of these media report no label at all while carrying 651 and 611 tracks.
- Status bytes `0x6f` (USB) and `0x73` (SD) behave exactly as `vcdj.adoc`
  documents them — `04` when no media is present, `00` when it is loaded, and
  `02` or `03` while it is being unmounted. S15b is a clean worked example of
  the whole cycle, and the transitions line up with what was done on the deck:

  ```
  t= 0.0s  0x6f=04  0x73=00     no USB; SD loaded
  t=13.0s  0x6f=00  0x73=00     USB inserted
  t=54.8s  0x6f=00  0x73=02  ┐
  t=57.4s  0x6f=00  0x73=03  ├ SD ejected first: unmounting, then gone
  t=59.6s  0x6f=00  0x73=04  ┘
  t=68.2s  0x6f=02  0x73=04  ┐
  t=69.7s  0x6f=03  0x73=04  ├ then the USB, the same way
  t=69.9s  0x6f=04  0x73=04  ┘
  ```

  The `02` and `03` states last only a second or two each, so a parser that
  treats these bytes as a boolean will see media flap during an eject.
- `LOAD SETTINGS` (S16a) is a UDP `0x35`/`0x36` exchange that reads
  `PIONEER/MYSETTING.DAT`. Both packets are in that capture, which is only 605
  packets long and easy to read end to end.

**NFS**

- **A CDJ-2000NXS serves NFS** — nfsd 2049, mountd 48276, portmapper 111, the
  same numbers seen on an XDJ. It exports to the **whole link-local subnet**
  (`169.254.0.0/255.255.0.0`), which is why a passive client can read it without
  announcing itself at all.
- USB is `/C/`, SD is `/B/`, and the export paths are UTF-16LE while the groups
  are ASCII.
- Real decks use **8192-byte** reads — the NFSv2 maximum — and simply rely on IP
  fragmentation, five or six fragments per reply on a 1500-byte MTU. Anything
  analysing these captures must reassemble fragments: ignoring them under-counted
  one transfer by 1800×, which would have made the audio look like it never
  crossed the wire.
- Audio is **streamed over NFS during playback**, not downloaded ahead of time.
- `NFSERR_ACCES` on `MNT` means **the slot is empty**, not that the client was
  refused.

**dbserver / menus**

- S20 exercises the full menu surface in one sitting — 4 475 dbserver packets:
  GENRE, ARTIST, ALBUM, LABEL, BITRATE, HISTORY and KEY, drill-downs all the way
  to track level, SEARCH with text typed on the deck, and all twelve sort modes
  including the Camelot key ordering. Both ends are real hardware, so it has the
  authentic **replies** as well as the requests.
- Sorting selects the item's **second column**, and drill-down requests form a
  grid of the shape `0x1000 | depth << 8 | category`.
- One dbserver connection multiplexes **both** slots, distinguished by a
  descriptor byte, rather than one connection per slot. S15b is the direct
  evidence: SD and USB both browsed and loaded from, over a **single** TCP
  stream.
- S05 is the one to read for connection setup — it is the only capture here that
  contains the full handshake, with the port query on TCP 12523 and the dbserver
  connection to 1051 both opening inside the capture window. In S15a, S15b and
  S20 the connection was already established before recording began.

## Provenance and processing

**What was excluded.** A parallel set of captures exists in which a Mac
impersonated a player and served media to a CDJ. None of those are here — that
traffic is only as authentic as the implementation that produced it, and the
point of this directory is ground truth.

The Mac does appear on the wire in **S04** and **S4b**, purely as an ordinary
NFS *client* reading from a live deck. It never claims a device number or emits
DJ-Link traffic in those two, so the deck's side of the conversation — which is
what the NFS findings above rest on — is genuine hardware output.

**What was modified.** Exactly one file:
`S13-format-ground-truth/run.pcapng`. The captured bytes of NFS datagrams (and
of every IP fragment belonging to them) were truncated to a 96-byte frame,
dropping the streamed audio payload and taking the file from 61 MB to 8.6 MB.

- All 53 368 records are still present, in order, with their original timestamps.
- Every record's **original length is preserved** in the record header, so read
  sizes, offsets, fragmentation and timing all read correctly.
- The 5 029 non-NFS records — every DJ-Link, dbserver, mount and portmap packet —
  are **byte-identical** to the original.

The effect is the same as having captured with a small snaplen on NFS only.
Wireshark will mark the truncated frames as such. Every other file in this
directory is an untouched copy of the original capture.

Filenames use `.pcap` for the single-interface `bridge1` captures and `.pcapng`
for the multi-interface `pktap` ones, matching what `tcpdump` actually wrote.

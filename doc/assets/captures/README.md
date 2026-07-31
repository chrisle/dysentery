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

## Things in here that may be new

Pulled out because they were surprising, not because they are a complete list.
The `NOTES.md` files carry the detail and the reasoning.

**Discovery / keep-alive (UDP 50000)**

- Keep-alive cadence is **2.003 s**, not the 1.5 s that is documented.
- The name field really is `CDJ-2000nexus` with that exact casing — previously
  inferred, since no published capture contained a literal CDJ-2000 name field.
- Byte `0x25` of the keep-alive means **"was I first on this network?"**. It is
  latched once at boot and never re-evaluated, and the same latch drives the
  stage-3 repeat count: a deck booting alone sends **three** ClaimNumber packets,
  a deck joining an occupied network sends **one**. S01/S1b/S02/S2c are the four
  captures that isolate this — the variable was confounded twice before S2c
  settled it as a prediction.
- S13 has deck B in **AUTO** numbering, including the type-`05` "number in use"
  reply, alongside deck A on a manual number.

**Status (UDP 50002)**

- These decks send **284-byte** (`0x11c`) status packets on firmware 1.44.
  Published tables map `0xd4` to "Nexus" and `0x11c`/`0x124` to "newer firmware /
  Nexus 2", so **length does not identify the generation** — a parser keying
  behaviour off packet length will mis-classify a plain nexus on current firmware.
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

  Two things fall out: **slot 2 is SD and slot 3 is USB**, and **an unnamed
  volume is not an empty one** — two of these media report no label at all while
  carrying 651 and 611 tracks. Occupancy has to key on the track count.
- Media state is visible in the status packet at offsets `0x6f` and `0x73`, and
  it is **not a binary flag**. S15b captures both slots being ejected, and the
  bytes step through intermediate values on the way:

  ```
  t= 0.0s  0x6f=04 0x73=00      both settled
  t=13.0s  0x6f=00 0x73=00
  t=54.8s  0x6f=00 0x73=02   ┐
  t=57.4s  0x6f=00 0x73=03   ├ 02 and 03 appear only in transition,
  t=59.6s  0x6f=00 0x73=04   ┘ for a couple of seconds each
  t=68.2s  0x6f=02 0x73=04
  t=69.7s  0x6f=03 0x73=04
  t=69.9s  0x6f=04 0x73=04
  ```

  `04` is the settled state and `00` the other extreme; `02` and `03` look like a
  mount-and-scan progression. A parser treating these as boolean will see media
  flap.
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

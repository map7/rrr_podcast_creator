# rrr_podcast_creator

Rebuilds a podcast RSS feed for a Triple R (3RRR Melbourne) program whose
official podcast feed has stopped updating, so it can be read in a normal
podcast client. Written for *Byte Into IT*, but the program slug is an
argument, so it works for any program on the site.

## Why

Triple R publishes some programs as podcasts, with real mp3 files on S3.
For Byte Into IT that stopped on 6 February 2025. The show kept airing and
the episodes are still on the website, but the podcast feed at
`/explore/podcasts/byte-into-it/feed.xml` has not moved since.

The website plays the newer episodes from the station's on-demand archive,
which is a different system to the podcast mp3s.

## How the audio URL is found

This is the only interesting part. The audio URL does not appear in the
episode page at all. The page carries a `data-view-playable` attribute:

```json
{
  "type": "broadcast_episode",
  "player_item_id": 429413,
  "data": { "timestamp": "20260930190000", "duration": 3600 }
}
```

The player builds a Wowza HLS URL from that at runtime:

```
https://playback.rrr.org.au/od-p1v1/_definst_/mp4:YYYYMM/YYYYMMDDHHMM_<seconds>_128.mp4/playlist.m3u8
```

with

| part       | source                                        |
|------------|-----------------------------------------------|
| `YYYYMM`   | first 6 digits of `timestamp`                 |
| `YYYYMMDDHHMM` | first 12 digits of `timestamp`             |
| `<seconds>`| `duration` + 360                              |
| `128`      | bitrate, constant so far                      |

The `+ 360` is the catch. The page reports a 3600 second slot, but the
recorded file is 3960 seconds: RRR's encoder adds six minutes of padding.
Checked against every episode that has audio, `+ 360` returns HTTP 200 and
3600, 3660, 3900 and 4200 all return 404, so the padding is fixed rather
than per-episode.

Nothing else is needed. There is no API call to resolve the stream: the
player goes straight from the page data to the chunklist.

## What it produces

RSS 2.0 with iTunes tags. Each episode with audio gets an `<enclosure>`
pointing at `playlist.m3u8`, typed `application/vnd.apple.mpegurl`.

Only episodes inside the on-demand window have audio, currently about 45,
going back to October 2025. Older episodes are listed with a note and no
enclosure, or omitted entirely with `--playable-only`.

That leaves a gap between February 2025, where the podcast feed stopped,
and the start of the on-demand window. Those episodes have no audio from
either source. Everything before February 2025 is still available from the
original podcast feed, which remains online even though it no longer
updates, so subscribing to both covers the whole run.

## Usage

```sh
rrr-feed byte-into-it -o ~/.local/share/rrr-feeds/byte-into-it.xml
rrr-feed byte-into-it --playable-only -o ~/.local/share/rrr-feeds/byte-into-it.xml
rrr-feed some-other-program --verbose -o /tmp/other.xml
```

Python 3, standard library only, so it runs under a systemd unit without a
virtualenv. The output file is written through a temporary file and renamed,
so a reader never sees a half-written feed.

## Install

```sh
ln -s ~/code/rrr_podcast_creator/rrr-feed ~/.local/bin/rrr-feed
cp systemd/rrr-feed.* ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now rrr-feed.timer
```

The timer rebuilds the feed daily at 07:30 with `Persistent=true`, so a
missed run catches up at the next login.

## Playing it

The enclosure is an HLS playlist, not a file. mpv and ffmpeg handle it
directly; wget and curl cannot fetch a playlist as audio. To download an
episode, let ffmpeg stitch the segments and copy the AAC through without
re-encoding:

```sh
ffmpeg -i "<playlist.m3u8>" -c copy -bsf:a aac_adtstoasc episode.m4a
```

## Caveats

This scrapes HTML, so a site redesign will break it. The failure mode is
that no episodes are found and the previous feed file is left alone.

The `+ 360` padding and the constant `128` bitrate are observed, not
documented. If episodes stop resolving, check those first against a live
episode in the browser.

For personal use. It reads the same public pages a browser does and does not
republish anything.

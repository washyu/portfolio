---
title: "From Bluetooth Fuzz to Bit-Perfect: Building a Raspberry Pi DAC HAT Audio Player"
date: 2026-04-22
draft: false
tags: ["raspberry-pi", "audio", "homelab", "plex", "hifi", "dac", "queen"]
description: "How a couple of flat-sounding Queen CDs led me down a rabbit hole of bad CD masters, Pi DAC HATs, and replacing Bluetooth with a proper hi-fi endpoint."
---

# From Bluetooth Fuzz to Bit-Perfect: Building a Raspberry Pi DAC HAT Audio Player

*A homelab engineer's journey from muffled Queen albums to a proper hi-fi endpoint*

---

## The Problem: When Good Albums Sound Bad

I've been building a CD library, picking up physical discs from local shops around Tacoma, ripping them with ARM on a homelab VM, and streaming through Plex to my Sharp SA-150 amp. Recently I picked up Queen's *A Night at the Opera* and *Jazz*, and something sounded off. It sounded muffled, flat, it was off.

The culprit turned out to not be my playback chain at all. It was the masters. EMI's early CD releases of Queen's catalog (and the Hollywood Records remasters that followed in the 1990s) were essentially vinyl transfers. The source material was already compressed and EQ'd for the physical limitations of a record, and rather than going back to the original multitrack tapes and remastering for CD's wider dynamic range, they largely just digitized what was already there. The result is CDs that don't sound particularly better than vinyl, because they're not actually using what CD audio can do.

The version worth hunting is the 2011 remasters supervised by Bob Ludwig. Those went back to the tapes, restored dynamic range, and were mastered with CD as the actual target format. The difference is audible.

That research rabbit hole had a side effect: I started thinking more carefully about my playback chain. If I was going to invest time tracking down correct pressings and building a library worth listening to, was Bluetooth really the right last link in that chain?

---

## The Options I Considered

My first instinct was a WiiM Pro Plus. At around $220, it's a proper network streaming endpoint with Wi-Fi or ethernet, AirPlay 2, Spotify Connect, Tidal, and a significantly upgraded AKM DAC over the entry-level Mini. Plug it into the SA-150 via RCA and you have a no-compromise, wire-it-and-forget-it solution. The ethernet angle was compelling too, since the amp sits right next to the homelab rack.

But I had a spare Raspberry Pi 4 sitting in a drawer, and a homelab full of infrastructure I've already built. The Pi 4 is actually preferred for audio HAT use because the Pi 5's higher power draw and faster clocks can introduce more electrical noise into a sensitive analog circuit. And I was already running a media stack: TrueNAS for storage, Plex for the music library, ARM for ripping. Adding a Pi-based streaming endpoint felt like it fit naturally into the architecture rather than introducing an external device.

So the question shifted from *buy something* to *which HAT do I put on this Pi?*

---

## Choosing the Right HAT

The Pi DAC HAT market is honestly pretty good right now. The main players I looked at:

**HiFiBerry DAC2 Pro** (~$45) is the brand everyone recommends. It uses the same TI PCM5122 DAC chip as most mid-range HATs, but with dual master clocks, one for the 44.1kHz family and one for the 48kHz family, which means no resampling and genuinely lower jitter. It's the safe choice with a solid reputation and HiFiBerryOS available if you want a turnkey software experience.

**IQaudio DAC+** (~$35) is the Raspberry Pi Foundation's own branded HAT, acquired from IQaudio a few years back. First-party driver support, clean implementation, and it would be running in 20 minutes.

**Innomaker HiFi DAC HAT** (under $30) was the dark horse. It showed up as an Amazon listing that looked generic but turned out to be from Innomaker, a legitimate brand that's been making Pi accessories for years. The spec sheet was worth reading carefully:

- TI PCM5122 DAC, same chip as the HiFiBerry, rated at 112dB SNR and -78dB THD+N
- Dual oscillators (45.158MHz and 49.152MHz), the same dual-clock architecture the HiFiBerry Pro uses
- TPA6133 headphone amp with independent grounding, a real headphone amp chip rather than a resistor network
- Film capacitors in the output stage, a genuine (if modest) quality indicator over ceramic caps
- Gold-plated RCA line-level outputs at 2.1Vrms
- Onboard EEPROM for plug-and-play device tree overlay auto-detection
- Reserved IR port on GPIO26

At that price point with those internals, the Innomaker punches well above its weight. The specs that matter, SNR and THD+N, are identical to the more expensive HiFiBerry because those numbers are largely determined by the PCM5122 chip itself. A competent implementation hits those numbers regardless of brand name on the board.

The one thing worth noting from reviews: the EEPROM plug-and-play claim is somewhat aspirational. In practice, reviewers had to manually add `dtoverlay=allo-boss-dac-pcm512x-audio` to `/boot/config.txt`. So the Innomaker presents itself to the kernel as an Allo Boss DAC clone rather than a HiFiBerry. That's actually good news since the Allo Boss was a well-regarded audiophile design before Allo went under, and it means broad software support. Any distro that supports Allo Boss works with this board. Adding one line to a config file is a 30-second task, not a dealbreaker.

A particularly useful review showed the board still running solid on moOde two years after purchase, while the reviewer's HiFiBerry DAC2 Pro had failed in the same timeframe. One data point, but build quality holding up over time matters.

---

## Initial Setup

Setup was straightforward. I flashed a fresh 64-bit Raspberry Pi OS using Pi Imager and seated the HAT on the Pi 4's 40-pin header. The main configuration step was adding the device tree overlay manually to the `[all]` section of `/boot/config.txt`:

```
dtoverlay=allo-boss-dac-pcm512x-audio
```

After a reboot, `aplay -l` confirmed the DAC registered correctly:

```
card 3: BossDAC [Boss DAC], device 0: Boss DAC HiFi pcm512x-hifi-0
```

There were a handful of `snd_soc_register_card() failed: -517` messages in the boot log, which looks alarming but is actually just `EPROBE_DEFER`. That's the kernel saying a dependency wasn't ready yet, so it retried and then succeeded. The DAC was registered and working.

For software, I went with Plexamp headless. Since my music library is already in Plex and the rips live on a TrueNAS NFS share that Plex already indexes, it was the obvious fit. No separate music server to maintain, no double-mounting the NFS share, and the Pi shows up as a player target directly in the Plex and Plexamp interfaces on any device. Plexamp runs as a systemd service so it's always up, and the web interface on the Pi's local IP lets you verify the output device configuration without needing a display attached.

---

## First Impressions

Honest take: the change wasn't the dramatic revelation I was half expecting.

With Plexamp pointed at the BossDAC output and the RCA cables running to the SA-150, the music still sounded flat on *A Night at the Opera*. But it was a better flat, and there was an immediate practical difference. The Bluetooth adapter was running at a lower output level than the DAC, so to get the same listening volume through Bluetooth I had been running the amp higher than it liked. That introduced a faint but noticeable hiss. With the Pi driving the RCA inputs directly, I turned the amp down to a more comfortable level and the hiss disappeared.

I also noticed the soundstage felt slightly fuller across the rest of my library, not a dramatic transformation, but a sense that things were sitting more correctly in the mix. Whether that's the cleaner signal path, the better output level matching, or just knowing the chain is right, I can't say definitively. But it felt like an improvement.

The Queen albums were still flat though, and that was the mastering, not the tech. The real difference came later when I picked up the 2011 remix version of *A Night at the Opera* from the library. That opened up the way I was expecting all along: the layered harmonies had actual space, the dynamics landed properly. But that was the Bob Ludwig remaster doing its job, not anything I built. The Pi just got out of the way and let it happen.

Overall it was a worthwhile change. The hiss is gone, the signal path is cleaner, and I'm not leaving anything on the table for the music that's actually well mastered.

---

## What's Next

The Pi setup is staying put. Plexamp does exactly what I need and I have no reason to change it. The music plays when I ask it to, the hiss is gone, and the SA-150 is getting a clean signal. That's the job done.

A few things I'm thinking about on the music side though. The full library shuffle is fine but it would be interesting to try building some playlists that recreate more of a radio station feel, a curated mix rather than just everything in random order. And after hearing what the 2011 Bob Ludwig remasters did for *A Night at the Opera*, I want to track down the rest of that Queen remaster series to complete the collection. The local shops and library sales are a good hunting ground for those.

The bigger addition I'm considering is a turntable. The SA-150 is a vintage amp and a record player would suit it well aesthetically and sonically. I'm thinking about scouring local pawnshops and Goodwills. There's no shortage of decent decks sitting in thrift stores waiting to be found, and that's half the fun of it.

---

*Tools used: Raspberry Pi 4, Innomaker HiFi DAC HAT (PCM5122), Sharp SA-150 amp, Plexamp headless, TrueNAS SCALE for storage, ARM for CD ripping*

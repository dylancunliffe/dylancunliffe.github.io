---
layout: post
title: MacLeod Free Rooms
subtitle: A Live "Which Classroom Is Empty Right Now" Finder for UBC's MacLeod Building
thumbnail-img: /assets/img/mcld-rooms-thumb.png
tags: [Web, JavaScript, Python, Data Parsing, GitHub Pages]
---

## Overview

Finding a quiet place to work in the MacLeod building is a small, constant problem. Study space is tight, the lounges are loud, and the general-purpose classrooms sit empty for large parts of the day — but nobody knows which ones, or for how long, without pulling up eight separate timetables and doing the arithmetic.

I built a small static site that does the arithmetic. It shows, for each of the eight MacLeod classrooms, whether it is free right now, how long it stays free, what's coming in next, and — for the rooms that are in use — when they open up. Tap a room for its full day; pick a date and time to plan ahead.

**Try it: [dylancunliffe.github.io/mcld-rooms](https://dylancunliffe.github.io/mcld-rooms/)**

Key points:

- **Pure static site** — one HTML file plus a generated data file. No backend, no framework, no build step; it runs on GitHub Pages for free and works on a phone
- **Timetable parser** in Python that turns the university's room-timetable exports into a compact JSON schedule, de-duplicating multi-room and multi-instructor rows and reading the academic week calendar from the export itself
- **Correct in any timezone** — all "now" calculations are pinned to Vancouver local time, so it doesn't lie to someone checking from elsewhere
- **Refreshable in one command** — ad-hoc bookings get added all term, so the data is a re-export away from current

### System Flow

1. Export the list-view room timetables from UBC's Scientia web timetable (needs a campus login, so this step stays manual)
2. `build.py` parses the exports, merges duplicates, expands week ranges (`3-7, 9-16`) into explicit week lists, and writes `schedule.js`
3. The page loads `schedule.js`, works out the current academic week and weekday, and merges each room's overlapping bookings into busy blocks
4. Rooms are sorted free-first (longest free window on top), then busy rooms by soonest-free, and re-evaluated every 30 seconds

---

## Repository 📂

Source, parser, and refresh instructions:
**[Github Link](https://github.com/dylancunliffe/mcld-rooms)**

---

## Getting the Data Right

The first attempt used the timetable's *grid* view — the familiar week-at-a-glance chart. It looked complete and it wasn't: the export silently clipped every day at noon, so a room with a 5–8 pm lecture appeared free all evening. Screenshots of a chart are not data.

The *list* view exports a plain table — activity, weeks, location, day, start, end — which is exactly what a parser wants. A few things still needed handling:

- **Multi-room bookings** appear once per room, and **multi-instructor sections** once per instructor, so a single lecture can be four rows. The parser keys on (room, day, start, end, weeks, name) and merges the rest.
- **Week numbers are relative** to the timetable year's week 1. Rather than hard-code the date, the parser reads it from the export header and refuses to run if the header isn't a Monday or if two files disagree — the failure mode where a stale export sneaks in and shifts every booking by a week is worth guarding against.
- **Bookings overlap.** Two sections back-to-back, a tutorial inside a lecture slot, a maintenance block over everything — the page merges each room's day into non-overlapping busy intervals first, then answers "free until when?" against those.

> ![MacLeod free rooms screenshot](/assets/img/mcld-rooms-thumb.png)
*Image: the live page mid-morning on a Monday*

---

## Keeping It Honest

The site only knows about timetabled bookings. It says so in the footer, along with the date of the last export, because a study-spot finder that quietly drifts out of date is worse than none. Refreshing is deliberate but quick: re-save the list view into `raw/`, run `python build.py`, push. The parser prints per-room event counts so a missing export is obvious before it goes live.

# Pepitune

*Personal project, 2026 · solo build · code is private; this is a write-up.*

**An AI web app that organises an entire Spotify library: it tags every song by genre,
language and mood at ~1,000 songs a minute, then turns those tags into playlists,
insights and a chat assistant.**

| | |
|---|---|
| **Role** | Product, design and full-stack development, solo |
| **Stack** | Next.js 14, TypeScript, Supabase (Postgres), Vercel, Gemini via OpenRouter, Whisper |
| **Scale** | 10+ external data sources · 62 API endpoints · 59 database migrations · ~1,000 songs/min |
| **Status** | Live at [pepitune.ai](https://pepitune.ai), in private beta |

## The problem

Spotify only knows genre per artist, not per song, and knows nothing about language or
mood. With thousands of liked songs, organising by hand is impossible.

## Features

*Each feature: what it does, then what makes it hard. Animations are from the app's landing page and use its example library.*

### 1. Library sync

Connect Spotify and the whole library comes in: songs, playlists, albums and artists.

**Under the hood:** Spotify OAuth, with paginated sync of the full library into
Postgres. Row-level security in Postgres means users only ever see their own data. Per-user AI quotas and rate limiting keep the paid APIs under control.

<img src="../images/pepitune-home.gif" alt="Overview page" width="100%">

### 2. Automatic tagging, the core

Every song gets a **language, genre, sub-genre and mood**, and you can filter by any mix
of them (e.g. Italian + Pop + Energetic).

**Under the hood:** a 10-phase pipeline that orchestrates **10+ services**: lyrics
providers, Shazam audio matching, Spotify, Genius and Apple Music catalogues, Whisper
speech-to-text and an LLM. It runs 100 songs concurrently, falls back to the next
source when one misses, tries free sources before paid ones (e.g. a 30,000-word
dictionary before any API for language), and ends with a **supervisor LLM** that
re-checks inconsistent tags before saving.

<img src="../images/pepitune-detection.gif" alt="Songs with language, genre, sub-genre and mood tags" width="100%">

### 3. Playlists

All playlists in one place, including new ones built from tags, synced back to Spotify
in one click.

**Under the hood:** playlists are written back through the Spotify Web API, with tracks
added in batches and a custom cover uploaded. A playlist made in Pepitune is a real
Spotify playlist, not a copy.

<img src="../images/pepitune-playlists.gif" alt="Playlists page" width="100%">

### 4. Pepito, the AI assistant

Ask in plain English: "make a chill playlist for a rainy Sunday", "what's my top genre?",
"find energetic songs that aren't in any playlist".

**Under the hood:** the LLM answers with structured JSON actions, chosen from **19 action
types** (filter, stats, create/merge/delete playlists, edit tags, re-run detection…).
Each one is validated against a schema before it runs, and destructive actions wait for
the user to confirm. Vague requests like "workout" are matched against example songs,
then the LLM re-ranks the user's own tracks to pick the ones that fit.

<img src="../images/pepitune-pepito.gif" alt="Pepito chat" width="100%">

### 5. Analytics

A listening identity ("The Global Wanderer"), your mood mix, and a globe of where your
music comes from.

**Under the hood:** none of this exists in Spotify's data. It's computed from the tags
from step 2, aggregated per user in Postgres.

<img src="../images/pepitune-analytics.png" alt="Analytics page" width="100%">

## Architecture

```mermaid
flowchart LR
    U((User)) --> APP["Next.js app<br/>(Vercel)"]
    APP --> API["62 API routes<br/>(serverless)"]
    API --> DB[("Supabase<br/>Postgres + Auth + RLS")]
    API --> SP[Spotify Web API]
    API --> PIPE["Tagging pipeline"]
    PIPE --> EXT["Lyrics, Shazam,<br/>music catalogues"]
    PIPE --> AI["LLM (Gemini) +<br/>Whisper"]
    API --> AI
```

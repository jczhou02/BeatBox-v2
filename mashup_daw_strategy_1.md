# Mashup DAW — Architecture & `generate_mashup_timeline` Strategy

A working design document for a previously implemented stack of Next.js (TypeScript) + Supabase + Python web app that lets users build harmonic mashups of real songs, powered by Hooktheory-scraped metadata and Demucs stem separation.
Review this document with imtemt to question any design decisions and improve existing ideas with more technically sound practices!
---

## 1. System Overview

**Goal:** Manage the lifecycle of music DAW mashup projects for real users, from search → analysis → stem extraction → timeline synthesis → in-browser editing → save/return.

### High-Level Flow

1. **Search (Frontend / Next.js) --or-- Playlist:** Spotify powers the search UX- alternately, user can supply the link to their public playlists and we can target songs from there. Once, User picks a "anchor" song; queries against the master Hooktheory CSV public table on Supabase are run using **title likeness** and **artist likeness** (fuzzy matching, since the Hooktheory table's format varies slightly from Spotify's).
2. **Row fetch (Supabase):** Fetch the matching rows for the user's requested Spotify songs. Rows contain rich metadata: **BPM, chord progression (`cp` jsonb), melody, key, section timestamps/cues**.
3. **Backend processing (Python):** Rows are sent to the Python backend for:
   - YouTube audio extraction (allows users to simulate and hear their generated mixing ideas conveniently)
   - Demucs stem separation (vocals, drums, bass, other)
   - Personally tailored harmonic synthesis / mashup logic
4. **Timeline result (Backend → Frontend):** The backend returns a **JSON mashup timeline** — timestamp placements of each song piece, BPM, key, and other music annotations.
5. **Playback & editing (Frontend):** Web Audio API and/or Tone.js drive the DAW experience — playing, panning, moving, and editing the mashup audio stems.

### What to Do with the Extracted Stems

Stems are **binary audio artifacts**, not row data:
supabase example:
- **Storage layout:** One bucket (e.g. `stems`), keyed by a **canonical song identifier** (e.g. `stems/{song_id}/{vocals|drums|bass|other}.mp3`). Key by *song*, not by *user or project* — this is what makes the system scale when many users mash up the same songs.
- **Deduplication / cache-first:** Before running Demucs, check whether stems for that song already exist in Storage. Extract once, serve many. Demucs is the expensive step; storage of processed stems is the cache.
- **A `songs` / `stems_cache` table** in Postgres tracks which songs have been processed, their storage paths, duration, and processing status — so the backend can skip work and the frontend can stream immediately.
- **Delivery:** Frontend fetches stems via signed URLs (private bucket) or public URLs (public bucket) and loads them into Web Audio API / Tone.js buffers.
- **Free-tier realism:** Supabase free tier has limited storage (~1 GB). Use compressed formats (mp3/ogg, not wav), consider trimming stems to only the sections used, and add an eviction/LRU policy for rarely used songs if needed.

### Auth, Guest Workflow, and Project Persistence

- **Guest mode:** Unauthenticated users can use the full mashup DAW. Two good options:
  - Keep guest state entirely client-side (in-memory / localStorage of the timeline JSON) — stems are fetched read-only from the shared cache, so no auth is needed to *play*.
  - Or use **Supabase anonymous sign-ins**, which give the guest a real `auth.users` row that can later be **converted to a permanent account** (email/OAuth link), carrying their project with them.
- **Save requires registration:** When a guest wants to save, prompt sign-up. On success, persist the project. With anonymous auth, this is just linking an identity; with pure client-side guest state, you insert the project on first save.
- **Project persistence schema (Postgres):**
  - `projects` — `id`, `user_id` (FK → `auth.users`), `name`, `created_at`, `updated_at`
  - `project_tracks` / `project_stems` — which songs/stems a project uses (FK → songs/stems cache)
  - `projects.timeline` (jsonb) — the full timeline document: placements, timestamps, gain/pan, edits. A single jsonb column is simpler and perfectly fine at this scale versus fully normalized placement rows.
- **RLS (Row Level Security):** Enable on all user tables; policies like `user_id = auth.uid()` so users only see their own projects. The Hooktheory master table and stems cache are read-only public (or read-only to authenticated + anon roles).

---

## 2. Proposed Architecture & Workflow (Detailed)

A refined end-to-end workflow incorporating Supabase for stem and project management.

### 2.1 Frontend (Next.js): Search & Selection

- User searches for songs.
- The Next.js frontend calls a Next.js API route (`/api/search`).
- The API route queries the **Spotify API**.
- Based on Spotify results (track ID, artist name, title), the API route queries the Supabase `hooktheory_data` table using the Supabase JS client (`supabase.from('hooktheory_data').select()` with `.ilike()` or similar for fuzzy matching).
- The API route returns relevant Hooktheory row(s) (IDs, BPM, chords, melody cues) to the frontend.
- User selects the desired songs on the frontend.

### 2.2 Backend Request (Next.js API Route → Python)

- Frontend sends the selected Hooktheory song identifiers (and possibly Spotify IDs / YouTube URLs needed for download) to a Next.js API route (`/api/process-mashup`).
- This API route acts as a **proxy/orchestrator**: it forwards the request containing the song identifiers to the Python backend service (Flask or FastAPI).

### 2.3 Python Backend: Processing & Stem Handling

**Receive request:** the Python backend receives the list of songs to process.

**Check for existing stems (crucial optimization):**

- For each requested song, create a unique identifier (e.g., hash the YouTube URL, or use the Spotify Track ID).
- Query a `source_audio_stems` table to see if stems for this exact source identifier already exist in Supabase Storage. This table maps `source_identifier` → `storage_path` per stem type.
- **If stems exist:** retrieve their storage paths from `source_audio_stems`. No re-download or re-processing needed.
- **If stems don't exist:**
  1. Use the Hooktheory data / Spotify ID to find and download the audio source (e.g., YouTube via `yt-dlp`).
  2. Run **Demucs** to extract stems (vocals, drums, bass, other).
  3. **Upload stems to Supabase Storage** using `supabase-py`, at a deterministic, unique path (e.g., `stems/[source_identifier_hash]/vocals.mp3`, `stems/[source_identifier_hash]/drums.mp3`). This ensures each unique stem is stored only **once**, regardless of how many users or projects use it.
  4. **Record stem metadata:** insert rows into `source_audio_stems` storing the `source_identifier` and `storage_path` for each uploaded stem type.

**Mashup logic:** perform harmonic synthesis, timeline generation, etc., using the Hooktheory data. This produces the JSON defining the mashup (timestamp placements, BPM adjustments, segment info).

**Prepare & send response:** return to the Next.js API route —

- the generated timeline/placement JSON, and
- the Supabase Storage **paths** for all required stems (both newly generated and pre-existing).

### 2.4 Frontend (Next.js): Loading & Playback

- The `/api/process-mashup` route receives the Python backend's response and forwards the timeline JSON + stem storage paths to the browser.
- **Get public URLs:** on the frontend, use the Supabase JS client: `supabase.storage.from('your-bucket-name').getPublicUrl('path/to/stem.mp3')`. Ensure the bucket has public read access configured (or use signed URLs for more security — public URLs are simpler initially).
- **Load audio** with Web Audio API / Tone.js:
  - Fetch the audio data from the public URLs.
  - Load into buffers (`AudioBuffer` in Web Audio API; `Tone.Player` / `Tone.Sampler` in Tone.js).
  - Use the timeline JSON to schedule playback, apply panning, volume, and effects — the interactive DAW experience.

### 2.5 Guest Workflow & Saving

- A guest goes through steps 2.1–2.4 and interacts with the mashup in-browser, using stems fetched from public URLs in Supabase Storage. Stems are stored efficiently thanks to the de-duplication in 2.3.
- **Save action — if the guest clicks "Save":**
  - Prompt Sign Up / Log In via **Supabase Auth** (`@supabase/auth-helpers-nextjs` works well here).
  - Once authenticated, the frontend sends the current project state (timeline JSON + list of stem storage paths in use) to `/api/projects/save`.
  - **Backend save logic:**
    - The route verifies the user is authenticated (Supabase server-side helpers).
    - It creates a row in the `projects` table: `id`, `user_id` (from `auth.uid()`), `name`, `timeline_json`, `created_at`, `updated_at`.
    - A linking table (`project_stems`) associates the `project_id` with the `source_identifier` / `storage_path` of each stem used in that arrangement — tracking which stems belong to which project even though the stem *files* are shared.
    - **RLS** on `projects` (`auth.uid() = user_id`) ensures users only access their own projects.

### 2.6 Logged-in User Workflow

- **Loading an existing project:**
  1. User logs in.
  2. Frontend calls `/api/projects` to fetch the user's projects (query `projects` where `user_id` matches).
  3. User selects a project to load.
  4. Frontend calls `/api/projects/[projectId]` to get that project's `timeline_json` and associated stem `storage_path`s (joining `projects` and `project_stems`).
  5. Proceed as in 2.4 (get public URLs, load audio).
- **Saving:** same as 2.5's save action, but may **update** an existing project row instead of creating a new one.

### 2.7 Supabase Setup

**Auth:** Supabase Auth for user management.

**Database tables:**

| Table | Columns | Notes |
|---|---|---|
| `hooktheory_data` | Status	artist	title	section	Chord Progression	Key	Scale	BPM	Meter	Danceability	Energy	Loudness	Acousticness	Instrumentalness	Liveness	Valence	Duration (ms)	Genres	Time Signature	cp	Melody	YouTube ID	Start Timestamp (s)	End Timestamp (s)	id	source_track_id	cp_compare	beatUnit | Public master table |
| `profiles` | `user_id` (FK → `auth.users`), extra user info | Optional |
| `source_audio_stems` | `id`, `source_identifier` (text, unique — e.g. YouTube ID hash or Spotify ID), `stem_type` (text, e.g. `'vocals'`), `storage_path` (text), `created_at` | **Crucial for de-duplication** |
| `projects` | `id` (uuid, PK), `user_id` (uuid, FK → `auth.users`), `name` (text), `description` (text, optional), `timeline_json` (jsonb), `created_at`, `updated_at` | Enable **RLS** for user ownership |
| `project_stems` | `id`, `project_id` (uuid, FK → `projects`), `stem_storage_path` (text, references paths in `source_audio_stems`), `project_specific_settings` (jsonb, optional — e.g. volume/pan for this stem in this project) | Linking table; RLS based on project ownership |

**Storage:**

- Create a bucket (e.g., `stems`).
- Set access policies — start with **public read** for ease of development. Tighten later with signed URLs or storage RLS if needed. Public URLs for stems are common and efficient when the raw stems aren't highly sensitive IP; the project *arrangement* (timeline JSON) is what's protected by RLS in the database.
- Organize files with the deterministic paths above: `stems/[source_identifier_hash]/[stem_type].mp3`.

### 2.8 Benefits of This Approach

- **Scalability:** stems are stored centrally and efficiently; processing happens only once per unique source song.
- **Cost-effectiveness (free tier):** Supabase Storage is more cost-effective for files than DB storage. Deduplication drastically reduces storage needs, keeping you within free/low tiers longer. Database usage for metadata is minimal.
- **Guest workflow:** handled seamlessly without scattering temporary files. Stems are generated and stored permanently (but efficiently) from the start.
- **Data integrity:** project data (timeline, stem references) is stored securely in the database with user ownership enforced by RLS.
- **Performance:** the frontend loads stems directly from Supabase's CDN-backed Storage; the Python backend focuses solely on processing, not file serving.
- **Best practices:** leverages Supabase Auth, Database (with RLS), and Storage as intended.

### 2.9 Key Takeaway

**Don't send stems back to the frontend from Python.** Instead:

1. Python uploads stems to Supabase Storage (if they don't already exist).
2. Python returns the **storage paths** (and timeline JSON) to the Next.js backend.
3. The Next.js backend sends the storage paths (and timeline JSON) to the browser.
4. The browser uses the Supabase JS client to get public URLs from the paths and loads audio via Web Audio API / Tone.js.

---

## 3. `generate_mashup_timeline` — Section-Matching Strategy

### Context & Assumptions

- Input: `requested_tracks`, guaranteed to contain **at least two tracks**. The POST `/mashup` `route.ts` handler either:
  - enhances the user's requested tracks via `fetchCompatibleSections()`, or
  - already has a sufficient number of tracks, depending on the `mode` parameter in the frontend's POST request.
- **Exactly one track has `anchor: true`.** The resulting mashup timeline draws heavily from the anchor track's chord progression (the `cp` jsonb column fetched from Supabase).

### Core Algorithm

Iterate through the **anchor track's sections in order** (e.g. Intro → Verse → Pre-Chorus → Chorus → Bridge → Instrumental — some tracks may be missing sections). For **each anchor section**, find the best-matching section from the **non-anchor tracks** (iterate their sections, or build a dictionary/lookup for speed), using a **priority-ranked elimination process**:

Start with a **candidate pool** of all non-anchor track sections. Apply the rules below **in order**; after each rule, keep only candidates that pass. If exactly one candidate remains, stop — that's the match. If more than one remains, proceed to the next rule.

#### Rule 1 — Key Compatibility (Camelot Wheel)

Accept a candidate if its key is harmonically compatible with the anchor section's key, using the **Camelot wheel**: major and minor keys form a circular relationship labeled **1–12**, each with an **A (minor)** and **B (major)** variant.

A candidate key matches when it is any of:

- **Same number, same letter** — identical key
- **Same number, different letter** (nA ↔ nB) — relative major/minor
- **±1 number, same letter** — adjacent on the wheel (12 and 1 wrap around)
- **±7 semitone relationship** (as specified: plus/minus 7 also accepted)

#### Rule 2 — Section Role Matching

If multiple candidates remain, prefer candidates whose section plays the **same structural role** as the anchor section, classified via:

```python
section_mapping = {
    'Bridge':                'Misc',
    'Chorus':                'Drop',
    'Chorus 2':              'Drop',
    'Chorus 3':              'Drop',
    'Chorus Lead-Out':       'Drop',
    'Instrumental':          'Drop',
    'Intro':                 'Misc',
    'Intro and Verse':       'Misc',
    'Outro':                 'Misc',
    'Outro 2':               'Misc',
    'Pre-Chorus':            'Build-Up',
    'Pre-Chorus and Chorus': 'Drop',
    'Pre-Outro':             'Misc',
    'Solo':                  'Drop',
    'Solo 1':                'Drop',
    'Solo 2':                'Drop',
    'Verse':                 'Build-Up',
    'Verse and Pre-Chorus':  'Build-Up',
}
```

Map both the anchor section and each candidate section through `section_mapping`; keep candidates whose mapped class (`Drop` / `Build-Up` / `Misc`) equals the anchor's.

#### Rule 3 — Chord Progression Similarity

If candidates still remain, compare the **Chord Progression** metadata:

- **Exact match** on the chord number sequence, or
- **Fuzzy match:** subset containment, rotations, or similar patterns/progressions (e.g. shared n-grams of chords, longest common subsequence).

Keep the candidate(s) with the strongest progression similarity.

#### Rule 4 — Sensible Fallback Assignment

If candidates still remain, assign remaining tracks to the anchor section using a clean, deterministic strategy (good coding practice — e.g. round-robin across non-anchor tracks so every track gets used, or least-recently-used track first).

#### Rule 5 — Random Tiebreak

If more than one candidate *still* remains, choose one at random.

### Process Notes

- The selection is a **filter-down pipeline**: for a given anchor section, each rule prunes the candidate pool (add/remove candidates by match criteria per track); once all tracks have been evaluated for a rule, if `len(pool) > 1`, advance to the next rule and repeat.
- Run this **entire workflow once per anchor section, in section order**, guaranteeing every anchor section is paired with one (or several) non-anchor sections that should mix well.
- **Output:** a coded representation of the matched sections + metadata (anchor section, matched track/section, keys, BPMs, chord progressions) — the input for the *next* helper function.



## 4. Table-based Single Track Compatible/Mixable Songs Seeker
A standalone feature that may also act as a alternative step to the spotify query step where users want to find songs to run the mashup api on: users should be able to give a single target song (we have access to its metadata- i.e, key,bpm,cp,melody,etc) and we should return a modern table like view with compatible songs based on the user's targeted parameters or attributes. for instance, i know for now we should at least support this parameters/modes: lyrical (finds specific songs and their lyrical sections that contain the phrase or word the user wants). by default, we can assume the qualifying candidates or base filtered pool of song choices to choose from at least follow the camelot key matching rule explained earlier. Lyrical data for songs needs to be something we have to figure out. I only know so far that musixmatch api is a solid approach to gaining access to lyrics, I am also open to fetching and storing this as a new column data as well.
## 5. Tech Stack Summary (open to more modern,practical suggestions)

| Layer | Technology | Role |
|---|---|---|
| Frontend | TypeScript | Search UI, DAW timeline editor, stem playback/pan/edit |
| Search | Spotify API + fuzzy title/artist matching | Song discovery mapped to Hooktheory rows |
| Database | Supabase Postgres (+ RLS) | Hooktheory master table, users, projects, timelines (jsonb), stems cache index |
| Storage | Supabase Storage | Extracted stems, keyed per-song for dedup/reuse |
| Auth | Supabase Auth (incl. anonymous sign-ins) | Guest → registered upgrade path, project ownership |
| Backend | Python (Demucs, YouTube extraction) | Stem separation, harmonic synthesis, timeline generation |

## Final Notes Considerations
We should be open to any suggestions for database, and fullstack architecture choices based on your judgment.
Spotify api for developers features seems to be not feasible since their is a limited user count for features that require logging in. This is why when users want to see how they can create a dj set from their spotify playlists they themselves will need to supply a link to their public spotify playlist. 
There is room for AI integration. Besides the rule based matching algorithm (which was described a bit earlier and can be seen in action in the python backend code of the beatbox repo), we can also implement our own AI to intelligently choose the tools we already discussed (keys, bpm, lyrical, melodic or chord phrasing matching) along with hugging face musical phrase detection to decide this for the user (gatekept as a premium,paid feature in our platform); more details can be explored in future conversations.

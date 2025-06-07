# SnapCon v1-Lite — Product Requirements Document  
*Date: 7 Jun 2025* · *Repo: single Next.js 14*

---

## 1 · Purpose  
Provide a **zero-install, QR-first unconference tool** that:

* lets organisers spin up an event in minutes;  
* uses AI to transform raw attendee interests into clear session topics;  
* produces a conflict-free schedule for a fixed set of rooms;  
* keeps everyone updated in real time;  
* preserves notes + ratings and generates an end-of-day recap.

---

## 2 · Problem / Opportunity  
Whiteboards and sticky-notes break down beyond ~60 people, slow the process, and lose knowledge. A mobile-web workflow with lightweight AI removes friction, scales cleanly, and captures outcomes without extra hardware, apps, or logins.

---

## 3 · Goals & KPIs

| Goal                       | Target (v1) |
|----------------------------|-------------|
| Join speed                 | ≤ 15 s median from QR scan |
| Topic coverage             | ≥ 85 % interests merged |
| Schedule confidence        | 0 room/time clashes at publish |
| Participant engagement     | ≥ 90 % cast ≥1 vote |
| Knowledge capture          | ≥ 70 % sessions with ≥1 takeaway |
| Satisfaction (NPS)         | > 60 |

---

## 4 · Personas

| Persona                | Core Needs |
|------------------------|-----------|
| **Participant “Pat”**  | Join fast, propose ideas, avoid clashes, get recap |
| **Host “Olivia”**      | Energise room, spotlight top ideas, final-approve schedule |
| **Assistant “Alex”**   | Rapidly add a host-requested topic from a tablet |
| **Facilitator “Sam”**  | Claim a session, collect notes, solicit feedback |

---

## 5 · End-to-End Flow (concise)

| Min | Action                                                                                                 | Driver |
|-----|--------------------------------------------------------------------------------------------------------|--------|
| 00  | Host: “Scan this QR, type your name, list things you’d love to discuss.” Live Wall projected.          | Host |
| 01-03 | Interests stream in → **AI seeds 3-5 ✨ starter cards**.                                              | AI |
| 03-08 | Attendees vote ▲; every 60 s **AI spawns new ✨ spin-off cards** from fast-rising votes.              | AI |
| 04-09 | Host polls room for extra ideas; Assistant taps **“+ Add Topic”** (📣 badge).                         | Host / Asst. |
| 08  | Host pauses wall; top cards shown; proposers tap **“Claim”** to facilitate (card turns green).         | Host + Participants |
| 10  | Host presses **“Build Schedule.”** ILP solver assigns all claimed / AI / 📣 topics into fixed rooms.   | Host |
| 10-11 | Host drags overflow sessions to larger rooms → instant update (toast + vibration) to phones.          | Host |
| 11  | Host taps **“Publish.”** Personal agendas pushed; sessions begin.                                      | Host |
| During | Notes & ★ ratings entered; host can still move sessions; toasts keep phones synced.                | Participants |
| End | System compiles notes/ratings ⇒ **Recap PDF** downloadable (SMTP e-mail optional).                     | System |

---

## 6 · Functional Requirements

| ID  | Feature                              | Key Requirements & Acceptance Criteria |
|-----|--------------------------------------|----------------------------------------|
| **F1** | **QR-based Join**               | Single join URL → QR poster. Form: **Name (required)**, Email (optional). 24 h JWT cookie. **AC:** 95 % test users reach Wall ≤ 15 s on 3 G. |
| **F2** | **Interest Capture**            | 1-5 bullet inputs, autosave. Optional weight slider 1-5. **AC:** interest row saved ≤ 1 s; chip counter updates. |
| **F3** | **AI Topic Suggestions**        | After ≥ 5 interests, HDBSCAN + GPT-4o label (≤ 6 words). ✨ badge until ≥ 3 votes. Every 60 s check vote velocity → spawn spin-off. **AC:** new ✨ card ≤ 30 s after trigger; profanity filtered. |
| **F4** | **Manual Host Topic (📣)**      | Host/assistant add title via admin wall; 📣 badge drops after ≥ 3 votes. **AC:** card visible to all clients < 1 s. |
| **F5** | **Live Voting**                 | One-tap up-vote/un-vote; optimistic UI; WebSocket ≤ 250 ms. **AC:** totals converge across 100 clients < 1 s. |
| **F6** | **Schedule Builder**            | Fixed rooms (name, capacity). ILP solver maximises attendance, respects capacity & facilitator conflicts; unplaceable topics go to Overflow. **AC:** publish yields 0 conflicts; solver ≤ 2 s for 1 k interests. |
| **F7** | **Realtime Grid UI**            | Columns = rooms; drag card: left/right → room, up/down → time. Capacity badge colours (green/amber/red). **AC:** move propagates to all clients ≤ 300 ms. |
| **F8** | **Notifications**               | PWA toast + vibration on schedule changes / publish. Recap download link shown; optional SMTP e-mail if organiser configures creds. **AC:** 95 % pushes ≤ 2 s; recap ready ≤ 6 h. |
| **F9** | **Session Workspace**           | CRDT pad for notes; star rating + takeaway; “Add Facilitator” toggle. **AC:** edits from two devices merge without conflict. |

---

## 7 · AI Pipeline

1. **Embedding** – `text-embedding-3-large`; stored in `Interest.vector` (`pgvector`).  
2. **Initial Clustering** – HDBSCAN when first ≥ 5 interests.  
3. **Label Generation** – GPT-4o prompt: “Summarise in ≤ 6 words …”.  
4. **Vote-Velocity Spin-off** – every 60 s spawn “Deep-dive” card when votes/sec > 2× mean.  
5. **Schedule Optimisation** – ILP via `javascript-lp-solver`; constraints: fixed rooms, capacity, no facilitator overlap.  
*Fallback:* TF-IDF labels + greedy room assignment if OpenAI fails.

---

## 8 · Data Model (Prisma – simplified)

```prisma
model Room        { id String @id; eventId String; name String; capacity Int; sessions Session[] }
model Participant { id String @id; eventId String; name String; email String?; joinMethod String }
model Interest    { id String @id; userId String; raw Text; weight Int; vector Vector @db.Vector(1536) }
model Topic       { id String @id; eventId String; label String; status String } // 'suggested' 'manual' 'confirmed'
model Vote        { id String @id; userId String; topicId String }
model Session     { id String @id; topicId String; roomId String; startAt DateTime; endAt DateTime; facilitatorId String? }
model Note        { id String @id; sessionId String; userId String; body Text }
model Rating      { id String @id; sessionId String; userId String; stars Int; takeaway Text? }
```

## 9 · External Dependencies

| Service        | Purpose                     | Status |
|----------------|-----------------------------|--------|
| **OpenAI API** | Embeddings + GPT-4o labels  | **Required** – guarded by rate-limit logic with TF-IDF fallback. |
| Google One-Tap | *Optional* quick auth       | Disabled by default; organiser can enable via a checkbox (adds a 15-line JS snippet). |
| SMTP           | *Optional* recap email      | Organiser supplies host/user/pass; if omitted, recap is download-only. |

_No other third-party APIs are used in v1-Lite._

---

## 10 · Security & Compliance Highlights

* **Transport** – HTTPS only; HSTS and TLS 1.2+.
* **Sessions** – Strict same-site JWT cookies (24 h); HTTP-only.
* **Data Protection**  
  * Interests contain PII → encrypted at rest with a per-event AES-256 key.  
  * Automatic hard-delete of all event data and recaps after **30 days**.
* **Rate-Limiting** – Basic IP + JWT limits on all POST routes to mitigate abuse.
* **Privacy** – GDPR/CCPA notice; self-serve “Delete my data” link in recap page.
* **Accessibility** – Mobile web meets WCAG 2.1 AA (contrast, keyboard nav, aria labels).

---

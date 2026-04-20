# Carebot Deployable Chatbot — Feature Documentation

**CS 495 Capstone Project | University of Alabama**
**Last Updated: April 2026**

---

## Table of Contents

1. [User Account Types](#1-user-account-types)
2. [Chat — Queries & Responses](#2-chat--queries--responses)
3. [User Feedback](#3-user-feedback)
4. [Sorting & Filtering Feedback (Admin)](#4-sorting--filtering-feedback-admin)
5. [Feedback Graphs & Analytics Dashboard](#5-feedback-graphs--analytics-dashboard)
6. [Text-to-Speech (TTS)](#6-text-to-speech-tts)
7. [Speech-to-Text (STT)](#7-speech-to-text-stt)

---

## 1. User Account Types

| Role | Login Required | Access |
|---|---|---|
| **Anonymous / Public** | No | Chat, submit resource recommendations, leave feedback, view dashboard |
| **Staff / Moderator** | Yes — `/accounts/login/` | All of the above + `/manage/` section: review/approve/reject resource submissions, similarity search, resource merge tools |
| **Django Admin (Superuser)** | Yes — `/admin/` | Full database access, Gemini-powered feedback theme classification, user/permission management |

### How account types link to other features

- The **public user** role is the entry point for all other user-facing features: chat responses are what generate feedback records, and that feedback is what appears in the admin panel and dashboard charts.
- **Staff** accounts exist specifically to act on resource recommendations submitted by public users — a submission by a public user shows up in the `/manage/` queue for a staff member to approve or reject.
- **Django Admin** accounts are required to run Gemini theme analysis (see [Section 4](#4-sorting--filtering-feedback-admin)), which is what populates the theme breakdown chart visible to everyone on the dashboard (see [Section 5](#5-feedback-graphs--analytics-dashboard)).

---

## 2. Chat — Queries & Responses

**Sprint: Previous Group**

### What the chatbot queries

The chatbot uses two sources to answer questions:

- **Google Gemini API** — natural language understanding and general healthcare knowledge. The model is configured via the `GEMINI_MODEL` environment variable (default: `models/gemini-2.5-flash`).
- **SQL Database** — structured provider records including hospitals, nursing homes, home health agencies, practitioners, and medical suppliers located in Alabama.

### What types of questions it responds to

The chatbot is scoped to **Alabama residents seeking healthcare services in Alabama**. It handles two response modes:

1. **General knowledge answers** — plain-text responses about healthcare topics, benefits, and services (e.g., "What is Medicare?", "How do I apply for food stamps?").
2. **Resource lookups** — database-driven searches that return provider listings filtered by resource type and/or location (city or county).

Supported resource types include (but are not limited to): support groups, food stamps, counseling, Medicare, nursing homes, hospitals, specialists, housing assistance, hospice, and 40+ additional categories.

> **Out-of-scope requests** — queries for healthcare resources outside Alabama are declined. The chatbot will inform the user it can only assist with Alabama-based services.

### How chat results link to other features

- Every chatbot response includes a **"Leave feedback"** button — this is the entry point for the feedback system described in [Section 3](#3-user-feedback).
- Every chatbot response also includes a **🔊 replay button** that triggers the Text-to-Speech feature described in [Section 6](#6-text-to-speech-tts).
- The user's location is recorded with each chat request (if permitted). This location data is what populates the heatmap and county request table on the analytics dashboard (see [Section 5](#5-feedback-graphs--analytics-dashboard)).
- Chat messages are stored with a `conversation_id` and `message_id` that are used to associate feedback records with the specific response that was rated.

---

## 3. User Feedback

**Sprint: Sprint 1**

Feedback can be left on any chatbot response. No account is required.

### How to give feedback

1. After a chatbot response appears, click the **"Leave feedback"** button below the message.
2. Click **👍** if the response was helpful. A thank-you message appears immediately — no further action needed.
3. Click **👎** if the response was not helpful. An optional comment box will appear.
4. Optionally type a comment explaining what was wrong (e.g., wrong info, unclear answer).
5. Click **Submit** to send your feedback.

Feedback is submitted to `/api/feedback/` and stored alongside the message and conversation IDs.

### How feedback links to other features

- Submitted feedback is stored in the database and immediately reflected in the **analytics dashboard** at `/dashboard/` — the feedback ratio chart, feedback-over-time chart, and feedback-by-county chart all update with new submissions (see [Section 5](#5-feedback-graphs--analytics-dashboard)).
- Thumbs-down feedback with comments can be reviewed and classified by a Django Admin in the admin panel at `/admin/chat/chatfeedback/` (see [Section 4](#4-sorting--filtering-feedback-admin)).
- Feedback records are tied to the specific chat message and conversation they came from, so admins can trace a piece of feedback back to the exact exchange that prompted it.

---

## 4. Sorting & Filtering Feedback (Admin)

**Sprint: Sprint 2**

Feedback records are managed in the Django admin panel at **`/admin/chat/chatfeedback/`**. A Django Admin (Superuser) account is required.

### Filtering options

| Filter | Options |
|---|---|
| **Rating** | Thumbs Up / Thumbs Down |
| **Date** | Filter by `created_at` date |
| **Theme tags** | `wrong info`, `out of date`, `unhelpful`, `unclear`, `other`, `Not yet analyzed` |
| **Comment text** | Full-text search on the comment field |

### Sorting

Click any column header to sort by that field. Available sort columns: **ID**, **Rating**, **Comment**, **Themes**, **Created At**.

### Gemini theme analysis

The admin list page shows a **theme summary** at the top breaking down negative feedback by category. To classify unanalyzed thumbs-down feedback automatically:

1. Select one or more feedback records (or select all).
2. Choose **"Analyze themes with Gemini (thumbs down only)"** from the Actions dropdown.
3. Click **Go**. Gemini will assign theme tags and save them to each record.

Themes are displayed as color-coded badges on each feedback row.

### How admin feedback tools link to other features

- Feedback records displayed here originate from the **👍/👎 buttons** on the chat page (see [Section 3](#3-user-feedback)) — every record in this list was submitted by a public user after receiving a chatbot response.
- Running the Gemini theme analysis action updates the `themes` field on each record. Those theme values are what the **Feedback Theme Breakdown** chart on the analytics dashboard reads from — running the analysis is what keeps that chart populated and up to date (see [Section 5](#5-feedback-graphs--analytics-dashboard)).
- Only thumbs-down feedback with comments is meaningful for theme analysis; thumbs-up records appear in this list but the Gemini action skips them automatically.

---

## 5. Feedback Graphs & Analytics Dashboard

**Sprint: Sprint 2**

The analytics dashboard is publicly accessible at **`/dashboard/`**. No login is required.

### Available charts and visualizations

| Chart | Type | What it shows |
|---|---|---|
| **Request Heatmap** | Leaflet.js map | Geographic distribution of chat requests across Alabama counties |
| **Location of Requests** | Table | County-by-county request counts |
| **Feedback Ratio** | Pie/doughnut chart | Overall thumbs-up vs. thumbs-down totals |
| **Feedback Theme Breakdown** | Horizontal bar chart | Count and percentage for each negative feedback theme (`wrong info`, `out of date`, `unhelpful`, `unclear`, `other`) |
| **Feedback Over Time** | Line chart | Daily positive (green) and negative (red) feedback percentages |
| **Feedback by County** | Grouped bar chart | Thumbs-up and thumbs-down counts side by side per county |

### How the dashboard links to other features

- All feedback data shown in the charts (ratio, over-time, by-county, theme breakdown) comes from records submitted via the **chat feedback buttons** (see [Section 3](#3-user-feedback)). The charts will be empty if no feedback has been submitted.
- The **Feedback Theme Breakdown** chart only shows meaningful data after a Django Admin has run the Gemini theme analysis action on the feedback records (see [Section 4](#4-sorting--filtering-feedback-admin)). Unanalyzed records do not appear in this chart.
- The **Request Heatmap** and **Location of Requests** table are driven by location data recorded during chat sessions — each chat request logs a county if the user's location is available.

---

## 6. Text-to-Speech (TTS)

**Sprint: Sprint 1**

TTS uses the browser's native **Web Speech API** — no extra software or plugins required. A female English voice is preferred; the first available English voice is used as a fallback.

### How to enable TTS

1. Click the **⚙️ (gear)** button in the chat interface to open voice settings.
2. Check the **"TTS On"** checkbox to enable automatic read-aloud for all new chatbot responses.

### Adjusting speech rate

In the voice settings panel, use the **−** and **+** buttons to decrease or increase the reading speed in 0.1× increments. The current rate is displayed between the buttons (e.g., `1.00×`). The range is **0.5× to 2.0×**.

### Replaying a response

Every chatbot message has a **🔊 replay button** beneath it. Click it to hear that specific response read aloud. Clicking it again while it is playing will stop playback.

> TTS will not auto-play for error messages even if the toggle is on.

### How TTS links to other features

- The 🔊 replay button is attached to every **chatbot response** on the chat page — TTS reads the same text the chatbot produced through the Gemini API or database lookup (see [Section 2](#2-chat--queries--responses)).
- TTS and STT are both controlled from the same **⚙️ voice settings panel**, making them companion features. A common workflow is to use STT to speak a question and TTS to hear the answer read back (see [Section 7](#7-speech-to-text-stt)).

---

## 7. Speech-to-Text (STT)

**Sprint: Sprint 1**

STT uses the browser's native **Web Speech Recognition API** — no extra software or plugins required. Recognized speech is transcribed in English (US).

### How to use STT

1. Click the **⚙️ (gear)** button to open voice settings.
2. Click the **🎤 (microphone)** button. The icon changes to 🔴 to indicate the microphone is active.
3. Speak your question clearly.
4. When you stop speaking, the transcribed text is automatically placed in the chat input field.
5. Review the text and press **Send** as normal.

### Notes

- STT captures a **single utterance** — it stops listening after you finish speaking or after **7 seconds** of silence.
- Click the microphone button again at any time to cancel listening early.
- If the browser does not support the Web Speech API, the microphone button will not function. Chrome and Edge have the best compatibility.

### How STT links to other features

- STT populates the **chat input field** — the transcribed text is sent to the chatbot exactly as if the user had typed it, and is processed through the same Gemini API and database lookup pipeline (see [Section 2](#2-chat--queries--responses)).
- STT and TTS share the **⚙️ voice settings panel**. Using them together enables a fully hands-free interaction: speak a question with STT, receive a chatbot response, and have it read back via TTS.

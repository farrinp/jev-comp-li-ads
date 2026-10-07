# Build your own LinkedIn competitor-ad tracker with Claude Code.

This guide builds a Chrome extension that quietly records every competitor ad you browse in the
LinkedIn Ad Library, then uses an AI model to tag each one. You get one spreadsheet row per ad, covering
format, call to action, funnel stage, message theme, tone, the emotion it's aiming for, and what the
viewer is meant to take away.

You don't write any code. You paste one prompt into Claude Code, and it builds and tests the extension.
Then you load the extension into Chrome and start browsing.

## What you need

- **Claude Code.** See https://claude.com/claude-code.
- **Google Chrome** on a desktop computer.
- **An OpenRouter account with some credit,** and an API key from https://openrouter.ai/keys. The
  extension tags ads with Jev, which is on OpenRouter as `typesafe/jev-router`. You can switch to any
  other OpenRouter model in the extension's settings.
- **Node.js,** if you want to run the extension's tests. Claude Code will tell you if it's missing.

## Step 1: Start Claude Code in an empty folder

Make a new folder, open a terminal in it, and run:

    claude

## Step 2: Paste this prompt

Copy the whole block and paste it into Claude Code as your first message. It's long on purpose. It
records how LinkedIn's pages are built and the problems hit along the way, so Claude Code gets it right
first time.

```text
Build a Manifest V3 Chrome extension called "Jev Ad Library Tagger" for the LinkedIn Ad Library
(https://www.linkedin.com/ad-library/). As I browse competitors' ads, save each ad into a database inside
the extension, tag it with Jev via OpenRouter, and let me export everything as CSV. Build it in this
folder: manifest.json, src/, test/, package.json and README.md. No build step: plain JavaScript, ES
modules for the background worker and popup, classic scripts for content scripts. Never write my API key
into any file; I'll enter it in the extension's popup.

== Connecting to Jev ==
- OpenRouter chat completions: POST https://openrouter.ai/api/v1/chat/completions, Bearer key, headers
  HTTP-Referer https://www.linkedin.com/ad-library/ and X-Title "Jev Ad Library Tagger". temperature 0.
- Default model "typesafe/jev-router" (Jev's OpenRouter id), changeable in the popup.
- Send 8 ads per request. Use response_format json_schema (strict) with enums for every fixed list. If
  OpenRouter returns 400 mentioning response_format/json_schema/structured, retry once without the schema.
- Validate every reply yourself: strip code fences, parse, match values case-insensitively to the allowed
  lists, drop tags for ad ids not in the batch, and reject an ad's tags if any value is outside its list.
- 401/403 = bad key, 402 = no credit: pause tagging (leave ads pending, show the problem in the popup and
  the page badge). 429/5xx/network errors: count an attempt and retry via the alarm. After 3 failed
  attempts an ad is marked failed; a "Retry failed" button re-queues them.

== Tags (Jev fills these; keep the lists at the top of src/jev.js) ==
- format: Single image, Video, Carousel, Document, Text, Spotlight, Follower, Event, Message, Conversation,
  Thought leader, Job, Other. If the ad is posted by a person on a company's behalf (posted_as), use
  "Thought leader" whatever LinkedIn's label says. Otherwise use LinkedIn's label, mapped.
- cta: Learn more, Register / Attend, Download, Sign up, Get started, Start free trial, Request demo,
  Contact sales / Get quote, Buy / Shop, Subscribe, Apply, Follow, Join, Other, Unknown. Use the real
  button if known; otherwise an explicit ask in headline/text ("Get started", "Register", "Book a demo",
  "Try it free"; asks for a price or quote are "Contact sales / Get quote"); otherwise "Unknown" (never
  "None": LinkedIn ads nearly always have a button we didn't see).
- funnel_stage: awareness (brand, ideas, thought leadership, news; no real ask), consideration (webinars,
  events, guides, reports, case studies, "learn more"), conversion (demo, trial, pricing, quote, contact
  sales, buy, "get started" or sign up for the product). Judge from the ask first. Recruitment ads
  ("Apply") are awareness unless they sell a product.
- user_learning: free text, one plain-English sentence of at most 20 words saying what a viewer takes away,
  whatever language the ad is in. State the takeaway, not a description of the ad. Collapse whitespace,
  cap at 200 characters.
- message_theme: Product feature, Customer story, Thought leadership, Research or report, How-to or
  education, Event or webinar, Offer or pricing, Brand or mission, Award or recognition, Partnership,
  Company news, Recruitment, Other.
- user_emotion (likely emotion in the target viewer): Curiosity, Trust, Confidence, Inspiration,
  Excitement, Reassurance, Concern, Urgency, Fear of missing out, Belonging, Pride, Amusement, Neutral
  (only for purely factual ads).
- tone: Professional, Conversational, Authoritative, Inspirational, Technical, Empathetic, Playful, Bold,
  Urgent, Celebratory.
- Keep a TAG_VERSION constant. Store it on each tagged ad. When the worker starts, re-queue tagged ads
  whose version differs, so changing the instructions re-tags everything.
- Send Jev per ad: ad_id, advertiser, posted_as, linkedin_format, text, headline, cta_button,
  image_description, has_video (omit empty fields, truncate long fields to 1500 chars).

== Reading LinkedIn's pages (this was the live markup when written; LinkedIn may have changed it) ==
Search results (/ad-library/search?...):
- Each ad is li.search-result-item containing .base-ad-preview-card. The ad id comes from the link
  a[href*="/ad-library/detail/<id>"].
- The card's aria-label is "<Advertiser>, <Format> Ad, View details"; take the last segment ending in "Ad"
  as linkedin_format ("Single Image Ad", "Video Ad", "Document Ad", ...).
- Header: .self-center. Its first line is who posted. If a line reads "Promoted by <Company>", the ad is a
  thought-leader ad: advertiser = that company, posted_as = the person.
- Body: .commentary__content (search cards truncate long text with "…"). Headline:
  .sponsored-content-headline h2 (join several with " | "). Image description: alt of
  img.ad-preview__dynamic-dimensions-image. Video: presence of video or a video tracking attribute.
- Document ads have no headline. On search cards the card has class
  sponsored-update-native-document-preview and the title is .bg-color-canvas span.font-semibold. On detail
  pages the title is doc.title inside the JSON in iframe[data-native-document-config].
- Search cards do NOT show the ad's button. "Load more results" appends more cards to the same list.
Detail pages (/ad-library/detail/<id>):
- Card: .ad-detail-left-rail .base-ad-preview-card. Button: [data-tracking-control-name$="_cta"]
  (e.g. "Attend", "Apply", "Learn more"; thought-leader and document ads usually have none).
- Right rail .ad-detail-right-rail: format is the first <p> ending in "Ad"; "Paid for by X" is
  .about-ad__paying-entity; advertiser link [data-tracking-control-name="ad_library_about_ad_advertiser"].
- Mark ads read from a detail page detailChecked = true.
- Normalise text: non-breaking spaces to spaces, collapse runs of spaces, trim around newlines. innerText
  falls back to textContent on parsed (unrendered) documents, so the same code must work on DOMParser output.

== Content script (src/extract.js pure DOM reading + src/content.js) ==
- Runs on https://www.linkedin.com/ad-library/*. Scan on load and on DOM mutations (debounce 700 ms).
  Send only ads whose fields changed since last sent from this page.
- Show a small fixed badge bottom-right in a closed shadow root: "Jev: N ads saved · M tagging", plus
  "finding buttons" and any problem. Stop quietly if the extension context is invalidated.
- Background button lookups (on by default, toggle in popup settings): for ads on the current search page
  whose detail page hasn't been read, fetch /ad-library/detail/<id> same-origin, parse with DOMParser, run
  the detail extractor, save. One request every 6 seconds: LinkedIn returned 429 on the 5th back-to-back
  request but allowed one every 6 s. On 429, 999 or 5xx pause lookups for 10 minutes. If a page is
  unreadable or gone, save {adId, detailChecked: true} so it isn't fetched again. Never crawl beyond the
  ads on the page I'm viewing, and never go faster than this.

== Storage and background worker (src/db.js, src/records.js, src/background.js) ==
- IndexedDB database "jev-ad-library", store "ads" keyed by LinkedIn ad id, index on status
  (pending / tagged / error).
- Merging a re-seen ad: a blank or false incoming field never overwrites a stored one (search cards lack
  what detail pages have); keep firstSeen and the first found_on URL; update lastSeen. If a field Jev reads
  changes (advertiser, posted_as, linkedin_format, body, headline, cta_button, image description,
  has_video), set the ad back to pending. paid_by alone doesn't trigger a re-tag.
- Queue: pick pending ads; when lookups are on, skip ads whose detail page hasn't been read unless first
  seen over 10 minutes ago, so each ad is tagged once with its button. A once-a-minute chrome.alarms alarm
  resumes the queue after the service worker sleeps.
- When Jev answers, apply the tags to the ad AS STORED NOW in one transaction, never to the copy that was
  sent: skip ads deleted meanwhile (so "Delete all" can't be undone by an in-flight batch), and skip ads
  whose Jev-read fields changed meanwhile (they stay pending and get re-tagged with the new text).
- Wrap every message reply as {ok: true, value} or {ok: false, error}. Do not signal errors with an "error"
  field on the value: the stats object has an "error" count, which would read as an error.
- Toolbar badge shows the number of pending ads.
- Key and settings in chrome.storage.local (say in the README it's local and unsynced).

== Popup (src/popup.html/css/js, ~360 px wide, light and dark) ==
- Four stat tiles: saved, tagged, waiting, failed.
- One row of buttons: Export CSV (primary), Retry failed (disabled when none), Delete all (danger, right
  aligned). Delete needs two clicks within 4 s: first click turns it into "Confirm delete" and says
  "Nothing is deleted yet. Click Confirm delete to remove all N saved ads."; afterwards say "Deleted N ads.
  Any Ad Library tab you reload will save its ads again."
- Latest 8 ads: advertiser · LinkedIn format, headline, and tag chips (format, cta, stage, theme,
  emotion, tone) or "waiting for Jev" / "failed: reason".
- Settings (collapsed unless no key): API key (password field, never shown back), model, the
  button-lookup checkbox with "Slower: one LinkedIn page every 6 seconds".
- Refresh every 2 s. Build DOM with textContent, never innerHTML with ad text.

== CSV export ==
- File linkedin-ads-YYYY-MM-DD.csv, UTF-8 with BOM, CRLF, RFC 4180 quoting. Prefix cells starting with =, +,
  -, @, tab or CR with ' (ad text is written by other companies). Booleans as yes/no.
- Columns, in order: ad_id, creative_group, ads_in_group, advertiser, posted_as, paid_by, linkedin_format,
  cta_button, headline, body, image_description, has_video, detail_url, found_on, first_seen, last_seen,
  tag_status, tag_error, then all of Jev's columns last: jev_format, jev_cta, jev_funnel_stage,
  jev_user_learning, jev_message_theme, jev_user_emotion, jev_tone, jev_model.
- creative_group (G1, G2… by first appearance) groups ads with the same advertiser, format, headline and
  first 80 characters of body (lower-cased, whitespace collapsed, trailing "…" removed), because
  advertisers run one creative under many ad ids and search cards truncate text that detail pages don't.
  ads_in_group is the group's size.

== Tests (npm test) ==
- test/run.mjs, no dependencies: merging rules, retry/give-up, payload schema and enums, reply validation,
  schema fallback, auth vs transient errors, CSV quoting/formula guard/BOM/column order, creative grouping
  (a truncated and a full copy of the same ad share a group), flags never flip back to false, TAG_VERSION
  re-tagging, prompt contains the thought-leader, quote and Unknown rules.
- test/background.test.mjs with fake-indexeddb (devDependency): load the real background.js with stubbed
  chrome.* APIs and a fetch whose replies the test releases by hand. Cover: save and tag; Delete all while
  a batch is with Jev leaves the database and CSV empty; a detail-page update arriving mid-request survives
  and is re-tagged; tagging waits for the button lookup; needsDetail returns only unread ads; old-version
  tags are re-tagged; the stats reply isn't mistaken for an error. Confirm the race tests fail against
  the naive "put the sent copies back" implementation.
- Run the tests and fix failures before finishing.

== Check it against the live site ==
LinkedIn changes its markup, so the selectors above may be out of date. Before finishing:
- If you can drive a browser, run the actual src/extract.js against a live search page and a detail page,
  including a thought-leader ad, a video ad and a document ad, and against a detail page fetched with
  fetch() and DOMParser.
- If you can't, give me one short snippet to paste into Chrome's DevTools console on a search results
  page and one for a detail page. Each should run the extractor and print what it found. Use what I paste
  back to fix the selectors.
- If you can view pages, check the popup at its real width with sample data (stub chrome.runtime in a
  local preview) in its normal, "Confirm delete" and settings states.

== README.md and handover ==
README: what it does; install (chrome://extensions → Developer mode → Load unpacked → choose this folder;
pin it; paste an OpenRouter key from https://openrouter.ai/keys under Settings; reload the extension
after any change); how to use; the column table; the tag lists and where to edit them; behaviour
(batching, retries, pausing on key problems, lookups and their pace, waiting for buttons, local storage,
formula guard); running tests (npm install once, then npm test).
When you're done, tell me in plain words how to install it, what you verified, and anything you couldn't.
```

Claude Code will ask permission before running commands. Read each one before you approve it. When it
finishes, it tells you what it checked and how to install the extension.

## Step 3: Install the extension

1. In Chrome, go to `chrome://extensions` and turn on **Developer mode**, top right.
2. Click **Load unpacked** and choose the folder Claude Code built.
3. Pin the extension from Chrome's puzzle-piece menu.
4. Open its popup, expand **Settings**, paste your OpenRouter key and click **Save**.

If you change anything later, click the circular reload arrow on the extension's card in
`chrome://extensions`.

## Step 4: Collect ads

1. Go to https://www.linkedin.com/ad-library/ and search for one competitor by company name. A keyword
   search can also match similarly named companies.
2. Scroll, and click **Load more results** until you have the period you care about. A counter in the
   bottom-right corner shows how many ads are saved and how many are still being tagged.
3. Leave the tab open for a few minutes. In the background, the extension reads each ad's detail page,
   one every 6 seconds, to get its real button. Allow about 2½ minutes per 24 ads.
4. When the popup shows nothing waiting, click **Export CSV**.
5. Repeat for other competitors. Ads build up in one database, so one export can cover several. Click
   **Delete all** to start a fresh study.

## Step 5: Make sense of the export

Open the CSV in Google Sheets, Excel or Numbers. LinkedIn's data comes first, and Jev's tags are the
columns starting with `jev_` at the end.

- **Funnel mix.** Pivot advertiser against jev_funnel_stage. A competitor spending most of its ads on
  awareness is building a brand. One heavy on conversion is harvesting demand.
- **Count creatives, not ad IDs.** Advertisers run one creative under many ad IDs for different
  audiences and tests. Count distinct creative_group values to see how many messages they're running, and
  use ads_in_group to see which ones they back hardest. In testing, 96 ad IDs turned out to be 31
  creatives.
- **What they ask for.** Compare cta_button, the real button, with jev_cta, the normalised ask. Lots of
  "Learn more" and few "Request demo" buttons says a lot about how a competitor sells.
- **What they talk about.** jev_message_theme shows the content mix: customer stories, product features,
  events, research, recruitment. Filter by theme to read every customer story a competitor runs.
- **How they sound.** Cross jev_tone with jev_user_emotion to map each competitor's voice.
- **What they claim.** jev_user_learning is one sentence per ad on what the viewer should take away.
  Read them per competitor to find the claims they repeat, and the gaps your own messaging could own.
- **Formats.** linkedin_format and jev_format show the creative mix: video, image, document, and
  thought-leader posts from executives.
- **Who pays.** paid_by sometimes names an agency rather than the brand.

## Make it yours

Ask Claude Code, in the same folder, for changes in plain English. For example:

- "Add a column for the target persona, chosen from: founders, marketers, engineers, executives."
- "Change the message themes to fit the cybersecurity market."
- "Use a different OpenRouter model by default."

Saved ads are tagged again automatically when the tag instructions change.

## Good to know

- **It only records what you browse.** It's a record of the ads you looked at, not a full archive.
- **first_seen and last_seen are when you saw an ad,** not when it ran.
- **Tags are a model's judgement.** Spot-check a sample before you present numbers. Use cta_button
  rather than jev_cta when you need the literal button.
- **Some ads have no button.** Thought-leader and document ads usually don't, so their call to action
  comes from the text, or is "Unknown".
- **Your data stays on your computer,** in your Chrome profile, along with your API key. Only ad text is
  sent to OpenRouter for tagging. Removing the extension deletes the database, so export first.
- **Use it responsibly.** The extension only reads public Ad Library pages you open yourself, plus one
  detail page every 6 seconds for the ads on screen. It backs off if LinkedIn asks it to slow down. Check
  that your use fits LinkedIn's terms and your company's policies.

## If something goes wrong

| What you see | What to do |
|---|---|
| The counter never appears, or says 0 ads saved | LinkedIn has probably changed its page layout. Ask Claude Code to re-check the selectors. It will give you a console snippet to run on the page. |
| "OpenRouter rejected the API key" | Paste a fresh key from https://openrouter.ai/keys into Settings and save it. |
| "Out of credit" | Add credit to your OpenRouter account. Tagging resumes by itself. |
| "LinkedIn asked to slow down" | Nothing. Lookups resume after 10 minutes, and tagging carries on without buttons in the meantime. |
| Ads stuck on "waiting" | Keep the Ad Library tab open. Ads wait up to 10 minutes for their button before being tagged without it. |
| Deleted ads come back | An open Ad Library tab was reloaded, and it saved its ads again. Close the tab before deleting. |

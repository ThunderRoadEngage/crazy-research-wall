# Jaryd Atadero: seed dataset (Crazy Research Wall)

Built 2026-11-23 (America/Chicago) by Research Analyst for SuperBot and Mystic.

## Scope
This dataset covers the disappearance of 3-year-old Jaryd Atadero on Oct 2, 1999, on the Big South Trail (Comanche Peak Wilderness, Roosevelt National Forest, Poudre Canyon, Larimer County, CO). It runs through the 1999 search, the June 2003 clothing and remains recoveries, the DNA work and death certificate, the legislation that followed (PL 106-324, Colorado AMBER Alert), the hoaxes, and the 2025 death of Allyn Atadero.

## Files
| file | contents |
|---|---|
| entities.csv | 136 entities (id, type, name, properties as `key=value;...`, notes). Id prefixes: P (person), L (location), O (organization), E (event), D (document), V (vehicle), X (object). |
| relationships.csv | 168 relationships (id R###, from_id, to_id, rel_type, date, status = confirmed/disputed/speculative, confidence 0–1) |
| evidence.csv | 54 evidence rows (id EV###, claim, supports = `;`-separated entity/relationship ids, source_title, source_url, source_date, source_type, reliability) |
| contradictions.md | 26 documented disagreements between sources, cited by evidence id and URL |
| media.csv + media/ | 21 images downloaded from Mystic's case page (see the sensitivity note below) |
| scripts/build.py | Regenerates the three CSVs and validates ids. Edit this file, then run `python3 scripts/build.py`. |
| _raw/ | Fetched snapshots of the user's case page (HTML/text) and the two research PDFs with pdftotext output |

## Method
- Public sources only, gathered with WebSearch, WebFetch, and curl. Every URL in evidence.csv was either fetched or returned as a search result; rows built from search excerpts only say so in source_title (Coloradoan 2025, Greeley Tribune 2003, Dalton FH obituary).
- Every entity and relationship is cited by at least one evidence row. The build script checks that every id referenced exists and that nothing is left unsupported.
- `status` meanings: **confirmed** = stated by official or reputable news sources; **disputed** = sources conflict, or the claim is an allegation; **speculative** = only a low-reliability or single interested-party source supports it (forum, or the user's own write-up where it couldn't be checked against an independent source).
- Mystic's write-up (Thunder Street Ministry case page plus "Full Disclosure" Parts One and Two) is logged as `source_type=research` at reliability 0.5 and flagged as the user's own compilation. Its claims attributed to the LCSO supplemental report or to Allyn's 2017 book are recorded as the write-up's account of those documents. They have not been checked against the originals.
- Respect: the victim was a child and family members are living. Family appear only in their public roles. No addresses or contact details are included. The one hiking-group member who matters here appears only under the write-up's pseudonym "Janet". The remains photos are described from their captions only and were not viewed.

## Reliability scale (evidence.reliability)
- 0.90–0.95: official records (Public Law, Congressional Record, CBI, USFS)
- 0.80–0.85: contemporaneous wire or major news (AP, CNN, Denver Post 1999–2003)
- 0.75–0.80: later retrospective news built on interviews or the SAR report (Coloradoan 2019, 9News, Denver7, Greeley)
- 0.60–0.65: Wikipedia and search-excerpt-only sources
- 0.50: the user's own research write-up (secondary; cites the Sheriff's report and Allyn's book). Individual allegations are scored 0.30–0.40.
- 0.30: CanAm Missing Project / Missing 411 (an advocacy-oriented secondhand account)

## Sources
Coloradoan 2019 (Blumhardt, with the SAR-report timeline) · AP 2019 · Coloradoan 2025 (excerpt) · Denver7 2024 · 9News 2003 and 2019 · Denver Post 1999-10-07 and 2000-10-03 · CNN 1999-10-05 · LA Times/AP 1999-10-05, 10-07, 10-09 · Deseret/AP 1999-10-06, 10-07 · CBS 1999-10 and 2003-06-11 · Plainview Herald/AP 2003-06-09 · Greeley Tribune 2003-06-11 and 2007-11-18 · Congressional Record vol. 146 p. 16127 · Public Law 106-324 · CBI AMBER Alert page · USFS Big South Trailhead · Wikipedia · Dalton FH obituary (excerpt) · CanAm Missing Project · Thunder Street Ministry case page, Full Disclosure Parts One and Two. Full URLs are in evidence.csv.

## Media (sensitivity note)
All 21 images come from https://thunderroadengage.github.io/thunder-road-engage/cases/jaryd-atadero/: one portrait, 2 recovery-site photos, 8 lab photos of the jacket, and 10 photos of the recovered skull fragment. The 10 remains photos (`remains-*.jpg`) show a child's remains. The source page hides them by default, so they should stay hidden by default in the app. These files are third-party images included for private research reference; source URLs for each file are listed in `media.csv`. The write-up credits some evidence photos to LCSO. Their provenance and rights are not otherwise documented.

## Pending: Larimer County Sheriff's report (user providing 2026-11-24)
Mystic will provide the Sheriff's report tomorrow. It is entity **D002**: LCSO Supplemental Field Incident Report 99-06-59.S02, cited by TRE1. When it arrives, add it as a primary `official` source and re-score the TRE-derived rows. It may resolve these items:
- C01, C12: the exact distances and elevations of the last-seen point, the clothing site, and the remains site, and whether that slope was searched by air before the crash.
- C05, C06: what DOW and trackers concluded about the tracks (track age, bear against child), the dive teams' probability-of-detection figures, and the sheriff's stated working theory.
- C15, C16, C17: the group's composition and splits, witness statements, the exact times (departure, the 11:20/11:30 sightings, the scream), and whether there was a Poudre Falls stop.
- C09, C10, C11: search size, duration, and the suspension date.
- C19, C18: hair and CBI lab results, and the chain of custody for the tooth and skull.
- C24, C23: how the Mesa Verde tip was handled, the Fort Collins woman's charge, and how family and volunteers were given trail access.
- Fills gaps: the fishermen's statements, the "older couple" and the runner the write-up mentions, and the tracker "Seaux"'s reports.

## Known gaps
- The SAR report, Sheriff's report, NecroSearch report, CBI lab reports, death certificate text, and Air Force mishap report are not publicly posted. They are known only through news or the write-up's quotes.
- The coroner's name was not verified. A search synthesis named one, but it couldn't be traced to a fetched source, so it was left out.
- Dr. Michael Baden's 2003 Fox News comments (TRE/TRE1) could not be verified. They are logged as speculative.
- TRE1's claim that a cadaver dog search off Pingree Park Rd found a teen murder victim in a mine shaft could not be verified. It was not added.
- The identities of the two fishermen and the 11 group members were never published. Rod "Seaux" Larreau's name appears only in the write-up.
- Allyn's books (2008, 2017) were not read directly. They are cited through Wikipedia and TRE1.
- The date of the Ritter proclamation and the date the Fort Collins woman was charged are unknown.

## Biggest open questions
1. Where are the exact last-seen, clothing, and remains sites, and was that slope covered by air or foot in 1999? (C01, C12)
2. Is the official "undetermined, probable mountain lion attack" finding consistent with the clothing evidence: no blood, pants inside out, shoes nearly unweathered, no lion hair? Or do the NecroSearch caveats and the specialists' doubts weaken it? (C04, C19, C20)
3. What did the DNA work actually show? That covers the 86% initial result, the alleged mixed profile in the tooth, and when, by whom, and how the 100% match was made. (C18)
4. What did the hairs on the clothing come from, and why was that result not shared with the family? (C19)
5. Were the Mesa Verde sighting and other tips properly run down, and what was the Fort Collins woman charged with? (C24)
6. Why do the accounts of the finders' prior familiarity with the area differ, and when did they first see the items? (C13, C03)

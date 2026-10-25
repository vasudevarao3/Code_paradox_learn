# 80-Day Plan: System Design, ML & AI, LLM & GenAI (Code Paradox)

Oct 24, 2026 · @Vasu

## Overview

You can finish all three Code Paradox tracks in 80 days (Sun Oct 25, 2026 to Tue Jan 12, 2027) with about 8 to 9 hours a week: 1 hour on weekdays, 2 hours on Saturday, 1 hour on Sunday. No day goes above 2 hours.

**What the Software System Design track contains** (read from the [Learning](https://codeparadox.io/learning) and [Roadmaps](https://codeparadox.io/roadmaps) pages):

- Foundations: 10 sections, 97 chapters, 40 labs (Compute & Traffic, Data & Storage, Caching & Delivery)
- Advanced: 10 sections, about 60 chapters, 40 labs (Events & Communication, Service & Resilience, Production Operations)
- Expert: 25 full design problems in 5 groups (Core Backend Systems 5, Consumer Systems at Scale 5, High-Integrity Platforms 4, Search & Platform Services 5, Infrastructure Systems 6), of which 13 are the most commonly asked (Netflix, WhatsApp, Uber, YouTube, Twitter/X Feed, Stripe, Dropbox, Instagram, Ticketmaster, Slack, URL Shortener, Notification Platform, Distributed Rate Limiter)

**Notes:**

- ML & AI System Design (10 sections, 46 chapters, 40 labs) and LLM & GenAI Systems (4 sections, 45 chapters, 8 labs) are scheduled from the section lists you sent. Chapter-heavy days in those two tracks run up to 1.25 hr; nothing goes over 2 hr.
- The site lists no chapter or lab durations. I assumed about 6 minutes per chapter (so 8 to 9 chapters fit the 50-minute reading block) and 15 minutes per lab. Design problems have site durations: URL Shortener 20, Notification Platform 22, Rate Limiter 22, Netflix 24, Twitter/X Feed 24, Stripe 24, Ticketmaster 24, Dropbox 25, Instagram 25, Uber 26, Slack 26, WhatsApp 28, YouTube 30 minutes. The 6-chapter size is shown for the first Advanced sections; I assumed the other Advanced sections match.
- On the site, Reliability Patterns belongs to Advanced stage 2 (Service & Resilience). Production Operations holds Autoscaling, Observability, Deployment and Cost. The schedule order doesn't change.

## Daily rhythm

Every week uses the same template, so you never have to decide what to do today: weekdays are for reading chapters, Friday to Sunday are for labs and design problems.

| Day | Time | What you do |
| --- | --- | --- |
| Mon to Thu | 1 hr | Read one section's chapters (about 50 min), then write 3 bullets of notes (10 min): the idea, one trade-off, one number worth remembering |
| Fri | 1 hr | Do 4 labs |
| Sat | 2 hr | Do 4 to 8 labs, or design problems in the checkpoint weeks |
| Sun | 1 hr | Finish remaining labs, then a 10-minute review of the week's notes |

**Three habits that keep it consistent:**

- Start each Monday with 5 minutes recalling last week's 3-bullet notes without looking.
- Fixed time slot: pick one (for example 6:30 to 7:30 pm) and keep it every day.
- In Advanced weeks (Nov 16 to Nov 29) each weekday adds 1 lab to the chapter hour, so the weekend carries fewer labs. Weekdays stay at 1 hr.

## Phase map

Software System Design takes the first 6 weeks because it is the base for the other two tracks; ML & AI gets 2 weeks, LLM & GenAI gets 11 days, and the last 12 days hold the remaining Expert problems, mocks and final review.

| Phase | Dates | Days | Target | Done when |
| --- | --- | --- | --- | --- |
| 0. Kickoff | Sun Oct 25 | 1 | Set up notes and tracker | Notes template ready |
| 1. Software System Design | Mon Oct 26 to Sun Dec 6 | 2 to 43 | 20 sections, 80 labs, 13 popular design problems | You can design a URL shortener, a feed and a chat app out loud in 25 min |
| 2. ML & AI System Design | Mon Dec 7 to Sun Dec 20 | 44 to 57 | 10 sections, 46 chapters, 40 labs | Roadmap shows all sections and labs done |
| 3. LLM & GenAI Systems | Mon Dec 21 to Thu Dec 31 | 58 to 68 | 4 sections, 45 chapters, 8 labs | Roadmap shows all sections and labs done |
| 4. Remaining Expert and mocks | Fri Jan 1 to Tue Jan 12 | 69 to 80 | 12 remaining Expert problems, 3 mocks, final cheat sheet | Expert shows 25 of 25 |

## Phase 1: Software System Design, day by day

Weeks 1 to 3 cover Foundations, weeks 4 and 5 cover Advanced, week 6 covers Expert design problems. Tick each row off as you finish it.

| Day | Date | Time | Do |
| --- | --- | --- | --- |
| 1 | Sun Oct 25 | 1 hr | Kickoff: skim all three tracks on Learning, create your notes doc with the 3-bullet template, bookmark Roadmaps to watch the solved counters |
| 2 | Mon Oct 26 | 1 hr | Servers & Compute: chapters 1 to 8 |
| 3 | Tue Oct 27 | 1 hr | Servers & Compute: chapters 9 to 16 |
| 4 | Wed Oct 28 | 1 hr | Load Balancers & Traffic Routing: chapters 1 to 9 |
| 5 | Thu Oct 29 | 1 hr | Application State & Sessions: chapters 1 to 9 |
| 6 | Fri Oct 30 | 1 hr | Servers & Compute: 4 labs |
| 7 | Sat Oct 31 | 1.5 hr | Load Balancers: 4 labs, then redraw client to load balancer to servers to session store from memory |
| 8 | Sun Nov 1 | 1 hr | Application State: 4 labs, then 10-minute week review |
| 9 | Mon Nov 2 | 1 hr | Relational Databases: chapters 1 to 9 |
| 10 | Tue Nov 3 | 1 hr | Database Scaling & Replication: chapters 1 to 9 |
| 11 | Wed Nov 4 | 1 hr | Partitioning & Sharding: chapters 1 to 9 |
| 12 | Thu Nov 5 | 1 hr | Object Storage & File Systems: chapters 1 to 9 |
| 13 | Fri Nov 6 | 1 hr | Relational Databases: 4 labs |
| 14 | Sat Nov 7 | 2 hr | Database Scaling & Replication: 4 labs, then Partitioning & Sharding: 4 labs |
| 15 | Sun Nov 8 | 1 hr | Object Storage & File Systems: 4 labs, then 10-minute week review |
| 16 | Mon Nov 9 | 1 hr | Caching: chapters 1 to 9 |
| 17 | Tue Nov 10 | 1 hr | CDNs & Edge Delivery: chapters 1 to 9 |
| 18 | Wed Nov 11 | 1 hr | Queues & Background Processing: chapters 1 to 9 |
| 19 | Thu Nov 12 | 1 hr | Checkpoint: Design URL Shortener (20 min), then Design Notification Platform (22 min), then 15 min of notes |
| 20 | Fri Nov 13 | 1 hr | Caching: 4 labs |
| 21 | Sat Nov 14 | 2 hr | CDNs & Edge Delivery: 4 labs, then Queues & Background Processing: 4 labs |
| 22 | Sun Nov 15 | 1 hr | Design Distributed Rate Limiter (22 min), then start a one-page Foundations cheat sheet (30 min) |
| 23 | Mon Nov 16 | 1 hr | Pub/Sub & Event-Driven Systems: chapters 1 to 6, lab 1 |
| 24 | Tue Nov 17 | 1 hr | Idempotency, Ordering & Delivery Guarantees: chapters 1 to 6, lab 1 |
| 25 | Wed Nov 18 | 1 hr | APIs & Service Communication: chapters 1 to 6, lab 1 |
| 26 | Thu Nov 19 | 1 hr | Service Boundaries & Microservices: chapters 1 to 6, lab 1 |
| 27 | Fri Nov 20 | 1 hr | Consistency & Distributed Data: chapters 1 to 6, lab 1 |
| 28 | Sat Nov 21 | 2 hr | 8 labs: remaining 3 of Pub/Sub, 3 of Idempotency, 2 of APIs |
| 29 | Sun Nov 22 | 2 hr | 7 labs: last 1 of APIs, 3 of Service Boundaries, 3 of Consistency |
| 30 | Mon Nov 23 | 1 hr | Reliability Patterns: chapters 1 to 6, lab 1 |
| 31 | Tue Nov 24 | 1 hr | Autoscaling & Capacity Management: chapters 1 to 6, lab 1 |
| 32 | Wed Nov 25 | 1 hr | Observability & Production Diagnostics: chapters 1 to 6, lab 1 |
| 33 | Thu Nov 26 | 1 hr | Deployment & Release Architecture: chapters 1 to 6, lab 1 |
| 34 | Fri Nov 27 | 1 hr | Cost & Architecture Optimization: chapters 1 to 6, lab 1 |
| 35 | Sat Nov 28 | 2 hr | 8 labs: remaining 3 of Reliability, 3 of Autoscaling, 2 of Observability |
| 36 | Sun Nov 29 | 2 hr | 7 labs: last 1 of Observability, 3 of Deployment, 3 of Cost |
| 37 | Mon Nov 30 | 1 hr | Expert: Design Dropbox and Design Instagram (about 25 min each), 10 min notes |
| 38 | Tue Dec 1 | 1 hr | Expert: Design Ticketmaster and Design Slack |
| 39 | Wed Dec 2 | 1 hr | Expert: Design Netflix and Design WhatsApp |
| 40 | Thu Dec 3 | 1 hr | Expert: Design Uber and Design Twitter/X Feed |
| 41 | Fri Dec 4 | 1 hr | Expert: Design YouTube and Design Stripe |
| 42 | Sat Dec 5 | 2 hr | Redo your 2 weakest problems from memory, out loud, 25 min each; then finish the Software cheat sheet (components, numbers, trade-offs) |
| 43 | Sun Dec 6 | 1 hr | Phase 1 checkpoint: redo URL Shortener cold in 20 min; list 5 weak topics to revisit in the Jan buffer |

## Phases 2 to 4: ML & AI, LLM & GenAI, and the buffer

ML & AI System Design takes 14 days: chapters first (days 44 to 48), then labs (days 49 to 56), then a wrap-up. LLM & GenAI Systems has few labs but long chapter lists, so it takes 11 days. The freed days move 7 Expert problems into Jan 1 to Jan 3 and keep real buffer and mocks at the end.

**ML & AI System Design** (S1 ML System Architecture Fundamentals 5 chapters, S2 Data Collection & Labeling 4, S3 Data Pipelines & Preprocessing 5, S4 Feature Engineering & Feature Stores 5, S5 Training Infrastructure 5, S6 Experiment Tracking & Reproducibility 5, S7 Model Evaluation & Validation 5, S8 Model Registry & Versioning 4, S9 Batch Inference 4, S10 Online Inference & Model Serving 4; every section has 4 labs)

| Day | Date | Time | Do |
| --- | --- | --- | --- |
| 44 | Mon Dec 7 | 1.25 hr | S1 chapters 1 to 5, S2 chapters 1 to 4 |
| 45 | Tue Dec 8 | 1.25 hr | S3 and S4 chapters (10) |
| 46 | Wed Dec 9 | 1.25 hr | S5 and S6 chapters (10) |
| 47 | Thu Dec 10 | 1.25 hr | S7 and S8 chapters (9) |
| 48 | Fri Dec 11 | 1 hr | S9 and S10 chapters (8) |
| 49 | Sat Dec 12 | 2 hr | S1 labs (4), S2 labs (4) |
| 50 | Sun Dec 13 | 1 hr | S3 labs (4) |
| 51 | Mon Dec 14 | 1 hr | S4 labs (4) |
| 52 | Tue Dec 15 | 1 hr | S5 labs (4) |
| 53 | Wed Dec 16 | 1 hr | S6 labs (4) |
| 54 | Thu Dec 17 | 1 hr | S7 labs (4) |
| 55 | Fri Dec 18 | 1 hr | S8 labs (4) |
| 56 | Sat Dec 19 | 2 hr | S9 labs (4), S10 labs (4) |
| 57 | Sun Dec 20 | 1 hr | Redraw the full ML pipeline from memory (data, features, training, registry, serving); write a one-page ML cheat sheet |

**LLM & GenAI Systems** (RAG Foundations 13 chapters and 3 labs, Serving and Cost 12 and 2, Quality Gates 11 and 1, Agents That Take Action 9 and 2)

| Day | Date | Time | Do |
| --- | --- | --- | --- |
| 58 | Mon Dec 21 | 1 hr | RAG Foundations: chapters 1 to 7 |
| 59 | Tue Dec 22 | 1 hr | RAG Foundations: chapters 8 to 13 |
| 60 | Wed Dec 23 | 1 hr | Serving and Cost: chapters 1 to 6 |
| 61 | Thu Dec 24 | 1 hr | Serving and Cost: chapters 7 to 12 |
| 62 | Fri Dec 25 | 1.25 hr | Quality Gates: chapters 1 to 11 (swap with Sat if you want the day off) |
| 63 | Sat Dec 26 | 2 hr | Agents That Take Action: chapters 1 to 9, then RAG Foundations lab 1 |
| 64 | Sun Dec 27 | 1 hr | RAG Foundations labs 2 and 3 |
| 65 | Mon Dec 28 | 1 hr | Serving and Cost labs 1 and 2 |
| 66 | Tue Dec 29 | 1 hr | Quality Gates lab 1 |
| 67 | Wed Dec 30 | 1 hr | Agents That Take Action labs 1 and 2 |
| 68 | Thu Dec 31 | 1 hr | LLM cheat sheet (RAG, serving and cost, quality gates, agents); redraw a RAG architecture from memory |

If an LLM lab takes more than an hour, finish it the next day and shift the cheat sheet to Jan 3.

**Remaining 12 Expert problems, mocks and final review (days 69 to 80)**

The public pages name only 13 of the 25 Expert problems, so the 12 names below are placeholders. Swap in the real names from the Expert groups (Core Backend, Consumer, High-Integrity, Search & Platform, Infrastructure) when you open them.

| Day | Date | Time | Do |
| --- | --- | --- | --- |
| 69 | Fri Jan 1 | 1 hr | Expert, timed 25 min each: Google Drive, Search Autocomplete |
| 70 | Sat Jan 2 | 2 hr | Expert: Full-Text Search, Collaborative Editor, Webhook Platform |
| 71 | Sun Jan 3 | 1 hr | Expert: Distributed Scheduler, Metrics Platform |
| 72 | Mon Jan 4 | 1 hr | Expert: Logging Platform, Feature Flag Platform |
| 73 | Tue Jan 5 | 1 hr | Expert: Distributed Cache, Object Storage Service |
| 74 | Wed Jan 6 | 1 hr | Expert: the Kafka-like streaming platform (the 25th problem); use the rest of the hour to catch up |
| 75 | Thu Jan 7 | 1 hr | Mock 1: an ML & AI system design question from that track's labs, 35 min out loud, 15 min self-review |
| 76 | Fri Jan 8 | 1 hr | Mock 2: an LLM system (for example a RAG chatbot), same format |
| 77 | Sat Jan 9 | 2 hr | Mock 3: redo your 2 weakest Expert problems cold, 25 min each, then fix the gaps |
| 78 | Sun Jan 10 | 1 hr | Merge the three cheat sheets into one 2-page sheet |
| 79 | Mon Jan 11 | 1 hr | Catch-up day; redo your weakest problem from memory |
| 80 | Tue Jan 12 | 1 hr | Final check: roadmaps show all sections, labs and 25 of 25 Expert done (fewer only if you cut Expert problems under the catch-up rules), then stop and celebrate |

## If you miss days

Never double up; slide the plan instead, and cut optional items before labs.

- **Missed 1 to 2 days:** do the missed row on the next free weekend slot and push nothing else. Sat and Sun have slack.
- **Behind by 3 or more days at a Sunday review:** skip the optional items first (the last Expert problems (Jan 4 to 6), then the day-7 redraw), never the labs.
- **A track runs longer than planned:** use the Jan 11 catch-up day first, then drop the last Expert problems, before extending your daily hours.
- **Ahead of schedule:** use the extra time on Expert problems, not on starting the next track early, so each phase gets its full review.

**Weekly check, every Sunday (10 min):** count the rows ticked this week, check the solved counters on [Roadmaps](https://codeparadox.io/roadmaps), and write the one topic you would still struggle to explain.

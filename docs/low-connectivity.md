# Low-latency and low-bandwidth recommendations

Design agreed: 2026-10-09. Status: recommendations and provisional targets; no production optimisation measured here.

Preserve content, visual identity, navigation, accessibility, and chatbot evidence standards. Improve delivery, request count, rendering order, and repeat access so citizens receive useful information before optional media and chat code load.

## Historical baseline

The [September report](../promisetracker_docs/testing/accessibility-low-connectivity-report.md) analysed source and built files, not throttled production. It estimated 3–5.5 MB uncompressed homepage traffic. Establish current compressed wire sizes before claiming savings.

| Asset | Historical bytes | Implication |
|---|---:|---|
| Homepage `VidCv.png` background | 2,797,827 | Largest immediate opportunity |
| Single initial JavaScript bundle | 823,068 | Route splitting needed |
| Six OTF/TTF fonts | 1,011,872 | Review formats, weights, duplicate declarations |
| Single CSS bundle | 211,110 | Audit unused/global rules and route dependencies |

## Priorities

| Priority | Change | Quality guardrail | Expected effect |
|---|---|---|---|
| P0 | Generate responsive AVIF/WebP variants of the large image, with fallback | Compare detail, crops, gradients, and readability at supported sizes | A suitable variant under 200 KB would remove about 2.6 MB from this asset; target, not measured saving |
| P0 | Use responsive sizes and reserve image dimensions/aspect ratio | Preserve sharpness at supported densities and prevent layout shifts | Avoid desktop-sized downloads on small screens |
| P0 | Consolidate font declarations and ship licensed WOFF2 subsets | Preserve branding and required characters, including Nigerian names; check fallback shifts | Reduce font bytes and redundant requests |
| P0 | Split routes and defer chat code until opened | Preserve navigation, loading feedback, and error recovery | Reduce initial JS and CPU work |
| P0 | Compress HTML/JS/CSS/JSON and cache hashed public assets at the edge | Verify headers; exclude personalised responses | Reduce transfer and origin round trips |
| P1 | Lazy-load below-fold images; defer optional video and analytics | Keep hero/LCP image eager; retain media controls and alternatives | Reduce initial discretionary downloads |
| P1 | Paginate public APIs, return rendered fields, deduplicate/cancel obsolete reads | Preserve totals, filters, deep links, and complete-data access | Reduce unnecessary payload and requests |
| P1 | Cache previously viewed public pages with bounded storage | Label saved/as-of dates; exclude private/chat routes | Useful repeat/offline reads |
| P1 | Make headings, primary text, and summaries available early | Keep interactive flows progressively enhanced | Faster useful rendering; evaluate prerendering if needed |
| P2 | Offer a data-saving preference for optional media | Preserve facts, citations, ratings, and access to full media | Give users control without removing content |

Responsive images need visual checks at actual display sizes. Do not lazy-load the LCP image. Follow [web.dev responsive-image guidance](https://web.dev/learn/design/responsive-images) and [LCP guidance](https://web.dev/articles/optimize-lcp).

Use long-lived immutable caching for hashed public assets; revalidate HTML for new bundles. Published JSON can use ETag/Last-Modified validation and short freshness rules aligned with publication changes. A 304 saves body bytes but still costs a network round trip. See [MDN HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching).

Offline caches should allowlist public routes, cap entries/bytes, evict old content, clean up obsolete versions, and label dates. Use a short bounded network attempt before falling back to saved content on intermittent links. Service workers cannot perform a new grounded RAG query without connectivity. See [MDN offline operation](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Offline_and_background_operation).

## Chat latency without weaker evidence

Measure network delay, queue age, embedding/retrieval, reranking, generation, and client polling separately. Optimise the measured slow stage. Bound concurrency to actual capacity, reuse validated embeddings, keep provider connections warm where supported, and reuse eligible answers against the active evidence snapshot. Avoid model generation for simple greetings and verified deterministic lookups; avoid context rewriting for standalone questions without relevant history.

Send concise grounded answers with essential citations and offer more detail on request. Never trim away ratings, caveats, or source links. Keep evidence gating and reranking unless controlled evaluation demonstrates an alternative maintains quality. Model substitution or smaller retrieval budgets require per-slice quality evaluation, not latency-only comparison.

Retain job IDs across reconnects; back off conditional polling instead of resending questions. Give immediate local sending feedback and confirmed queued status after admission. Include polling delay in end-to-end measurements. The standalone page avoids full-site media while retaining source access, accessibility, and the shared answer renderer.

## Provisional engineering budgets

These are starting targets to validate on community devices and networks, not universal guarantees.

| Measure | Target | Conditions |
|---|---|---|
| First mobile public-page transfer | ≤500 KB compressed through first useful view | Cold cache; include critical data; count deferred traffic separately |
| Initial JavaScript | ≤150 KB compressed | Per entry route; unopened chat excluded |
| Required initial fonts | ≤100 KB total | Required glyphs and branding preserved |
| Standalone chat before first question | ≤200 KB compressed | Cold cache; no full-site media |
| Repeat unchanged public page | ≤100 KB transfer | Warm cache; include validation/data requests |
| LCP on fast-3G test | ≤4 seconds | 1.6 Mbps down, 750 Kbps up, 150 ms RTT, mobile CPU throttle |
| Useful heading/text on weak-link test | ≤8 seconds | 400 Kbps down, 100 Kbps up, 400 ms RTT |
| Durable browser admission p95 | ≤2 seconds server-side | Representative staging concurrency; network reported separately |
| Grounded answer p95 | ≤10 seconds server-side including queue | Report cached/uncached and provider/model slices separately |
| Publication-to-query visibility | ≤5 minutes | Detection through activation and cache invalidation |

At 400 Kbps, 500 KB takes about ten seconds to transfer before overhead. Critical text must therefore arrive before the entire first-view budget finishes. Do not promise a four-second complete load on that profile.

## Verification and quality gates

1. Establish current compressed bytes, request counts, response headers, cold/warm waterfalls, and page/chat timings for homepage, tracker, promise detail, pledge flow, embedded chat, and standalone chat.
2. Test both stated network profiles, a representative lower-end Android phone, and intermittent/offline conditions. Keep device/browser, viewport, CPU throttle, and cache state consistent.
3. Compare screenshots, typography, image detail, layout shifts, content, navigation, keyboard operation, headings/landmarks, focus, form errors, screen-reader chat announcements, and motion preferences.
4. Exercise freshness labels, service-worker upgrades, eviction, offline failures, and reconnect; confirm private/chat responses never enter public caches.
5. Compare the same adjudicated RAG cases before and after optimisation. Rating fidelity, valid grounded citations, publication eligibility, archive correctness, and appropriate abstention block release when they regress.
6. Record absolute values and baseline changes, ship gradually, and keep a compatible rollback path.

Recheck the older audit's clickable divs, missing headings/landmarks, and skip-link gaps against current source. Correct verified gaps alongside performance work so every control remains usable.

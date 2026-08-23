# Research Principles

These principles apply to ALL research agents. Every agent must follow them.

## Iterative Deep Research

Each agent performs **5-8 web searches minimum**, organized in sequential rounds that drill deeper:
- Round 1: Broad overview queries
- Round 2: Drill-down into specific findings from Round 1
- Round 3: Cross-reference and validate
- Round 4: Reality check and edge cases

Do NOT stop after a single query. The first search gives you the surface — the follow-ups give you the insight.

## Source Quality Tiers

Rank every finding by source reliability:

| Tier | Source Type | Use For |
|------|-----------|---------|
| **Tier 1** | Industry reports (Gartner, Forrester, McKinsey, IBISWorld), SEC filings, government data, Crunchbase verified data | Hard numbers, market share, verified metrics |
| **Tier 2** | Reputable tech press (TechCrunch, Bloomberg, WSJ), company press releases, investor presentations, PitchBook | Funding data, company news, expert opinions |
| **Tier 3** | Blog posts, Reddit threads, individual reviews, social media, founder interviews | Sentiment, narrative hooks, anecdotal evidence |

For pitch research, Tier 2 and Tier 3 sources are especially valuable — investor blog posts, demo day recaps, and founder stories reveal what narratives resonate and what language investors respond to.

**KR branch — if the target market is Korea:** read `../startup-design/references/kr-research-sources.md` §2 before ranking anything. The tiers above are US-shaped — Gartner, SEC filings, Crunchbase, and G2 have little or no coverage of Korean startups, and the substitutes rank differently (KOSIS · DART · 중기부·창업진흥원 공고 for Tier 1, 벤처투자정보센터 · 혁신의숲 · 플래텀 · 벤처스퀘어 for Tier 2, 네이버 카페·블로그 · 앱스토어 리뷰 · 잡플래닛 for Tier 3). §1 also grades every source by machine accessibility: `thevc.kr` returns 403 to automated fetches, so a failed fetch there means "not collected" and must be reported that way — never as "no data exists".

## Pitch-Specific Search Strategies

When researching investor preferences:
- Search for "{investor/fund name} portfolio", "{fund} thesis", "{investor} blog pitch advice"
- Look for demo day recaps and pitch competition results in the relevant space

When researching comparable narratives:
- Search for "{competitor} pitch deck", "{similar company} fundraise", "{space} demo day"
- Look for "X for Y" analogies that have worked for similar companies

## Cross-Referencing

Never trust a single source for important claims. For every key finding:
- Look for 2-3 independent sources
- If sources agree: note convergence and cite all
- If sources disagree: note both, explain the discrepancy, and state which you trust more and why

## Quantification

Vague claims are worthless — and investors see through them instantly:
- Bad: "They raised a big round"
- Good: "$15M Series A led by a16z in March 2025, $3M seed in 2023"
- Bad: "The market is growing"
- Good: "$4.2B in 2025, projected to reach $8.1B by 2028 at 12.3% CAGR (source: Grand View Research)"

## Dating

Always note when data was published. Flag anything older than 12 months as potentially outdated. Fundraising landscapes shift fast — a market correction or new competitor can change everything in weeks.

## Handling Research Failures

Sometimes WebSearch won't find what you need:

1. **Try alternative queries.** Rephrase, use synonyms, try different angles. At least 3 variations before declaring a gap.
2. **Use proxy data.** If you can't find a company's exact metrics, estimate from team size, funding, pricing × estimated customers. Show your math.
3. **Declare the gap explicitly.** Write: "DATA GAP: Could not find reliable data on [X]. Closest proxy: [Y]. Confidence: Low."
4. **Never fabricate.** An honest "unknown" is infinitely more valuable than a made-up number.
5. **Suggest how to fill the gap.** Recommend specific research the founder can do to fill the gap before the pitch.

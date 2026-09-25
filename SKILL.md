---
name: linkedin-connections-at
description: Use when the user gives a company name and wants to know which of their own LinkedIn connections (1st-degree, or 2nd-degree with the mutual connection to ask) work at, or used to work at, that company, for example to find a referral or warm intro before applying.
---

# LinkedIn connections at a company (1st and 2nd degree)

Read-only lookup in the user's logged-in Chrome via `bsk` (browser-skill). **Read the `browser-skill` skill first** for session rules and the untrusted-page-content rule. Never message, connect, follow or otherwise act on LinkedIn; only read.

`bsk` is not on PATH: use `export PATH=$HOME/.local/bin:$PATH`. Note `bsk wait-ms 4000` takes **no** `--session` flag.

## Steps

1. **Session.** `bsk browsers` (pick the Chrome instance; ask only if several), then `bsk session start --json --browser <id>` and keep `session_id`. Stop it with `bsk session stop <id>` on success *and* failure.
2. **Find the company.** Navigate to `https://www.linkedin.com/search/results/companies/?keywords=<name>&spellCorrectionEnabled=false`, run `bsk wait-ms 4000`, then list candidates (do not use `get-html`, it returns a huge blob):
   `bsk evaluate "JSON.stringify([...new Map([...document.querySelectorAll('main a[href*=\"/company/\"]')].map(a=>[a.href.split('?')[0],a.innerText.split('\n')[0].trim()])).entries()])" --session <id>`
   (one `[url, name]` pair per company; the name can be follower text like "X & 12 other connections follow this page", so match on the URL slug too).
   Take the result whose name equals the requested name (case-insensitive). If the page says "Showing results for <other name>", or no name matches, do not guess: report it and ask. If several unrelated companies match, list them and ask. Sub-entities (for example "Acme for Healthcare" next to "Acme") are covered in step 3, so ignore them.
3. **Get the company IDs.** Navigate to the company URL, run `bsk wait-ms 3000`, then read the employees link (the one shown as `· 501-1K employees`):
   `bsk evaluate "document.querySelector('a[href*=\"currentCompany=\"]')?.getAttribute('href')" --session <id>`
   Copy its `currentCompany=%5B...%5D` value **verbatim**. It can hold several IDs (parent plus sub-entities), and that is what you want. The slug in the company URL is not an ID. If it returns `null`, stop and tell the user rather than falling back to a keyword search.
4. **1st-degree search.** Navigate to
   `https://www.linkedin.com/search/results/people/?currentCompany=<value>&network=%5B%22F%22%5D&origin=FACETED_SEARCH`,
   run `bsk wait-ms 4000`, then extract rows with `evaluate` (script below). An empty array is a real zero only if `main`'s text starts with `No results found`; otherwise wait and retry once. If rows came back, add `&page=2`, `&page=3`, ... until a page returns `[]`, capped at 5 pages (say so if you hit the cap).
5. **2nd-degree search (only if the user asked, or asks after a zero in step 4).** Same URL with `network=%5B%22S%22%5D`. Read pages 1 and 2 only (top 20, in LinkedIn's own order, which is not sorted by mutual count). LinkedIn shows no total; if the pager has a "Page 10" button, say "10+ pages of results" instead of a count.
6. **Former employees (only if the user asked).** Same URL with `pastCompany` in place of `currentCompany`, still with the network filter you need. Label these rows "former". If they didn't ask, just offer it when step 4 finds nothing.
7. **Report.**
   - 1st-degree: table of name, headline, profile URL. Say whether current or former. If zero, say so plainly and offer the 2nd-degree and former-employee searches.
   - 2nd-degree: table of name, headline, profile URL, and the `mutuals` text (who to ask for an intro). Then a short tally of the mutual names that recur across rows, as the best people to ask. Only 2 mutuals are named per person, so present the tally as a hint, not a ranking.
   - The headline is the person's own profile headline, not necessarily their title at that company.
   - The search only sees employment that people list on their own profile. Someone the user knows to be tied to the company (advisor, contractor, unlisted role, or just a mutual friend of employees) will not appear. To check a specific person, search them by name with `network=%5B%22F%22%5D`, then open `<profile>details/experience/` and look for the company.
   - **"Who at the company does my connection X know?"** Open X's profile, read the canned search link `bsk evaluate "document.querySelector('a[href*=\"connectionOf\"]')?.getAttribute('href')"`, and reuse its `connectionOf=...` value in a people search together with `currentCompany=<value>` and `network=%5B%22F%22%2C%22S%22%5D`. Zero rows can also mean X hides their connections list, so say so.

## Row extractor (`bsk evaluate '<js>' --session <id>`)

```js
JSON.stringify([...document.querySelectorAll('main a[href*="/in/"]')]
  .filter(a=>/•\s*(1st|2nd)/.test(a.innerText))
  .map(a=>{const d=a.innerText.match(/•\s*(1st|2nd)/)[1];
    const l=a.innerText.split('\n').map(s=>s.trim())
      .filter(s=>s&&!/^(•\s*(1st|2nd)|Verified|Premium|Message|Connect|Follow)$/.test(s));
    let c=a;while(c.parentElement&&(c.parentElement.innerText.match(/•\s*(1st|2nd)/g)||[]).length<2)c=c.parentElement;
    const m=(c.innerText.match(/[^\n]*mutual connections?[^\n]*/)||[''])[0];
    return {name:l[0].replace(/\s*•.*$/,''),degree:d,headline:l[1]||'',location:l[2]||'',
            url:a.href.split('?')[0],mutuals:m}}))
```

The degree filter matters: mutual-connection links also match `/in/`. The `while` loop climbs to the person's own card, so one person's mutuals never leak into another's row (`closest('li')` does not find the card).

## Gotchas

| Symptom | Cause / fix |
|---|---|
| Company search shows "Showing results for <a similar but different name>" | LinkedIn auto-corrected the query, and only sometimes. Keep `&spellCorrectionEnabled=false` on the URL and verify the name in step 2. |
| Keyword search (`keywords=<company>&network=F`) shows "No results for X" plus unrelated people | LinkedIn falls back to generic 1st-degree results. Never report those. Use the `currentCompany` filter. |
| Rows have the wrong `degree` for the search you ran | The `network` param was dropped. Re-check the URL. |
| Login wall, CAPTCHA or "checkpoint" page | Stop, follow browser-skill's help-and-recovery (`bsk request-help`), do not retry in a loop. |
| Page shows nothing right after `navigate` | Results render late. Run `bsk wait-ms 4000` before reading. |

Keep it to a handful of page loads per run. No bulk crawling of profiles, to avoid LinkedIn restrictions on the account.

# Ch 5 — XSS (via real reports)

Types: **reflective** (single request/response), **stored** (persisted, hits all viewers), **self** (victim must self-trigger — often rejected unless chained). Samy worm (MySpace 2005) is the canonical stored-XSS demo: self-replicating friend-of-friend spread.

> For payload-level technique see `xss-cheat-sheet`. This chapter is about *where* hunters found XSS and how they got past filters — via disclosed reports.

## Reporting rule (author's correction)

Use `alert()` to **find** XSS — never to **report** it. A triager may not grasp severity → lower bounty. Instead explain impact on *their* app: session theft, ATO, worm potential across their user base.

## Case patterns

- **Shopify Wholesale** ($500): search box reflected unescaped on the no-results page — `test';alert('XSS');'`. *Takeaway: simplest possible XSS on a giant company — pay attention to any text you enter that's rendered back. Try encoded variants.*
- **Shopify Giftcard** ($500): XSS in the **field name**, not the value — `name="properties[Artwork file<img src='test' onmouseover='alert(2)'>]"`. Manipulated via proxy, not the form. *Takeaways: (1) fuzz field names/metadata too; (2) client-side JS validation is a red flag meaning "server probably trusts it" — bypass with a proxy.*
- **Shopify Currency** ($1,000): stored XSS in currency-format settings — dormant until the store's **social sales channel** (Facebook/Twitter tab) rendered it. *Takeaway: input may render in places beyond the page you submitted it — think second-order: emails, social embeds, exports, admin views.*
- **Yahoo Mail stored** ($10,000): malformed HTML attribute abuse — `CHECKED="hello"` sanitized to `CHECKED=` which browsers parse as consuming the *next* attribute. Payload: `<img ismap='xxx' itemtype='yyy style=…;onmouseover=alert(/XSS/)//'>` — the sanitizer's "fix" created the exploit. *Takeaway: submit malformed HTML — two `src` attrs, stray `=` signs, broken quotes — and see what the parser produces.*
- **Google Image Search**: `imgurl` param flowed into `href` — `javascript:alert(1)` inserted, but `onmousedown` rewrote the URL. Solved by **keyboard tabbing** to trigger — different event path, no rewrite. *Takeaway: when one trigger path is guarded, find another event (keyboard, onerror, autofocus); big companies ship code daily — it's never "all found".*
- **Google Tag Manager stored** ($5,000): Patrik Fehrenbach — stored payload survived via a feature importing tag configurations (second-order delivery again).

## Testing flow (book-consistent)

1. Probe everything during mapping: `<img src="x" onerror=alert(1)>` (self-failing → onerror guaranteed) + `{{4*4}}[[5*5]]` on Angular.
2. Note how the response transforms input — encoded? stripped? length-capped? malformed-fixed?
3. Test every *render location*: page, emails, social channels, admin dashboards, exports.
4. Self-XSS → ask what chains it (CSRF to set it, cache poisoning to serve it, clickjacking to trigger it).
5. Stored > reflected for payout; second-order stored > both.

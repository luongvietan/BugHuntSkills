# Ch 2 — Basic Hacking: CMS

~62% of the internet runs a CMS; ~39% is WordPress. Don't hand-test a CMS — run the platform-specific scanner, then check known misconfigs.

## Scanner-per-CMS table

| CMS | Tool | Command |
|-----|------|---------|
| WordPress | wpscan | `wpscan --url <URL>` |
| Drupal | droopescan | `python3 droopescan scan drupal -u <URL> -t 32` |
| Joomla | joomscan | `perl joomscan.pl -u <URL>` |
| Adobe AEM | aemhacker | `python aem_hacker.py -u <URL> --host <YOUR_PUBLIC_IP>` |
| Magento | magescan | (github.com/steverobbins/magescan) |

## Per-CMS notes

- **WordPress**: mostly auto-patched by managed vendors, but **plugins** are frequently vulnerable (many need creds). **Always check `/wp-content/uploads/` for directory listing** — emails, passwords, paid products leak there constantly.
- **Drupal**: droopescan also scans other CMSs.
- **Joomla**: "a mess" — worst security posture of the big three.
- **Adobe AEM**: ~instant win. Riddled with public vulns (SSRF, RCE…). SSRF tests need a public IP the target can reach back to.
- **Unknown CMS**: ExploitDB for CVEs → GitHub for a dedicated scanner (search `<cms> vulnerability scanner`) → if nothing, move on unless hunting 0-days.

## Play

1. Fingerprint CMS (Wappalyzer, generator meta tag, footer).
2. Run the dedicated scanner.
3. Manual checks: uploads dir listing, default admin panels, default creds.
4. Known CVEs → PoC (ch01 cycle).

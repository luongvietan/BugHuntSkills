# sources.md — owasp-wstg

## Sources

- **Book/PDF**: OWASP Web Security Testing Guide **v4.2** (frozen PDF in the
  source repo) — successor to OTGv4.
- **Live authority**: https://owasp.org/www-project-web-security-testing-guide/
  — stable release stream; test IDs `WSTG-<CAT>-<NN>` are versioned and stable.
- **Upstream repo**: https://github.com/OWASP/wstg — where post-freeze
  additions land before a numbered release.
- Methods-refresh source register: `docs/superpowers/2026-09-23-bug-bounty-source-register.md`
  (BugBounty workspace) — pins v4.2-as-stable + v5.0-in-development status.

## Provenance & edition policy

- Chapters are synthesized from the **v4.2 frozen** text — the current stable
  edition (verified 2026-09-23).
- **v5.0** exists only as upstream development drafts — deliberately NOT
  covered; unversioned draft guidance would destabilize the test-ID catalog.
- `WSTG-BUSL-10` (Test Payment Functionality, §4.10.10) was added to live
  WSTG after the v4.2 PDF froze — synthesized in ch10 with provenance marked
  in-file; it is the only post-freeze test included.

## Review history

- 2026-09-23: stable-edition status verified (v4.2 current, v5 dev-only);
  sources.md created (methods-refresh T5).
- Next review trigger: WSTG v5.0 stable release.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A static GitHub Pages site for **LAPSE Design**, an interior design studio in Johor Bahru, Malaysia. It currently hosts only privacy policy pages, published so they can be linked from social media (Instagram/Facebook) and WhatsApp.

There is no build system, package manager, linter, or test suite — the HTML files are hand-written and served as-is by GitHub Pages. To preview a page, just open the `.html` file in a browser.

## Files

- `privacy.html.html` — Privacy policy in Simplified Chinese (`lang="zh-CN"`), the primary audience's language.
- `Privacy Policy.html.html` — The same policy in English (`lang="en"`).

The two files are translations of each other and share identical inline CSS. **Any content change to one must be mirrored in the other**, and the "Last updated" date at the top of the body must be bumped in both.

## Conventions and gotchas

- **Double `.html.html` extensions and the space in `Privacy Policy.html.html` are intentional-as-deployed**: published URLs depend on these exact filenames. Do not rename the files unless the user explicitly asks, since that would break existing links shared externally.
- All styling is inline `<style>` in each file's `<head>` — there are no shared CSS/JS assets. Keep it that way; if you change styles, apply the same change to both files.
- The contact section (section 9 / 联系我们) contains placeholder values like `[Your email address]` / `[您的邮箱地址]`. Do not invent real contact details to fill them in.
- The site has no `index.html`; the policy pages are accessed by direct URL only.
- Content references Malaysia's Personal Data Protection Act 2010 (PDPA). Keep legal wording consistent between the two language versions when editing.

## Git

- Default branch: `main`.
- Commit directly per the task's designated branch instructions; there is no CI to wait on.

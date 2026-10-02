# WHATWG vs RFC 3986 URL Parser Differential

Paste URLs and see, field by field, where your browser's WHATWG URL parser and a strict RFC 3986 grammar disagree on host, userinfo, port and path, including strings one accepts and the other rejects.

**Live demo:** https://0xelitesystem.github.io/whatwg-rfc3986-url-parser-differential/

## Use

1. Open the live demo, or open `index.html` from a local clone. It works from `file://` with no server.
2. Paste URLs, one per line. The box splits on line feed only and never trims, so a leading space or a tab is part of the input. Click **Load example set** to see the built-in cases.
3. Optionally add a base URL. Relative lines are then resolved against it on both sides.
4. Read each row. Every line gets three readings side by side:
   - **A. This browser's URL()**, the native `URL` object, which is the WHATWG URL Standard as implemented by the engine you are running. The page prints `navigator.userAgent` so you know which engine that is.
   - **B. RFC 3986 strict.** A verdict of absolute URI, relative reference or invalid, built from the Appendix A grammar. An invalid string shows the first offending code point, its index, the component and the ABNF rule. A valid one shows its raw components, then the section 6.2.2 and 6.2.3 normalization steps one at a time, then an http(s) scheme check from RFC 9110 section 4.2.
   - **C. Appendix B split.** The RFC's own regular expression, which RFC 3986 section 4.1 describes as "an example of a non-validating URI-reference parser that will take any given string and extract the URI components".
5. Read the badges. Each component (scheme, userinfo, host, port, path) is classed as AGREE, AGREE AFTER RFC NORMALIZATION (with the 6.2.x step named), SAME IP, DIFFERENT TEXT, or DISAGREE, and the row rolls up to the worst of those, or to RFC REJECTS, BROWSER ACCEPTS, BROWSER FAILS, RFC ACCEPTS, or BOTH REJECT. A **HOST SPLIT** flag marks any row where the browser's host differs from the RFC strict host or the Appendix B host. **USERINFO PRESENT** marks an http(s) URI carrying userinfo.

Two toggles. **Strict RFC 3986 5.2.2 resolver** is on by default; untick it to take the non-strict branch, which drops a reference scheme equal to the base scheme (so `http:g` resolves to `http://a/b/c/g` against the RFC example base). **Decode escapes** is off by default; when on, `<BS>t <BS>r <BS>n <BS><BS> <BS>uXXXX` (where `<BS>` is one backslash) are decoded so you can type a tab, CR or any code point. With it off, a backslash is literal. Lines over 8192 characters are refused with a visible error.

### Real use cases

- **Auditing an SSRF guard or redirect allowlist.** Paste the URLs your check approved. A HOST SPLIT row is a string where the host your browser, or any WHATWG based fetch, connects to is not the host another reader extracts. If your allowlist uses a lenient splitter and your fetch uses a WHATWG engine, that row is the bypass shape.
- **Choosing where to reject.** Column B shows whether a strict RFC 3986 check would have refused the string before any host was extracted, and names the rule that fails.
- **Gating http and https.** RFC 3986 is generic syntax, so `http:/evil.com/` and `http:///good.com/` are valid URIs. The RFC 9110 line says they are not valid http URIs: one has no `//` authority and the other has an empty host.
- **Comparing cache keys and normalizers.** A row that agrees only after normalization tells you which RFC 6.2 step your comparison has to perform. Numeric hosts are not resolved on the RFC side; `0x7f.1` stays a reg-name there.
- **Testing reference resolution.** Put `http://a/b/c/d;p?q` in the base field and paste relative references.
- **Knowing your engine.** The self test runs the 42 RFC 3986 section 5.4 examples through the browser's `URL()` and lists each mismatch live, classed as equivalent under 6.2.3, equal to the RFC's own backward-compatibility alternative, or an engine deviation. The probe table checks eleven parser rules from the WHATWG standard. Open the page in every browser you ship to; the results differ by engine, which is why no count is printed here.

### What the self test proves

It runs on every load and shows a pass or fail panel. If any HARD check fails, the page shows a banner saying its results cannot be trusted.

- The 42 reference resolution vectors from RFC 3986 sections 5.4.1 and 5.4.2, through the in-page strict resolver, must match exactly. Non-strict `http:g` must give `http://a/b/c/g`.
- The two section 5.2.4 `remove_dot_segments` traces must give `/a/g` and `mid/6`, step by step.
- Validator fixtures cover every example, all nine IPv6address alternatives valid and invalid, IPvFuture with upper and lower case `v`, zone identifiers, bad percent-encodings, and empty userinfo.
- The regex verdict must equal the component diagnostic verdict on every fixture.
- The RFC 9110 layer must reject `http:/evil.com/`, `https:evil.com` and `http:///good.com/` for the right reason.
- Every backslash fixture must still hold its U+005C characters, so a fixture cannot silently turn into an ordinary URL.
- A positive control is designed to fail; if it passes, the harness is broken.
- A 10,008 character hostile string must validate in under 50 ms.

## Why this exists

An allowlist check and a fetch that read the same string under different rules can see different hosts. `http://evil.com<BS>@good.com/` is the clean example. In a WHATWG engine a backslash in a special URL ends the authority, so the host is `evil.com`. A lenient splitter ends userinfo at the `@`, so the host is `good.com`. And a strict RFC 3986 check rejects the string outright: U+005C is in none of the character classes the grammar allows, so the string is not a URI at all. Several of the strings that cause host splits never pass a strict check.

The specifications already say what to do. RFC 3986 section 7.4, on IP address literals:

> If this filtering is performed, literals should be converted to numeric form and filtered based on the numeric value, and not on a prefix or suffix of the string form.

RFC 9110 section 4.2.4:

> Before making use of an "http" or "https" URI reference received from an untrusted source, a recipient SHOULD parse for userinfo and treat its presence as an error; it is likely being used to obscure the authority for the sake of phishing attacks.

This page shows each reading next to the rule that produced it, so you can see which reader your code is using and where it parts ways with the others.

It is one HTML file with no dependencies, no build step, no telemetry and no network access of any kind. MIT licensed. Fork it, vendor it, run it behind your firewall.

### Measured elsewhere

The page itself never claims Python or PHP behaviour. For the backslash URL only, these are the results measured on 2026-10-01 with the exact commands below. Run them yourself; versions change.

Python 3.14.7, `urllib.parse.urlsplit`, prints `good.com`:

```
python -c "from urllib.parse import urlsplit; print(urlsplit('http://evil.com' + chr(92) + '@good.com/').hostname)"
```

PHP 8.5.6, `parse_url`, prints `string(8) "good.com"`:

```
php -r 'var_dump(parse_url("http://evil.com" . chr(92) . "@good.com/", PHP_URL_HOST));'
```

PHP 8.5.6, `Uri\Rfc3986\Uri::parse`, prints `NULL` (the string is rejected):

```
php -r 'var_dump(Uri\Rfc3986\Uri::parse("http://evil.com" . chr(92) . "@good.com/"));'
```

PHP 8.5.6, `Uri\WhatWg\Url::parse`, prints `string(8) "evil.com"`:

```
php -r 'var_dump(Uri\WhatWg\Url::parse("http://evil.com" . chr(92) . "@good.com/")->getAsciiHost());'
```

The two lenient splitters are not RFC 3986 parsers. They read `good.com`; the strict RFC 3986 parser rejects the string; the WHATWG parser reads `evil.com`.

### Sources

- RFC 3986, Uniform Resource Identifier (URI): Generic Syntax. STD 66, January 2005, Internet Standard, updated by RFC 7320 and RFC 8820 ([rfc-editor.org/info/rfc3986](https://www.rfc-editor.org/info/rfc3986/)). Sections used: 2.3, 3.2, 3.2.2, 4.1, 5.2, 5.4, 6.2.2, 6.2.3, 7.4, Appendix A and Appendix B. Text: https://www.rfc-editor.org/rfc/rfc3986.txt
- RFC 9110, HTTP Semantics. STD 97, June 2022. Sections 4.2.1, 4.2.2 and 4.2.4. Text: https://www.rfc-editor.org/rfc/rfc9110.txt
- WHATWG URL Standard, a Living Standard. Anchors: [#concept-basic-url-parser](https://url.spec.whatwg.org/#concept-basic-url-parser), [#authority-state](https://url.spec.whatwg.org/#authority-state), [#special-authority-ignore-slashes-state](https://url.spec.whatwg.org/#special-authority-ignore-slashes-state), [#port-state](https://url.spec.whatwg.org/#port-state), [#concept-ipv4-parser](https://url.spec.whatwg.org/#concept-ipv4-parser), [#ends-in-a-number-checker](https://url.spec.whatwg.org/#ends-in-a-number-checker), [#double-dot-path-segment](https://url.spec.whatwg.org/#double-dot-path-segment), [#forbidden-host-code-point](https://url.spec.whatwg.org/#forbidden-host-code-point) and [#special-scheme](https://url.spec.whatwg.org/#special-scheme). WHATWG text is paraphrased on the page, never quoted.

This tool implements the published RFC 3986 text; errata are not applied. The 42 reference-resolution vectors are quoted from RFC 3986 section 5.4, Copyright (C) The Internet Society (2005). RFC 9110 quotations are from RFC 9110, Copyright (c) 2022 IETF Trust and the persons identified as the document authors. Behaviour is implemented from the published specifications; no code is reproduced here.

This is an independent tool. It is not affiliated with, or endorsed by, the WHATWG, the IETF or any browser vendor.

## Privacy

Everything runs in your browser. Nothing is uploaded, logged or sent anywhere. There is no analytics, no font fetch, no CDN reference and no `fetch` call in the source, and a Content-Security-Policy in the page head blocks every outbound connection. `new URL()` parses a string; it makes no network request. Inputs are treated as hostile and rendered as text, never as markup or links. The only thing written to storage is your light or dark theme preference.

## Run locally

```
git clone https://github.com/0xelitesystem/whatwg-rfc3986-url-parser-differential.git
cd whatwg-rfc3986-url-parser-differential
```

Open `index.html` directly in a browser, or serve it:

```
python -m http.server 8000
```

Then visit http://localhost:8000/.

## Build

There is no build. It is a single self-contained `index.html` with inline CSS and JavaScript. Editing the file is the whole workflow.

## License

MIT. Copyright (c) 2026 0xelitesystem.

## Related

- [url-parser-and-query-editor](https://github.com/0xelitesystem/url-parser-and-query-editor) breaks one URL into its parts and edits the query string. This tool asks a different question: whether two parsers agree on those parts at all.
- [agent-sandbox-linter](https://github.com/0xelitesystem/agent-sandbox-linter) reads an AI agent's command allowlist and host allowlist. A host allowlist is only as good as the parser that extracts the host it checks, which is what this tool tests.
- [yaml-type-coercion-auditor](https://github.com/0xelitesystem/yaml-type-coercion-auditor) is the same idea for YAML: one input, several parsers, every place they disagree.

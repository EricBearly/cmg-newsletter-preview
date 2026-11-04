# CMG e-newsletter builder templates

Three HTML email templates with the editorial copy, the pictures and the ad
creative taken out and replaced by merge fields. They are the same markup as
the approved samples, so anything that renders in the sample renders here.

- `AR-Today-AM-template.html`
- `AR-Today-PM-template.html`
- `SubPrime-template.html`

## Syntax

Mustache. Every mainstream stack has an implementation — PHP (`mustache.php`),
JS (`mustache.js`, Handlebars), Python (`chevron`, `pystache`), Ruby — and
Handlebars and Liquid are close enough to port in an afternoon.

```
{{field}}                  a value
{{#thing}} ... {{/thing}}  renders once for a hash, once per item for a list,
                           and not at all when the value is absent, false or
                           an empty list
{{.}}                      the current item, where a list holds plain strings
```

**Turn HTML escaping off in your engine.** Mustache engines escape `{{field}}`
by default, and these templates use double braces throughout because the copy
carries entities (`&rsquo;`, `&amp;`). Left on, readers see a literal `&rsquo;`.
Tested with the `chevron` engine: default settings mangled the copy, and with
escaping off the output matched our reference renderer on all three templates.

| Engine | Setting |
|---|---|
| mustache.php | `new Mustache_Engine(['escape' => fn($v) => $v])` |
| mustache.js | `Mustache.escape = t => t` |
| Handlebars | `Handlebars.compile(tpl, { noEscape: true })` |

Since nothing is escaped on the way out, sanitize on the way *in*: anything an
editor types must be cleaned before it reaches the template.

## Ads

Every ad position is a section. Pass nothing for a position and the whole
block — background, rule, "Sponsored" label and all — is gone, with the
blocks either side closing up against each other. Nothing is left behind.

| Field | Where | Shape |
|---|---|---|
| `AD1` | top leaderboard, full bleed | banner |
| `AD2`–`AD7` | in content, numbers vary per title | text ad, or banner |

A **banner** takes `url`, `img_desktop`, `img_mobile`, `alt`.
A **text ad** takes `url`, `logo`, `name`, `head`, `body`, `cta`.

Which slots exist differs per title — AM has 1–7, PM and SubPrime have 1–6.
Each block's HTML comment names its own slot. The numbers are field names, not
reading order: in SubPrime, banner `AD4` sits between `AD2` and `AD3`.

Creative sizes are in the rate sheet; the short version is 1300×160 supplied
for the desktop banner, 750×124 for mobile, and a 340×340 logo or image for
a text ad.

## Editorial

| Field | Notes |
|---|---|
| `LEAD_URL` `LEAD_CAT` `LEAD_HEAD` `LEAD_IMG` `LEAD_ALT` | the lead story |
| `LEAD_DECK` `LEAD_BYLINE` | optional; leave empty and the row still renders, so pass the empty string |
| `LEAD_BODY` | a **list** of paragraphs, one row each |
| `STORIES_1` `STORIES_2` `STORIES_3` | the runs of stories between the fixed ad positions |
| `HAS_PICKS` + `PICKS` | Editor's Picks cards |
| `HAS_HEADLINES` + `HEADLINES` | the More headlines list (AM, SubPrime) |
| `HAS_EARLIER` + `EARLIER`, `HAS_PODCASTS` + `PODCASTS` | PM only |
| `PROMO` | the house promo: `url`, `logo`, `when`, `head`, `body` |
| `ISSUE_DATE` | the date strip |
| `VIEW_IN_BROWSER_URL` `ISSUE_URL` | the hosted copy of this edition |
| `FORWARD_URL` `RECIPIENT_EMAIL` `UNSUBSCRIBE_URL` | **supplied by the sending platform.** Pass that platform's own merge tag as the value, so it survives into the composed email |

**An empty field renders as nothing, with no error.** A blank `UNSUBSCRIBE_URL`
produces a dead "Opt out" link, so make it a required field in your builder.

A story item is `{URL, CAT, HEAD, BODY, IMG, ALT}`. **Leave `IMG` empty and
the thumbnail cell disappears** and the brief runs full width — that is how
the stories below Editor's Picks are set, per house rule.

Why three story lists rather than one: the ad positions are fixed by the
design, and stories flow between them. Put two stories in a run or five and
the ads stay where they belong.

### The `sep` flag

`HEADLINES`, `EARLIER` and `PODCASTS` draw a hairline rule above each item.
Mustache has no index, so **set `sep: true` on every item except the first**.
Without it you get a rule immediately under the section heading.

## A worked example

```php
$data = [
  'ISSUE_DATE' => 'Tuesday, Oct. 6, 2026',
  'LEAD_HEAD'  => 'Wholesale used-car prices drop more than 3% in September',
  'LEAD_BODY'  => ['First paragraph.', 'Second paragraph.'],
  'STORIES_1'  => [
     ['URL' => '...', 'CAT' => 'Retail', 'HEAD' => '...', 'BODY' => '...',
      'IMG' => 'https://.../thumb.jpg', 'ALT' => ''],
  ],
  'HAS_HEADLINES' => true,
  'HEADLINES' => [
     ['CAT' => 'Auctions', 'URL' => '...', 'HEAD' => '...'],
     ['CAT' => 'Retail',   'URL' => '...', 'HEAD' => '...', 'sep' => true],
  ],
  'AD1' => ['url' => '...', 'img_desktop' => '...', 'img_mobile' => '...',
            'alt' => 'Advertisement'],
  // AD2..AD7 omitted entirely: those blocks will not render
];
```

## Before a send

The AR Today masthead art is referenced as `logos/AR-lockup-white.png` plus
`logos/today-orange.png` (AM) or `logos/today-pm-orange.png` (PM); SubPrime's is
already a hosted URL. **Host those files on autoremarketing.com and point each
`src` at its URL** — a relative path will not resolve in an inbox. "Today" is an
image because Gmail and desktop Outlook will not load the Oswald web font.
The same goes for any image still pointing at a local path.

Nothing in these templates assumes a particular sending platform. Open-tracking
pixels, list-unsubscribe headers and link wrapping are the ESP's job and should
be added by whatever tool ends up sending.

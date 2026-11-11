# WordPress newsletter builder: build notes

An optional plugin for Cherokee Media Group's WordPress site. Editors enter the
copy for an issue, pick the ads, and get a finished HTML email. It does not
depend on any particular email sending tool.

**Status: these are notes for a build. No plugin has been written.** The approach
comes from the newsletter tool running on Modern Public Fleet, a separate
WordPress site. That code isn't in this package and is built around one sending
platform, so treat it as a source of lessons rather than something to copy.
Nobody has reviewed Cherokee's WordPress install for this, so check every
assumption below against it.

## Decisions already made

| Decision | Detail |
|---|---|
| Stored as a composed email | Each issue's record is its finished HTML. Changing a template later never alters an issue already composed. |
| Sender left open | The sending tool is undecided, so nothing may assume one. Sending plugs in later. |
| Unsold ads disappear | The templates already do this. The builder only has to pass nothing for that position. |
| Optional | The templates work without this plugin. |

**One assumption to confirm:** editors need somewhere to keep typed copy while
drafting, so these notes keep the fields as a working draft and treat the
composed HTML as the record. If the fields should be discarded after composing,
that is a small change.

## What an editor does

1. Create an issue and choose the title and date.
2. Fill the stories, either by picking a published post or by typing.
3. Fill each ad position they sold, and tick it on.
4. Fill Editor's Picks and the other standing sections.
5. Press **Compose**. The plugin checks the issue, then renders and stores the HTML.
6. Preview at desktop and mobile width.
7. Take the HTML to the sending tool (download or copy), or send it through a sender adapter if one exists.

## Titles and sections

Templates exist for the first three. AR Canada, BHPH Report and Auto Fin Journal
exist only as finished samples so far; `tools/build_templates.py` needs extending
to cover them before the builder can.

| Title | Schedule | Ad positions | Sections |
|---|---|---|---|
| AR Today (AM) | Mon to Sat; Wednesday is Dealer Insights | 7 | Lead, 3 story runs, Editor's Picks, More Headlines, house promo |
| AR Today PM | Mon to Fri | 6 | Lead, 2 to 4 stories, Earlier Today, Editor's Picks, Latest Podcasts, house promo |
| SubPrime News Update | Mon, Wed, Fri | 6 | Lead, 2 story runs, Editor's Picks, More Headlines, house promo |
| AR Canada | Weekly | 5 | Lead, stories, Editor's Picks, Latest Podcasts, house promo |
| BHPH Report News | Tue, Thu | 5 | Lead, stories, Editor's Picks, Latest Podcasts, house promo |
| Auto Fin Journal | Mon, Wed | 5 | Lead, stories, Editor's Picks, Latest Podcasts, house promo |

Every merge field is listed in `template-fields.md`. Ad sizes and character limits
are in `ad-specs.md`.

## Suggested data model

A custom post type per issue, such as `cmg_issue`:

| Store | Holds |
|---|---|
| Post status | Draft, then composed, then sent |
| `_cmg_title` | Which newsletter |
| `_cmg_fields` | The working draft of every field, as JSON |
| `_cmg_composed_html` | The finished email, the record of the issue |
| `_cmg_template_sha` | Hash of the template used, so you can tell which version made an issue |
| `_cmg_composed_at`, `_cmg_composed_by` | When and who |

Rules for the composed record:
- Composing again replaces the HTML only until the issue is marked sent. After that it is locked.
- To reuse a sent issue, duplicate it into a new draft.
- Editing a template never touches a stored issue.

Keep the templates as files in the plugin rather than editable in admin, so
changes go through version control.

## Compose step

**Stories.** Let an editor pick a published post, which fills the fields, then edit
them. Copy the values into the issue instead of keeping a live link to the post, so
a later edit to the article cannot silently change a newsletter.

**Rendering.** Use a Mustache engine with HTML escaping turned off (see
`template-fields.md`). Since nothing is escaped on output, sanitize on input:
`sanitize_text_field` or `wp_kses` for copy, `esc_url_raw` for links.

**Sender merge tags.** `RECIPIENT_EMAIL`, `FORWARD_URL` and `UNSUBSCRIBE_URL` belong
to the sending tool. Make these a per-title setting holding that tool's own tags
and pass them in as values, so they survive into the composed HTML.

**Refuse to compose when:**
- `UNSUBSCRIBE_URL` is empty, because an empty field renders as a dead link with no error.
- Any `{{` survives the render.
- An ad is ticked on but missing a required field.
- An image `src` is not an absolute `https://` URL.
- A story has no headline or link.

**Warn, without blocking, when:**
- The HTML is close to Gmail's clipping point of about 102 KB. Filled samples currently run 52 to 61 KB.
- Copy is over the limits in `ad-specs.md`.

## Ad positions

Lessons from the Modern Public Fleet tool:
- **Each position gets an on/off toggle, and it defaults to off.** An old issue, an unticked box and a fresh draft should all read as off.
- **A position renders only if it is ticked on and complete.** Both are needed. An empty top banner reached a draft there before this rule existed.
- **Call-to-action text gets a default** and stays out of the completeness check. A sponsor shouldn't lose a position over a field with a sensible fallback.
- A text ad and a banner take different fields, so the position's form has to change with its type.

## Sending, left open

Define a small interface and ship with no sending at all. Download and copy are
enough to start.

```php
// Illustrative only: names are suggestions.
interface CMG_Sender_Adapter {
    public function label(): string;
    public function create_draft( string $html, array $meta ); // returns the tool's ID
    public function send_test( string $draft_id, string $address );
}
add_filter( 'cmg_newsletter_sender_adapters', function ( $adapters ) { return $adapters; } );
```

**Keep real sends behind a config constant that is off by default,** and restrict
them to administrators. Test sends can go to editors.

**Check these before choosing a tool, because they affect the adapter:**
- How it handles merge tags, unsubscribe and forward-to-a-friend.
- Whether it rewrites links for click tracking, and what that does to merge tags inside URLs.
- Whether it supports sending in timed waves.

## Lessons from running a newsletter tool in production

- **Duplicate-send guard.** Warn at two levels if the same list was mailed recently. Enforce it on the server, since a browser prompt can be skipped. The list is only a stand-in for the people: two overlapping lists can email the same person twice without tripping a list-based guard.
- **"Unknown" must be loud.** A guard that can't work out its answer looks identical to one with nothing to report. Treat "could not determine" as a warning, never a pass.
- **Duplicating an issue must drop the sender's draft ID,** or syncing the copy overwrites the original's draft. In WordPress, `get_post_meta` with no key returns serialized values, so use `wp_slash( maybe_unserialize( $v ) )` when copying.
- **`esc_url()` percent-encodes square brackets.** A sender merge tag like `[[LINK]]` in a URL becomes `%5B%5BLINK%5D%5D` and never gets substituted. It is invisible when sending and only shows up for the recipient. Check for it in the composed HTML.
- **A readiness check before every send.** Fetch every link and image, check for unfilled fields, the unsubscribe link and the postal address, and report HTML size. It must be read-only. Treat LinkedIn's 999 and Facebook's error pages as warnings, because they block data centres.
- **Fetch images from outside your server.** A check run on the host once reported images as fine when every reader outside got a 401, because the server could reach a staging sibling.
- **Sends can flatten the site.** Mail security scanners open every link within minutes. Modern Public Fleet took thousands of 5xx errors on send days until sends went out in timed waves and front-end tracking calls got lighter. Plan for this on a small host.
- **Images may need a CDN.** A response that sets a cookie isn't shared-cached by Gmail's image proxy, which produced 10 to 20 second image loads there. Moving email images to a cookie-free CDN host fixed it.
- **Image weight.** Serve at twice the drawn width. Baseline JPEG, not progressive, because older Outlook is inconsistent. Flatten PNGs onto white if they convert to JPEG, or transparency turns black. Don't flatten the masthead art, which is transparent on purpose.
- **Admin labels.** Strings pass through `esc_html`, so writing `&amp;` in one shows a literal `&amp;` on screen.

## Compliance

Every email needs a working opt-out and the postal address, which the templates
carry in the footer. AR Canada reaches Canadian readers, so CASL applies as well as
CAN-SPAM. Have whoever owns compliance at Cherokee confirm the requirements.

## Before building: questions for Cherokee

1. Which sending tool, or none yet?
2. Where will email images and the masthead art be hosted?
3. Who may compose, and who may send?
4. Is the site's PHP version and plugin policy compatible with a bundled Mustache library?
5. Which ad server fills the banners? The samples use `servedby.cherokeemediagroup2.com`. If those positions stay ad-served, the builder may only need a zone setting rather than an image field.

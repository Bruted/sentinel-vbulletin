# Redeyed Sentinel for vBulletin 5 / 4

Adds the **Redeyed Sentinel** CAPTCHA (self-hosted CAPTCHA + IP reputation)
to your vBulletin registration form. Free to install and **inert until you
configure your Secret Key** — with an empty Secret Key it never blocks
registration ("fail open").

- **Compatibility:** vBulletin 4.x and 5.x
- **Author:** Redeyed Corporation — <https://redeyed.com>
- **License:** MIT (2026)

## Files

```
product-redeyed_sentinel.xml   The vBulletin product (settings + hooks)
README.md
LICENSE
```

## Installation

1. Log in to the **AdminCP**.
2. Go to **Plugins & Products → Manage Products** (vB4) or
   **Products & Plugins → Manage Products** (vB5).
3. Click **[Add/Import Product]**.
4. Under **Import Product XML File**, choose
   `product-redeyed_sentinel.xml` and click **Import**.

You should now see **Redeyed Sentinel** in the product list.

## Configuration

Go to **AdminCP → Settings → Options** and open the
**Redeyed Sentinel** option group, then set:

| Setting        | Where to get it                                                   |
|----------------|-------------------------------------------------------------------|
| **Site Key**   | Redeyed Lab → **Sentinel → Sites** (public key)                   |
| **Secret Key** | Redeyed Lab → **Sentinel → Sites** (secret key, shown once)       |
| **Base URL**   | Defaults to `https://redeyed.com`; change only if self-hosting    |

Both keys come from the same place — **Redeyed Lab → Sentinel → Sites**. The
**Secret Key** is displayed **only once** when the site is created, so copy it
right away.

### Optional widget customization

The same option group has four **optional** settings that customise the widget's
look and behaviour. Each is **empty by default**; when empty, nothing changes and
the widget renders exactly as before. When set, each is emitted as a `data-*`
attribute on the `<div class="sentinel-captcha">` element (HTML-escaped the same
way as the Site Key).

| Setting          | Attribute         | Values                                                        |
|------------------|-------------------|---------------------------------------------------------------|
| **Widget Type**  | `data-widget`     | `behavioral`, `checkbox`, `press_hold`, `image_pick`, …       |
| **Theme**        | `data-theme`      | `auto`, `light`, `dark`                                       |
| **Colour Scheme**| `data-scheme`     | a colour scheme name to match your forum styling             |
| **Difficulty**   | `data-difficulty` | `easy`, `medium`, `hard`, `max`, or `1`–`6`                  |

> **Difficulty only raises the challenge.** `data-difficulty` sets a *minimum*
> challenge strength: it only **raises** difficulty above Sentinel's adaptive
> baseline, it never lowers it. Leave it empty to let Sentinel adapt
> automatically.

Leaving any of these empty keeps the Sentinel default for that aspect, so the
feature is fully backward compatible.

The **Site Key** is public and appears in the registration page HTML.
The **Secret Key** is secret: it is only ever sent server-side inside the
verification request body and is never printed to the page.

> While the **Secret Key** is empty, Sentinel is **inert**: no verification is
> performed and registration is never blocked. (The Site Key alone still
> renders the widget, but nothing is enforced without a Secret Key.)

## How it works

- **Render** (hook `register_form_complete`): injects
  `<script src="{base_url}/sentinel.js" async></script>` and
  `<div class="sentinel-captcha" data-sitekey="{site_key}"></div>` into the
  registration page. The widget adds a hidden `sentinel-token` input. When the
  optional customization settings are configured, the div also carries
  `data-widget`, `data-theme`, `data-scheme`, and/or `data-difficulty`
  attributes (each rendered only when its setting is non-empty).
- **Verify** (hook `register_addmember_process`): reads the posted
  `sentinel-token`, then POSTs (via cURL) to `{base_url}/sentinel/siteverify`
  with JSON body `{"secret":"…","response":"…","remoteip":"…"}` (the
  `remoteip` is optional). This is the reCAPTCHA/Turnstile-style siteverify
  flow: the per-site **Secret Key** authenticates the call in the request
  body — there is no `X-Api-Key` header. Registration proceeds only when the
  response decodes to top-level `success === true`; otherwise an error is
  added / an exception is thrown to stop registration.

## Hook-location assumptions (you may need to adjust)

The product uses widely-compatible hook locations, but vBulletin's
registration flow varies between versions and themes:

- **`register_form_complete`** (render) — fires while building the
  registration page in both vB4 and vB5. If your theme is fully
  template-driven and does not echo plugin output here, add the widget
  markup to your **register** template manually, e.g.:

  ```html
  <div class="redeyed-sentinel" style="margin:10px 0;">
    <script src="https://redeyed.com/sentinel.js" async></script>
    <div class="sentinel-captcha" data-sitekey="YOUR_SITE_KEY"></div>
  </div>
  ```

  (On vB5 the plugin also exposes `data.redeyed_sentinel_widget` when the
  hook provides a `$data` array, so you can render
  `{{ data.redeyed_sentinel_widget }}` from the template.)

- **`register_addmember_process`** (verify) — fires on submit before the
  account is saved in both vB4 and vB5. Some vB5 builds route registration
  through a controller that exposes the validation step as
  `register_addmember_complete` or a different controller hook instead. If
  verification never triggers, edit the product XML and change the
  `<hookname>` of the **"Redeyed Sentinel: verify token"** plugin to the
  hook your build fires on submit, then re-import the product.

## Security notes

- The **Secret Key** is transmitted only in the server-side siteverify
  request body (`secret`) and is never echoed in HTML or error messages.
- The optional `remoteip` is a best-effort client IP, validated with
  `filter_var(..., FILTER_VALIDATE_IP)` before being sent.
- All output is escaped with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')`.
- TLS peer/host verification is enabled on the verify request.
- If cURL is unavailable while the **Secret Key** is set, verification fails
  and registration is blocked (fail **closed**) so it is never skipped.

## Uninstalling

AdminCP → Manage Products → **Redeyed Sentinel** → **Delete**. This
removes the settings and hooks added by the product.

## Changelog

### 1.0.2

- **Added optional widget customization.** Four new (optional, empty-by-default)
  settings in the **Redeyed Sentinel** option group emit matching `data-*`
  attributes on the widget div only when set:
  - **Widget Type** → `data-widget` (`behavioral` | `checkbox` | `press_hold` |
    `image_pick` | …)
  - **Theme** → `data-theme` (`auto` | `light` | `dark`)
  - **Colour Scheme** → `data-scheme`
  - **Difficulty** → `data-difficulty` (`easy` | `medium` | `hard` | `max` or
    `1`–`6`) — only **raises** challenge strength above the adaptive baseline,
    never lowers it.
- Each value is HTML-escaped the same way as the Site Key. Backward compatible:
  with all four empty the rendered markup is unchanged.

### 1.0.1

- **Fixed server-side verification** to use the reCAPTCHA/Turnstile-style
  siteverify flow:
  - Endpoint changed from `{base_url}/api/v1/verify` to
    `{base_url}/sentinel/siteverify`.
  - Removed the `X-Api-Key` header. The developer API key is no longer used.
  - Renamed the **API Key** setting (`redeyed_sentinel_api_key`) to
    **Secret Key** (`redeyed_sentinel_secret_key`).
  - Request body changed from `{"site_key","token"}` to
    `{"secret","response"}` plus an optional `remoteip`.
  - A request now passes only when the response has top-level
    `success === true`.
  - Fail-open is now based on the **Secret Key** being present (the Site Key
    still renders the widget).

> **Upgrading from 1.0.0:** re-import the product XML, then open
> **AdminCP → Settings → Options → Redeyed Sentinel** and paste your per-site
> **Secret Key** (from Redeyed Lab → Sentinel → Sites) into the new field.

### 1.0.0

- Initial release.

# Redeyed Sentinel for vBulletin 5 / 4

Adds the **Redeyed Sentinel** CAPTCHA (self-hosted CAPTCHA + IP reputation)
to your vBulletin registration form. Free to install and **inert until you
configure your keys** — with empty keys it never blocks registration
("fail open").

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
| **Site Key**   | Redeyed Lab → Developer → **Sentinel Sites** (public key)         |
| **API Key**    | Redeyed Lab → Developer → **API Keys** (secret key)               |
| **Base URL**   | Defaults to `https://redeyed.com`; change only if self-hosting    |

The **Site Key** is public and appears in the registration page HTML.
The **API Key** is secret: it is only ever sent server-side in the
`X-Api-Key` request header and is never printed to the page.

> While either key is empty, Sentinel is **inert**: no widget is shown and
> registration is never blocked.

## How it works

- **Render** (hook `register_form_complete`): injects
  `<script src="{base_url}/sentinel.js" async></script>` and
  `<div class="sentinel-captcha" data-sitekey="{site_key}"></div>` into the
  registration page. The widget adds a hidden `sentinel-token` input.
- **Verify** (hook `register_addmember_process`): reads the posted
  `sentinel-token`, then POSTs (via cURL) to `{base_url}/api/v1/verify`
  with header `X-Api-Key: {api_key}` and JSON body
  `{"site_key":"…","token":"…"}`. Registration proceeds only when the
  response decodes to `data.success === true` (or `success === true`);
  otherwise an error is added / an exception is thrown to stop
  registration.

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

- The API key is transmitted only in the `X-Api-Key` header and is never
  echoed in HTML or error messages.
- All output is escaped with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')`.
- TLS peer/host verification is enabled on the verify request.
- If cURL is unavailable while keys are set, verification fails and
  registration is blocked (fail **closed**) so it is never skipped.

## Uninstalling

AdminCP → Manage Products → **Redeyed Sentinel** → **Delete**. This
removes the settings and hooks added by the product.

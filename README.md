# AGAME

A browser-based game library with an admin dashboard and a Watch section for browsing movies, shows, seasons, and episodes.

The user-facing page is a single HTML file. It loads its game catalogue from the admin website and checks site availability through Supabase. It requires an internet connection; it is not an offline application.

## Features

- Searchable game library.
- Grey admin dashboard for managing catalogue entries, artwork, visibility, and submissions.
- Red shutdown and green restore controls with a configurable maintenance message.
- Site availability checks approximately every 10 seconds while the user page is visible.
- Watch interface with title search, season selection, and episode selection.
- Movy playback links that open in a new tab.
- A Supabase Edge Function for Watch metadata requests.

## Project layout

The latest Watch update package uses this layout:

```text
main/
  index.html                         # User-facing games and Watch page
admin/
  _headers                           # Public catalogue CORS rule
supabase/
  functions/
    catalog/
      index.ts                       # Watch metadata proxy
```

The complete admin deployment also needs these files from the admin project package:

```text
index.html                           # Admin dashboard, not the public games page
catalog.json                         # Extracted game catalogue and embedded data tables
_headers                             # Netlify response headers
```

The admin and user pages are both named `index.html` in their respective deployment packages. Keep them in separate folders/projects so one does not overwrite the other.

## Requirements

- A browser with JavaScript enabled and an internet connection.
- A Supabase project with the existing database schema, authentication, storage configuration, and access policies used by the app.
- An administrator account listed in `ag_admins`.
- A host for the admin page and public catalogue. The supplied header configuration targets Netlify.
- The `catalog` Supabase Edge Function for Watch metadata.

This repository is not a complete database bootstrap. The availability setup SQL only configures the site settings table; it does not create the full catalogue, submissions, admin membership, or artwork storage setup.

## Configure your own deployment

The supplied HTML is configured for an existing deployment. Before publishing a fork, replace its live service URLs and publishable key with your own values.

Check these settings in both HTML pages:

| Setting | Purpose |
| --- | --- |
| `SB_URL` | Your Supabase project URL |
| `SB_KEY` | Your Supabase publishable key |
| `catalogURL` | Absolute HTTPS URL of the admin-hosted `catalog.json` |
| Availability bootstrap `url` and `key` | Supabase connection used before loading the user interface |

Also review hard-coded artwork, storage, and catalogue URLs. The main page contains multiple scripts, so updating only one Supabase URL/key occurrence may leave other requests pointing at the original project.

Never put a Supabase secret/service-role key, database password, deployment token, or saved access token in the HTML or repository.

## Deploy the admin project

1. Place the admin `index.html`, extracted `catalog.json`, and `_headers` at the site's published root.
2. Deploy that folder to Netlify.
3. Open the deployed admin page and sign in with an authorized administrator account.
4. Confirm that `/catalog.json` returns JSON rather than an HTML fallback page.

The `_headers` file must have no extension and contain:

```text
/catalog.json
  Access-Control-Allow-Origin: *
```

If `_headers` already exists, merge this rule instead of replacing other rules. For build-based sites, put the file in the folder that is copied into the final publish directory.

This rule intentionally allows browsers to read the public catalogue from other origins, including the standalone user page. It is not an authentication mechanism and should not be copied onto private data endpoints.

### Catalogue format

Use the extracted catalogue supplied with the updated page. Its structure is:

```json
{
  "games": [],
  "embedded": {}
}
```

This is a structural example, not a working replacement catalogue. The HTML requires the populated embedded data tables from the matching extracted file. The original array-only catalogue is not interchangeable with this file for the standalone loader.

## Deploy or open the user page

1. Set `catalogURL` to your admin site's public `catalog.json` URL.
2. Ensure the admin site's CORS header is deployed.
3. Open the user-facing `main/index.html` in a browser, or deploy it as the main site's `index.html`.

The latest standalone version does not require a neighbouring JSON file or the earlier Netlify catalogue proxy rule. It fetches the admin-hosted JSON directly.

Browser restrictions and individual game providers may affect local-file operation. Test the browsers and games you intend to support.

## Configure shutdown and restore

If the availability table is not already configured, run the supplied `setup.sql` from the admin project package in your existing Supabase project's SQL Editor.

The admin dashboard updates `ag_site_settings`. The user page checks the `main` settings record and displays the configured maintenance message when `is_down` is enabled.

- The admin page stays available so an administrator can restore the site.
- Open user pages check for changes approximately every 10 seconds while visible and when returning to the tab.
- If availability cannot be verified, the page remains blocked rather than assuming the site is online.

Shutdown is a client-side maintenance feature, not a server-level access lock. Anyone controlling a downloaded copy can remove that check. It does not revoke downloaded files or prevent direct access to third-party URLs.

## Configure Watch metadata

1. Open the existing Supabase Edge Function named `catalog`.
2. Replace its `index.ts` with `supabase/functions/catalog/index.ts` from the latest update.
3. Redeploy it with gateway authentication configured to accept the requests sent by the app.
4. Test title search, show details, and season episodes.

Deploying the HTML alone does not replace the Edge Function.

The function supports:

| Action | Parameters | Response field |
| --- | --- | --- |
| `search` | `q` | `results` |
| `details` | TV `id` | `seasons` |
| `season` | TV `id`, `season` | `episodes` |

The latest replacement uses `https://db.wecollege.net/3/`, a metadata service observed in use by Movy. It does not require a TMDB token in this version. It is a third-party dependency without a verified official public API contract, and its availability or access rules can change.

Movie and episode playback opens on Movy in a new tab. The app does not assume Movy supports iframe playback and does not host the linked video content. The updated Watch HTML and function no longer reference Rosin or its previous Springwater host.

## Security

- Enforce database access through Supabase grants and Row Level Security, not through hidden buttons or client-side admin checks.
- Allow anonymous reads only for data intentionally made public.
- Restrict catalogue edits and availability updates to authorized administrators.
- Prevent users from granting themselves admin membership.
- Protect private submissions and artwork storage operations with appropriate policies.
- Consider server-side rate limiting and usage monitoring for public function endpoints.
- Treat all frontend URLs and publishable keys as visible to users.

Supabase publishable keys may be included in browser code, but that does not make permissive database policies safe. Never expose secret/service-role keys.

The generated updates have not received a full security audit.

## Troubleshooting

### Catalogue request returns 404

Check that `catalogURL` points to the public JSON file, not a hosting dashboard URL. Verify the filename is exactly `catalog.json` and the file is included in the admin deployment.

### NetworkError or a CORS error

Check the failed URL in browser Developer Tools. For catalogue requests, verify that `_headers` is deployed on the admin project and that the JSON response includes `Access-Control-Allow-Origin: *`.

### Invalid catalogue format or missing embedded tables

Use the populated extracted `catalog.json` matching the updated HTML. Do not replace it with the original array-only file or the empty structural example above.

### Episode picker reports a provider error

Check the Supabase `catalog` function logs. Confirm the replacement function was deployed. A third-party metadata service can reject or fail requests independently of the game catalogue's Netlify CORS configuration.

### Site remains temporarily unavailable after restoring

Inspect the exact failing request shown by the page. Restoring changes the availability setting; it does not fix a missing catalogue, blocked request, or database access policy.

## Testing status

Inline JavaScript in the generated user page was syntax-checked. Metadata search, show details, and season endpoints returned responses during development inspection.

The replacement Edge Function was not compiled or deployed in that environment. Full browser testing, local-file testing, deployed-function testing, and video playback verification are still required.

## Third-party content and licensing

Third-party games, artwork, metadata, and video services remain subject to their owners' terms and rights. Links and catalogue entries do not grant redistribution rights. Only publish bundled assets and code that you are authorized to redistribute.

No license is selected by this README. If you intend to release your own code as open source, choose an appropriate license and add a `LICENSE` file after reviewing the rights to included third-party material.

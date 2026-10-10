# Connections Hub overhaul, privacy image proxy, Pro/Social tags, mobile Studio fix

## Current state (checked)
- Only **Spotify** actually works today. Every other tile, including Bluesky, YouTube, Xbox, Discord and Steam, is a "Binnenkort" placeholder.
- Strava, GitHub, WakaTime, Letterboxd, Chess.com, Lichess, Home Assistant and RSS are **not in the hub yet**. The briefing calls them "already built", but they will be added as new tiles.
- Xbox shows a generic controller because the icon set has no Xbox logo.

## 1. Categories (final mapping)
```text
Populair             top picks across all categories
Muziek               Spotify (live), Apple Music, Last.fm, SoundCloud, Deezer, YouTube Music
Gaming               Steam, Xbox, PlayStation, Riot, Epic, Battle.net, Roblox
Streaming            Twitch, YouTube (live: handle/channel ID), Kick
Socials              Bluesky (live), W.social (live), Instagram, TikTok, X, Discord,
                     Reddit, Mastodon, Threads, Snapchat, Tumblr, Pixelfed
Werk & Code          GitHub (live), GitLab (live), WakaTime (live), LinkedIn
Gezondheid & Fitness Strava, Oura, Garmin, Apple Health, Google Fit
Design & Portfolio   VSCO, Behance, Dribbble, Figma, ArtStation, Pinterest
Cultuur & Vrije Tijd Letterboxd, Chess.com, Lichess, StoryGraph, Goodreads
Smart Home & Zelf-Gehost  Home Assistant, RSS / Blog, Matrix / Element
Support & Betalingen Bitcoin Lightning, Monero, Ethereum (live: paste address -> QR/button);
                     Stripe, Buy Me a Coffee, PayPal
Alles
```
Anything not marked live shows "Binnenkort". The category chips scroll sideways on mobile.

## 2. What becomes live in this round
- **Bluesky and W.social:** connect with just a handle, using the AT Protocol engine you already have. The profile shows a live card.
- **YouTube:** a handle (like @mkbhd) or a channel ID shows the latest video from the public feed. No Google sign-in needed.
- **GitHub and GitLab:** a public username shows "Actief vandaag" or the latest public commit, using the public API.
- **WakaTime:** a public profile username shows something like "Coding in TypeScript · 2 u vandaag", from the public stats endpoint.
- **Crypto:** a pasted public address is format-checked per coin. The profile shows a QR code and a "Steun" button. Only public addresses are accepted, never keys.

## 3. Sign-in shortcut and verified badge
- If someone signed in with GitHub, GitLab, Google or Bluesky, that tile shows **"Activeer"** with their known handle filled in. One tap turns it on.
- Cards that come from a real sign-in get a small lock or check badge on the public profile. Typed-in handles never get that badge, and the server decides which is which.

## 4. Pro / Social / Beide tag
- Each connection gets a three-way selector: Pro, Social or Beide (the default). It's saved per connection, and the public profile shows only the cards that match the profile being viewed.

## 5. Privacy image proxy
- Album art, thumbnails and avatars from other services load through ROUT itself, so visitors never contact those companies directly.
- Only image types from a fixed list of known image hosts are allowed. The proxy also enforces a size limit and a timeout, refuses internal addresses, caches images and sends no cookies.

## 6. Mobile Studio fixes (checked at 375px)
- **Top action bar:** wraps onto two rows with spacing. The link and copy button sit on row 1; undo/redo, "Bekijk live profiel", "Verwerpen" and "Publiceren" sit on row 2, and "Publiceren" stays fully visible.
- **"Verbindingen & Integraties" header:** the title shortens with "…" when space runs out, and the badge or switch keeps its size on the right.
- **Bottom bar:** "Concept automatisch opgeslagen" turns into an icon only on very narrow screens, and the preview button keeps its full size.
- Screenshots at 375px and desktop width before finishing.

## 7. Database (Neon)
- One new update file adds a connections table. Each row holds the user, provider, handle or address, the Pro/Social/Beide tag, visible on/off, a verified flag and a short cached status. Spotify keeps its current storage.
- You run it with your migration login, as with earlier updates. Until then the hub still renders, and saving shows a clear "database update needed" notice.

## Not in this round
- Strava and the other "Binnenkort" services stay as placeholders. Most need developer keys from those services.
- Secret keys get added through the secure form once you're ready. Nothing in this plan needs a new key except optional GitHub/GitLab tokens for higher rate limits.

## Technical details
- `src/lib/integrations-catalog.ts`: new categories, items, `connectMode` (`oauth` | `handle` | `address` | `soon`), and an inline official Xbox SVG.
- New `db/NN_user_integrations.sql` (idempotent): `user_integrations(user_id, provider, handle, audience check in ('pro','social','both'), visible, verified, status_json, status_at, unique(user_id, provider))`, plus a `runSchemaEnsure` safety net.
- `src/lib/integrations.functions.ts`: `listMyIntegrations`, `saveIntegration` (zod-validated per provider; `verified` only set server-side from Better Auth linked accounts), `removeIntegration`, and a public `getProfileIntegrations` that respects `publicProfile` and audience.
- `src/lib/integrations/fetchers.server.ts`: YouTube RSS, GitHub/GitLab public events, WakaTime public stats, ATProto profile. Status is cached in `status_json` for about 10 minutes and rate-limited through `hitRateLimit`.
- `src/routes/api_.public.proxy-image.ts`: host allowlist, https only, content-type `image/*`, max 5 MB, 8 s timeout, no forwarded headers, `Cache-Control: public, max-age=86400`, cached in the internal bucket.
- `IntegrationsAccordion.tsx`: Activeer/Koppel logic, handle/address dialog with audience selector, verified badge.
- `ProfileEditor.tsx` around lines 860–880 and 1813: `flex flex-wrap gap-2`, `min-w-0`/`truncate`, `shrink-0`, and `hidden min-[380px]:inline` for the autosave text.
- Tests: audience filtering, verified-only-from-SSO, crypto address validation, proxy host allowlist and private-IP refusal.
- `AGENTS.md`: one rule each for the image proxy and the integrations table.

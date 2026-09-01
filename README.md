# iframe-poc-partner

**Part of a two-repo platform/partner `postMessage` POC.** This repo simulates a **partner** (third-party content provider) whose pages are embedded in a cross-origin iframe by the platform. The partner is required to load the platform's `browser-event-relay.js` event relay script from the platform repo by reference.

## Architecture

```
┌─────────────────────────────────────┐
│    PLATFORM PAGE (Vercel)           │
│    (iframe-poc-platform)            │
│                                     │
│  ┌──────────────────────────────┐   │
│  │  This Partner Iframe Content │   │
│  │  (cross-origin, GitHub Pages)│   │
│  │                              │   │
│  │  Loads browser-event-relay   │   │
│  │  .js from platform by URL    │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
         ▲
         │ postMessage() to platform
         │
  ┌──────┴─────────────┐
  │ YOUR CODE HERE     │
  │                    │
  │ index.html or      │
  │ index-dev.html     │
  │                    │
  │ (your content)     │
  │                    │
  │ The platform's     │
  │ event relay script    │
  │ auto-initializes   │
  │ and monitors       │
  │ user activity      │
  └────────────────────┘
```

## What This Repo Is

This repo is **example third-party content** that demonstrates how a partner would embed
the platform's required event relay script. In production:

- **This repo plays the "partner"** — arbitrary third-party content (e.g., a widget, sidebar, embedded app)
- **You modify `index.html` and `index-dev.html`** to add your own content
- **You must load the platform's `browser-event-relay.js` script** — this is the platform's requirement for embedding your content in their iframe
- The script auto-initializes and detects user activity (clicks, typing, scrolling, etc.)
- It sends messages back to the platform page via `postMessage()`

## Files

| File | Purpose |
|---|---|
| `index.html` | Example partner content (production) — loads platform's event relay script |
| `index-dev.html` | Example partner content (dev/preview) — loads platform's dev event relay script + dev banner |

## How the Event Relay Script Works (Read-Only for Partner)

The platform's `browser-event-relay.js` (loaded cross-origin via `<script src>`):

1. Detects user activity inside the partner iframe (click, keypress, scroll, mousemove)
2. Derives the platform's origin from `document.referrer` (automatically available cross-origin)
3. Extracts the **relay token** (`iframeId`) from the iframe's URL parameters
4. Sends minimal messages via `window.parent.postMessage()` to the platform (including the relay token)
5. Supports lifecycle management (`init()`, `cleanup()`) if your partner content is a single-page app

**You don't need to do anything** — the script auto-initializes on load. The platform automatically appends the relay token as a URL parameter when loading your iframe, so no configuration is needed. If your content is an SPA that unmounts/remounts, see **[PARTNER_INTEGRATION.md](../iframe-poc-platform/PARTNER_INTEGRATION.md)** in the platform repo for lifecycle management.

## Integration Checklist

To integrate as a real platform partner:

1. **Copy `index.html` from this repo** as a template
2. **Replace the placeholder content** with your actual app/widget/content
3. **Keep the `<script src>` tag** pointing to the platform's `browser-event-relay.js`:
   ```html
   <!-- Production (communicates with platform at https://iframe-poc-platform.vercel.app) -->
   <script src="https://iframe-poc-platform.vercel.app/browser-event-relay.min.js"></script>
   ```
4. **Update CSP** to allow scripts from the platform's domain:
   ```html
   <meta http-equiv="Content-Security-Policy" content="script-src 'self' https://iframe-poc-platform.vercel.app; ...">
   ```
5. **Ensure `referrerpolicy`** is not explicitly set to `no-referrer` (the platform's iframe uses `strict-origin`, so referrer will be available)
6. **For SPAs**: manage lifecycle on component mount/unmount:
   ```javascript
   // On mount
   if (!window.BrowserEventRelay.isInitialized()) {
     window.BrowserEventRelay.init();
   }
   
   // On unmount
   window.BrowserEventRelay.cleanup();
   ```

## Message Format (Information Only)

The event relay script sends this message format to the platform. You don't need to send messages yourself — the script handles it:

```javascript
{
  source: "browser-event-relay",
  type: "PARTNER_RELAYED_CLICK",     // or other event types
  timestamp: 1691743200000
}
```

For the full message contract and security details, see **[README.md](../iframe-poc-platform/README.md)** in the platform repo.

## Local Development

### Serve Both Repos Locally (Cross-Origin Testing)

For true cross-origin testing, run both repos on different ports:

```bash
# Terminal 1: Serve this partner repo on port 8000
python3 -m http.server 8000

# Terminal 2: Serve platform repo on port 8001
cd ../iframe-poc-platform
python3 -m http.server 8001

# Terminal 3: Open browser
# http://localhost:8001
```

The platform's dev `env-config.js` should point to `http://localhost:8000`:

```javascript
dev: {
    PARTNER_URL: 'http://localhost:8000/index.html',
    PARTNER_ORIGIN: 'http://localhost:8000'
}
```

Or use `index-dev.html`:

```javascript
dev: {
    PARTNER_URL: 'http://localhost:8000/index-dev.html',
    PARTNER_ORIGIN: 'http://localhost:8000'
}
```

### Verify Integration

Open browser console and:

1. Check that the event relay script loaded:
   ```javascript
   window.BrowserEventRelay  // Should be defined
   window.BrowserEventRelay.getVersion()  // Should return version
   ```

2. Check that parent origin was detected:
   ```javascript
   console.log(document.referrer)  // Should show platform's origin
   ```

3. Enable event relay script debug output:
   ```javascript
   // The script logs to console during init, look for "[browser-event-relay]" messages
   ```

4. Interact with the partner content (click, type, scroll) and check:
   - Platform header turns GREEN (ACTIVE)
   - Platform countdown timer counts down
   - Platform console shows `[Usage Meter]` messages when debug is enabled

## Deployment

This partner content is deployed separately from the platform. When deploying:

1. **Update event relay script URL** if the platform deployment domain changes
2. **Update CSP header** to allow scripts from the platform's new domain
3. **Notify the platform** of your new deployed URL so they can update their `env-config.js`

See the platform's [README.md](../iframe-poc-platform/README.md) for how the platform points to partner content.

## Development & Testing

### index.html vs. index-dev.html

- **`index.html`**: Production partner content, loads platform's production event relay script
- **`index-dev.html`**: Development partner content (includes blue dev banner), loads platform's dev/preview event relay script

The platform's `env-config.js` decides which one to load based on environment.

### Update the Dev Script URL

If you're testing with a local platform deployment or a different Vercel preview branch, update `index-dev.html`:

```html
<!-- Current -->
<script src="https://iframe-poc-platform-git-develop-phil-skaroulis-projects.vercel.app/browser-event-relay.min.js"></script>

<!-- To test local platform on port 8001 -->
<script src="http://localhost:8001/browser-event-relay.min.js"></script>
```

## Troubleshooting

### Event relay script not loading

**Check CSP header:**
```html
<!-- Should allow the platform's domain -->
<meta http-equiv="Content-Security-Policy" content="script-src 'self' https://iframe-poc-platform.vercel.app; ...">
```

**Check browser console (Network tab):**
- Is the script URL accessible? Try visiting it directly in a new tab.
- Is CSP blocking it? Look for CSP violation messages in console.

### `window.BrowserEventRelay` is undefined

- Verify the event relay script URL is correct and accessible
- Verify CSP allows the script load
- Check browser console for errors during script load

### Messages not being sent to platform

1. Verify the event relay script loaded: `window.BrowserEventRelay` should exist
2. Verify parent origin was detected: `console.log(document.referrer)` in partner console
3. If referrer is empty:
   - Platform must set `referrerpolicy="strict-origin"` on the iframe (default is `no-referrer`)
   - Partner CSP must not suppress referrer

### No platform activity when interacting with partner content

- Confirm you're interacting **inside the partner iframe** (click, type, etc.)
- Check platform console (run `window.UsageMeter.setDebug(true)`) for message traffic
- Verify both repos are running and the partner URL is correct

## For More Details

- **Platform's complete documentation**: [iframe-poc-platform README.md](../iframe-poc-platform/README.md)
- **Integration guide** (lifecycle management for SPAs): [PARTNER_INTEGRATION.md](../iframe-poc-platform/PARTNER_INTEGRATION.md)

## Technical Notes

- The event relay script is pure JavaScript (no dependencies or polyfills)
- It's minified (~1KB) for fast loading
- **IE9+ compatible** — uses manual URL/query parsing, no passive event options
- Automatically derives trust from `document.referrer` (no configuration needed)
- Handles lifecycle management for single-page apps
- Supported: IE9+, Chrome 32+, Safari 10+, Firefox 26+

## License

MIT

# Beam — direct device-to-device file & text sharing

A single-file web app. Connect two devices (scan a QR code, or type a code),
and either side can offer files or text to the other. Transfers happen
**directly between the two devices** over a WebRTC data channel — no file
ever touches a server.

## What's new in this version

1. **Fully bidirectional.** Once connected, both devices can add files/text
   and send to each other — there's no fixed "sender"/"receiver" role
   anymore. Both panels ("Your items" / "Their items") are live on both
   ends at the same time.
2. **Fixed the stuck status message.** The old "waiting for sender to press
   send" text no longer exists — status is now tracked per item
   (`available` → `requested` → `receiving X%` → `done`), so it always
   reflects what's actually happening.
3. **Metadata-first offers.** Adding something to your queue no longer sends
   the actual bytes. It broadcasts just the name/size/type (and a thumbnail
   for images) to the other device. The other side decides whether to
   download each item individually, or hit **Download all**.
4. **Automatic on the same network.** Right after connecting, Beam checks
   (via WebRTC's own connection stats) whether the direct link is a
   same-network ("host-to-host") connection or one that had to cross the
   open internet. If you're on the same LAN, files start downloading
   automatically as soon as they're offered. If you're on different
   networks, it falls back to the manual select/download flow from point 3.
   This detection is best-effort — on an unusual network setup it may not
   be able to tell, in which case it defaults to manual download.
5. **Parallel or queued transfer, your choice.** A "parallel transfers"
   selector (1–4, default **2**) controls how many items move at once. Extra
   requested items simply queue up until a slot frees.
6. **Thumbnails + inline preview.** Image files show a small thumbnail as
   soon as they're offered (generated locally, sent as a tiny JPEG
   alongside the metadata — no need to download the full file to see what
   it is). After an image is fully received, tap **View** to see it full
   size in an in-page preview, or **Save** to download it.

## Why there's still one external service involved

Two devices on different networks can't find each other on their own — they
need a few kilobytes of "here's how to reach me" info exchanged once, up
front. That's the **only** thing that touches an outside party:

- **Signaling (handshake only):** uses PeerJS's free public broker
  (`0.peerjs.com`), plus Google's public STUN server for NAT traversal.
  Neither ever sees your files or their metadata — only tiny
  connection-setup messages.
- **File / text data, and the file-list metadata itself:** flows only over
  the direct `RTCDataChannel` between the two browsers.
- **Large files:** if the receiving browser supports the File System Access
  API (Chrome/Edge desktop, Chrome on Android), it streams straight to disk
  instead of buffering the whole file in RAM. Otherwise it falls back to
  building a downloadable Blob in memory (fine for iOS Safari for
  reasonably sized files).

**One caveat:** on rare, very restrictive networks (symmetric NAT, locked-down
corporate firewalls) a direct P2P connection can fail to establish at all —
there's no TURN relay fallback in this version, so the connection will just
not connect rather than silently sending data through a relay.

## How transfer actually works under the hood

- All control messages (offers, download requests, start/done signals) go
  over one PeerJS `DataConnection` as JSON objects.
- File bytes are sliced into 16KB chunks. Each chunk gets a single-byte
  "slot" number prefixed to it before sending, so multiple files can be
  interleaved over that same one data channel — this is what makes
  parallel transfer possible without opening multiple connections.
- The sender pauses sending (checking `bufferedAmount` on the underlying
  `RTCDataChannel`) whenever the outgoing buffer gets too full, so a large
  file can't blow up either browser's memory.

## Deploying it

The app itself (the HTML/JS) still needs to be loaded by both browsers from
*some* URL — that's normal for any web app and is separate from the "no
server for my data" goal. Free static hosting works well:

**GitHub Pages**
1. Create a new repo, add `index.html` to it.
2. Repo Settings → Pages → Deploy from branch → `main` / root.
3. Your app is live at `https://<username>.github.io/<repo>/`.

**Netlify / Vercel (drag-and-drop, no git needed)**
1. Go to netlify.com (or vercel.com) → drag the folder containing
   `index.html` onto the dashboard.
2. You get a live HTTPS URL instantly.

HTTPS is required — camera-based QR scanning and clipboard access need a
secure context, and `localhost` works too for same-device testing.

## Using it

1. Open the app on either device. It starts broadcasting immediately — a
   QR code and a plain-text room code appear.
2. On the other device, scan the QR code, or open the app and type the code
   into the "connect to a code" box.
3. Once connected, both sides see "Your items" (add files/text here) and
   "Their items" (what the other device has offered).
4. Add something on either side — the other device sees it appear
   instantly as an available item. On the same network, it starts
   downloading automatically; otherwise, tap **Download** (or
   **Download all**) on the receiving side.

## Notes / possible next steps

- Only one active connection at a time — connecting to a new device closes
  any previous one.
- No encryption layer beyond WebRTC's own mandatory DTLS encryption on the
  data channel, which is already end-to-end between the two devices.
- Room IDs are random 7-character codes prefixed `beam-`, live only as long
  as that tab stays open.
- Same-network detection relies on WebRTC connection stats and is
  best-effort; it can occasionally misjudge on unusual router/VPN setups.

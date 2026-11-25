# Natoor: Wi-Fi Security (ناطور) · Security design

**[Get it on the App Store](https://apps.apple.com/app/id6837992617)** · iPhone · English and Arabic · by [Eng. Rawan Alfarraj](https://github.com/r1alfarraj)

Natoor checks a home Wi-Fi network: which devices are on it, which services they make reachable, and what to fix. A network scanner is a dual-use tool, so it was designed from the start to be **safe to point at a home network and useless as an attack tool**. This document explains those design decisions. The app's source code is not published.

## Threat model in one paragraph

The person using Natoor owns or manages the network. The risks to design against are: scanning hosts the user has no right to scan, hurting fragile home devices (cameras, printers, old routers) with aggressive traffic, the app itself becoming an attack surface (hostile replies from devices on the network), and leaking what it learns about the household. Each section below addresses one of these.

## 1. Scope: only the network you are on

- **Wi-Fi only, never cellular.** Network connections require the Wi-Fi interface and explicitly forbid cellular.
- **Private address ranges only.** Before anything is sent, the scan scope is built from the iPhone's real interface address and subnet mask. If the subnet is not entirely inside a private range (10/8, 172.16/12, 192.168/16, or carrier-grade NAT 100.64/10), the scan refuses to start. A misconfigured network that hands out public addresses cannot turn Natoor into an internet scanner.
- **No guessing.** The scope comes from the actual subnet and the gateway the system reports. It never assumes a /24 or a router at `.1`. Link-local, point-to-point and host routes are rejected.
- **Bounded size.** At most 256 IPv4 targets per automatic scan. On larger subnets it takes the aligned block around the iPhone and says the result is partial.
- **Stops when the network changes.** If Wi-Fi drops or the phone joins a different network mid-scan, the scan stops and its scope is invalidated. Results are never mixed across networks.

## 2. Gentle on devices

Home devices are often fragile, so every connection goes through one scheduler with rate limits, caps on connections per device and time budgets.

| Stage | Limits |
| --- | --- |
| Finding devices | 4 common ports per address, 700 ms timeout, no retries. An address stops being probed as soon as it answers. |
| Checking ports | 24 to 32 connections at once, 64 new per second overall, at most 4 at once and 8 per second to any one device. No retries: silence is reported as inconclusive. |
| Whole Home Check | stops at 60 seconds or 8,000 connection attempts and then says "partial", never "complete" |
| Full check of one device | only started by the user, for one device at a time; up to 64 connections per second, in resumable 20-minute slices |
| Gentle mode | the full device check uses one connection at a time |

Everything backs off for 2 seconds when iOS reports resource errors.

## 3. Read-only by design

Natoor observes. It never logs in, sends exploit payloads, guesses passwords or changes settings.

- **Service checks are narrow.** For SSH it only reads the server's greeting line and sends nothing. For web services it makes one request for the root page, keeps the status, `Server` header and page title, and reads at most 16 KiB. Redirects are not followed, scripts never run, and nothing is submitted.
- **The router is read-only in the code itself.** The only router operation that exists is `GetGenericPortMappingEntry`, which lists port forwards. There is no add, delete or configuration action to call, by mistake or otherwise.
- **Router requests are pinned to the router.** Description and control URLs from the router's UPnP replies are followed only if their host is the gateway itself. A malicious device cannot redirect Natoor to another host.
- **Strict response limits.** Router inspection is capped at 128 entries, 10 seconds and 256 KiB.
- **Changing the Wi-Fi password is left to the user.** iOS cannot change a router's password, and Natoor does not try. It generates a strong password with the system's cryptographic random source, checks strength offline, and puts it on the clipboard as local-only (no Universal Clipboard) with a 2-minute expiry.

## 4. Treating the network as hostile input

Devices on a home network can be compromised, so their replies are treated as untrusted:

- **No XML external entities.** The UPnP and SOAP parsers have external entity resolution switched off, which blocks XXE attacks from a hostile router.
- **Size caps everywhere.** Every network read has a byte limit and a deadline.
- **Sanitised text.** Device names, banners and page titles are cleaned and length-limited before display.

## 5. Privacy

- **No accounts, analytics, ads or crash-reporting SDKs**, and no server of mine. The App Store privacy label is "Data Not Collected", and the privacy manifest declares no tracking.
- **Results stay on the iPhone**, in one file protected by iOS data protection. You choose how long scan history is kept (90 days by default), and *Delete all data* removes everything.
- **Shared reports are redacted by default.** Addresses become stable placeholders and device names are left out unless you choose to include them.
- **The speed test is the only internet traffic, and it asks first.** Before the first test, a consent screen names the provider (Cloudflare), explains what it can see (your public IP address, like any website), states the data use (25 MB Lite, 150 MB Standard), and confirms it runs on Wi-Fi only. No scan data, device list or identifier is sent.

## 6. Honest results

- Every finding records how it was established: **observed**, **inferred** or **reported by the router**.
- Checks that were not run are never implied. IPv6 targets, UDP services and TLS configuration are not assessed in this version, and the app does not claim they were.
- There is no firmware database, so Natoor never says a device is up to date. It shows where to look for updates instead.
- *Verify fix* re-checks only the specific service a finding was about, on the same network.

## Reporting a security issue

If you find a security problem in Natoor, please report it privately by email, using the address in the contact section of [r1alfarraj.com](https://r1alfarraj.com/#contact). Please don't open a public issue. I'll confirm receipt and keep you updated until it's fixed.

---

Natoor is built in SwiftUI (Swift 6) for iOS 18 and later. See also my [vulnerability research](https://github.com/r1alfarraj/vulnerability-research).

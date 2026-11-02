# Mobile Bug Bounty Lab Setup

A practical, step-by-step guide to setting up a physical-device mobile security testing lab, coming from a web application security background. Built on real devices (not emulators/simulators) to avoid the common pitfalls of virtualized environments.

> **Status**: Work in progress. Sections marked `TODO` will be filled in as the lab setup progresses.

## Hardware

| Device | Model | Why |
|---|---|---|
| Android | Google Pixel 8a | Long OS update support window, clean AOSP-based firmware, well-documented factory images |
| iOS | iPhone 13 | Long iOS update support window, no jailbreak required for most testing workflows |

## Network Setup

### Goal

Give the lab devices (and the testing PC) a US-based IP address for programs where most private invitations are US-only, **without** routing the entire household's traffic through a VPN, and **without** breaking local proxy interception (Burp) between the testing PC and the mobile devices.

### Hardware

- Main router: provided by the ISP.
- Secondary router: **TP-Link Archer AX58**, set to operate in a secondary role as an access point extending the same home SSID.
- VPN provider: NordVPN (paid plan).

### Key design decision: per-device VPN, not a separate SSID

The initial instinct was to create a dedicated SSID for the testing devices with the VPN applied only to that network. **This turned out to be unnecessary and actively counterproductive.**

- The TP-Link VPN Client feature applies per-device (by MAC address), via a **Device List** inside the VPN Client settings — not per-SSID.
- A separate SSID (especially a Guest Network) introduces client isolation between devices on different network segments, which breaks local Burp proxy interception between the testing PC and the mobile devices.
- **Conclusion**: keep all devices on the same, single home SSID. Add only the specific MAC addresses of the testing PC (wired) and the mobile devices (Wi-Fi) to the VPN Client's Device List, and toggle VPN access per device from there.

### Setup steps

1. Set the Archer AX58 to **Router mode** (not Access Point/Range Extender mode) — this is required for the VPN Client feature to be available at all; bridge/AP mode disables most Layer 3 functionality on TP-Link firmware.
2. Connect it to the ISP router via a LAN-to-WAN cable (double NAT setup).
3. Configure the **VPN Client** with NordVPN using the manual OpenVPN/WireGuard credentials from the NordVPN service configuration panel (not your regular account login).
4. Keep the SSID identical to the main household network — **do not** create a separate SSID/Guest Network for this.
5. In the VPN Client's **Device List**, add the testing PC and both mobile devices by MAC address, and enable the VPN toggle only for those three.
6. Leave every other household device untouched — they continue to use the ISP router's normal connection.

### Gotchas

- If Burp proxy traffic between the PC and a mobile device mysteriously fails (timeouts, nothing showing up in Burp's HTTP history), check whether the device in question is on a Guest Network / isolated SSID — client isolation silently blocks local traffic between devices even when the VPN itself works fine.
- `TODO`: document whether/how to configure split tunneling if a future firmware update on the AX58 supports excluding the local LAN range from the VPN tunnel.

## iOS Setup (iPhone 13, no jailbreak)

### Why no jailbreak

Jailbreak tooling availability depends entirely on the current iOS version and whether a public exploit exists for it — unreliable long-term for a lab that needs to stay usable across iOS updates. The non-jailbreak workflow below trades some convenience (per-app patching instead of a system-wide Frida server) for long-term stability.

### 1. Dedicated Apple ID

- Create a new, dedicated Apple ID (never reuse a personal one) — avoids the country-change restrictions Apple imposes on accounts with existing purchase history.
- A **free** Apple ID is sufficient to start: signing certificates last 7 days and up to 3 apps can be signed simultaneously. Upgrade to a paid Apple Developer account (99 USD/year, 1-year certificates, no 3-app limit) only if the 7-day re-signing cycle becomes a real friction point.

### 2. US phone number for Apple ID verification

Apple requires a +1 number for account verification. Two important distinctions:

- **VoIP numbers** (free apps like TextNow): low success rate with Apple's verification (~20-40%), frequently rejected.
- **Non-VoIP numbers from a real carrier via eSIM** (e.g. Tello, running on the T-Mobile network): much higher success rate (~95-99%), because Apple can't distinguish them from a physical US number.

Setup used: **Tello**, minimal monthly plan (~5 USD/month: 100 minutes + unlimited SMS), eSIM.

- Tello's plan is a **recurring monthly subscription**, not a PAYG balance — it auto-renews as long as the payment method stays valid.
- If the eSIM activation gets stuck while trying to activate from outside the US, connect through a US-exit VPN (NordVPN in this case) during the signup/activation process — budget MVNOs often gate activation on the apparent origin IP.
- **E911 requirement**: Wi-Fi Calling (needed to receive SMS from outside the US without roaming charges) requires registering a real US residential address for emergency dispatch purposes. Virtual mailbox / CMRA addresses are typically rejected by the carrier's address validation. Use either an address tied to a real contact in the US, or a legitimate residential-address service (e.g. iPostal1, Anytime Mailbox) — not an arbitrary public location (restaurant, library, etc.), since that address is where emergency services would actually be dispatched if 911 is ever dialed by accident from that line.
- Once E911 is resolved, enable **Wi-Fi Calling** in the Tello dashboard and on the iPhone (Settings → Cellular → [Tello line] → Wi-Fi Calling). This allows receiving SMS over home Wi-Fi with **no roaming charges**, since it registers over the internet rather than over a local cell tower.
- Keep **International Roaming disabled** in the Tello dashboard — this makes it physically impossible to incur cellular roaming charges; without it, the line simply shows "no service" over cellular outside the US.

### 3. Developer Mode

- Settings → Privacy & Security → Developer Mode is **hidden by default** and only appears after the first attempt to install a developer-signed app (e.g. via Sideloadly). It doesn't need to be pre-enabled manually in most cases — just proceed with the first sideload attempt and iOS will prompt for it.
- Requires a device restart plus a confirmation dialog after reboot to fully enable.

### 4. Burp Suite certificate + proxy

1. In Burp: **Proxy → Proxy settings → Proxy listeners**, edit the listener and set "Bind to address" to **All interfaces** (default is loopback-only, which blocks the phone from connecting). Save the config inside a **saved project file** (not a Temporary Project) — listener binding settings are tied to the project file and are lost on close with temporary projects.
2. On the iPhone: Settings → Wi-Fi → (i) next to the connected network → HTTP Proxy → Manual → enter the PC's local IP and Burp's listening port.
3. On the iPhone, open Safari and go to `http://burp` to download Burp's CA certificate; install the resulting configuration profile.
4. **Critical step people often miss**: Settings → General → About → Certificate Trust Settings → enable full trust for the installed CA certificate. Without this, the certificate is installed but not trusted by apps, and SSL errors persist.
5. Make sure Windows Firewall allows inbound connections on Burp's listening port, or the phone will simply time out trying to reach the proxy.

### 5. Frida Gadget injection (for SSL-pinned apps, non-jailbroken)

Since there's no jailbreak, there's no system-wide `frida-server`. Instead, the Frida Gadget (a `.dylib`) is injected directly into the target app's IPA before signing and installing.

1. Install **Sideloadly** on the testing PC — on Windows, this requires the **official Apple iTunes installer** (not the Microsoft Store version), for the Apple Mobile Device Support drivers/services Sideloadly depends on.
2. Install `ipatool` (`pip install ipatool`) and authenticate with the dedicated US Apple ID — this allows downloading unencrypted IPAs directly from the App Store.
3. Download the Frida Gadget `.dylib` matching the exact installed `frida-tools` version (`frida --version`) from the Frida GitHub releases page — version mismatches between the gadget and the client break the connection.
4. **Windows-specific note**: `objection patchipa` requires macOS (Xcode codesign tooling) for the signing step. On Windows, skip Objection for the patching step and use **Sideloadly's built-in dylib injection** instead (Advanced options → enable "Inject dylibs/Frameworks" → add the Frida Gadget `.dylib`). Sideloadly handles both the injection and the final signing in one step, no macOS needed. Objection's *client* (`objection explore`, SSL pinning bypass commands, etc.) still works fine from Windows once the gadget is running — only the IPA patching/signing step needed macOS tooling.
5. Validate the full pipeline against a free throwaway app before using it on a real target: download its IPA, inject the gadget via Sideloadly, install, open the app, and confirm from the PC with `frida-ps -U` that the process shows up.

## Android Setup (Pixel 8a)

`TODO` — rooting with Magisk, Zygisk/Shamiko, moving the Burp cert to the system trust store, frida-server.

## Testing Tools

`TODO` — Burp Suite, Objection, MobSF, jadx, apktool, Drozer, Ghidra.

## Methodology / Workflow

`TODO` — recon → static analysis → dynamic analysis → IPC surface → deep links → native libraries.

## Notes on Scope for Bug Bounty

`TODO` — what's typically in/out of scope (local storage without a realistic attack vector, exported components, deep links, hardcoded secrets, etc.).

## References

- [OWASP MASTG (Mobile Application Security Testing Guide)](https://mas.owasp.org/MASTG/)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [HackTricks — Android Applications Basics](https://book.hacktricks.xyz/mobile-pentesting/android-app-pentesting/android-applications-basics)
- [HackTricks — Android APK Checklist](https://book.hacktricks.xyz/mobile-pentesting/android-checklist)
- [HackTricks — iOS Pentesting Checklist](https://book.hacktricks.xyz/mobile-pentesting/ios-pentesting-checklist)
- [Drozer (WithSecure/Reversec)](https://labs.withsecure.com/tools/drozer/)

# Akile DNS (AKDNS) Guide: What the Free Beta Gives You, How the Script Rewrites Your Resolver, and Where It Stops

AKDNS is Akile's DNS-layer unlock service: you point a Linux box's resolver at it, and requests for streaming and AI domains get answered with an IP from a node that can reach those services. Nothing is tunnelled, no client is installed, and unconfigured domains resolve exactly as before. It is in open beta and free right now, and an account that isn't an Akile customer can still register **10 IPs** at no cost.

That last number is the part most people want first, so here it is up front. Below: what the beta actually includes, which eight regions the nodes sit in, the one command that installs it, and the part that matters most if this is going on a machine you care about. The "set as system DNS" option does not append a nameserver. It disables your resolver stack, writes a static `/etc/resolv.conf`, and locks the file with `chattr +i`.

> **Quick take:** If you have a spare Linux VPS and want Netflix, Disney+, ChatGPT or Claude to see a different region without running a proxy, AKDNS costs nothing to try during the beta. Budget ten minutes and read the script before you pipe it into bash.
>
> 👉 [Create an Akile account to reach the AKDNS console](https://akile.io/register?aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)

## The plans: one tier, and it's free

Akile has not published a paid AKDNS tier, a price, a currency, or a renewal rate. What exists today is the open beta, and the official announcement on LowEndTalk (19 October 2026) lists exactly one commercial term: free, with 10 IPs for people who aren't Akile customers.

| Plan | Price | IP allowance | Node regions | Services | Get started |
| --- | --- | --- | --- | --- | --- |
| Open beta (current) | Free, no subscription | 10 IPs for non-Akile accounts | 8: Hong Kong, Japan, US, Singapore, Korea, Taiwan, Germany, UK | 60+ | [Register and open the console](https://akile.io/register?aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |

Two things the table can't show honestly. First, the beta has no announced end date, and Akile has said nothing about what AKDNS will cost afterwards, so treat the free period as open-ended but not guaranteed. Second, a third-party write-up from May 2026 states that AkileCloud customers who log in via OAuth can add unlimited server IPs. The October official post only confirms the 10-IP allowance for non-customers, so if unlimited IPs is the deciding factor for you, verify it in the console after signing in rather than planning around a six-month-old article.

The 10 IPs are per account, and rules are applied per IP. Each machine gets its own list of services to unlock; adding a second server doesn't inherit the first one's settings.

## What the script actually changes on your box

This is the section the promotional posts skip, and it's the one that decides whether you're comfortable running this on production.

A normal "change your DNS" instruction means editing `/etc/resolv.conf` and hoping nothing overwrites it. AKDNS's installer takes a different route because on most modern distros that file isn't really yours: it's a symlink into systemd-resolved, or a stub that NetworkManager regenerates, or something netplan renders at boot. Appending a nameserver to it survives until the next DHCP lease.

The installer's approach, as documented in the public `akile-network/aktools` repository (script version 3.0.0 at time of writing):

- It detects which backend is managing DNS and stops it. systemd-resolved gets disabled, NetworkManager is set to `dns=none`, resolvconf is turned off, and netplan configuration is processed.
- It writes a static `/etc/resolv.conf` pointing at whichever node won the latency test.
- It applies `chattr +i` to that file so dhclient, cloud-init or a reboot can't replace it.
- If `nsswitch.conf` is missing the `dns` entry, the script adds it, so the resolver is actually consulted.

Before any of that, it copies your existing configuration into a backup directory. The restore option in the menu is the inverse of the whole sequence: unlock the file, put back `resolv.conf`, the systemd-resolved config and any drop-ins, the NetworkManager connection settings keyed by UUID, netplan files and `nsswitch.conf`, then re-enable the services it stopped and run a resolution check to confirm you're back where you started. If the backup step fails, the script aborts rather than proceeding, which is the right failure mode.

There are two modes. Temporary leaves the change in place until a reboot or a NetworkManager restart undoes it, and is what you want for a first look. Permanent is the full takeover described above. On a box where you'll later want to point DNS somewhere else, remember the immutable flag is still set: unlock it or run the restore first, or your next DNS change will silently fail.

One boundary worth stating plainly, since it's in Akile's own post: the script performs connectivity checks and DNS configuration. It installs no proxy and doesn't touch how your traffic is routed. Anything you didn't select for unlocking resolves through normal DNS and leaves over your normal path.

## Installing it: six steps, one command

Run this as root or with sudo on the Linux machine you want to configure:


wget -qO- https://raw.githubusercontent.com/akile-network/aktools/refs/heads/main/akdns.sh | bash


It drops you into an interactive menu. The flow that works:

1. Add the machine's public IP in the AKDNS console at `dns.akile.ai/console` first. The script can't register the IP for you.
2. Run the baseline unlock check to see what the box can reach before any changes.
3. Run the DNS speed test. The script queries the regional nodes (default test domain `www.google.com`, five queries, one-second timeout) and ranks them by average latency.
4. Set the chosen node as system DNS. The menu has a combined "speed test and apply" option if you'd rather do both in one go.
5. In the console, tick the services this IP should unlock.
6. Re-run the unlock check to confirm the result changed.

If piping a remote script into bash bothers you, that instinct is correct and Akile says as much in the announcement. The script is public in the `aktools` repository; read it before running it. The console login uses your Akile account, which is why the entry point below goes through registration.

👉 [Sign up for Akile to add your first IP to the AKDNS console](https://akile.io/register?aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)

One caveat on step 2 and step 6: the unlock check inside the menu is the third-party open-source `1-stream/RegionRestrictionCheck` script, and the tool prints a notice saying the script and its results aren't official AKDNS output. It injects AKDNS context before running, and aborts if that injection fails rather than running an unlabelled third-party script. Treat the output as a useful third-party check, not a vendor guarantee.

## When UDP 53 is blocked

Many VPS providers and networks filter or poison plain DNS on port 53, which would make a DNS-based unlock service useless. The script handles this: if every UDP query times out, it retries the same tests over TCP, and if TCP works it applies the configuration with `options use-vc` written into `resolv.conf` to force DNS over TCP.

There's an exception you should know before you start. On Alpine and other musl libc systems, the script warns that musl may not honour `options use-vc`, so forced TCP may simply not take effect. If you're on Alpine, do the temporary-mode test first and confirm resolution actually works before committing to the permanent takeover.

## Where AKDNS stops

The scope is narrower than "unlock everything", and the honest limits are:

- **Only the domains you selected.** Unlocking is per-IP and per-service. A service you didn't tick resolves normally, which is also why the vendor says websites, SSH, mail and databases are unaffected.
- **Beta stability.** Service coverage is still being added, and a beta can change under you. Don't build a revenue-critical workflow on it without a fallback.
- **Resolver takeover is invasive by design.** Disabling systemd-resolved and locking `resolv.conf` is what makes it reliable across distros, and it's also what will confuse you in six months when a DNS change doesn't stick.
- **Eight regions only.** AkileCloud's VPS network spans 18 countries and 50+ locations, but the AKDNS unlock nodes are Hong Kong, Japan, US, Singapore, Korea, Taiwan, Germany and UK. Don't read the hosting footprint as the unlock footprint.
- **No published pricing.** Free today, undetermined later, with no end date announced.

Who shouldn't bother: anyone who needs a specific residential IP for a service that blocks datacentre ranges, anyone who needs all traffic routed through another country rather than a handful of domains, and anyone running a DNS setup they don't fully understand and can't restore by hand if the script's restore path fails.

## FAQ

**How is this different from a VPN?**
A VPN takes over your routing and sends all traffic through a tunnel. AKDNS answers DNS queries differently for the domains you picked, so those requests egress via an unlock-capable node while everything else stays on your normal path. Lower overhead, narrower scope, and no encryption layer.

**Does it survive a reboot?**
In permanent mode, yes, by design: the static `resolv.conf` is locked with `chattr +i` so DHCP clients and cloud-init can't overwrite it. In temporary mode, no, and a NetworkManager restart will also revert it.

**Can I use it on a server that isn't from Akile?**
Yes. The beta is explicitly open to non-Akile customers with a 10-IP allowance, and there's no restriction on provider or carrier.

**Can I run it on a router and share it with the LAN?**
Any device that lets you set a custom DNS server can point at AKDNS, so a soft router or home gateway can pass the unlocking to devices behind it. That's a third-party description of the use case rather than something Akile's announcement spells out, so test it on one client first.

**How do I undo it?**
Run the installer again and pick the restore option. It backs up before restoring and aborts if the backup fails.

**How long is it free?**
No end date has been announced, and no future pricing has been published.

## How to decide

If you have one Linux VPS and a short list of services you want to reach from a different region, the beta costs you nothing but ten minutes and a config backup. Start in temporary mode, confirm the unlock check changes, then decide whether the permanent takeover is worth the locked `resolv.conf`.

If you're running multiple machines, count them first. Ten IPs covers a small fleet; beyond that, the unlimited-IP claim for Akile account holders is worth confirming in the console before you assume it applies.

Two things to check before you start: that your provider doesn't block outbound TCP 53 as well as UDP (the script will tell you during the speed test), and that you know how to unlock a file with `chattr -i` in case you need to change DNS by hand later.

👉 [Register an Akile account and set up AKDNS on your first IP](https://akile.io/register?aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)

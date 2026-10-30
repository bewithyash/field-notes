---
title: "From a Locked-Down Citrix Desktop to a Domain Shell"
date: August 2026
source: https://blog.imperonlabs.com/blog/from-a-locked-down-citrix-desktop-to-a-domain-shell.html
---

Red Team

# From a Locked-Down Citrix Desktop to a Domain Shell

By Imperonlabs Research · August 2026 · 11 min read

A financial institution gave contractors a published Citrix desktop and called it containment. No command prompt, no visible drives, network isolation, the works. Every control in it looked reasonable on its own. Chained in sequence, they walked me from a sealed kiosk to a shell on an internal domain host in five steps, and not one of the steps needed an exploit.

**Scope note.** Sanitized from a real red team engagement. Paths, account names, and directories are examples. This was authorized offensive testing against a client environment.

Kiosk and locked desktop breakouts are rarely about a single clever trick. They are about a set of restrictions that each assume a different layer is doing the real enforcing, so that stepping around them one at a time defeats the whole model. This environment had five such assumptions. Here they are in the order I used them.

## Step 1: the drive that was hidden, not denied

The file manager showed no C drive. Hidden drives are a favorite kiosk control and a weak one, because hiding something from the GUI is not the same as denying access to it. I typed the path directly into the address bar:

```
C:\
```

The drive opened. The restriction lived in the interface, not in the filesystem permissions, so anything that bypassed the interface bypassed the restriction. Now I could enumerate the disk.

## Step 2: the provisioning leftovers

Browsing the drive turned up a directory left behind from imaging the machine:

```
C:\TempCitrix\RDP-W7\
```

Inside were `.rdp` connection files from provisioning, readable, with permissive access controls, pointing at internal hosts. Provisioning artifacts like these are ready made shortcuts into the network, and they survive because nobody owns cleaning them up after the image is built. They told me where to knock next.

## Step 3: RDP with an account that should not have had it

I opened one of the leftover `.rdp` files and authenticated with an ordinary personal domain account:

```
DOMAIN\jdoe
```

This account had no business holding remote desktop rights, and yet the connection succeeded. Worse, the session greeted me with:

```
Another user is signed in. If you continue, they will be disconnected.
```

That one message is three findings. Remote desktop logon rights were granted far too broadly, sessions were not isolated per user, and I could forcibly disconnect a legitimate user on demand. I was now on an internal host, as a normal domain user, on a system I was never meant to reach remotely.

Five controls, each assuming the next one holds

1 Drives hidden in the GUI
Bypass: type C:\ into the address bar

2 Provisioning cleanup skipped
Bypass: read leftover .rdp files

3 RDP logon rights too broad
Bypass: connect as an ordinary domain user

4 cmd blocked in Citrix only
Bypass: launch cmd from Win + R

5 Sessions not isolated
Result: shell on an internal host, users evictable

No single control was the failure. Each one held at one layer and was absent at the layer that mattered, so the chain defeated the model.

## Step 4: the shell block that only covered one door

Citrix application restrictions stopped me launching `cmd.exe` or `powershell.exe` directly. The Run dialog did not care:

```
Win + R  →  cmd  →  Enter
```

The application control layer blocked the shortcuts it knew about and left the Run dialog as an open execution path outside its reach. The block was real at one layer and completely absent at the one that mattered. I had a command prompt.

## Step 5: standing on the internal network

From the shell on the internal host, a few standard commands turned the foothold into situational awareness:

```
hostname
ipconfig
net localgroup administrators
whoami /groups
```

That gave me the internal hostname, the corporate subnet layout, local administrator group membership, and my own group memberships and privileges. Combined with the RDP credentials and the ability to evict live sessions, this is a solid base for lateral movement, not a curiosity. The containment that was supposed to hold a third party inside a sealed desktop was gone.

## How to fix it properly

Fix all five, because any one left in place keeps the chain alive. Remove or lock down provisioning directories with NTFS permissions restricted to administrators, so leftover `.rdp` files are gone or unreadable. Enforce remote desktop logon rights explicitly through a dedicated group, strip broad domain groups and Authenticated Users from the permission, and front RDP with an RD Gateway or firewall rules that limit it to management subnets. Configure Citrix and RDP for strict per user session assignment so shared sessions and disconnect attacks are impossible. Enforce application execution with AppLocker or WDAC that covers `cmd.exe` and `powershell.exe` through allowlisting, not GUI hiding, so the Run dialog is not a bypass. And stop treating interface visibility as a control: restrict access with permission objects and policy, never by hiding an icon.

## How to approach a locked desktop on any engagement

1. Assume every GUI restriction is cosmetic until proven otherwise. Try the address bar, dialogs, and file pickers to reach what the menus hide.
2. Hunt for provisioning and imaging leftovers. Temp directories, saved connection files, and installer logs are shortcuts into the network.
3. Test whether your low value account has rights it should not, especially remote logon rights, before assuming you need to escalate.
4. Look for a second door to the same capability. If one launch path for a shell is blocked, another almost always is not.
5. Chain the weaknesses. Each control assumes another is enforcing, so map the assumptions and step through the gaps in order.

The one line to remember
Containment is only as strong as its weakest assumption. Every layer here trusted another layer to enforce, and stepping around them one at a time defeated the whole model without a single exploit.

### Would your locked-down desktop survive a curious contractor for an afternoon?

Imperonlabs tests kiosk and Citrix builds the way a real insider would, chaining the small assumptions that each look fine alone. Book a scoping call and we will find the breakout path before someone else does.

[Request an assessment](https://imperonlabs.com/#contact)

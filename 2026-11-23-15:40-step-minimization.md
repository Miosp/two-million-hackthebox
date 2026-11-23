# 15:40 — Step-minimization tests: can the chain be shortened?

Priority per user: reasoning-first, then length. Every candidate shortcut tested live.

## Front of chain — can we drop invite / register / login?
| Test | Result | Conclusion |
|---|---|---|
| PUT settings/update, no cookie | 401 | session required — login stays |
| POST vpn/generate, no cookie | 401 | admin session required — mass-assign stays |
| POST register, junk invite code | 302 → /register?error=Code+is+invalid! | code validated server-side — invite stays |
| POST invite/generate, no cookie | 200 | only unauth endpoint of the four |

Every hop is enforced server-side. Five requests to RCE is the floor:
invite → register → login → is_admin → vpn/generate.

## Local privesc — anything cheaper than the kernel bug?
Full sweep as www-data on 10.129.148.45:
- SUID binaries: stock Ubuntu 22.04 set (su, sudo, passwd, mount, chfn/chsh, gpasswd,
  newgrp, umount, fusermount3, dbus helper, polkit-agent-helper, snap-confine +
  snap core20 copies). No pkexec installed → PwnKit N/A. sudo useless (admin not in sudoers).
- getcap -r /: nothing.
- Cron: only stock logrotate/man-db/anacron placeholders; nothing custom, nothing writable.
- Kernel alternates: Dirty Pipe patched in 5.15.70 (fixed 5.15.25);
  GameOver(lay) = same effort class as CVE-2023-0386, no gain.
→ CVE-2023-0386 (OverlayFS/FUSE) confirmed as the only real root step.
  Box gcc can't compile (cc1 exec fails) → off-box build is required, not optional.

## Structural shortening (kept)
- Root flag does NOT need the admin hop: www-data → CVE-2023-0386 → root directly.
  `su admin` is only for user.txt.
- One-shot variant (no listener): cmd injection runs `echo PASS | su admin -c "cat user.txt"`
  with output exfiltrated via cert CN (base32, no slashes); root flag via gc rewritten to
  `cat /root/root.txt > /tmp/f` + CN exfil. ~4 curls, but blind (harder to debug);
  kept as note, interactive shell remains the demo path.

## Verdict
Current DEMO.md chain is step-minimal given server-side checks. Improvements = add
reasoning (why each step) rather than removing steps.

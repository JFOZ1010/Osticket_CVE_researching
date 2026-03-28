# CVE-2026-XXXX — BOLA / IDOR in osTicket `ajax.tickets.php`

> **Broken Object Level Authorization (BOLA) — Insecure Direct Object Reference (IDOR)**  
> `include/ajax.tickets.php` → `viewField()` function  
> Reported by [@JF0x0r](https://github.com/JFOZ1010) · March 27, 2026

---

## At a Glance

| Field | Details |
|---|---|
| **Vulnerability** | BOLA / IDOR (Broken Object Level Authorization) |
| **Target** | osTicket v1.18-git — commit `2570d69` |
| **Component** | `include/ajax.tickets.php` |
| **Function** | `viewField()` — lines 805–806 |
| **Endpoint** | `GET /scp/ajax.php/tickets/{ticket_id}/field/{field_id}/view` |
| **CVSS 4.0 Score** | **8.2 HIGH** — `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:N/VA:N` |
| **CWE** | CWE-862 (Missing Authorization), CWE-639 (Auth Bypass via User-Controlled Key) |
| **Status** | Responsibly disclosed — awaiting patch |

---

## Who Am I

I'm Juan Felipe Oz ([@JF0x0r](https://github.com/JFOZ1010)), a security researcher passionate about open source security. I don't do this for bounties — I do it because I believe the tools people rely on should be safe. When I find something, I report it responsibly, document it properly, and share it publicly once it's fixed.

---

## What I Found

While doing a manual code review of osTicket's AJAX subsystem, I noticed something off in `ajax.tickets.php`. The `viewField()` function handles requests to view ticket field data — and it does retrieve the ticket object and validates the field exists. But it **never checks whether the requesting agent actually has permission to access that ticket**.

No `checkStaffPerm()`. No department validation. Nothing.

This means any authenticated agent — even one strictly limited to a single department — can read ticket fields from **any other department** in the system, just by knowing or guessing the `ticket_id` and `field_id`. Those are sequential integers. Easy to enumerate.

What makes this particularly clear-cut is the comparison with `editField()`, the sister function right above it in the same file. `editField()` correctly calls `$ticket->checkStaffPerm($thisstaff, Ticket::PERM_EDIT)` and returns HTTP 403 on violation. The fix was already implemented for writes — it was simply never applied to reads.

---

## Proof of Concept — Live Demo

I recorded a full end-to-end demonstration of the exploit in a controlled lab environment:

[![PoC Video — BOLA/IDOR osTicket](https://img.shields.io/badge/▶%20Watch%20PoC-Google%20Drive-blue?style=for-the-badge&logo=google-drive)](https://drive.google.com/file/d/1xpHmI-3ZFSYQc7qeRtKGVTEbbtanohl7/view)

The video walks through:
- Lab setup with two isolated departments (Dept-A and Dept-B)
- Agent `agent_a` authenticated with access restricted to Dept-A only
- Crafting the unauthorized request targeting a Dept-B confidential ticket
- The server returning **HTTP 200** with the restricted ticket field data exposed
- Replay after the patch showing **HTTP 403 — Permission denied**

The `exploit.py` script in this repository automates the full chain (authentication → enumeration → unauthorized field access) and was used during the assessment to confirm the issue scales beyond manual testing.

---

## Impact

- **Sensitive data disclosure** — any agent can read confidential ticket fields across all departments
- **Horizontal privilege escalation** — departmental boundaries are completely bypassed
- **Mass enumeration** — sequential `ticket_id` / `field_id` integers make bulk scraping trivial
- **Multi-tenant confidentiality breach** — defeats the core design principle of osTicket's department isolation model

---

## The Fix

A single line insertion in `viewField()`, immediately after the ticket object is retrieved — mirroring exactly what `editField()` already does correctly. Full technical details, diff, and CVSS breakdown are in the attached report.

📄 [`BOLA_IDOR_osTicket_Report_v2.pdf`](./BOLA_IDOR_osTicket_Report_v2.pdf)

---

## Disclosure Timeline

| Date | Event |
|---|---|
| March 27, 2026 | Vulnerability discovered and documented |
| March 27, 2026 | Report sent to `security@osticket.com` |
| — | Awaiting response from osTicket team |
| — | CVE assignment pending via GitHub CNA |

---

## Files in This Repo

```
.
├── README.md                        # This file
├── BOLA_IDOR_osTicket_Report_v2.pdf # Full technical disclosure report
├── exploit.py                       # PoC automation script
└── PoC_osTicket.mov                 # Local copy of the demo video
```

---

## Responsible Disclosure

I reported this privately to the osTicket security team before publishing anything. This repository was made public only after the responsible disclosure window. If you are an osTicket maintainer and have questions, feel free to reach out directly via GitHub.

---

<p align="center">
  Found by <a href="https://github.com/JFOZ1010">@JF0x0r</a> · Open source security matters.
</p>
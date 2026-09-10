# Vulnerability Management Portfolio Project

**Author:** Vishal · **Tool:** Tenable Vulnerability Management (Nessus) · **Environment:** The Cyber Range

In this project I carry out the technical core of a vulnerability management program — asset discovery, authenticated and unauthenticated scanning, CIS/STIG compliance auditing, and two independently verified **discover → assess → remediate → verify** remediation cycles — against a mixed Windows/Linux lab environment using Tenable Vulnerability Management.

***Inception state:*** a set of unassessed lab hosts with no scan history, including a Windows server running an end-of-life third-party application and three unhardened security settings.

***Completion state:*** every host scanned both unauthenticated and authenticated, a CIS/STIG compliance baseline established, and two hosts brought from active Critical/High/Medium findings to a fully verified clean state — each fix confirmed by re-scan, not assumed.

> **Scope note:** this document covers the scanning and remediation *execution* phases of the Cyber Range's "Implementing a Full VM Program" curriculum. It does not include the policy drafting, stakeholder buy-in meetings, or Change Advisory Board process from that same curriculum — those are separate deliverables not reflected in the scan data this report is built from.

---

## Technology Utilized

- Tenable Vulnerability Management (Nessus scan engine)
- CIS Benchmark / DISA STIG compliance auditing
- Windows Server, Windows 11 (target hosts)
- Ubuntu Linux, Samba, vsftpd (target hosts)

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [Objective & Scope](#objective--scope)
- [Methodology](#methodology)
- [Phase 1 — Network Discovery](#phase-1--network-discovery)
- [Phase 2 — Linux Host Assessment](#phase-2--linux-host-assessment-unauthenticated-vs-authenticated)
- [Phase 3 — Windows Host Assessment](#phase-3--windows-host-assessment-unauthenticated-vs-a-credential-failure)
- [Phase 4 — Windows 11 CIS/STIG Compliance Scan](#phase-4--windows-11-cisstig-compliance-scan)
- [Phase 5 — Full Remediation Lifecycle (Vulnerable Windows Server)](#phase-5--full-remediation-lifecycle-vulnerable-windows-server)
- [Phase 6 — Manual Remediation on Linux](#phase-6--manual-vulnerability-remediation-on-linux-smb-signing)
- [Remediation Effort Summary](#remediation-effort-summary)
- [Aggregate Findings Across the Engagement](#aggregate-findings-across-the-engagement)
- [Lessons Learned](#lessons-learned)
- [Skills Demonstrated](#skills-demonstrated)
- [Conclusion](#conclusion)

---

## Executive Summary

This project documents a full vulnerability management cycle carried out against a mixed Windows/Linux lab environment using Tenable Vulnerability Management: asset discovery, unauthenticated and authenticated scanning, a CIS compliance audit, and — as the centerpiece — a complete **discover → assess → remediate → verify** lifecycle against a deliberately vulnerable Windows server.

That server started with **2 Critical and 8 High-severity findings** (an end-of-life, unsupported version of Wireshark) plus three OS-hardening gaps (SMB signing, RDP Network Level Authentication, and legacy LAN Manager authentication). Through targeted remediation — patch/software removal followed by three individually verified configuration fixes — the server was brought to **0 Critical, 0 High**, with only a self-signed lab certificate and an informational ICMP finding left outstanding.

| Metric | Before | After |
|---|---|---|
| Critical findings | 2 | 0 |
| High findings | 8 | 0 |
| Medium findings (actionable) | 3 | 0 |
| Total findings | 23 | 3 |

A second, smaller remediation exercise reinforced the same discipline: an exposed SMB/FTP file-sharing service on a separate host was found (SMB signing not required, FTP accepting cleartext credentials), manually remediated by taking the services offline, and confirmed clean on re-scan.

---

## Objective & Scope

The goal was to simulate the vulnerability management lifecycle an analyst performs in a real SOC/IT environment:

1. Discover assets on the network.
2. Scan them both **unauthenticated** (attacker's-eye view) and **authenticated** (defender's-eye view), to compare visibility.
3. Run a **compliance/hardening audit** (CIS benchmark) against a Windows 11 host.
4. Take one intentionally vulnerable server through a **full remediation cycle**, re-scanning after each fix to verify it actually worked — not just assuming it did.
5. Manually remediate a separately discovered exposure on a Linux host and independently verify the fix via re-scan.

**In scope:** 10 lab hosts across two subnets (10.0.0.0/24 and 10.2.0.0/24), scanned with Tenable Vulnerability Management between June 29 and July 11, 2026.

This mirrors the discover → assess & prioritize → change-managed remediation → verify lifecycle taught in the Cyber Range's "Implementing a Full VM Program" curriculum. Phases 1–4 below cover the scanning fundamentals (discovery, authenticated vs. unauthenticated scanning, compliance/STIG auditing); Phases 5–6 cover the remediation lifecycle — Phase 5 walks through curriculum Remediations #1–#6 against the vulnerable Windows server, and Phase 6 is the corresponding manual-remediation exercise on Linux.

---

## Methodology

| Phase | Activity |
|---|---|
| 1 | Network/subnet discovery |
| 2 | Linux host assessment — unauthenticated vs. authenticated (root vs. limited privilege) |
| 3 | Windows host assessment — unauthenticated vs. authenticated |
| 4 | Windows 11 CIS/STIG compliance scan |
| 5 | Full remediation lifecycle on a vulnerable Windows server — Remediations #1–#6 (7 scans) |
| 6 | Manual vulnerability remediation on Linux — exposed SMB/FTP service, single-finding fix and verification (3 scans) |

All scans were run from Tenable Vulnerability Management. Severity is Tenable's standard Critical/High/Medium/Low/Info scale.

---

## Phase 1 — Network Discovery

A subnet discovery scan against `10.0.0.0/24` identified four live hosts (`10.0.0.10`, `10.0.0.100`, `10.0.0.198`, `10.0.0.8`) — this scan covered one /24 segment rather than the full Cyber Range address space, so it's a partial pass at the curriculum's "Discovery Scan: Entire Cyber Range Subnet" exercise. It was a lightweight ping/host-discovery pass — no vulnerability findings, just confirming what was alive before deeper scanning.

---

## Phase 2 — Linux Host Assessment: Unauthenticated vs. Authenticated

This phase demonstrates why **credentialed scanning matters**: the same host looks very different depending on how much access the scanner has.

| Scan | Target | Auth | Critical | High | Medium | Low | Info | Total |
|---|---|---|---|---|---|---|---|---|
| Unauthenticated | 10.2.0.42 | None | 0 | 0 | 0 | 1 | 21 | 22 |
| Authenticated (root) | linux-auth-unauth | Root creds | 2 | 2 | 5 | 1 | 61 | 71 |
| Authenticated (limited) | linux-auth-unauth | Non-root creds | 2 | 2 | 5 | 1 | 57 | 67 |

**Unauthenticated** — Only network-visible information: SSH banner, OS fingerprint, an ICMP timestamp disclosure, and (on this particular host) a SonicWall SonicOS identification. No package-level vulnerability data.

**Authenticated with root** — With valid, fully-privileged credentials, Tenable enumerated installed packages and cross-referenced them against Ubuntu Security Notices, surfacing real, actionable vulnerabilities:

- **Critical:** OpenSSL vulnerabilities (USN-8414-1); Vim vulnerabilities (USN-8451-1)
- **High:** SQLite vulnerabilities (USN-8480-1); NSS vulnerability (USN-8481-1)
- **Medium:** Vim (USN-8415-1), systemd (USN-8402-1), Inetutils (USN-8387-1), libxml2 (USN-8456-1), tar (USN-8477-1)

**Authenticated with limited privilege** — Same underlying vulnerabilities were still detected (package versions don't require elevated access to read), but Tenable logged an **"Insufficient Privilege"** credential status and a few info-level plugins that require command execution (e.g., enumerating running processes with elevation) came back empty. Net effect: 4 fewer Info-level findings than the root-credentialed scan, even though the vulnerability picture itself was unchanged.

**Takeaway:** unauthenticated scanning barely scratches the surface; authenticated scanning is where real vulnerability data comes from — but the *scope* of the service account still matters for full visibility into system state.

---

## Phase 3 — Windows Host Assessment: Unauthenticated vs. a Credential Failure

| Scan | Target | Credential status | Medium | Low | Info | Total |
|---|---|---|---|---|---|---|
| Unauthenticated | 10.2.0.33 | No credentials provided | 4 | 1 | 41 | 46 |
| "Authenticated" attempt | 10.2.0.33 | **Failure for provided credentials** | 4 | 1 | 41 | 46 |

This pairing is a useful real-world lesson rather than a clean before/after: the second scan *intended* to be authenticated, but Tenable logged a credential failure, and the results came back identical to the unauthenticated scan. In both cases, the findings were limited to network-visible issues: an untrusted/self-signed SSL certificate, and deprecated TLS 1.0/1.1 support.

**Takeaway:** a scan configured with credentials isn't the same as a scan that successfully authenticated. Tenable's **"Target Credential Status by Authentication Protocol"** plugin is worth checking on every credentialed scan — an analyst who doesn't verify it can walk away thinking a host was thoroughly assessed when it wasn't.

---

## Phase 4 — Windows 11 CIS/STIG Compliance Scan

A credentialed scan against a Windows 11 host (`vishal-stig-tes`) using a STIG-aligned template returned **5 High, 3 Medium, 2 Low, 135 Info** findings. Notably, even on a security-baseline-hardened image, the third-party application layer still had real gaps:

- Windows Defender antivirus signature definitions out of date
- Microsoft Teams for Desktop — remote code execution (pre-August 2025 build)
- Microsoft Edge (Chromium) — three separate multi-vulnerability advisories, plus one CVE-specific Medium finding
- Microsoft Teams — elevation of privilege (pre-July 2025 build)

**Takeaway:** OS-level hardening (STIG/CIS baselines) and third-party application patching are two different problems. A well-configured OS image can still ship high-severity risk through outdated bundled apps like Teams and Edge.

---

## Phase 5 — Full Remediation Lifecycle: Vulnerable Windows Server

This is the core of the project: one server, seven scans, tracking a real fix from discovery to verified closure — covering curriculum Remediations #1 through #6 in sequence.

### Timeline

| # | Scan | Remediation (per curriculum) | Timestamp (UTC) | Crit | High | Med | Low | Info | Total |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Initial baseline scan | — (pre-remediation) | Jul 5, 15:56 | 2 | 8 | 12 | 1 | 0 | 23 |
| 2 | CIS compliance audit — "OS Update" policy | **#1 — Windows OS Update** | Jul 6, 04:20 | 2 | 8 | 12 | 1 | 154 | 177 |
| 3 | CIS compliance audit — "Secure Config" policy | **#2 — Windows OS Secure Config** | Jul 6, 05:21 | 2 | 8 | 12 | 1 | 155 | 178 |
| 4 | After third-party software removal | **#3 — Third Party Software Removal** | Jul 6, 07:00 | **0** | **0** | 5 | 0 | 152 | 157 |
| 5 | After re-enabling SMB signing | **#4 — Re-enable SMB Signing** | Jul 6, 07:50 | 0 | 0 | **2** | 1 | 0 | 3 |
| 6 | Verification pass — RDP NLA | **#5 — Re-enable RDP NLA** | Jul 6, 19:19 | 0 | 0 | 3 | 1 | 151 | 155 |
| 7 | After restoring LAN Manager auth level | **#6 — Restore LAN Manager Auth** | Jul 6, 20:49 | 0 | 0 | **2** | 1 | 0 | 3 |

### Step-by-step narrative

**1. Baseline (Scan 1).** The server carried an unsupported, end-of-life copy of Wireshark 2.2.x — 2 Critical and 8 High findings, each a distinct multi-vulnerability advisory spanning versions 2.2.2 through 2.2.17. On top of that, three OS-level misconfigurations sat in the Medium tier: SMB signing not required, RDP not enforcing Network Level Authentication, and legacy LM/NTLMv1 authentication enabled — all classic lateral-movement and credential-relay risk factors.

**Remediations #1–#2 — Windows OS Update & Secure Config (Scans 2–3).** Two CIS benchmark audit runs — one against an "OS Update" policy, one against a "Secure Configuration" policy — verified these two remediations and, in the process, established a compliance baseline for the host: together they flagged roughly **72 failed hardening controls**, spanning:
- Password and account lockout policy (minimum length, age, history, lockout threshold)
- Audit policy (application group management, file share auditing, IPsec driver auditing)
- Privacy and lock-screen settings (camera/slideshow on lock screen, diagnostic data level)
- App Installer restrictions (malware scan override, certificate validation bypass)
- Event log configuration (max size, overwrite behavior)

Neither remediation targets third-party software or the SMB/RDP/LAN Manager settings, so the Critical/High Wireshark findings and the three Medium misconfigurations from baseline were still present in both scans, exactly as expected — those are addressed by Remediations #3–#6 below.

**Remediation #3 — Third Party Software Removal (Scan 4).** Removing Wireshark eliminated every Critical and High finding in one step — the fastest, highest-impact fix in the whole project. Five Medium findings remained: two related to the lab's self-signed SSL certificate (expected/accepted risk in this environment) and the three OS hardening gaps identified at baseline. A fuller credentialed scan also ran this time (152 Info items), giving much deeper visibility into installed software, patch level, and configuration.

**Remediation #4 — Re-enable SMB Signing (Scan 5).** After enforcing SMB signing, this targeted re-scan came back with just 3 findings total — SMB signing, RDP NLA, *and* LM/NTLMv1 authentication had all dropped off the list, leaving only the two SSL certificate findings and the ICMP timestamp disclosure.

**Remediation #5 — Re-enable RDP NLA (Scan 6).** A full re-scan roughly 11 hours later confirmed SMB signing and NLA were still holding — but the LM/NTLMv1 authentication finding had **reappeared**, a good reminder that hardening settings can drift or get reverted and need periodic re-verification, not a one-time check.

**Remediation #6 — Restore LAN Manager Auth (Scan 7).** Re-applying the LAN Manager authentication level restriction (disabling LM/NTLMv1) brought the server back to its cleanest state: **0 Critical, 0 High, 2 Medium** (both the accepted self-signed cert findings), **1 Low** (ICMP timestamp). This is the final, verified state of the server.

### Before / After (Scan 1 → Scan 7)

<p float="left">
  <img src="images/baseline-4.jpg" width="46%" alt="Baseline scan executive summary - 2 Critical, 8 High, 12 Medium" />
  <img src="images/final-remediated-4.jpg" width="46%" alt="Final remediated scan executive summary - 0 Critical, 0 High, 2 Medium" />
</p>

*Left: initial baseline scan (23 findings, 2 Critical / 8 High). Right: final verified state after full remediation (3 findings, 0 Critical / 0 High).*

---

## Phase 6 — Manual Vulnerability Remediation on Linux (SMB Signing)

A second, standalone remediation exercise — smaller in scope than Phase 5, but a cleaner "single finding, manually fixed, independently verified" case study, corresponding to the curriculum's "Manual Vulnerability Remediation on Linux" exercise (the target runs Samba and vsftpd, not native Windows). Target `10.2.0.41`, three scans across roughly 70 minutes on July 11, 2026. (Credentials failed to authenticate on all three scans — per Phase 3's lesson, that's worth flagging rather than glossing over — so everything here was found via unauthenticated, network-facing checks.)

| # | Scan | Timestamp (UTC) | Medium | Low | Info | Total |
|---|---|---|---|---|---|---|
| 1 | Baseline | Jul 11, 11:09 | 0 | 1 | 23 | 24 |
| 2 | Deeper scan — services discovered | Jul 11, 11:54 | 1 | 2 | 37 | 40 |
| 3 | After manual remediation | Jul 11, 12:19 | 0 | 1 | 23 | 24 |

**Scan 1 (baseline).** Only SSH and basic OS/protocol fingerprinting were visible — a narrow footprint, no FTP or SMB services detected, no Medium findings.

**Scan 2 (services discovered).** A follow-up scan picked up two additional listening services that Scan 1 hadn't surfaced: **vsftpd** (FTP) and **Samba** (SMB/CIFS file sharing). That visibility brought two real findings with it:
- **Medium — SMB Signing not required** (plugin 57608): the same finding class seen on the Phase 5 Windows server, this time on the SMB service exposed by Samba — without signing enforced, the service is exposed to man-in-the-middle relay attacks.
- **Low — FTP Supports Cleartext Authentication** (plugin 34324): vsftpd was accepting credentials in plaintext, meaning any FTP login on this network segment could be sniffed.

**Scan 3 (post-remediation verification).** Every FTP- and SMB-related plugin that fired in Scan 2 — including the base service-detection plugins (FTP Server Detection, Samba Server Detection, vsftpd Detection, SMB shares/version/dialect enumeration) — is entirely absent, and the results match Scan 1 almost line for line. That's a meaningfully different fix than Phase 5's: rather than reconfiguring SMB signing while leaving the service running, **the FTP and SMB services themselves were taken offline** (stopped or firewalled). No listening service, no exposure, no finding — attack-surface reduction rather than in-place hardening.

<p float="left">
  <img src="images/smb-finding-4.jpg" width="46%" alt="Scan 2 - SMB Signing not required and FTP cleartext auth findings" />
  <img src="images/smb-remediated-4.jpg" width="46%" alt="Scan 3 - clean state after FTP and SMB services taken offline" />
</p>

*Left: Scan 2, the moment FTP and SMB exposure was discovered. Right: Scan 3, verified clean after remediation — both services no longer detectable.*

---

## Remediation Effort Summary

**Vulnerable Windows server (Phase 5):** total findings dropped from 23 to 3 — an **87% reduction**. Critical and High findings were both eliminated entirely at Remediation #3 (**100% reduction each**, Third-Party Software Removal). Medium findings fell from 12 to 2 (**83% reduction**) — the two that remain are the lab's self-signed SSL certificate, an accepted risk rather than an open gap.

| Severity | Before (Scan 1) | After (Scan 7) | Reduction |
|---|---|---|---|
| Critical | 2 | 0 | 100% |
| High | 8 | 0 | 100% |
| Medium | 12 | 2 | 83% |
| Low | 1 | 1 | 0% (accepted risk) |
| **Total** | **23** | **3** | **87%** |

**Linux SMB/FTP host (Phase 6):** from the point of discovery (Scan 2, 40 findings) to verified remediation (Scan 3, 24 findings), both the Medium (SMB signing) and the newly-introduced Low (FTP cleartext auth) findings were fully resolved — **100% reduction** on each — by taking the exposed services offline.

---

## Aggregate Findings Across the Engagement

| Asset / Scan Group | Critical | High | Medium | Low |
|---|---|---|---|---|
| Subnet discovery (4 hosts) | 0 | 0 | 0 | 0 |
| Linux host (root creds) | 2 | 2 | 5 | 1 |
| Windows host 10.2.0.33 | 0 | 0 | 4 | 1 |
| Windows 11 STIG scan | 0 | 5 | 3 | 2 |
| Vulnerable server — baseline | 2 | 8 | 12 | 1 |
| Vulnerable server — final | **0** | **0** | 2 | 1 |
| 10.2.0.41 — services exposed (Scan 2) | 0 | 0 | 1 | 2 |
| 10.2.0.41 — final (Scan 3) | 0 | 0 | **0** | 1 |

---

## Lessons Learned

- **Credentialed scanning is non-negotiable for real vulnerability data.** The unauthenticated Linux scan returned zero actionable vulnerabilities; the same host with root credentials returned 9 Critical/High findings tied to real USN advisories.
- **Verify credential status, don't assume it.** The Windows 10.2.0.33 "authenticated" scan silently failed to authenticate and returned unauthenticated-level results — a trap that's easy to miss without checking the credential status plugin.
- **Fix, then verify — don't just fix.** Re-scanning after each remediation step (rather than trusting the change was applied correctly) caught the LM/NTLMv1 setting reverting between Scan 5 and Scan 6, which would otherwise have gone unnoticed.
- **Compliance and vulnerability findings are complementary, not redundant.** The CIS audit surfaced ~72 configuration gaps that a standard vulnerability scan wouldn't have flagged (password policy, audit policy, privacy settings) — neither view alone gives the full risk picture.
- **The biggest win came from the simplest fix.** Removing one end-of-life third-party application (Wireshark) eliminated 100% of Critical and High findings on the server in a single step, before any OS-hardening work even began.
- **The same misconfiguration showed up twice, on two different hosts.** "SMB Signing not required" appeared on both the Phase 5 Windows server and the Phase 6 Samba host — a reminder to check for a finding pattern across the environment, not just fix it in the one place it was first spotted.
- **Removing exposure beats reconfiguring it when the service isn't needed.** Phase 5 fixed SMB signing in place (service stays up, setting corrected); Phase 6 took the exposed FTP/SMB services offline entirely. Both are valid, verified fixes — but attack-surface reduction is the stronger option whenever the service doesn't need to be there at all.

## Skills Demonstrated

Tenable Vulnerability Management (scan configuration, credentialed vs. uncredentialed scanning, CIS/STIG compliance auditing) • vulnerability triage and prioritization by severity • Windows hardening (SMB signing, RDP NLA, LAN Manager authentication levels) • Linux service hardening and attack-surface reduction (Samba/FTP) • third-party software risk management • verification-based remediation workflow • Linux and Windows patch/USN advisory analysis

---

## Conclusion

Across six phases, ten lab hosts, and 17 individual scans, this project walked through the full vulnerability management lifecycle — from network discovery to credentialed deep-dive scanning to compliance auditing to two independently verified remediation cycles. The centerpiece result speaks for itself: a server that started at 2 Critical / 8 High findings was brought to a clean, verified 0 Critical / 0 High state through targeted, re-scan-confirmed remediation. The second case — an exposed SMB/FTP service manually taken offline and confirmed clean — shows the same discipline applied at smaller scale: find it, fix it, prove it, on re-scan rather than on faith. That verification loop is the core of the process a vulnerability management analyst runs in a production environment.

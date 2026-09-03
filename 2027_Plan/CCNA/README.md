# CCNA — Prep Material for v2.0

Cisco announced **CCNA 200-301 v2.0** on **May 20, 2026** — the biggest blueprint change to CCNA
in over 6 years. It goes live **February 3, 2027**; the current v1.1 exam retires the day before
(**February 2, 2027**). Since this plan's target year is 2027, **study for v2.0 from the start**
rather than the outgoing version — no point learning content that's about to be retired.

**Caveat worth keeping in mind**: this blueprint was only published a few months ago as of writing
(August 2026), official Cisco Press cert guides for it aren't out yet, and third-party sources
don't 100% agree on the exact domain percentages below — treat the numbers as "close, community-
derived" rather than gospel, and cross-check
[Cisco's own official exam topics page](https://www.cisco.com/site/us/en/learn/training-certifications/exams/ccna.html)
closer to when real study time starts.

---

## Exam Fundamentals (unchanged from v1.1)

| | |
|---|---|
| **Exam code** | 200-301 (same code — the certification you earn doesn't change) |
| **Duration** | 120 minutes |
| **Questions** | ~100 |
| **Passing score** | 825 / 1000 |
| **Domains** | **5** (down from 6 in v1.1) |

---

## What Actually Changed vs. v1.1

The exam code and duration are the same. What Cisco asks you to *do* with the knowledge is not.

### 1. Troubleshooting is back — the headline change
v1.1 had **zero** exam topics using "troubleshoot" or "diagnose" as the action verb. v2.0 has
roughly **28–30% of topics** at that level. Many topics that used to top out at "describe" or
"explain" now go all the way to "configure → verify → diagnose → troubleshoot."

**Practical consequence**: passive reading/video-watching alone won't cut it this time — lab
work has to happen *while* studying, not after. The recommended approach from multiple sources:
deliberately break a config, then read the output and find your own mistake, rather than only
ever building configs that work first try.

### 2. Domains consolidated 6 → 5, weight shifted toward switching/infrastructure
"Automation and Programmability" disappears as its own domain — folded into a new
**AI and Network Operations** domain. IP Services also gets folded into a broader domain rather
than standing alone. Switching + Network Infrastructure together now make up roughly **50%** of
the exam (up from ~40% in v1.1).

### 3. AI is now explicitly on the blueprint
New topics include: distinguishing **generative AI** (produces text/configs) from **agentic AI**
(actually takes actions on the network), constructing effective AI prompts (data, format,
persona, instructions), and evaluating AI-generated recommendations for correctness rather than
trusting them blindly.

### 4. Notable removals
- REST, CRUD, JSON, and Terraform fundamentals
- WLC (Wireless LAN Controller) GUI configuration
- Wireless architecture theory (as a standalone topic)
- Python is no longer heavily tested

(Note: removed as a *standalone exam topic* doesn't always mean gone completely — some of this
remains as background/prerequisite knowledge for other topics that are still tested.)

---

## Domain Breakdown (community-sourced — verify against Cisco's official PDF before finalizing a study plan)

| Domain | Weight | Notes |
|---|---|---|
| **Network Infrastructure & Connectivity** (Network Fundamentals) | ~25% | Core networking fundamentals — OSI/TCP-IP models, addressing, Ethernet, cabling |
| **Switching and Network Access** | ~25% | Switch-to-switch/router connectivity, L2/L3 interfaces, 802.1Q trunks, EtherChannel, SVIs, VLANs, PoE, wireless APs, VoIP phones, CDP/LLDP, Rapid PVST+, troubleshooting via `show` commands |
| **IP Routing** | ~20% | Routing fundamentals and protocols |
| **Network Services and Security** (folds in what was IP Services in v1.1) | ~20% | DHCP, NAT, NTP, DNS diagnostics (all 6 primary record types), Layer 2 security protections |
| **AI and Network Operations** (replaces Automation and Programmability) | ~10% | Agentic AI in network ops, AI prompt construction, SNMP, Ansible-based config management, syslog interpretation, network management approaches (device/cloud/controller-based) |

---

## Study Plan

- [ ] Download and read **both** the v1.1 and v2.0 official exam-topics PDFs from the Cisco
      Learning Network side by side — see exactly what moved, what's new, what's gone, straight
      from the source rather than a third-party summary (including this one).
- [ ] Foundations first (weeks 1–4): binary/subnetting, OSI/TCP-IP models, Ethernet
      fundamentals — this hasn't changed and everything else builds on it.
- [ ] Core configuration (weeks 5–8): VLANs, STP/Rapid PVST+, routing basics, DHCP, NAT, NTP,
      DNS — build it, then deliberately break it and diagnose the break, not just build-and-move-on.
- [ ] Security fundamentals + Layer 2 protections (weeks 9–10) — given weight moved here.
- [ ] AI and Network Operations (weeks 11–12) — genuinely new territory: generative vs. agentic
      AI, prompt construction, Ansible, syslog. Lightest-weighted domain (10%) but entirely new,
      so budget real time rather than treating it as an afterthought.
- [ ] Full timed practice exams once all domains are covered — given the troubleshooting shift,
      prioritize labs/practice exams that actually simulate broken configs, not just recall-style
      question banks.
- [ ] Book the exam for on/after **February 3, 2027**.

---

## Resources

- **Cisco's official exam topics page** (primary source, check for updates as go-live nears):
  https://www.cisco.com/site/us/en/learn/training-certifications/exams/ccna.html
- **Wendell Odom / certskills.com** — detailed verb-by-verb blueprint diff (v1.1 → v2.0) and
  plans for the next official Cisco Press cert guide:
  - https://www.certskills.com/ccna26-01/
  - https://www.certskills.com/ccna26-02/
  - https://www.certskills.com/ccna26-03/
- **Anthony Sequeira / ajsnetworking.com** — domain-by-domain walkthrough of what's new:
  https://www.ajsnetworking.com/new-ccna-v2-2027/
- **certempire.com** — concise summary of exam fundamentals + added/removed topics:
  https://certempire.com/ccna-200-301-v2-0-update/
- **ccnatraining.com** — practical "test now or wait" guidance (this plan already answers that:
  wait, study v2.0 from the start):
  https://ccnatraining.com/ccna-v2-0-is-coming-in-2027-should-you-test-now-or-wait/

---

## Notes

Update this file as Cisco's own materials firm up — the exact domain percentages here are
third-party interpretations of a blueprint published only a few months ago (May 2026), and
official Cisco Press cert guides for v2.0 aren't published yet. Re-check the official exam
topics PDF before locking in a final study schedule, and log real progress/practice-exam scores
here once study actually starts.

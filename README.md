# Operating OpenShift in an Air-Gapped Banking Environment: What's Different

Most Kubernetes content assumes you have internet access — pull images from Docker Hub, `apt update` when you need a package, and Google your way through an unfamiliar error. None of that was true in the environment I worked in. This was **OpenShift**, running inside an **RBI-regulated banking infrastructure**, with Production and QA fully air-gapped from the outside world.

This article covers what actually changes operationally when you take Kubernetes-style platform engineering and put it behind an air gap — not the theory, but the day-to-day friction.

---

## Why Air-Gapped Environments Exist in Banking

In regulated banking infrastructure, air-gapping isn't a design preference, it's a compliance requirement. Production and QA had no direct route to the public internet. Every image, every package, every dependency had to be deliberately brought in through a controlled path. The tradeoff is explicit: you give up convenience for a dramatically reduced attack surface, which regulators and security teams require in this kind of environment.

---

## Environment Layout: DC, MZ, DMZ, and DR-DMZ

Our OpenShift footprint spanned four zones — **DC, MZ, DMZ, and DR-DMZ** — each isolated from the others with *no direct connection between them*. Practically, this meant logging into each environment independently and repeating tasks manually, zone by zone. There was no single control plane view or tooling that let you push a change once and have it propagate. If a fix was needed in all four zones, you did it four times, individually, with individual verification each time.

This is one of the most understated costs of this kind of setup, not the complexity of any single task, but the multiplication of that task across isolated zones with zero shortcuts.

---

## Getting Images In: Mirror Registry + Manual Pipeline Handoff

Since QA and Production had no internet access, we used an **internal mirror registry** as the image source of truth. The actual flow: images were built through a GitLab pipeline in the open dev environment, and once validated, the image sync into the air-gapped mirror registry was **manually triggered**, not automatic. That manual trigger was a deliberate control point, not an oversight — a human decision point before anything crossed into the regulated zone.

This meant our CI/CD wasn't a fully automated pipeline in the cloud-native sense. It was open-environment automation up to a boundary, then a manual gate, then air-gapped deployment.

---

## Troubleshooting Without Google

When something broke and there was no Stack Overflow to lean on, the process was: **Red Hat's official documentation first**, since RHEL/OpenShift was the only supported OS/platform on our specific IBM hardware. Beyond that, we relied on an **internal NAS**, literal folders of `.txt` and `.docx` files with screenshots, documenting every environment-specific activity we'd ever performed, no matter how small.

That NAS became our institutional memory. Every task, even minor ones, got documented because the next person troubleshooting the same issue wouldn't have Google to fall back on either. This is a very different documentation discipline than most cloud-native teams practice, where a quick search often substitutes for internal docs.

---

## The Cost of Change: Port and URL Whitelisting

Any new integration requiring a port to be opened or a URL/IP whitelisted went through a **security-first approval process**: we had to provide concrete evidence and complete testing proving no attack vector existed before that port or URL was approved. On average, this took **3 working days** per request.

In a typical cloud environment, opening a port or whitelisting an endpoint might take minutes. Here, it required a documented security justification, review, and sign-off, a direct tradeoff between agility and the security posture the environment demanded.

---

## Patching and CVE Remediation Under CAB Approval

Software updates and CVE remediation didn't happen unilaterally. Every patch required **approval from a Senior Manager, followed by a General Manager**, before it could be applied. This is standard change-management discipline in regulated banking, but it fundamentally changes your operational cadence — patch cycles are measured against an approval chain, not just testing readiness.

---

## What This Teaches You That Cloud-Native Doesn't

Working in this environment forces a different kind of engineering discipline. You can't lean on convenience — no quick pulls from public registries, no instant Google answers, no one-click propagation across environments. Every change becomes deliberate, documented, and justified. It also sharpens your understanding of *why* certain security controls exist, rather than treating them as friction to route around.

Engineers who've only worked in fully cloud-native, internet-connected environments often underestimate how much operational overhead exists in regulated, air-gapped infrastructure, and conversely, engineers from this background bring a security-first instinct that's valuable anywhere.

---

## Conclusion

Air-gapped OpenShift in a banking environment isn't just "Kubernetes but slower." It's a fundamentally different operating model: mirror registries instead of public pulls, manual gates instead of full automation, Red Hat docs and internal NAS instead of Stack Overflow, and multi-day approval chains instead of self-service changes. None of it is accidental complexity, it's complexity by design, in service of a security posture that regulated industries require. Understanding that tradeoff is the real lesson.

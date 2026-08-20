# Migrating Datadog Across Tenants: A Zero-Downtime, Cutover-Based Approach to Org Consolidation

As part of a broader infrastructure centralization effort, our organization made the decision to consolidate monitoring under a single Datadog tenant. What used to be spread across multiple tenants for different departments was being brought together into one, and I was responsible for executing the migration for our environment, solo, end to end.

This article walks through how I approached it, what broke, and what I'd tell another engineer facing the same task.

---

## Why the Migration Was Needed

The trigger wasn't technical debt or a broken setup, it was organizational. The company was centralizing infrastructure monitoring that had historically been fragmented across several tenants tied to different departments. Beyond that, I wasn't involved in the higher-level rationale, which is fairly common when you're the engineer executing a migration decided above your team.

What mattered for my scope was straightforward: move our environment's Datadog footprint — cleanly, without duplicate billing, and without gaps in monitoring coverage — from **Tenant 1** to **Tenant 2**.

---

## Scope and Environment

Our environment ran on **AWS** with **EKS** as the core compute layer, and **MSK** for Kafka-based streaming workloads. The Datadog components in scope for migration were:

- **Agent configuration** across all hosts
- **Infrastructure monitoring** for EC2, EKS, and MSK
- **Dashboards** — we used Datadog's standard, out-of-the-box dashboards rather than heavily customized ones, which simplified this part significantly
- **Monitors** — alerting rules covering infrastructure and application health
- **Integrations** — AWS, EKS, and related service integrations
- **Tags** — our existing tagging conventions, which needed to carry over unchanged

One constraint shaped everything else: *dual shipping to both tenants simultaneously wasn't approved*, due to cost. That single decision defined the entire migration strategy.

---

## Migration Approach

With dual shipping off the table, the only viable strategy was a **cutover-based migration** rather than a gradual parallel-run. That meant no long soak period where both tenants received live data — Tenant 1 needed to be actively transitioned to Tenant 2 in a controlled window, not run in parallel indefinitely.

For monitors specifically, I used **Datadog CLI** commands to migrate them programmatically rather than recreating each one manually. Given the number of monitors involved, manual recreation would have been slow and error-prone — CLI-based migration let me move them in bulk while preserving their logic.

Integrations were handled through **Terraform**, which was already how we managed infrastructure-as-code elsewhere in our stack, though I'll be transparent that the exact implementation details there are fuzzier in my memory than the monitor migration itself.

Dashboards were the easiest part of the scope, precisely because we relied on Datadog's standard dashboards rather than heavily customized ones. That meant less to migrate and less that could break.

---

## Technical Implementation

The most operationally sensitive part of the migration was **API key and App key rotation**. Rather than manually touching individual hosts, our EKS workloads pulled Datadog credentials from **AWS Secrets Manager**. That meant the actual "cutover" for agent connectivity came down to updating the API/App key values in Secrets Manager — EKS picked up the new tenant's credentials through the existing secrets injection pipeline, without needing per-host manual intervention.

This was a deliberate design advantage from how the environment was already architected — centralizing credential management in Secrets Manager meant a tenant switch didn't require touching every node individually.

Tags were preserved *as-is* during migration — no restructuring, no renaming conventions. This mattered because tags are what keep dashboards, monitors, and filtering logic meaningful; changing tag structure mid-migration would have added an unnecessary second variable to troubleshoot if something broke.

---

## Challenges and Troubleshooting

The main issue I hit during the migration was **alert duplication**. During the transition window, both Tenant 1 and Tenant 2 monitors were technically capable of firing on the same underlying conditions — which meant teams risked getting paged twice for the same incident, or getting conflicting signals about which tenant was "live."

The fix was operationally simple but required discipline: I **muted Tenant 1 monitors** and kept them muted until the migration was verified 100% complete on Tenant 2. This avoided duplicate alerting while still keeping Tenant 1's monitor definitions intact as a fallback reference until I was confident Tenant 2 was fully operational.

In hindsight, this is the kind of issue that's obvious once you hit it, but easy to underestimate beforehand if you're focused primarily on data migration mechanics rather than alerting behavior during the transition window.

---

## Validation and Cutover

Since this was a solo migration without a dedicated QA phase for the migration itself, validation came down to **manual spot-checks and checklist-based verification** across the scope — agent connectivity, monitor status, dashboard rendering, and integration health in Tenant 2 — before I made the call to decommission Tenant 1.

There was no automated validation suite here. It was methodical, manual verification against a checklist covering each component in scope, repeated until I was confident nothing was silently broken.

---

## Key Lessons

The biggest structural lesson from this migration is that **cutover-based migrations demand more upfront discipline than parallel-run migrations**, because you don't get the safety net of comparing both tenants side-by-side over time. Every validation step has to be deliberate, not opportunistic.

The second lesson is around **alert hygiene during transitions**. It's easy to plan for data migration and forget that alerting logic on both sides can independently fire during the switch. Muting the old tenant's monitors proactively, rather than reactively after getting duplicate pages, should be a standard step in any similar migration, not an afterthought.

---

## Recommendations

If you're planning a similar Datadog tenant migration:

- **Decide dual-shipping vs. cutover early** — it fundamentally shapes your entire migration plan, not just the mechanics
- **Use Datadog CLI/API for monitor migration** rather than manual recreation if you're dealing with more than a handful of monitors
- **Centralize credential management** (like Secrets Manager) ahead of time if possible — it turns tenant cutover into a config change rather than a fleet-wide operation
- **Proactively mute old-tenant monitors** before cutover completion; don't wait for duplicate alerts to force the decision
- **Build a validation checklist before you start**, not during — solo migrations benefit enormously from having a clear "done" definition upfront

---

## Conclusion

Migrating Datadog across tenants isn't just a data-copy exercise, it's an exercise in managing risk under a constraint — in this case, no dual shipping. The technical mechanics (CLI-based monitor migration, Terraform-managed integrations, Secrets Manager-driven credential rotation) mattered, but the real engineering judgment was in sequencing the cutover safely and catching the alerting overlap before it became a production noise problem.

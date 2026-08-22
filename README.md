# Two Tools, Two Worlds: Lessons from Operating Prometheus and Datadog in Production

I have operated Prometheus with Grafana in two distinct production environments — a cloud-native AWS shop and a regulated, air-gapped bank — and I have run Datadog at scale during a legacy server migration for a telecom SaaS platform. These were not bake-offs or proof-of-concepts. They were multi-month operational commitments where the wrong choice meant either a compliance violation, a surprise invoice, or a 3 a.m. page that should never have fired.
This article is what I wish I had read before committing to either stack.

# Why the Migration Was Needed

The need was never about Prometheus being "bad" or Datadog being "better." It was about constraints.
At a fintech services firm, we ran on AWS with EKS, EC2, and S3. The team was small, the workloads were container-native, and we needed observability without a recurring SaaS bill. Prometheus and Grafana were the natural choice.
Later, at a central bank, the constraint was regulatory. The RBI mandates air-gapped infrastructure. No outbound internet from production. That forced us onto Heal APM, an enterprise-approved Prometheus-and-Grafana bundle. The tool was selected by compliance, not by engineering preference.
Most recently, at a telecom client, the constraint was time and consolidation. The organization already had Datadog, but it was tied to a legacy server instance. We needed to migrate to a new Datadog tenant, move 80+ hosts and 9 production monitors, and stop paying for two environments simultaneously. There was no appetite to rebuild dashboards in Prometheus. The business had already bought the car; my job was to swap the engine without stalling it.

# Scope and Environment

## Environment 1: Cloud-Native AWS (Prometheus/Grafana)
AWS EKS, EC2, S3, VPC
~15% reduction in unplanned downtime after implementing Prometheus scraping and Grafana alerting
Jenkins for CI/CD
Full control over scrape configs, recording rules, and Alertmanager routing

## Environment 2: Regulated On-Premises Bank (Prometheus/Grafana via Heal APM)
800+ KVMs, VMs, LPARs, and SANs across DC, MZ, DMZ, and DR-DMZ zones
OpenShift 4.14 clusters across four environments
Air-gapped. No external endpoints. All container images and RPMs imported manually
Heal APM provided the Prometheus backend and Grafana frontend as an RBI-approved package

## Environment 3: Enterprise Telecom SaaS (Datadog)
AWS EKS, EC2, EFS, MSK (Kafka), VPC
80+ hosts: 50+ production, 30+ QA
9 production monitors migrated in under two weeks
Terraform and GitLab CI/CD for infrastructure-as-code
Legacy-to-new Datadog server migration to eliminate dual billing


# Migration Approach

The only true "migration" I led was the Datadog tenant cutover. The approach was simple in concept: build parity, automate what moves, then switch.
Phase 1: Inventory. We audited every monitor, dashboard, and integration on the legacy server. We found nine production monitors that mattered for paging, plus a long tail of dashboards used by application teams.
Phase 2: Automation. I wrote scripts to export monitor definitions and recreate them on the new Datadog server. Host-level migration was agent-based: update the Datadog Agent configuration to point to the new endpoint, roll through environments.
Phase 3: Parallel Run. We ran both tenants briefly to confirm metric continuity. This was the riskiest part because it meant dual billing. We kept the overlap as short as possible — a matter of days, not weeks.
Phase 4: Cutover. We executed a switchover rather than a slow drain. One change to the agent endpoint, a validation cycle, and then we decommissioned the legacy hosts from the old tenant. The cost saving was immediate.

# Technical Implementation

Prometheus/Grafana: The Build Path
In both Prometheus environments, the pattern was similar. Deploy Prometheus server or use a bundled distribution. Configure scrape_configs for Kubernetes API servers, kubelet, cAdvisor, and application pods via ServiceMonitor CRDs. Grafana connected to Prometheus as a data source. Alerts went through Alertmanager to email or Slack.
In the AWS environment, we ran Prometheus inside the EKS cluster, scraped node-exporter and kube-state-metrics, and built dashboards for pod resource utilization. We used HPA based on custom metrics, which contributed to a 15% infrastructure cost reduction and 20% better resource utilization.
In the bank, the implementation was heavier. Because we were air-gapped, every exporter image had to be scanned, imported, and signed. We could not pull from Docker Hub. We ran Prometheus on OpenShift with persistent volumes backed by enterprise SAN storage. Grafana LDAP integration tied into the bank's identity stack. The setup worked, but every upgrade required a change request and a maintenance window.

# Datadog: The Buy Path

Datadog's implementation was agent-first. We deployed the Datadog Agent as a DaemonSet on EKS and as a standard package on EC2. For MSK (Kafka), we used JMX-based integration metrics. Terraform defined the monitors and alert policies, versioned in GitLab. This was a significant advantage: monitor-as-code was easier in Datadog with Terraform than it was in my Prometheus environments, where recording rules were YAML in Git and Grafana dashboards were often click-ops.
Tag strategy mattered. We enforced env, service, team, and cluster tags early. Without that discipline, Datadog becomes expensive noise. With it, we could slice dashboards by team and route alerts accurately.

# Challenges and Troubleshooting

## Alert Noise in Datadog
Out of the box, Datadog is eager to alert. After migration, we had pages firing on CPU spikes that lasted 90 seconds. I spent the first week tuning thresholds, adding evaluation windows, and suppressing non-actionable monitors. The learning: Datadog's default monitors are built for visibility, not for sleep. You need to treat alert tuning as a dedicated work stream, not an afterthought.
## Air-Gapped Prometheus Limitations
At the bank, a scrape target started failing intermittently. Root cause: the SAN storage backing Prometheus was experiencing latency spikes, and the local TSDB was hitting write timeouts. Diagnosis required correlating Prometheus target_scrape_pool_sync_total with storage team metrics — a multi-team effort. Fix: moved Prometheus WAL to faster local SSD and increased scrape timeouts for slow targets. Lesson: in air-gapped environments, you own the entire stack down to the spindle. There is no vendor support ticket to absorb the blame.
## EKS Control-Plane Visibility
With Prometheus on EKS, you do not get managed control plane metrics out of the box. API server latency or etcd size requires CloudWatch or a proxy setup. Datadog's EKS integration, by contrast, ingests control plane metrics through the AWS integration. This is a genuine gap in self-hosted Prometheus on managed Kubernetes. We worked around it with CloudWatch exporter, but it was additional toil.
## Cost Surprises
Datadog's per-host billing is predictable until it is not. Log ingestion and custom metrics are where bills inflate. During the parallel run, we watched the invoice closely. The moment we confirmed metric parity, we cut over. Delaying would have burned budget with zero operational benefit.

# Validation and Cutover

For the Datadog migration, validation had four gates:
1)Telemetry Parity. We compared key metrics (CPU, memory, disk, network) between old and new tenants for the same host over a 48-hour window. Discrepancies above 2% were investigated.
2)Dashboard Accuracy. Application teams verified their dashboards loaded correctly and time series were continuous post-cutover.
3)Monitor Integrity. We triggered synthetic failures in QA to confirm monitors still paged through the correct PagerDuty integrations.
4)Billing Verification. We confirmed the legacy tenant host count dropped to zero and the new tenant reflected the full fleet.
The cutover itself was scripted. Update the Datadog Agent site parameter, restart, validate in the new tenant, and deregister from the old. We did QA first, then production in batches by cluster. Zero downtime.

# Key Lessons

## Cost Ownership Is Part of the SRE Role
With Prometheus, cost is infrastructure: EC2, EBS, and your time. With Datadog, cost is a line item that scales with your fleet. I learned to treat Datadog pricing as a first-class constraint. The dual-billing period during migration was a necessary risk, but we measured it in days and dollars, not assumptions.
## Alert Hygiene Is Non-Negotiable
Both tools will lie to you if you let them. Prometheus Alertmanager can spam just as easily as Datadog if your routing tree is flat. The difference is that Datadog's ease of creating monitors makes it easier to create noise. I now require a runbook link and an explicit "who pages" field before any monitor is marked as critical.
## Compliance Is a Forcing Function
The bank did not choose Heal APM because it was technically superior. It chose it because RBI compliance required an approved, air-gapped solution. If you work in regulated industries, your observability choice may be made by audit, not by benchmark. Plan for that.
## The "Build vs Buy" Reality
Prometheus gives you control. You own the scrape logic, the retention, the HA setup. That control costs engineering hours. Datadog gives you speed. You get APM, RUM, and cloud integrations in hours, not weeks. That speed costs recurring license fees. Neither is free. The right choice depends on which budget you have — engineering time or operational budget.
## Terraform for Monitors Is a Game Changer
Defining Datadog monitors in Terraform forced rigor. Every threshold change was a merge request. In my Prometheus environments, Grafana dashboards were too often edited in the UI and lost during migrations. I now apply Terraform-style discipline to both tools.

# Recommendations

Choose Prometheus/Grafana when:
You are in an air-gapped or regulated environment.
Your team has the expertise to manage TSDB retention, HA Pair setup, and Alertmanager routing.
You need deep customization of scrape behavior or metric relabeling.
Your primary cost constraint is SaaS subscription, not engineering headcount.
Choose Datadog when:
You need to observe managed services (EKS control plane, MSK, RDS) without building exporters.
You want monitor-as-code with Terraform and cloud-native integrations out of the box.
Your organization values speed of implementation over per-host licensing cost.
You need APM and distributed tracing without running a separate Jaeger stack.
Avoid Hybrid Confusion
Running both to compare them long-term is expensive and operationally confusing. If you must overlap, define a hard cutoff date. We kept our Datadog parallel run to a few days. Anything longer becomes technical debt and budget leakage.

# Conclusion

I did not migrate from Prometheus to Datadog. I operated both because different employers had different constraints. Prometheus taught me to own the stack. Datadog taught me to manage vendor relationships, billing, and alert discipline. The best SREs I know are not religious about their tools. They are religious about understanding the trade-offs.
Pick the tool that fits your compliance, your budget, and your team's capacity to maintain it. Then commit. Half-measures in observability cost more than either stack.

## 5 Key Takeaways
1)Match the tool to your constraint — compliance, budget, or team size — not to a features checklist.

2)Datadog's speed comes with a billing model that rewards discipline; monitor your monitors before they monitor you.

3)Prometheus gives control but demands ownership of storage, HA, and scraping; it is not zero-cost just because it is open source.

4)Alert noise is a people problem, not a tool problem — enforce runbooks and ownership fields before any critical page.

5)Treat observability migration as a product delivery — inventory, automation, validation gates, and a hard cutover date.


#DevOps #SiteReliabilityEngineering #Observability #CloudInfrastructure #PrometheusvsDatadog


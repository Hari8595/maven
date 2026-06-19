# Cloud Infrastructure — Approval Request
*Employment 360 & Recruitment 360*

| Field | Detail | Field | Detail |
| --- | --- | --- | --- |
| To | Team Leader, Support Services | Date | ____ June 2026 |
| For review by | CEO and Finance Team | Status | Awaiting Approval |
| Prepared by | Hari Vishnu, Junior DevOps Engineer | Region | AWS Asia Pacific (Mumbai) |

Respected Sir,

I hope this note finds you well. I would be most grateful for a few minutes of your time to review the hosting plan for our two applications, Employment 360 and Recruitment 360, on Amazon Web Services (AWS). I have set out clearly what has already been put in place, what remains to be done, and the costs involved, so that the decisions resting with you are easy to make. Throughout this work I have done my best to follow the recognised security standards — the OWASP guidelines for application security and the CIS Benchmarks for cloud and server hardening — so that the platform is built safely from the very beginning.

I have written this in plain language so that it is easy to follow for everyone reviewing it. The first part explains what is being built and the secure foundation already in place; the middle sets out the few decisions on which your guidance would help, each with an honest recommendation offered purely for your consideration; and the latter part gives a complete and transparent cost summary at today's exchange rate, so that the finance team has full visibility. I am, of course, very happy to explain any part of this in person.

---

## 1. What We Are Building

Both applications will run on a single, well-secured AWS server using Docker containers. The applications keep their existing technology — Next.js for the frontends and Node.js (version 22) for the backends — so no change is required to the applications themselves. The backends use the Sequelize toolkit to talk to MySQL. There are two databases (one for each application), and both backends are able to reach both, since one application promotes records into the other and our HR colleagues work across the two. Redis runs in a container to support the message queue used for WhatsApp (through Twilio) and email (through SendGrid). Each application keeps its own settings file, so the two remain cleanly separated.

### 1A. Progress So Far — A Working Setup

I am glad to report that the containerised setup is now running successfully on the server. Both applications — each with its frontend and backend — start together with the database and the Redis message queue, all within Docker. The services restart on their own if anything stops, and they come back automatically after a reboot, which gives us a genuinely round-the-clock setup. I have also confirmed that both backends connect to both databases and that the message queue is working.

Along the way I worked through the usual practical matters — making the storage large enough for the container images, tightening the database access so only the application can reach it, and arranging the settings so that no password is ever placed inside a container image. The applications load their settings only when they run, which keeps the secrets safely out of the stored images. I mention this only to show that the groundwork has been tested in practice, not merely planned on paper. A few finishing items remain, which I have listed honestly in section 5.

- One AWS server running both applications, with automatic restart and recovery.
- A shared MySQL database and a Redis cache and message queue.
- Source code held in private GitHub repositories with automated security scanning on every change.
- A review-and-approval step before any change is released, and a staging check before going live.
- A daily database backup kept securely, with logs and snapshots retained for a defined period.
- Secure access with no open remote-login ports, and all passwords held in a protected vault.
- Monitoring with an automatic email alert if the server becomes unhealthy.

---

## 2. Secure Foundation Already in Place

So that you have confidence the platform is being built carefully, the following items have already been set up and tested. Security has been the first consideration at every step, and I have noted the purpose of each item simply.

### 2.1 Protecting the Source Code (GitHub)

- Both applications are kept in private repositories, so only authorised team members can see or change the code.
- Two-factor authentication is now compulsory for every member of our code organisation, so a stolen password alone is not enough to gain access.
- Ordinary members are read-only by default and cannot delete, rename, transfer or expose a repository; these sensitive actions are reserved for the administrator.
- Access is organised through teams (Developers, Senior Developer and DevOps) rather than person by person, which keeps it neat and easy to adjust when people join or leave.
- Every time code is changed, an automatic security check runs to look for weaknesses, accidentally exposed passwords, and unsafe third-party libraries.

### 2.2 Securing the AWS Account

- Multi-factor authentication has been enabled on the main (root) account, which is now kept aside and not used for daily work, as recommended.
- A separate administrator login has been created that issues secure, temporary credentials instead of permanent keys, which greatly lowers the risk of a key being leaked.
- A full audit trail (AWS CloudTrail) records every action in the account, with the records encrypted and protected from tampering.
- A billing alert has been set so that any unusual spending is noticed quickly.

### 2.3 A Private, Secure Network

- A private network has been created in the Mumbai region, separated from other systems.
- It is divided into a public area and a private area. Our server sits in the private area and has no direct doorway to the internet, which is an important protection.
- Necessary updates reach the server through a single controlled exit, and access to AWS services stays within our private network wherever possible.

### 2.4 The Application Server

- The server has no public address and cannot be reached directly from the internet.
- Its main disk is encrypted, and the database is kept on a second, separately encrypted disk, so the data is well protected and can be preserved on its own.
- I connect to the server through a secure, fully recorded AWS service that needs no open remote-login ports and no shared keys, which is safer than the traditional approach.
- The container software was installed from the official source and tightened according to the CIS hardening guidance.

### 2.5 Handling Passwords and Keys

- All passwords and secret keys are stored in a dedicated, encrypted AWS service and are never placed inside the application code. The server fetches them securely only at the moment they are needed.

---

## 3. AWS Services Used, and Their Purpose

| Service | Purpose |
| --- | --- |
| EC2 | The server that runs both applications. |
| VPC | A private network on AWS, isolated from other accounts. |
| Elastic IP | A fixed outbound address (useful if a vendor such as Twilio requires whitelisting). |
| NAT Gateway | Lets the server reach the internet for updates while remaining private. |
| VPC Endpoints | Allow secure access to AWS services without using the public internet. |
| EBS | The fast, encrypted disk on which the live database runs. |
| S3 | Secure storage for database backups, logs and uploaded files. |
| CloudWatch | Monitoring and logs; raises an alert if the server becomes unhealthy. |
| AWS Backup | Takes scheduled backup images of the server and data disk for recovery. |
| Secrets Manager | Stores database and Twilio passwords securely, out of the code. |
| Session Manager (SSM) | Secure, recorded access to the server without opening any login ports. |
| IAM / Identity Center | Controls access so each person and service receives only what they need. |

> **Clarification on the database:** the live MySQL data runs on the fast attached disk (EBS), not on S3, because a database needs rapid continuous access that S3 is not built for. S3 is used for the backups and copies, which is the correct and safe use of that service.

---

## 4. Security Tools We Use (All Genuine and Open-Source)

I have chosen well-established, industry-recognised tools to keep our applications secure. These are all genuine, widely-used and openly available, selected on their merit rather than only because they carry no licence fee — which they also do not. They follow the security guidance of OWASP (a globally respected web-security body) and the CIS Benchmarks. Importantly, the scanning runs automatically and keeps our source code within our own environment.

| Tool | Purpose | Nature |
| --- | --- | --- |
| Docker | Packages and runs the applications consistently | Open-source, industry standard |
| GitHub Actions | Builds, scans and deploys our code automatically | Built into GitHub |
| Semgrep | Scans our code for security weaknesses | Open-source, widely adopted |
| njsscan | Scans Node.js code for security issues | Open-source (OWASP) |
| Trivy | Scans application images for vulnerabilities | Open-source (Aqua Security) |
| Gitleaks | Detects passwords accidentally left in code | Open-source |
| npm audit & Retire.js | Check for vulnerable software libraries | Open-source |
| Dependabot | Alerts us to outdated or risky dependencies | Built into GitHub, free |
| OWASP ZAP | Tests the running application for weaknesses | Open-source (OWASP) |

---

## 5. The Stages That Remain

The stages below are still to be completed. They follow the same careful, security-first approach. I have highlighted the few places where your direction would be helpful.

### 5.1 Releasing the Applications (Staging First, Then Live)

As you advised, each update will first run in a staging area for checking, and only then be released to the live (production) environment. I have built in a safety net: if a live update does not pass its health check, the system automatically returns to the previous working version, so a faulty update does not cause downtime.

### 5.2 Monitoring and Logs

Server logs, application logs and security-related events such as login attempts will be gathered centrally and kept encrypted. Email alerts will be sent automatically if the server becomes unhealthy, runs short of resources, or shows unusual activity. This gives us early warning and a clear record.

### 5.3 Keeping Logs Affordably (Retention and Archiving)

To balance good record-keeping with cost, logs will follow a tiered approach: recent logs are kept ready to search, then moved automatically to cheaper storage after a set period (for example, thirty days), and finally placed in very low-cost archive storage before being removed at the end of the retention period.

**A small point for your guidance:** as our work relates to healthcare revenue cycle management, there may be a rule about how long certain logs must be retained. If the required period can kindly be confirmed, I will set the lifecycle correctly from the start.

### 5.4 Backups and Recovery

Daily database backups will be taken and kept in encrypted, private storage for a defined period, and regular snapshots of the data disk will be taken so the system can be restored if anything fails. I will also carry out a test restore to confirm the backups truly work when needed.

### 5.5 Automating the Release Process with Approvals

The release process will be automated so that every change is security-scanned before it can move forward. In keeping with your wishes, a change is reviewed before reaching staging, and a further approval is needed before it goes live. No permanent access keys are stored anywhere in this process; only secure, temporary credentials are used.

### 5.6 Final Security Review

As the last step, the account will be checked against the CIS security benchmark using AWS's own compliance tools, network activity logging will be switched on, and a final review will confirm that no gaps remain before the applications hold any live data.

---

## 6. Decisions Where Your Guidance Would Help

There are three choices on which I would respectfully request your direction. In each case I have offered an honest recommendation purely for your consideration; the final decision rests entirely with you.

### 6.1 Decision 1 — AWS Region (Mumbai vs Cheapest Region)

AWS does not have a region in Chennai; Chennai users are served from the Mumbai region with very low delay. The cheapest AWS regions are in the United States. The comparison below is for our server (t3.large, 2 vCPU, 8 GB), verified at AWS pricing in June 2026.

| Factor | Mumbai (ap-south-1) | Cheapest US Region (e.g. us-east-1) |
| --- | --- | --- |
| Server cost (On-Demand) | ~$66 / month (~₹6,270) | ~$61 / month (~₹5,800) |
| Difference | about ₹500–850 / month more | baseline (cheapest) |
| Delay to Indian users | 5–20 ms (excellent) | 180–220 ms (poor) |
| Data location | India | United States |
| Suitability | Recommended for staff-facing apps | Better only for non-user-facing work |

**With respect, my suggestion would be:** Mumbai. The cost difference is small, but Mumbai gives our Indian staff a much faster experience and keeps employee data within India. The US region saves a little but would make the applications noticeably slower for every user and place the data abroad.

**Decision requested:** Mumbai (recommended) ☐ / Cheapest US region ☐

### 6.2 Decision 2 — Public Entry Point (Cloudflare vs AWS Load Balancer / WAF)

Because the server is kept private for safety, a protected public entry point is genuinely needed so that staff can reach the applications securely over the internet — the server cannot simply be exposed by itself. The options below all direct visitors to the correct application and handle HTTPS; the choice affects cost and where the protection sits. I have shown the AWS Load Balancer (ALB) and the AWS Web Firewall (WAF) as separate items, because they are billed separately and WAF is optional — so that you can choose the level of protection you prefer.

| Factor | Cloudflare (Free plan) | AWS Load Balancer (ALB) | ALB + Web Firewall (WAF) |
| --- | --- | --- | --- |
| Approx. monthly cost | Free tier (₹0) | ~$28 / ~₹2,660 | ~$43 / ~₹4,085 |
| HTTPS / SSL | Included | Included | Included |
| Basic DDoS protection | Strong, free | Basic (AWS Shield Standard) | Basic (AWS Shield Standard) |
| Managed firewall rules (OWASP) | Basic rules included | Not included | Included (managed rule sets) |
| Fully within AWS | Partly | Yes | Yes |

**With respect, my suggestion would be:** begin with the Cloudflare free plan, which is sufficient for our current needs and saves the monthly cost, since it already provides HTTPS and strong DDoS and basic firewall protection at no charge. If a fully in-AWS setup is preferred, the AWS Load Balancer can be used, with the Web Firewall (WAF) added on top for OWASP-managed protection. All three are secure; this is a cost and preference choice, and the decision is entirely yours.

**Decision requested:** Cloudflare Free (recommended) ☐ / AWS Load Balancer only ☐ / AWS Load Balancer + WAF ☐

### 6.3 Decision 3 — Server Payment (Pay-As-You-Go vs Reserved)

The server can be paid for monthly (Pay-As-You-Go) or committed for one year in advance (Reserved), which is cheaper but requires a commitment.

| Factor | Pay-As-You-Go (On-Demand) | Reserved (1 year) |
| --- | --- | --- |
| Server cost (Mumbai t3.large) | ~$66 / month (~₹6,270) | ~$41 / month (~₹3,900) |
| Commitment | None — stop anytime | One year |
| Saving | — | ~₹2,400 / month once committed |

**With respect, my suggestion would be:** begin with Pay-As-You-Go for the first month or two. This lets us confirm the server size is correct without any commitment. Once we are confident, we can switch to Reserved and save about ₹2,400 per month.

**Decision requested:** Start Pay-As-You-Go, review for Reserved after 2 months ☐ / Reserve now ☐

---

## 7. Cost Summary (at Today's Exchange Rate)

For full transparency, the figures below use today's exchange rate of approximately ₹94.4 to 1 US Dollar (verified on 18 June 2026), and are based on the current AWS Mumbai pricing confirmed for June 2026. The rupee has held close to ₹94–95 through this month, so the totals are essentially unchanged from my earlier estimate. I have separated what is already running from what is planned, and I have been careful to include every item — including a few that were added as the work progressed — so that the picture is complete and there are no surprises later. The rupee value will move slightly with the exchange rate, as AWS bills in US Dollars.

### 7.1 Already Running

| Item | USD / month | Approx. ₹ / month |
| --- | --- | --- |
| Application server (EC2 t3.large, Pay-As-You-Go) | ~$66 | ~₹6,270 |
| Server disk (30 GB, encrypted) | ~$2.75 | ~₹260 |
| Database disk (30 GB, encrypted) | ~$2.75 | ~₹260 |
| Internet exit gateway (NAT) | ~$33 | ~₹3,135 |
| Public IP address (Elastic IP) | ~$3.60 | ~₹342 |
| **Private secure-access endpoints (3)** | **~$22** | **~₹2,090** |
| Audit trail and encryption keys | ~$2 | ~₹190 |
| Password / secret storage | ~$2 | ~₹190 |
| **Subtotal — running now** | **~$134** | **~₹12,740** |

**A candid note on one item:** the three private secure-access endpoints (about ₹2,090 per month) were added while solving a connection issue, to let the server reach AWS services privately without using the public internet. They improve security, but they do carry this cost. If preferred, they can be removed to save this amount, with the server then using the internet-exit gateway instead — a reasonable trade-off between cost and the added privacy. I wanted to be open about this rather than leave it unexplained.

### 7.2 Planned (Monitoring and Backups)

| Item | USD / month | Approx. ₹ / month |
| --- | --- | --- |
| Central logs (collection and storage) | ~$5 | ~₹475 |
| Backup and log-archive storage (S3) | ~$2 | ~₹190 |
| Data-disk snapshots (AWS Backup) | ~$3 | ~₹285 |
| **Subtotal — planned** | **~$10** | **~₹950** |

### 7.3 Usage-Based Costs (Grow With Use)

These items are charged by how much the applications are actually used, so they are small at first and rise gently as more staff use the system. I have shown the rate for each and a careful estimate at our expected early usage, so that nothing is hidden and the figures can be understood as activity grows.

| Item | Rate | Estimate (early usage) |
| --- | --- | --- |
| Internet-exit (NAT) data processing | ~$0.045 / GB | ~$0.50 (~₹48) |
| Data transfer out to the internet | ~$0.09 / GB | ~$1 (~₹95) |
| Log ingestion (CloudWatch) | ~$0.50 / GB | ~$1–2 (~₹95–190) |
| **Estimated subtotal (early usage)** | **—** | **~$3 (~₹285)** |

**Why these matter:** these usage-based charges are the part of an AWS bill that most often rises unexpectedly, simply because more people are using the applications. At our current scale they are very small, but I have set them out plainly so that, as adoption grows, any increase is understood rather than a surprise. The budget alerts described later will keep these visible month by month.

### 7.4 Optional — Public Entry Point

| Choice | USD / month | Approx. ₹ / month |
| --- | --- | --- |
| Cloudflare (free tier) — my suggestion | $0 | ₹0 |
| AWS Load Balancer (ALB) only | ~$28 | ~₹2,660 |
| Web Firewall (WAF) add-on — optional | ~$15 | ~₹1,425 |
| **ALB + WAF together** | **~$43** | **~₹4,085** |

### 7.5 Overall Picture

| Scenario | Approx. ₹ / month |
| --- | --- |
| **Everything running now (with Cloudflare Free)** | **~₹12,740** |
| With monitoring, backups and early usage added | ~₹13,950 |
| With AWS Load Balancer (instead of Cloudflare) | ~₹16,600 |
| **With AWS Load Balancer + Web Firewall (WAF)** | **~₹18,000** |
| If the server is later reserved for 1 year (a saving) | about ₹2,370 less |

I should mention honestly that this is a little higher than the very first rough estimate I shared earlier. The increase is because the setup became more secure as the work progressed — the private secure-access endpoints, the separate encrypted database disk, the audit trail, and the planned monitoring and backups all add small, genuine costs. I felt it was far better to show the true figure now than to present a lower number that the actual bill would later exceed. If a one-year reservation of the server is acceptable, a meaningful part of this can be reduced.

### 7.6 The Three Packages, Side by Side

To make the choice as easy as possible, the table below brings the realistic options together as complete packages, each including the monitoring and backups. This way the full monthly figure for each path can be seen at a glance, rather than added up from separate parts.

| | Package A — Lean (recommended) | Package B — In-AWS Edge | Package C — Maximum Protection |
| --- | --- | --- | --- |
| Public entry point | Cloudflare (free) | AWS Load Balancer | AWS Load Balancer + WAF |
| Server payment | Pay-As-You-Go | Pay-As-You-Go | Pay-As-You-Go |
| Monitoring & backups | Included | Included | Included |
| **Total per month (₹)** | **~₹13,950** | **~₹16,600** | **~₹18,000** |
| **Total per month ($)** | **~$147** | **~$175** | **~$189** |
| If server reserved (1 yr) | ~₹11,580 | ~₹14,230 | ~₹15,630 |

All three packages are secure and follow the OWASP and CIS standards; the difference is only in how much protection sits at the public entry point, and the cost that comes with it. Package A relies on Cloudflare's free tier, which already provides HTTPS, strong DDoS protection and basic firewall rules at no charge, and is, in my honest view, sufficient for our current needs.

### 7.7 Final Recommended Configuration and Cost

Bringing the recommendations together, the configuration I would respectfully propose — entirely for your consideration — is as follows. It keeps cost sensible while maintaining a genuinely secure setup.

| Item | Recommended choice |
| --- | --- |
| AWS Region | Mumbai (low delay, data within India) |
| Public entry point | Cloudflare Free (HTTPS + DDoS + basic firewall, no cost) |
| Server payment | Pay-As-You-Go for the first 1–2 months, then review for Reserved |
| Monitoring, logs and backups | Included from the start |
| **Recommended monthly cost (now)** | **approx. ₹13,950 (~$147) per month** |
| **Expected cost after reserving the server** | **approx. ₹11,580 (~$122) per month** |

**In short:** the recommended monthly figure to plan for is approximately ₹13,950 (around $147), reducing to roughly ₹11,580 once we are confident enough to reserve the server for a year. Should you prefer the additional in-AWS protection of the Load Balancer or the Web Firewall, the figure would rise to about ₹16,600 or ₹18,000 respectively, as shown above. Whichever you choose, I will set up budget alerts so the spend stays visible and within the limit you approve.

### 7.8 What Could Increase the Bill, and How We Control It

In the interest of full honesty, the items below most commonly cause an unexpected rise in an AWS bill, together with how I will keep them under control:

- Data processing and transfer (the internet-exit gateway and data-out) grow as more staff use the applications — this is normal and expected as adoption increases.
- Application logs can grow over time and add to the monitoring cost; I will set sensible retention so they do not accumulate indefinitely.
- Forgotten resources (such as an unused address or an old backup) can quietly add cost; I will review these regularly.
- The exchange rate affects the rupee figure, as AWS bills in US Dollars.

**To make sure none of these is a surprise,** I will set up AWS Budget alerts (email warnings at 50%, 80% and 100% of an agreed monthly limit) and automatic anomaly detection, and I can report the actual spend to you each month. This way, any increase is seen early and explained, rather than appearing unexpectedly at month end. The figures in this section have been checked against current AWS pricing and the exchange rate on the date of writing, so they are as accurate as I can make them today.

---

## 8. Summary of Decisions Requested

| No. | Decision | Options |
| --- | --- | --- |
| 1 | AWS Region | Mumbai (recommended) / Cheapest US region |
| 2 | Public Entry Point | Cloudflare Free (recommended) / AWS Load Balancer only / ALB + WAF |
| 3 | Server Payment | Start Pay-As-You-Go (recommended) / Reserve now |
| 4 | Log retention period | To be confirmed per healthcare requirement |
| 5 | Stronger branch-protection enforcement | Current plan works; full enforcement needs GitHub Team (~₹1,900/month) |
| 6 | Budget approval | Approve recommended ~₹13,950 / month (Package A) / Discuss |
| 7 | Approval to proceed with remaining stages | Approve / Discuss |

I would also kindly request the domain names for the two applications, the developer names and email addresses, and an email address to receive alerts, so that I may proceed smoothly once approved.

---

## 9. A Note on Reliability

In fairness, I should mention that the present design uses a single server. It is set up to recover automatically from common failures: a stopped application restarts within seconds, a server reboot restores all services within a few minutes, and a faulty update reverts to the working version on its own. A daily backup is retained, and an alert is sent the moment the server becomes unhealthy. I would note honestly that a single server cannot guarantee it will never go down; it recovers quickly and the data stays safe. Should fully uninterrupted availability be required in future, this would involve adding a second server, which I have noted as a possible later step to keep costs reasonable for now.

---

## 10. Approval and Sign-off

| Prepared by | Reviewed by | Recommended by | Approved by |
| --- | --- | --- | --- |
| Hari Vishnu (DevOps) | Team Leader | Finance Team | CEO |
| Signature / Date | Signature / Date | Signature / Date | Signature / Date |

Thank you very much for taking the time to read this, and for your guidance. I am committed to completing the remaining stages carefully and securely, and I will keep you updated at every step.

With sincere regards,

**Hari Vishnu**
*Junior DevOps Engineer, Support Services*

---
*Confidential — For Review and Approval · June 2026*

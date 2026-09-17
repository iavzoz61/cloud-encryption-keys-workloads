# cloud encryption: How It Actually Works, Who Holds Your Keys, and Where to Run Encrypted Workloads

Search for "cloud encryption" and you'll get two very different kinds of answers. Half the internet explains it as a basic concept — your files get scrambled before they hit someone else's server. The other half is vendor material that assumes you already know what AES-256 is and just need to buy something.

This guide covers both ends: what cloud encryption actually does, the decisions that matter (client-side vs. server-side, provider keys vs. your keys), what compliance frameworks expect from you, and — the part most guides skip — where you host encrypted workloads in the first place. Because encryption strategy and hosting strategy are not separate conversations. If you're running your own VMs or bare-metal servers somewhere like Sharktech, you control the whole encryption stack yourself, and that changes your options considerably.

## What Cloud Encryption Actually Means

Cloud encryption is the process of transforming data from its original readable format (plaintext) into an unreadable format (ciphertext) before it's transferred to and stored in the cloud. Without the encryption keys, the data is effectively useless — even if it's lost, stolen, or accidentally shared with the wrong party.

That's the whole pitch, and it's a good one. Encryption is considered one of the most effective components of a security strategy because it protects the data itself rather than just the perimeter around it. Beyond confidentiality, it also covers three practical concerns:

- **Compliance** — regulations like HIPAA and standards like FIPS require encrypting sensitive data
- **Tenant isolation** — protection against unauthorized access from other customers sharing the same public cloud
- **Breach exposure** — in select cases, encrypted data may exempt an organization from mandatory breach disclosure

One warning before we go further: don't confuse "my cloud provider encrypts things" with "my data is protected." Every reputable provider offers basic encryption, but security in the cloud follows a shared responsibility model. The provider secures the underlying infrastructure. You are responsible for the data and assets you store in that environment. Plenty of organizations learn this distinction the hard way.

## The Three States of Data, and How Each Gets Encrypted

Encryption isn't one switch you flip. It protects data in different states, and each state has its own standard toolkit.

### Data in transit

This is data moving between your device, your applications, and the cloud. A large share of it is encrypted automatically via HTTPS, which adds a security layer (SSL/TLS) on top of the standard IP protocol. If someone intercepts the session, they see noise. Google Cloud's documentation frames transit encryption around protocols like TLS and VPNs — if you're administering servers over SSH or syncing backups across regions, this is the layer doing the work.

### Data at rest

This is data sitting on storage — disks, volumes, object storage buckets. The standard technique here is symmetric encryption, typically AES-256, often combined with full-disk or volume-level encryption. Even if a drive is physically stolen or a snapshot leaks, the contents are unreadable without the key.

### Data in use

The hardest state. Data being processed by an application is generally decrypted in memory, which is why layered controls — access management, monitoring, microsegmentation — matter alongside encryption itself.

### The two algorithm families

- **Symmetric encryption**: the same key encrypts and decrypts. Faster, simpler, and the standard choice for bulk data. The tradeoff: anyone holding the key can read the data, so key handling becomes your single point of failure.
- **Asymmetric encryption**: two linked keys, one public and one private. Slower, but it solves the key-distribution problem and underpins most of the transit protection you use daily without thinking about it.

Most real-world cloud encryption stacks use both: asymmetric crypto to establish sessions and exchange keys, symmetric crypto for the actual payload.

## Client-Side vs. Server-Side Encryption: The Decision That Matters Most

This is where the marketing gloss usually stops and the actual risk assessment begins. The question is simple: who encrypts the data, and who holds the keys?

**Server-side encryption (SSE).** Data is encrypted by the provider after it reaches their server. Convenient — often a checkbox. But the provider manages the keys and performs the encryption. As AWS's own documentation puts it, with server-side encryption your data is protected by policies; with client-side encryption, you manage the key.

**Client-side encryption (CSE).** Data is encrypted on your device before it ever touches the network. This is what gives you end-to-end protection, in transit and at rest, from source to storage. It's also the model behind the "zero-knowledge" storage services — Sync.com, pCloud, Proton Drive, Internxt, Tresorit — that dominate the encrypted-cloud-storage comparison lists. The provider literally cannot read your files.

The practical difference shows up in two scenarios:

1. **The provider gets compromised.** With SSE, an attacker who reaches the provider's key infrastructure may reach your data. With CSE, they get ciphertext.
2. **You lose the key.** With CSE, encrypted data without the key is gone. Not "contact support" gone — mathematically gone. Key management stops being a nice-to-have and becomes the whole job.

If you're storing family photos, SSE plus a strong account is fine. If you're storing client records, health data, or anything covered by a regulation, the calculus shifts.

## Key Management: Provider-Managed, BYOK, and HYOK

Once you've picked an encryption model, the next question is where the keys live. There are three broad arrangements:

- **Provider-managed keys** — the default. The provider generates, stores, and rotates keys on your behalf. Lowest operational burden, lowest control.
- **BYOK (Bring Your Own Key)** — you generate keys and upload them to the provider's key management service. Better control, but note the distinction security vendors draw: with BYOK, keys are stored in the provider's KMS, where the provider retains technical access. You've brought the key, but you've also handed over its custody.
- **HYOK (Hold Your Own Key)** — keys never leave your own key management system, entirely outside the provider's boundary. The provider only ever sees handles or wrapped keys. Maximum control, maximum operational burden: key rotation, availability, and recovery are all your problem now, and if your KMS goes down, your data access goes with it.

The Cloud Security Alliance's guidance on key responsibility models frames this as a genuine tradeoff rather than a hierarchy — HYOK's security comes at a real operational cost, and it may only make sense for your most sensitive subset of data, not everything.

There's a middle path that most of this site's likely readers should seriously consider: **run the encrypted workload on infrastructure where you have root access in the first place.** On a self-managed VPS, a cloud instance, or a bare-metal server, you choose the OS, the encryption tooling, and the key storage. LUKS full-disk encryption on a Linux VM you administer yourself, with keys that never leave your control, is functionally a HYOK arrangement — without paying enterprise KMS prices for it.

## What Compliance Actually Asks of You

If you handle regulated data, encryption moves from "best practice" to "requirement." The specifics differ:

- **HIPAA** — for health data, the commonly cited minimum is AES-128, with AES-256 the most common implementation; PHI must be encrypted both at rest and in transit, and cloud service providers handling PHI need to sign a Business Associate Agreement.
- **GDPR** — doesn't mandate specific algorithms, but encryption is explicitly named as an appropriate technical measure, and encrypted personal data can affect breach-notification obligations.
- **FIPS** — U.S. federal contexts require FIPS-validated cryptographic modules.

The CrowdStrike overview also notes a detail worth remembering: encryption can, in select cases, absolve an organization of the need to disclose a breach. That's not a loophole to design around, but it explains why encryption shows up in so many compliance checklists — it changes the legal consequences of failure, not just the technical ones.

## A Working Checklist for Cloud Encryption

Pulling together what the vendor documentation, cloud provider guides, and security organizations consistently recommend:

1. **Classify your data first.** Decide what actually requires encryption — by sensitivity or by compliance obligation — before choosing tooling.
2. **Encrypt in transit and at rest.** TLS for movement, AES-256 for storage. One without the other is half a fence.
3. **Decide your key model explicitly.** Provider-managed, BYOK, or hold-your-own — make it a documented decision, not a default.
4. **Never lose the keys.** Back them up securely, because lost keys with client-side encryption mean unrecoverable data.
5. **Layer, don't rely.** Multi-factor authentication, access controls, monitoring, and network segmentation all back up encryption — a strong cipher behind a weak password is still a weak system.
6. **Know your backup story.** Encrypted primary storage with unencrypted backups is a common and painful oversight.

## Where You Host Encrypted Workloads Changes Your Options

Here's the bridge from theory to infrastructure. When you run on a hyperscaler, most of your encryption posture is negotiated with the provider — their KMS, their key custody rules, their egress pricing if you ever want to leave.

When you run on self-managed infrastructure, you own the stack. And that's the specific niche a provider like **Sharktech** occupies. Sharktech is a 20-year-old hosting provider (AS46844, its own ISP with public peering) offering OpenStack-based cloud, Proxmox-based Smart VPS, bare-metal servers, object storage, and encrypted backup, across five locations: Los Angeles, Las Vegas, Denver, Chicago, and Amsterdam. A few characteristics matter specifically for encryption-focused deployments:

- **Root/OS-level control** — Smart VPS and cloud instances are self-managed: you pick the OS (Linux images or your own ISOs), install LUKS/VeraCrypt/whatever you want, and store keys wherever you decide. Nobody at the provider can decrypt your volumes because nobody at the provider administers your OS.
- **Private networking and firewall security groups** — OpenStack features that isolate backend traffic between VMs from public exposure, so key distribution and replication traffic stays off the public internet.
- **No vendor lock-in** — you can download your server disk images at any time, via portal or API, for offsite backup or migration. Your data is portable, which matters when your whole security model is "we can leave whenever we want."
- **DDoS protection included** — all services include Sharktech's proprietary DDoS filtering (the Smart VPS product spec lists 60Gbps protection), which is an availability control rather than a confidentiality one, but encrypted services that are offline are still useless.
- **Encrypted backup as a service** — Acronis Cyber Protect hosting, with encryption, deduplication, and ransomware protection built in.

An independent HostAdvice review of the Smart VPS measured 6,000+ random IOPS and sub-millisecond network latency, calling it "one of the most technically impressive VPS offerings" their reviewer had tested — useful context if you're weighing whether budget self-managed infrastructure can actually carry a serious workload. On the community side, a LowEndTalk user documented a year of Sharktech DDoS protection absorbing repeated attacks on game servers ("3Gbit to 8Gbit... never skip a beat").

## Sharktech Plans and Pricing (Full Comparison)

The table below covers every service category Sharktech currently lists in its store, with pricing verified from the official product pages at the time of writing. All services include DDoS protection, the management panel, and 24/7 support.

| Service | Core Configuration | Price (USD) | Billing | Purchase |
| --- | --- | --- | --- | --- |
| **Cloud Applications Platform** | Pay-per-use: cloudlets $0.0035/hr, storage $0.00011/hr, network $0.0035/GB, IPv6 $0.001/hr | From $5.00/mo | Monthly, usage-based | [ View plan details](https://bit.ly/SharKTech) |
| **Acronis Cloud Backup** | 200GB encrypted cloud backup + sync & share; +$0.02/GB beyond; Windows/Linux/macOS | $4.00/mo | Monthly | [ Get encrypted backup](https://bit.ly/SharKTech) |
| **Object Storage (S3)** | From 1TB storage, 1TB–1PB bandwidth, 5 locations | From $6.00/mo | Monthly | [ Check S3 storage pricing](https://bit.ly/SharKTech) |
| **Smart VPS** | 2–128 vCPU (Xeon Gold), 4–256GB DDR4, 40GB–2TB NVMe, 4–304TB transfer, unlimited VMs from your resource pool | From $7.95/mo (up to 50% off long-term billing, ~$3.98/mo effective annually) | Monthly / Quarterly / Semi-Annual / Annual | [ Deploy a Smart VPS](https://bit.ly/SharKTech) |
| **Public Cloud — Small** | 4–16 vCPU, 8–32GB RAM, 300–2400GB SSD (+HDD/NVMe options), 20TB+ bandwidth, resource cap protects against overage | From $39.00/mo | Monthly, hourly overage | [ Compare cloud tiers](https://bit.ly/SharKTech) |
| **Public Cloud — Medium** | 8–32 vCPU, 16–64GB RAM, 800–6400GB SSD, 20TB+ bandwidth | From $79.00/mo | Monthly, hourly overage | [ Compare cloud tiers](https://bit.ly/SharKTech) |
| **Public Cloud — Large** | 32–128 vCPU, 64–256GB RAM, 1500–12000GB SSD, 20TB+ bandwidth | From $249.00/mo | Monthly, hourly overage | [ Compare cloud tiers](https://bit.ly/SharKTech) |
| **Public Cloud — Enterprise** | 64+ vCPU, 128GB+ RAM, 5000GB+ SSD, uncapped resources | From $499.00/mo | Monthly | [ Compare cloud tiers](https://bit.ly/SharKTech) |
| **Dedicated Cloud** | 8–512 vCPU, 16–1024GB RAM, SSD/HDD/NVMe tiers, 5–300TB transfer; fixed resources, prepaid | From $86.23/mo | Monthly | [ See Dedicated Cloud](https://bit.ly/SharKTech) |
| **Bare-Metal Servers** | Fully customizable CPU/RAM/GPU/storage, 1–40Gbps ports, hardware-level access; availability varies with stock | Custom quote | Monthly | [ Request a bare-metal config](https://bit.ly/SharKTech) |

A few notes that affect real bills:

- **Public Cloud overage rates**: CPU $0.0025/hr per core, RAM $0.0035/hr per GB, NVMe $0.00009/hr per GB, SSD $0.00006/hr, HDD $0.00002/hr. Extra IPv4 addresses are $1.50/mo each (the first one is free). Ingress is unlimited; egress is $0.002/GB beyond the included 5,000GB outgoing.
- **Public vs. Dedicated Cloud** is purely a billing model difference on the same OpenStack infrastructure: Public Cloud lets you burst past your plan and pays hourly overage; Dedicated Cloud gives you exactly what you ordered at a fixed monthly price.
- **Smart VPS billing discounts**: 25% off quarterly, 35% off semi-annually, 50% off annually. The marketing page shows the annual effective rate at $3.98/mo against the $7.95 monthly price.
- **No lock-in**: on any cloud service you can export your disk images and leave — which, again, is exactly the property you want when your encryption model depends on key portability.

If you're not sure whether a VPS is even the right size for an encryption-heavy workload, the company's own advice is reasonable: start with the smallest Smart VPS tier and upgrade through the portal as needed, since subscriptions can be adjusted without redeploying VMs. 👉 [See current Smart VPS pricing and billing options](https://bit.ly/SharKTech)

## Mapping Encryption Workloads to the Right Service

A quick editorial read on which tier fits which encryption scenario, based on the verified specs above:

**Personal encrypted storage / small projects.** A Smart VPS at $7.95/mo running Nextcloud with client-side encryption, or an S3 bucket at $6/mo for encrypted archives. This is the classic "replace Dropbox, own the keys" setup, and it's genuinely cheap.

**Team infrastructure with compliance needs.** Public Cloud Small ($39/mo) gives you 4–16 vCPU and up to 32GB RAM to split across as many VMs as the resource pool allows — a database VM on a private network, an app VM, a key-management VM that only the other two can reach. Security groups do the segmentation.

**Fixed, predictable production.** Dedicated Cloud from $86.23/mo — you get exactly the resources you pay for, no hourly overage surprises, which makes budgeting for a compliance-audited environment simpler.

**Maximum control.** Bare-metal. Full hardware access, your OS, your encryption, no hypervisor layer at all. Pricing is quote-based because configurations are fully customizable and stock varies.

**Backup layer for everything.** The $4/mo Acronis tier is the cheapest line item in the whole table and covers the item from the checklist above that people forget: encrypted primary storage with equally protected backups.

## Frequently Asked Questions

**Is cloud encryption the same as a VPN?**
No. A VPN encrypts data in transit across a network. Cloud encryption covers data both in transit and at rest in the cloud, and typically involves key management that persists beyond any single session. They complement each other — Sharktech, for instance, includes native VPN support in its cloud platform for bridging to on-premises infrastructure.

**Does my cloud provider's encryption protect me from the provider itself?**
Only if it's client-side with keys you hold. Server-side encryption protects against external theft and other tenants, but the provider's systems perform the encryption and typically hold the keys. That's precisely the distinction BYOK and HYOK models exist to address.

**What happens if I lose my encryption key?**
Encrypted data without the key is unrecoverable — this is by design. Treat key backup with the same seriousness as the encryption itself.

**Is AES-256 necessary, or is AES-128 enough?**
For HIPAA contexts, AES-128 is cited as the minimum standard and AES-256 is the most common choice. There's no meaningful performance penalty at cloud scale for using AES-256, so it's the sensible default.

**Do I need to encrypt backups too?**
Yes. An unencrypted backup of encrypted production data undoes the work. Either encrypt at the source (so backups inherit it) or use a backup service with encryption built in — which is what the Acronis offering provides.

## The Short Version

Cloud encryption works, it's mandated by most regulations that matter, and the math behind AES isn't the weak point. The decisions that actually determine your risk are organizational: who encrypts the data, who holds the keys, and whether your hosting arrangement gives you control over both. Provider-managed encryption is convenient; client-side encryption with self-managed keys is the strongest posture; and infrastructure with root access — whether that's a $7.95 VPS or a bare-metal server — is what makes the strong posture practical to run.

👉 [Compare Sharktech's full range of cloud, VPS, and bare-metal plans](https://bit.ly/SharKTech)

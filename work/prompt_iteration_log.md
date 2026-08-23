# AI Prompt Engineering Iteration Log & Cross-Model Audit

**Target Task from FL-01 Audit:** Automated Generation of Structured Cyber Threat Intelligence (CTI) Summaries and Indicator of Compromise (IOC) Extractions.  
**Business Goal:** Convert unstructured threat reports into structured JSON containing exact threat actors, CVEs, and verifiable IOCs (IPs, hashes, domains).

---

## Section 1: Naive Baseline (Version 0)

### Prompt
> *"Summarize this threat intelligence report and extract IOCs."*

### Raw Output Excerpt
> *"This report discusses a malware campaign targeting financial institutions using phishing emails. The attackers used malicious PDF attachments containing custom macros. **Key IOCs:** Some malicious IP addresses were used to communicate with C2 servers, along with MD5 file hashes. Make sure your firewall blocks suspicious traffic and update your antivirus software regularly."*

---

## Section 2: Five-Step Iterative Refinement Log

### Version 1: Added Technique — Role Assignment (Persona)

#### Prompt
> *"**You are a Principal Cyber Threat Intelligence (CTI) Analyst specializing in malware reverse engineering and SOC incident response.** Summarize this threat intelligence report and extract IOCs."*

#### Raw Output Excerpt
> *"**Executive Summary:** Threat actors are leveraging spear-phishing campaigns distributing malicious PDF droppers to execute remote code. **Extracted Indicators:**\n- Hash: `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` (SHA-256)\n- Network Indicator: Suspicious outbound HTTPS requests observed targeting external C2 nodes."*

#### Iteration Notes
* **Technique Applied:** Assigned an expert persona (*Principal CTI Analyst*).
* **Observed Output Difference:** Adopted a formal SOC incident response tone and completely stopped offering generic consumer advice (*"update your antivirus"*).
* **Failure Point Remaining:** Missing structured network indicators (IP addresses, specific CVEs) and lacking standard machine-readable formatting.

---

### Version 2: Added Technique — Context & Operational Motivation

#### Prompt
> *"You are a Principal Cyber Threat Intelligence (CTI) Analyst. Summarize this threat intelligence report and extract IOCs. **Context & Motivation: This output will be directly ingested by SOC Tier-1 analysts during live incident triage and used to configure automated SIEM/firewall blocklists. Accuracy is critical; missed IOCs or false positives disrupt enterprise operations.**"*

#### Raw Output Excerpt
> *"**Threat Triage Summary:** High-severity campaign targeting web servers. **Identified IOCs for Immediate Ingestion:**\n1. `192.0.2.45` (C2 IP Address)\n2. `malicious-domain.com` (Phishing Landing Page)\n3. `CVE-2023-38606` (Privilege Escalation Vulnerability)\n*Note: High confidence on network IOCs; evaluate hash against internal telemetry before blocking.*"*

#### Iteration Notes
* **Technique Applied:** Stated downstream operational usage (SOC triage & automated SIEM blocklists).
* **Observed Output Difference:** The model added confidence scores, operational warnings, and categorized indicators into distinct infrastructure types (IP vs. domain vs. CVE).
* **Failure Point Remaining:** Output remains in free-form prose and bullet points, making programmatic automated ingestion impossible.

---

### Version 3: Added Technique — Output Structure (JSON Schema)

#### Prompt
> *"You are a Principal Cyber Threat Intelligence (CTI) Analyst. Summarize this threat report and extract IOCs. Context: Ingested by SOC Tier-1 analysts to update SIEM blocklists. **Output Structure: Respond EXCLUSIVELY in valid JSON using this schema:**\n```json\n{\n  \"campaign_summary\": \"string\",\n  \"threat_actor\": \"string or null\",\n  \"cve_identifiers\": [\"string\"],\n  \"iocs\": {\n    \"ip_addresses\": [\"string\"],\n    \"domains\": [\"string\"],\n    \"file_hashes\": [\"string\"]\n  }\n}\n```"*

#### Raw Output Excerpt
> ```json
> {
>   "campaign_summary": "Spear-phishing campaign distributing macro-enabled PDFs to achieve initial access.",
>   "threat_actor": "APT29 (Suspected)",
>   "cve_identifiers": ["CVE-2023-38606"],
>   "iocs": {
>     "ip_addresses": ["192.0.2.45"],
>     "domains": ["malicious-domain.com"],
>     "file_hashes": ["e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"]
>   }
> }
> ```

#### Iteration Notes
* **Technique Applied:** Restricted output strictly to a predefined JSON schema.
* **Observed Output Difference:** Eliminated all conversational markdown fluff, yielding a deterministic data payload ready for automated scripts.
* **Failure Point Remaining:** The model hallucinated `"APT29 (Suspected)"` for the threat actor field despite the source text making no explicit mention of this group.

---

### Version 4: Added Technique — Few-Shot Examples

#### Prompt
> *"You are a Principal CTI Analyst. Extract threat intel into JSON.\n\n**Example 1 Input:** 'An unknown attacker targeted Apache servers using CVE-2021-44228. C2 traffic connected to 198.51.100.14.'\n**Example 1 Output:**\n```json\n{\"campaign_summary\": \"Attack on Apache servers exploiting Log4j.\", \"threat_actor\": null, \"cve_identifiers\": [\"CVE-2021-44228\"], \"iocs\": {\"ip_addresses\": [\"198.51.100.14\"], \"domains\": [], \"file_hashes\": []}}\n```\n\n**Now process this report using the same schema:** [Report Text]"*

#### Raw Output Excerpt
> ```json
> {
>   "campaign_summary": "Exploitation of web server vulnerabilities for unauthorized access.",
>   "threat_actor": null,
>   "cve_identifiers": ["CVE-2023-38606"],
>   "i
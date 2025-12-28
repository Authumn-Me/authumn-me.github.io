---

title = 'SMB Signing: A Critical Control Against NTLM Relaying and Lateral Movement'
date = '2025-12-28T08:10:36+01:00'
tags: ["windows", "security", "active directory"]
draft = true
cover:
  image: "cover.png"
  relative: true
  hiddenInSingle: true
  responsiveImages: false

---
Server Message Block (SMB) is one of the most fundamental protocols in Windows environments. It provides file sharing, remote management and a range of operating system functions that depend on integrity and trust. When SMB signing is not enforced, NTLM authentication can be relayed to SMB endpoints, allowing unauthorised access to systems that accept unsigned connections. This behaviour forms the basis of several real-world attack paths and continues to appear in security assessments.

Earlier analyses of [DHCPv6 poisoning](https://authumn.com/blog/dhcpv6-poisoning-an-overlooked-weakness-in-windows-networks) and [legacy name resolution protocols such as LLMNR, NBT-NS and mDNS](https://authumn.com/blog/llmnr-nbt-ns-and-mdns-legacy-name-resolution-protocols-that-open-the-door-to-credential-exposure/) demonstrate how Windows hosts may be deceived into sending authentication traffic to an attacker-controlled endpoint. When such traffic is not protected with modern safeguards, SMB becomes a high-value relay target. The combination of unauthenticated network discovery and a lack of SMB signing creates a powerful chain of compromise.

This post focuses on SMB signing as a defensive mechanism and uses supporting evidence to illustrate the risks that arise when it is disabled.

## **How SMB signing fits into the broader attack surface**

SMB signing ensures that SMB traffic cannot be modified or impersonated by an untrusted intermediary. When signing is enabled and enforced, the server verifies the authenticity of the SMB client’s messages, preventing NTLM authentication from being relayed.

When signing is not required, NTLM authentication received through poisoning techniques can be forwarded to SMB servers. Any endpoint that accepts unsigned SMB traffic becomes a potential pivot point for lateral movement.

This weakness is not theoretical. It is a predictable byproduct of:

- DHCPv6 poisoning redirecting authentication to a malicious host
- LLMNR, NBT-NS or mDNS poisoning impersonating requested names
- NTLM authentication being forwarded to SMB services lacking enforced signing
- …

The intersection of these behaviours is what makes SMB signing essential.

## **Observed behaviour during analysis**

Responder was used in the supporting evidence to demonstrate how poisoned requests generate authentication attempts that can be forwarded to SMB. The tool captured incoming traffic and prepared it for relaying.

The screenshot below show Responder collecting authentication requests that originate from hosts responding to poisoned network resolution mechanisms.

![alt text](ResponderSMB-1.png)

These captured credentials were then forwarded to SMB endpoints using ntlmrelayx, showing how systems without SMB signing enabled accept and process authentication that did not originate from the legitimate source.

![alt text](InterceptingSMB-1.png)

## **Relaying to SMB with a normal user account**

When an ordinary domain user account was relayed, the target system accepted the authentication attempt. Access was granted according to the permissions defined for that account, validating that SMB endpoints without signing are susceptible even to low-privilege relays.

The screenshot below demonstrates a successful SMB relay using a standard domain user identity.

![alt text](SMBInterceptionShowcase-1.png)

With this level of access, file shares become browsable and readable according to the user’s rights.

![alt text](SMBRelayingShares-1.png)

Although seemingly limited, such access often provides attackers with footholds, sensitive internal documentation or misconfigured administrative paths.

## **Relaying SMB authentication as a domain administrator**

Environments where SMB signing is disabled on critical systems present even more severe risks. In the evidence below, a domain administrator account was relayed to a system lacking SMB signing enforcement. The server accepted the authentication without challenge.

![alt text](SMBAdminRelay-1.png)

*The screenshot shows ntlmrelayx establishing authenticated SMB access with domain administrator privileges.*

This elevated access enables a wide range of high-impact actions, including:

### **Dumping the SAM database**

Windows stores local password hashes in the Security Account Manager (SAM). With administrative SMB access, these values become retrievable. The extracted hashes include NT hashes, which can be used directly to authenticate to various Windows services. In such cases, the password does not need to be cracked for the account to be compromised.

![alt text](SMBDumpSAM-1.png)

### **Retrieving LSA secrets and cached credentials**

The Local Security Authority (LSA) stores a range of sensitive values that include credentials used by services, scheduled tasks and system components. These LSA secrets may also contain cached domain logon data, which allows users to authenticate when domain controllers are unavailable.

In some environments, LSA secrets can include plaintext or partially recoverable passwords. This behaviour results from a separate underlying issue that falls outside the scope of this blog and will be addressed in a future post.

The ability to extract LSA secrets and cached credentials illustrates the impact of gaining administrative SMB access on systems where SMB signing is not enforced.

![alt text](SMBDumpLSA-1.png)

### **Remote code execution over SMB**

Administrative SMB access allows command execution on the target system using standard operating system functions when signing is not enforced.

These outcomes highlight the severity of NTLM relaying when SMB signing is absent. The attack path requires no credentials and no exploitation of software vulnerabilities. It relies solely on predictable network behaviour and missing protocol integrity. The examples shown above, such as share access, SAM extraction, LSA secret retrieval and remote code execution, represent only a subset of what becomes possible once SMB access is achieved through relaying. A wide range of additional actions can be performed across the environment, depending on the privileges of the relayed account and the configuration of the target systems. These examples are included to illustrate the impact, not to suggest that they are the only consequences of disabled SMB signing.

## **Why SMB signing remains essential**

The role of SMB signing is to enforce message integrity and protect NTLM authentication from redirection. When signing is fully enforced:

- NTLM authentication cannot be relayed
- SMB traffic cannot be modified or forged
- The attacker must control legitimate credentials
- Poisoning techniques lose their leverage over relaying SMB access

Without signing, SMB becomes one of the most powerful pivot points available in a Windows domain.

The issue persists because:

- Many legacy systems still disable signing for compatibility
- Administrators assume relaying attacks require rare conditions
- IPv6 poisoning and LLMNR/NBT-NS/mDNS poisoning are often overlooked
- NTLM remains widely enabled for backward compatibility

The combination makes SMB signing failures both common and dangerous.

## **Mitigation strategies**

### **Enable and enforce SMB signing**

SMB signing should be enabled and strictly enforced on both domain controllers and servers. Enforcing signing ensures that SMB messages cannot be altered or replayed by an intermediary, effectively preventing NTLM authentication from being relayed to SMB services. Systems that only have signing set to “supported” remain vulnerable, since they will still accept unsigned connections from clients that do not negotiate signing. Full enforcement is therefore essential to guarantee message integrity across the environment and to close one of the most common lateral movement paths in Windows networks.

SMB signing can be enforced centrally through Group Policy, allowing organisations to apply consistent protection across all systems. Configuring domain controllers and servers through GPO ensures that no machine remains accidentally exposed due to local configuration drift or legacy settings.

## **Conclusion**

SMB signing is a fundamental security control in Windows environments. Its absence allows NTLM authentication to be relayed to SMB services, enabling unauthorised access ranging from limited file share browsing to full domain administrative takeover.

When combined with poisoning techniques previously outlined in discussions of DHCPv6, LLMNR, NBT-NS and mDNS, the risk becomes systemic. These protocols redirect authentication traffic, and unsigned SMB endpoints accept it without verification.

The evidence presented demonstrates how both normal users and domain administrators can be relayed to vulnerable SMB services, leading to credential extraction, lateral movement and remote code execution. Enforcing SMB signing across all systems closes one of the most consistent and impactful attack paths in enterprise networks.
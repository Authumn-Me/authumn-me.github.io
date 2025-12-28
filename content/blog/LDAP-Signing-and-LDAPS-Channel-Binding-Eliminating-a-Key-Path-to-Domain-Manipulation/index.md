---

title : 'LDAP Signing and LDAPS Channel Binding: Eliminating a Key Path to Domain Manipulation'
date : '2025-12-28'
tags: ["windows", "security", "active directory"]
cover:
  image: "cover.png"
  relative: true
  hiddenInSingle: true
  responsiveImages: false

---

Active Directory relies heavily on LDAP and LDAPS for authentication, directory queries and system management. These protocols underpin a significant portion of domain operations, including user lookups, group membership resolution, computer account provisioning and policy distribution. When LDAP signing and LDAPS channel binding are not enforced, NTLM authentication can be silently relayed to directory services, creating opportunities for unauthorised changes within the domain.

Earlier discussions on [DHCPv6 poisoning](https://authumn.com/blog/dhcpv6-poisoning-an-overlooked-weakness-in-windows-networks) and [legacy name resolution protocols (LLMNR, NBT-NS and mDNS)](https://authumn.com/blog/llmnr-nbt-ns-and-mdns-legacy-name-resolution-protocols-that-open-the-door-to-credential-exposure/) highlight how Windows hosts can be deceived into sending authentication traffic to a rogue endpoint. LDAP services become natural relay targets when these fallback mechanisms remain active. Without LDAP signing or proper LDAPS channel binding, directories may accept authentication that did not originate from the legitimate client, enabling a range of impactful actions.

## **How LDAP signing and LDAPS channel binding work**

### **LDAP signing**

LDAP signing ensures that all LDAP communication is protected against manipulation by adding cryptographic integrity checks to each message. When signing is required, the server verifies that the data originates from the authenticated session and has not been altered in transit. If signing is not enforced, LDAP servers will continue to accept unsigned or only partially protected messages. This behaviour allows NTLM authentication to be relayed into LDAP because the server does not verify whether the incoming request truly originates from the client that performed the authentication.

### **LDAPS channel binding**

LDAPS channel binding adds an additional safeguard by linking the authentication process directly to the underlying TLS session. The server verifies that the NTLM authentication belongs to the same encrypted connection that was originally established between the client and the directory. If channel binding is not enforced, LDAPS will still accept NTLM authentication that arrives over a different connection, even though the session is encrypted. This allows attackers to relay NTLM credentials into LDAPS endpoints despite the presence of TLS, making directory write operations possible whenever the relayed user has sufficient permissions.

Both measures exist to guarantee that authentication originates from the device actually connected to the directory and not from an attacker acting as an intermediary.

## **Evidence of LDAP and LDAPS relaying**

Responder can be used to capture authentication traffic generated through poisoned name resolution or IPv6 redirection. Combined with ntlmrelayx, this allows the behaviour of LDAP and LDAPS endpoints to be demonstrated in environments where signing and channel binding are not enforced.

![](https://www.resilix.be/web/image/3828-23bd75c1/image.png?access_token=93ded74a-dd22-4d5e-a2cf-7ad3925cd7d7)

The screenshot below displays ntlmrelayx successfully authenticating to LDAP using relayed NTLM credentials.

![](https://www.resilix.be/web/image/3829-29e9d47e/image.png?access_token=110ac317-c838-4da3-b842-c6fc0bf87f05)

The next screenshot shows LDAPS accepting the relayed authentication. When channel binding is not enforced, this authentication can be relayed to perform various directory operations, depending on the permissions of the relayed user.

![](https://www.resilix.be/web/image/3831-23850a58/image.png?access_token=c5843f79-a1af-4c20-a504-bdcc0cf16ecc)

The last screenshot shows the help functionality of the interactive LDAPS shell. It lists the actions that can be carried out through an LDAPS relay, such as creating computer accounts, adding new users, retrieving LDAP domain information or modifying group memberships, among others.

![](https://www.resilix.be/web/image/3832-cf550a6a/image.png?access_token=29d8f160-4e70-428e-a42e-afacb768adcc)

These examples illustrate how credentials captured indirectly through network behaviour can result in directory-level access without the user’s awareness.

## **Impact of LDAP and LDAPS relaying**

LDAP relaying allows authentication to be redirected to directory services and processed under the security context of the relayed user. The resulting access level depends entirely on the privileges assigned to that account.

Common high-impact scenarios include:

### **1. Abusing default machine account creation rights**

In many Active Directory environments, regular users are allowed to create up to ten computer accounts. This behaviour is controlled by the ms-DS-MachineAccountQuota attribute.

If this attribute is not set to zero, LDAPs relaying can be used to create new machine accounts under the identity of the relayed user.

![](https://www.resilix.be/web/image/3833-4f524a40/image.png?access_token=d47e25db-4ca8-4705-aa24-a1a7563480a9)

### **2. Retrieving domain information**

LDAP relaying can expose a wide range of domain data such as user lists, group memberships and configuration attributes.

![](https://www.resilix.be/web/image/3835-e2bbc265/image.png?access_token=be14d89c-2a1c-40e0-90d5-6286e0c76e1a)

### **3. Interactive LDAP sessions with privilege escalation opportunities**

Many relaying demonstrations show how an interactive LDAP shell becomes available under the relayed user identity. Within this environment, LDAP StartTLS upgrades may allow extended operations, depending on server settings.

### **4. Writing to the directory through LDAPS relaying**

When LDAPS does not enforce channel binding, the server may allow authenticated write operations. The relayed account’s permissions determine which modifications are possible.

Typical writeable targets include:

- Creation of new user accounts
- Assigning group membership or rights
- Modifying attributes on existing objects
- Preparation steps toward broader domain compromise

These actions highlight the systemic impact of missing LDAP signing and channel binding, particularly when combined with earlier poisoning techniques.

## **These examples are not the full extent of LDAP relaying**

The demonstrated actions, such as creating machine accounts, retrieving domain information and performing directory modifications, represent only a small portion of what becomes possible when LDAP signing and channel binding are not enforced. LDAP relaying enables a wide attack surface that depends on privilege levels and Active Directory configuration. The examples presented in this blog illustrate the potential consequences but do not reflect the full range of LDAP relay capabilities.

## **Why LDAP signing and LDAPS channel binding matter**

Without LDAP signing or channel binding:

- NTLM authentication can be relayed into LDAP or LDAPS
- Directory operations may be performed without user awareness
- Poisoning attacks become significantly more impactful
- High-value accounts that authenticate on the network create immediate risk
- Lateral movement becomes trivial on systems lacking these protections

LDAP and LDAPS are central to domain operations. Ensuring the integrity of these protocols is critical for preventing silent privilege escalation.

## **Mitigation strategies**

### **Enforce LDAP signing**

LDAP signing must be set to “Required” on all domain controllers. The default “None” or “Negotiate” settings leave directories exposed to NTLM relaying.

### **Enforce LDAPS channel binding**

Channel binding ensures that authentication is tied to the TLS session and prevents relaying into LDAPS endpoints.

## **Conclusion**

LDAP signing and LDAPS channel binding are essential for ensuring that Active Directory does not accept authentication that has been silently redirected through poisoning techniques. Without these controls, authentication captured via DHCPv6 poisoning, LLMNR, NBT-NS or mDNS can be relayed to directory services and processed as if it were legitimate. The result ranges from metadata extraction to object creation, privilege assignment and potential domain compromise.

A future post will explore misconfigurations and authentication behaviours that result in plaintext or recoverable credentials within LDAP-related operations, extending the discussion on credential exposure beyond the relay mechanisms described here.
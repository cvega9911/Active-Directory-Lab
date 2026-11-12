# Active Directory Lab

I built a Windows domain from scratch on a used mini PC. There's a Windows Server 2025 domain controller, a Windows 11 client joined to it, a file share, and a few Group Policies you can actually see working. It all runs as VMs on Proxmox, and every screenshot in here is from my own build (mistakes included, there were a few).

I built this to get real hands-on time with the stuff help desk and sysadmin roles live in every day: AD users and groups, DNS, domain joins, shares, permissions and GPOs.

## The setup

| Piece | What it is |
|---|---|
| Host | Lenovo ThinkCentre M720q Tiny, i7-8700T, 32GB RAM, 1TB NVMe (bought used) |
| Hypervisor | Proxmox VE 9.2, reserved IP on my home router |
| dc01 (VM 100) | Windows Server 2025 Standard Evaluation (Desktop Experience), static IP 192.168.1.211 |
| win11-client (VM 101) | Windows 11 Enterprise Evaluation, gets its IP from the router, uses dc01 for DNS |
| Domain | `lab.local` (NetBIOS name `LAB`) |

```
            Home router (192.168.1.1)
            DHCP for the apartment, internet
                       |
          +------------+------------+
          |                         |
   Proxmox host (pve)          everything else
   192.168.1.146               in the apartment
          |
   +------+--------------------------+
   |                                 |
 dc01 (192.168.1.211)          win11-client (DHCP)
 AD DS, DNS, file share  <---  DNS points only at dc01
 lab.local                     joined to lab.local
```

The M720q came off eBay, so before anything else I ran MemTest86+ overnight (9 passes, zero errors) and checked the SSD in CrystalDiskInfo: 99% health, about 7.7TB written. Fine for a lab.

About the domain name: `.local` works for a Windows-only lab, but it's not what you'd pick for a real company (it can clash with Apple's Bonjour/mDNS, and it isn't a domain you can own). I'd use a subdomain of a real domain next time.

## 1. Proxmox

Proxmox went on bare metal (ext4 on the NVMe), with the box's IP reserved in my router so the web interface stays at the same address. A fresh install points at the paid enterprise repos, so the first update fails. I disabled those and added the free no-subscription repo.

![Proxmox dashboard after install](screenshots/01-proxmox-dashboard.png)

![Enterprise repos disabled, no-subscription repo enabled](screenshots/02-proxmox-no-subscription-repo.png)

## 2. Building dc01

I uploaded the Windows Server 2025 evaluation ISO plus the VirtIO driver ISO, then created the VM with q35, UEFI (OVMF) with an EFI disk, VirtIO SCSI and the QEMU guest agent turned on.

![VM system settings](screenshots/03-dc01-vm-system-settings.png)

I picked the Desktop Experience edition. The default option is Server Core (command line only), which I'll get to eventually, but not while learning AD.

![Picking Desktop Experience](screenshots/04-server-2025-desktop-experience.png)

Setup showed no disks at all. Windows doesn't ship a driver for the VirtIO disk controller, so I had to hit Load Driver and point it at the `vioscsi` folder on the VirtIO disc.

![No disks until the VirtIO driver is loaded](screenshots/05-no-disks-until-driver-loaded.png)

Once Windows was up I ran the VirtIO guest tools installer to get the network driver and the guest agent.

![VirtIO guest tools](screenshots/06-virtio-guest-tools.png)

Before touching AD: renamed the server to `dc01` (renaming a DC after promotion is a pain) and gave it a static IP. A domain controller shouldn't be moving around on you.

![Server renamed to dc01](screenshots/07-dc01-renamed.png)

![Static IP on dc01](screenshots/08-dc01-static-ip.png)

I took a Proxmox snapshot here (`pre-ad`) so I had a clean rollback point, and kept snapshotting after every big step (`ad-installed` after promotion, `ad-populated` after the users and groups).

![dc01 snapshot chain](screenshots/22-dc01-snapshot-chain.png)

## 3. Promoting dc01 to a domain controller

Added the Active Directory Domain Services role, then promoted the server into a brand new forest called `lab.local`.

![Adding the AD DS role](screenshots/09-adds-role.png)

![New forest, lab.local](screenshots/10-new-forest-lab-local.png)

The prerequisite check passed with two warnings. One is about IPv6 not being static (IPv4 is static, IPv6 is still whatever the router gives it, which is fine for a lab but not how you'd leave a real DC) and the other is a DNS delegation warning, which doesn't apply since there's no parent zone above `lab.local`.

![Prerequisites check](screenshots/11-prerequisites-check.png)

## 4. DNS

After the reboot, the promotion had already pointed dc01's DNS at itself (`127.0.0.1`). That's correct now that it's the DNS server for the domain.

![DNS pointed at itself after promotion](screenshots/12-dns-pointed-at-itself.png)

dc01 still needs to resolve things like google.com, so I added forwarders: my router and Cloudflare (`1.1.1.1`). Anything dc01 doesn't know about gets passed along.

![DNS forwarders](screenshots/13-dns-forwarders.png)

`nslookup` kept showing the server as "UnKnown," so I built a reverse lookup zone for `192.168.1.x` and added a PTR record for dc01 (a PTR record is just the reverse of an A record: IP to name instead of name to IP).

![Reverse lookup zone](screenshots/14-reverse-lookup-zone.png)

![PTR record for dc01](screenshots/15-ptr-record.png)

![Reverse lookup working](screenshots/16-nslookup-reverse.png)

(The "UnKnown" stuck around anyway because nslookup was asking over the IPv6 loopback `::1`, which has no reverse record. Lookups work fine, so I let that one go instead of chasing it down a rabbit hole.)

## 5. dcdiag (and reading the noise)

`dcdiag` passed almost everything, but failed `DFSREvent` and `SystemLog`.

![dcdiag primary tests](screenshots/17-dcdiag-primary-tests.png)

Digging into the SystemLog output, everything it flagged was logged within about a minute of the first boot after promotion: DNS lookups timing out before the DNS service was up (some of them using my router's `lan` suffix), WinRM starting before the domain was ready, the time service pointing out there's no external time source, and one error from the Security Account Manager failing a Kerberos request. That error is what actually fails the test, and it lines up with the server coming up before its own DNS did. Logins and the rest of the lab worked fine after that, and those two tests only look at the last 24 hours of events, so they should clear once those age out. (Re-running dcdiag to confirm is on my list.)

![The warnings, all from first boot](screenshots/18-dcdiag-systemlog-warnings.png)

![Partition and enterprise tests passing](screenshots/19-dcdiag-enterprise-tests.png)

That's the part I'd actually use at a help desk: figuring out which errors matter and which ones are just the server waking up.

## 6. Users, groups and OUs

I went to make an OU called "Users" and AD said no, there's already one. Turns out the built-in Users and Computers aren't OUs at all, they're plain containers, and you can't link Group Policy to them. That's the whole reason to make your own. I created:

- **Lab Users**, **Lab Computers**, **Servers**, **Groups**, and later **Staff**
- Test users John Smith (`jsmith`) and Mary Jones (`mjones`)
- Security groups Accounting and Marketing
- My own admin account in Domain Admins, so I'm not using the built-in Administrator for everything

![Security group membership](screenshots/20-security-group-members.png)

![My admin account in Domain Admins](screenshots/21-my-admin-account-domain-admins.png)

Side quest: I made a test OU with "Protect from accidental deletion" checked and then couldn't delete it. The setting lives on the Object tab, and that tab only shows up after you turn on View > Advanced Features. (Works exactly as advertised. It protected the OU from me.)

## 7. The Windows 11 client and the domain join

The client is VM 101: Windows 11 Enterprise evaluation, UEFI with a TPM, and VirtIO again. I pointed Load Driver at the Windows install disc first, which (shockingly) has no VirtIO drivers on it. The right disc was the one labeled `virtio-win`.

![Wrong disc, no drivers](screenshots/23-client-wrong-driver-disc.png)

A computer can only join a domain if it can find the DC, and it finds the DC through DNS. So the client keeps getting its IP from the router but uses dc01 (and only dc01) for DNS. I set that on the client itself and didn't touch the router, so the rest of the apartment keeps working normally.

![Client DNS pointed at dc01](screenshots/24-client-dns-pointed-at-dc.png)

That still failed. `nslookup lab.local` was going to the IPv6 DNS server my router hands out, which has obviously never heard of my lab domain.

![nslookup asking the wrong server](screenshots/25-client-nslookup-fails.png)

Asking dc01 directly worked, which proved the DC was fine and the client was just asking the wrong server first:

![Asking dc01 directly works](screenshots/26-client-nslookup-direct-to-dc.png)

The cause was IPv6. Windows was preferring the router's IPv6 DNS server over the IPv4 one I set. I turned off IPv6 on the client's adapter and flushed DNS, and dc01 became the only DNS server it knew about. (That's a lab shortcut. Microsoft doesn't recommend disabling IPv6, and the cleaner fix is giving the client's IPv6 settings dc01's address as DNS too.)

![DNS fixed, dc01 is the only DNS server](screenshots/27-client-dns-fixed.png)

Then the join itself told me my username was invalid. I retyped it a bunch of times before I noticed I was using a forward slash (`LAB/Administrator`) instead of a backslash. (In my defense, it was late.)

![The specified username is invalid](screenshots/28-join-username-invalid.png)

I ended up renaming the client and doing the join in PowerShell, mostly because you can see exactly what you typed. These were run separately, since each one restarts the machine:

```powershell
Rename-Computer -NewName WIN11-CLIENT -Restart
# after the reboot:
Add-Computer -DomainName lab.local -Credential LAB\Administrator -Restart
```

![Joining the domain with PowerShell](screenshots/29-add-computer-powershell.png)

After the restart I logged in as `LAB\jsmith`, a domain user that only exists on dc01.

![Logging in as a domain user](screenshots/30-domain-login-jsmith.png)

![Client joined to lab.local](screenshots/31-client-joined-lab-local.png)

New computers land in the built-in Computers container. Domain-wide policies still apply there, but you can't link a GPO to it, so I moved the client into Lab Computers where I can target it later. Then I snapshotted it.

![Client in the Lab Computers OU](screenshots/32-client-in-lab-computers-ou.png)

![Client snapshot](screenshots/33-client-snapshot.png)

## 8. A file share

I made a Staff OU and moved both test users into it. My admin account stays out of it on purpose, so the restrictive policies never land on me.

![Staff OU](screenshots/34-staff-ou.png)

Created the share from the command line. My first try failed because of one space after the comma (`",full"` and `", full"` are not the same thing to `net share`):

![net share syntax error](screenshots/35-net-share-syntax-error.png)

```cmd
mkdir C:\Shares\Company
net share Company=C:\Shares\Company /grant:"LAB\Domain Users",full
```

![Share created](screenshots/36-net-share-success.png)

Shares have two sets of permissions stacked on top of each other: share permissions (only checked when you come in over the network) and NTFS permissions (checked every time, network or not). Whichever is stricter wins. The share gave Domain Users Full, but the NTFS permissions were mostly read-only, so I gave Domain Users Modify on the folder. Net result over the network is Modify, which is what I wanted.

![NTFS Modify for Domain Users](screenshots/37-ntfs-modify.png)

## 9. Group Policy

Three policies:

| Policy | Linked to | What it does |
|---|---|---|
| Default Domain Policy | lab.local | 12 character minimum, complexity on, 90 day max age, 24 passwords remembered |
| Staff - Drive Map | Staff OU | Maps `\\dc01\Company` as the S: drive |
| Staff - Lockdown | Staff OU | Blocks Control Panel and the Settings app |

Password rules for domain accounts only take effect from a GPO linked to the domain itself (linking one to an OU does nothing for domain passwords), so I just set them in the Default Domain Policy. Different rules per group is a separate feature (fine-grained password policies) that I haven't touched yet.

![Password policy](screenshots/38-password-policy.png)

![Drive map GPO](screenshots/39-drive-map-gpo.png)

![Lockdown GPO](screenshots/40-lockdown-gpo.png)

![Both GPOs linked to Staff](screenshots/41-gpmc-staff-links.png)

## 10. Proving it works

Logged into the client as `jsmith` and ran `gpupdate /force`. The S: drive showed up in File Explorer:

![S: drive mapped](screenshots/42-client-s-drive-mapped.png)

`gpresult /r` shows both Staff GPOs applied to John Smith in the Staff OU:

![gpresult for jsmith](screenshots/43-gpresult-jsmith.png)

And Settings wouldn't open at all. Clicking it did nothing, and right clicking it doesn't even give you an Open option anymore:

![Settings blocked for jsmith](screenshots/44-settings-blocked.png)

For a quick control test, I ran `gpresult /r` as the domain Administrator on dc01 (that account isn't in the Staff OU). No user GPOs applied, so the Staff policies aren't leaking onto accounts outside Staff. A better test would be logging into the same client with my own admin account, which is on the list.

![gpresult for Administrator, nothing applied](screenshots/45-gpresult-admin-control.png)

## Problems I ran into

The quick version, for anyone skimming:

| Problem | Fix |
|---|---|
| Windows setup showed no disks | Load the `vioscsi` driver from the VirtIO disc |
| dcdiag failed DFSREvent and SystemLog | First-boot events, nothing actually broken |
| Client couldn't find lab.local | It was asking the router's IPv6 DNS server first |
| "The specified username is invalid" | `/` instead of `\` |
| `net share` printed its help text | One stray space after the comma |

## What I didn't do (on purpose)

- **No DHCP role on dc01.** My router already hands out addresses, and two DHCP servers on one network will fight each other. That waits until I've got my own firewall/router running.
- **No changes to my home router's DNS.** Only the lab machines use dc01, so nothing else in the apartment breaks if the lab is down.

## Notes

- No passwords appear anywhere in this repo or the screenshots.
- My public IPv6 addresses are blacked out in the screenshots. The `192.168.x.x` addresses are private and only mean something inside my apartment.
- Both Windows VMs are eval editions, so they'll expire eventually (that's what the snapshots are for).

## What's next

- Re-run dcdiag and point dc01 at an external time source (that NtpClient warning)
- Control test from the client with my own admin account
- More GPOs: a desktop wallpaper, software restrictions, a logon script
- Account lockout policy, and testing what a locked-out user looks like from the help desk side
- A small k3s cluster on the same Proxmox host
- Syncing users to Microsoft Entra ID with Entra Connect (needs a real UPN suffix added, since Entra won't take `@lab.local`)

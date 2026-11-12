# Active Directory Lab

A Windows domain built from scratch on a used mini PC: a Windows Server 2025 domain controller, a domain-joined Windows 11 client, a file share, and Group Policy that actually does something you can see. Everything runs as VMs on Proxmox, and every screenshot in here is from my own build (mistakes included, because those taught me the most).

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
            DHCP for the house, internet
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

Before any of this I put the M720q through its paces since it came off eBay: MemTest86+ ran 9 passes overnight with zero errors, and CrystalDiskInfo showed the SSD at 99% health with about 7.7TB written. Good enough to trust with a lab.

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

I took a Proxmox snapshot here (`pre-ad`) so I had a clean rollback point.

## 3. Promoting dc01 to a domain controller

Added the Active Directory Domain Services role, then promoted the server into a brand new forest called `lab.local`.

![Adding the AD DS role](screenshots/09-adds-role.png)

![New forest, lab.local](screenshots/10-new-forest-lab-local.png)

The prerequisite check passed with two warnings. One is about IPv6 not being static (my IPv4 is, which is what matters here) and the other is a DNS delegation warning, which doesn't apply since there's no parent zone above `lab.local`.

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

Digging into the SystemLog output, every single warning was stamped within the same minute as the first boot after promotion: DNS lookups timing out before the DNS service was up, WinRM starting before the domain was ready, and the time service pointing out there's no external time source. Those two tests just scan recent events, so they fail on startup noise and clear on their own once those events age out.

![The warnings, all from first boot](screenshots/18-dcdiag-systemlog-warnings.png)

![Partition and enterprise tests passing](screenshots/19-dcdiag-enterprise-tests.png)

Telling real problems apart from noise was probably the most useful thing I got out of this section.

## 6. Users, groups and OUs

I went to make an OU called "Users" and AD said no, there's already one. Turns out the built-in Users and Computers aren't OUs at all, they're plain containers, and you can't link Group Policy to them. That's the whole reason to make your own. I created:

- **Lab Users**, **Lab Computers**, **Servers**, **Groups**, and later **Staff**
- Test users John Smith (`jsmith`) and Mary Jones (`mjones`)
- Security groups Accounting and Marketing
- My own admin account in Domain Admins, so I'm not using the built-in Administrator for everything

![Security group membership](screenshots/20-security-group-members.png)

![My admin account in Domain Admins](screenshots/21-my-admin-account-domain-admins.png)

Side quest: I made a test OU with "Protect from accidental deletion" checked and then couldn't delete it. The setting lives on the Object tab, and that tab only shows up after you turn on View > Advanced Features. (It works exactly as advertised. It protected it from me.)

![dc01 snapshot chain](screenshots/22-dc01-snapshot-chain.png)

## 7. The Windows 11 client and the domain join

The client is VM 101: Windows 11 Enterprise evaluation, UEFI with a TPM, and VirtIO again. I pointed Load Driver at the Windows install disc first, which (shockingly) has no VirtIO drivers on it. The right disc was the one labeled `virtio-win`.

![Wrong disc, no drivers](screenshots/23-client-wrong-driver-disc.png)

A computer can only join a domain if it can find the DC, and it finds the DC through DNS. So the client keeps getting its IP from the router but uses dc01 (and only dc01) for DNS. I set that on the client itself and didn't touch the router, so the rest of the apartment keeps working normally.

![Client DNS pointed at dc01](screenshots/24-client-dns-pointed-at-dc.png)

That still failed. `nslookup lab.local` was going to my ISP's DNS server, which has obviously never heard of my lab domain.

![nslookup asking the wrong server](screenshots/25-client-nslookup-fails.png)

Asking dc01 directly worked, which proved the DC was fine and the client was just asking the wrong server first:

![Asking dc01 directly works](screenshots/26-client-nslookup-direct-to-dc.png)

The cause was IPv6. My router hands out an IPv6 DNS server, and Windows was preferring it over the IPv4 one I set. I turned off IPv6 on the client's adapter and flushed DNS, and dc01 became the only DNS server it knew about.

![DNS fixed, dc01 is the only DNS server](screenshots/27-client-dns-fixed.png)

Then the join itself told me my username was invalid. I tried every variation I could think of. The actual problem: I was typing `LAB/Administrator` with a forward slash instead of `LAB\Administrator`. (In my defense, it was late.)

![The specified username is invalid](screenshots/28-join-username-invalid.png)

I ended up doing the join in PowerShell with `Add-Computer`, mostly because you can see exactly what you typed.

```powershell
Rename-Computer -NewName WIN11-CLIENT -Restart
Add-Computer -DomainName lab.local -Credential LAB\Administrator -Restart
```

![Joining the domain with PowerShell](screenshots/29-add-computer-powershell.png)

After the restart I logged in as `LAB\jsmith`, a domain user that only exists on dc01.

![Logging in as a domain user](screenshots/30-domain-login-jsmith.png)

![Client joined to lab.local](screenshots/31-client-joined-lab-local.png)

New computers land in the built-in Computers container, so I moved it into Lab Computers where policies can reach it, then snapshotted the client.

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

A share has two layers of permissions: share permissions (who can get in over the network) and NTFS permissions (what they can do with the files). The more restrictive one wins. The share was Full for Domain Users, but NTFS only gave them read, so I added Modify. Net result is Modify, which is what I wanted.

![NTFS Modify for Domain Users](screenshots/37-ntfs-modify.png)

## 9. Group Policy

Three policies:

| Policy | Linked to | What it does |
|---|---|---|
| Default Domain Policy | lab.local | 12 character minimum, complexity on, 90 day max age, 24 passwords remembered |
| Staff - Drive Map | Staff OU | Maps `\\dc01\Company` as the S: drive |
| Staff - Lockdown | Staff OU | Blocks Control Panel and the Settings app |

The password policy has to live at the domain level. That's the only place Windows enforces password rules for domain accounts, which is why it went in the Default Domain Policy instead of its own GPO.

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

For the control test, I ran the same thing as the domain Administrator on dc01. No user GPOs applied, so the lockdown only hits the people it's meant for:

![gpresult for Administrator, nothing applied](screenshots/45-gpresult-admin-control.png)

## Problems I ran into

| Problem | Cause | Fix |
|---|---|---|
| Windows setup showed no disks | No built-in driver for the VirtIO disk controller | Load Driver from `vioscsi` on the VirtIO disc |
| "No drivers were found" on the client | Browsed the Windows install disc instead of the VirtIO disc | Picked the `virtio-win` drive |
| nslookup showed "UnKnown" | No reverse zone, then querying over IPv6 loopback | Added a reverse zone and PTR record, left the cosmetic part alone |
| dcdiag failed DFSREvent and SystemLog | Events logged during the first boot after promotion | Nothing broken, they clear as the events age out |
| Couldn't create an OU named Users | Built-in Users container already uses the name | Named mine Lab Users |
| Couldn't delete a test OU | Protect from accidental deletion | Advanced Features > Object tab > uncheck it |
| Client couldn't find lab.local | Windows preferred the router's IPv6 DNS server | Disabled IPv6 on the client adapter, flushed DNS |
| "The specified username is invalid" | Typed `/` instead of `\` | `LAB\Administrator`, joined with PowerShell |
| `net share` printed its help text | One space after the comma in `/grant` | Removed the space |

## What I didn't do (on purpose)

- **No DHCP role on dc01.** My router already hands out addresses, and two DHCP servers on one network will fight each other. That waits until I've got my own firewall/router running.
- **No changes to my home router's DNS.** Only the lab machines use dc01, so nothing else in the apartment breaks if the lab is down.

## Notes

- No passwords appear anywhere in this repo or the screenshots.
- My public IPv6 addresses are blacked out in the screenshots. The `192.168.x.x` addresses are private and only mean something inside my apartment.
- Both Windows VMs are evaluation editions, so this is a lab, not something to run long term. Snapshots make rebuilding easy.

## What's next

- Point dc01 at an external time source (that NtpClient warning)
- More GPOs: a desktop wallpaper, software restrictions, a logon script
- Account lockout policy, and testing what a locked-out user looks like from the help desk side
- A small k3s cluster on the same Proxmox host
- Syncing `lab.local` to Microsoft Entra ID with Entra Connect

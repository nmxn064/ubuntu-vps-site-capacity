# Ubuntu virtual server host: how many sites can one server handle, what specs do you need, and what does it cost per month?

One Ubuntu machine can serve a dozen domains at once, and the Apache feature that makes it possible is the virtual host. The server underneath is a separate decision. SpinServers' smallest KVM plan (2 vCPU, 2 GB DDR4, 40 GB RAID-10 SSD, 4 TB of monthly transfer on a 1 Gbps port, one IPv4 and one IPv6, billed monthly) is enough for one low-traffic site or two. The rest of this guide covers the full plan list, the seven steps that point several domains at one machine, and two costs that rarely make it onto a promo page: what happens to the price after the first term, and the SMTP port SpinServers shuts by default.

> **Quick pick:** For two to five small sites (a blog, a brochure site, a staging copy of each) go with the 4 vCPU / 4 GB / 80 GB configuration. Apache plus MySQL plus PHP on 2 GB of RAM leaves very little room once a second site and its database are running. Every plan bills monthly, includes one IPv4 and one IPv6 address, and limits you to name-based virtual hosting. 👉 [Check the current KVM VPS configurations and monthly prices](https://bit.ly/Spinservers).

## What SpinServers' KVM VPS line actually contains

The VPS catalog is KVM-based and splits into two families: six Simple Application Server configurations running from 2 to 16 vCPU, and one Advanced Application Server at 32 vCPU. Every configuration uses RAID-10 SSD storage, a 1 Gbps network port, and ships with one IPv4 plus one IPv6 address.

| Plan | vCPU / RAM | Storage | Monthly transfer | Price and term | Order |
| --- | --- | --- | --- | --- | --- |
| Simple Application Server | 2 vCPU / 2 GB DDR4 | 40 GB RAID-10 SSD | 4 TB | Monthly billing; live price on the order page | [2 vCPU plan](https://bit.ly/Spinservers) |
| Simple Application Server | 4 vCPU / 4 GB DDR4 | 80 GB RAID-10 SSD | 8 TB | Monthly billing; live price on the order page | [4 vCPU plan](https://bit.ly/Spinservers) |
| Simple Application Server | 6 vCPU / 8 GB DDR4 | 120 GB RAID-10 SSD | 8 TB | Monthly billing; live price on the order page | [6 vCPU plan](https://bit.ly/Spinservers) |
| Simple Application Server | 8 vCPU / 8 GB DDR4 | 160 GB RAID-10 SSD | 10 TB | Monthly billing; live price on the order page | [8 vCPU plan](https://bit.ly/Spinservers) |
| Simple Application Server | 12 vCPU / 12 GB DDR4 | 160 GB RAID-10 SSD | 10 TB | Monthly billing; live price on the order page | [12 vCPU plan](https://bit.ly/Spinservers) |
| Simple Application Server (Xeon) | 16 vCPU / 16 GB DDR4 | 320 GB RAID-10 SSD | 15 TB | Monthly billing; live price on the order page | [16 vCPU plan](https://bit.ly/Spinservers) |
| Advanced Application Server (Xeon) | 32 vCPU / 32 GB | 500 GB SSD | 20 TB | Monthly billing; live price on the order page | [32 vCPU plan](https://bit.ly/Spinservers) |

SpinServers doesn't publish a fixed per-plan price table on the page reviewed here. It lists the configurations and advertises a featured one from **$62.00 USD per month**, with the exact figure for each plan appearing at checkout. Confirm the number for the configuration you pick before you pay, because that is the only place it is stated.

Four things on that page matter more than the price column:

- **Port 25 is closed by default on every VPS.** If your sites send mail (contact forms, password resets, order confirmations), you need to open a ticket with a justification, and SpinServers reserves the right to refuse. Run transactional mail through an external SMTP relay instead of fighting for the port.
- **One IPv4 address per plan.** No add-on IP block is listed. That decides your hosting model for you, as explained below.
- **No automatic backup is mentioned.** Snapshot or back up `/var/www` and your databases yourself.
- **Monthly billing.** There is no multi-year prepay to commit to, which also means no discounted first term to compare against a renewal price.

## Virtual host or virtual server: the two words decide your config

A virtual server is the rented machine: a KVM guest with its own kernel, disk and IP addresses. A virtual host is one configuration block inside Apache that tells the web server which directory to serve for which domain. You rent the first to run the second.

Apache places no software limit on how many virtual hosts you can define. The ceiling is the machine: RAM consumed by Apache worker processes and MySQL, disk for each site's files and logs, CPU for PHP execution, bandwidth for traffic. A 2 GB VPS will comfortably serve a handful of static sites and start swapping once several WordPress installs with plugins share it.

There are two ways to map domains to sites:

- **Name-based hosting** uses one IP address for all domains and lets Apache pick the right site from the `Host` header of each request. HTTPS works through SNI, which every current browser supports.
- **IP-based hosting** gives each site its own IP address. It exists for legacy clients that don't send a hostname, and for the rare case where a site genuinely needs a dedicated address.

With one IPv4 per VPS, name-based hosting is the only practical option, and it is what nearly everyone wants anyway. If a client requires a dedicated IP, that requirement belongs in the hosting conversation before you order, not after.

## Setting up Apache virtual hosts on Ubuntu, step by step

You need an Ubuntu server, a non-root user with sudo privileges, Apache installed, and (for a public site) `A` records for `example.com` and `www.example.com` pointing at the server's public IP. These steps have been verified on Ubuntu 20.04, 22.04 and 24.04 LTS; the `sites-available` layout is identical across those releases.

**1. Install Apache and confirm it is running.**

bash
sudo apt update
sudo apt-get install apache2
systemctl status apache2


Loading the server's IP in a browser should show the default Apache page.

**2. Create one directory per site.**

bash
sudo mkdir -p /var/www/example.com/public_html
sudo mkdir -p /var/www/shop.example/public_html


`public_html` holds the files visitors request. The parent directory holds scripts and application code for that site.

**3. Set ownership so you can edit files and Apache can read them.**

bash
sudo chown -R $USER:$USER /var/www/example.com
sudo chmod -R 755 /var/www/example.com


The common alternative, `sudo chown -R www-data:www-data /var/www/example.com`, hands the whole tree to the web server user. It works, but every file you upload or edit afterwards needs sudo, and files created by your login user can end up unreadable to Apache. Owning the tree as `$USER` with 755 permissions keeps both sides working. Where an application must write (an uploads folder, a cache directory), loosen permissions on that one directory rather than the whole site.

**4. Write the virtual host file.**

bash
sudo nano /etc/apache2/sites-available/example.com.conf


apache
<VirtualHost *:80>
    ServerAdmin admin@example.com
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>


The file name must end in `.conf`. Ubuntu's Apache configuration only loads files with that extension from `sites-available`, so `example.com` silently does nothing. `<VirtualHost *:80>` catches every request arriving on port 80, the default HTTP port.

**5. Enable the new site and disable the default one.**

bash
sudo a2ensite example.com.conf
sudo a2dissite 000-default.conf


The command prints a reminder to reload Apache. Hold off until the syntax check passes.

**6. Test the configuration.**

bash
sudo apache2ctl configtest


You want `Syntax OK`. A broken file here means Apache refuses to restart cleanly, so fix it before moving on.

**7. Restart and load the domain.**

bash
sudo systemctl restart apache2


Repeat steps 2 through 5 for each additional domain.

## Three details most tutorials leave out

**The symlink between `sites-available` and `sites-enabled`.** Files live in `/etc/apache2/sites-available/`, but Apache reads `/etc/apache2/sites-enabled/`. `a2ensite` creates a symbolic link from the second directory to the file in the first; `a2dissite` deletes that link. So editing the file in `sites-available` changes the live site, and deleting the link takes the site offline without touching the file. `000-default.conf` is the fallback host that answers when no configured name matches.

**Testing before DNS has propagated.** You can verify a virtual host without owning the domain. On your own computer, open the hosts file (`sudo nano /etc/hosts` on macOS and Linux; `notepad %windir%\system32\drivers\etc\hosts` on Windows with an administrator prompt) and add a line mapping the server IP to the domain:


203.0.113.10 example.com www.example.com


Your browser now resolves that domain to the server exactly as DNS would. The change applies to your machine only, so delete those lines once testing is done or you will confuse yourself for weeks.

**`ServerName` and `ServerAlias` are not interchangeable.**

| Directive | What it does | Example |
| --- | --- | --- |
| `ServerName` | The canonical hostname for the virtual host. Apache matches it exactly. | `example.com` |
| `ServerAlias` | Additional hostnames that should land on the same virtual host. Wildcards allowed. | `www.example.com`, `*.example.com` |

A request matching either directive is served by that virtual host. Put the bare domain in `ServerName` and the `www` variant in `ServerAlias`, not the other way around.

## Matching the plan to the number of sites

The rule of thumb is that disk and RAM run out before CPU does, because each site brings its own files, database and log files.

- **One site, low traffic, static or a light WordPress install:** the 2 vCPU / 2 GB plan with 40 GB of storage. Watch memory once you add a caching plugin and MySQL.
- **Two to five sites on one machine:** 4 vCPU / 4 GB / 80 GB. This is where most multi-site setups belong, and it leaves headroom for a staging copy.
- **Ten or more sites, or a WooCommerce store with a database on the same box:** 8 vCPU / 8 GB / 160 GB, or 12 vCPU / 12 GB if traffic is steady.
- **The 32 vCPU Advanced plan** only makes sense when a single site is CPU-hungry (image processing, heavy PHP, search indexing). Virtual hosting doesn't need more cores; it needs more RAM and disk per additional site.

Bandwidth is rarely the constraint: 4 TB of monthly transfer on a 1 Gbps port covers a lot of HTML, CSS and images. Video hosting or large downloadable files change that calculation quickly.

If you are provisioning today and want the middle of the range, the 4 vCPU configuration is the one to start from. 👉 [See the 4 vCPU / 4 GB plan and its current monthly price](https://bit.ly/Spinservers).

## How SpinServers lines up with three other Ubuntu VPS options

| Provider | Plan | vCPU / RAM / storage | Advertised price | Term and renewal |
| --- | --- | --- | --- | --- |
| SpinServers | Simple Application Server | 4 / 4 GB / 80 GB RAID-10 SSD | Featured configuration from $62.00/mo; per-plan price at checkout | Monthly |
| Hostinger | KVM 1 | 1 / 4 GB / 50 GB NVMe | $6.49/mo | 24 months prepaid, $155.76 upfront, renews at $11.99/mo |
| Hostinger | KVM 2 | 2 / 8 GB / 100 GB NVMe | $8.79/mo | 24 months prepaid, $210.96 upfront, renews at $14.99/mo |
| Liquid Web | General Ubuntu VPS | Not itemized on the page | $17/mo | Term and renewal price not stated |
| Vultr | Ubuntu instances | Regional configurations | Not listed on the Ubuntu page | Varies by region |

The Hostinger numbers are the useful comparison because both halves of the price are published. KVM 1 at $6.49/mo becomes $11.99/mo when the first 24-month term ends, and you pay $155.76 on day one. Across four years that is $155.76 plus two renewals at $287.76 each, or $443.52 total, roughly $9.24/mo averaged out. The advertised headline is real, but only for the first term.

Liquid Web's page shows $17/mo for its General Ubuntu VPS and $34/mo for a CPU-focused plan, but the page dates from December 2024 and states no billing term or renewal rate. Treat those figures as historical rather than current.

Vultr's Ubuntu page lists 33 data center regions and images for Ubuntu 26.04, 24.04 and 22.04 LTS, plus the option to upload your own ISO. It doesn't show prices there, so any comparison has to be built region by region.

SpinServers sits differently: monthly billing with no prepay lock-in and no renewal cliff, but the per-plan figure is only visible at checkout.

## Before you place the order

- **Check whether your sites need to send email.** Port 25 is closed on every VPS by default, and opening it requires a ticket plus approval. An external SMTP relay solves this without a support conversation.
- **Confirm the Ubuntu version.** LTS releases receive five years of support; interim releases get nine months and then stop. Take 24.04 LTS or 26.04 LTS, not an interim build. The Apache steps above are verified through 24.04 LTS, and the `sites-available` / `a2ensite` layout is the same family, but check the directory after provisioning rather than assuming.
- **Ask about refunds before paying.** A 2026 third-party review of SpinServers' dedicated line notes that month-to-month services generally carry no refund policy. VPS terms aren't stated on the plan page, so get an answer in writing if that matters to you.
- **Plan your own backups.** Nothing on the VPS page promises automated backups or snapshots.
- **Count your IP needs.** One IPv4 per plan means name-based virtual hosting and SNI, full stop.

Which configuration to choose comes down to one number: how many sites will share the machine. One site or two, tight budget, mostly static content: 2 vCPU / 2 GB. Three to five sites with databases, or anything running WooCommerce: 4 vCPU / 4 GB. More than ten sites, or mail and cron jobs alongside them: 8 vCPU or 12 vCPU. Anything above that is buying cores that virtual hosting won't use.

👉 [Start with the KVM VPS configuration that fits your site count](https://bit.ly/Spinservers).

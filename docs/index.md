# Tractorbeam


## Open source multi-tier website backups

Technology is fragile. When you think of sites going down, you usually think of big things like hacks, but entire systems of critical functionality could get lost if someone trips over a power plug. A thorough backup strategy is crucial to ensure that you can get your site back online quickly no matter what happens, whether the issue is site hacks, server problems, or zombie attacks.

We couldn’t find a backup solution that did everything we needed so we built our own: Tractorbeam, an open source, multi-tier, automated website backup tool that backs up your site code, database and site files.


*   **Tractorbeam does automated multi-tier backups daily, weekly and monthly, or any time schedule you choose.**  \
By default, Tractorbeam keeps seven daily backups, four weekly backups and twelve monthly backups. You could even configure the Docker container that runs Tractorbeam to run hourly backups. 
*   **Tractorbeam can write backups to any S3-compatible hosting provider, including Amazon AWS and Google Cloud.** \
Tractorbeam can write backups to any S3 or Rsync destination you choose, 
*   **Tractorbeam keeps a set number of backups per timescale.**  \
The value of a backup decreases the older it gets. If you have a website problem you’re likely going to discover it pretty quickly, so the most recent backups are what you need. Unlike other rolling backup solutions, Tractorbeam automatically prunes old backups before it uploads them to S3 locations to save space. 
*   **Tractorbeam works on most types of hosting.** \
    Tractorbeam needs to access databases (MySQL or MariaDB) and site files through SSH, which is available on most hosts. We also support [Pantheon](https://pantheon.io/) and [platform.sh](https://platform.sh/), because Tractorbeam can access files through the command-line tools from those providers.** **
*   **Tractorbeam is free and open source**, **so you have complete ownership of and access to your backups.** \
    You only have to pay for the server Tractorbeam runs on, and the hosting of the backups themselves. For example, at Digital Ocean, you could pay $5 for [a droplet](https://www.digitalocean.com/products/droplets/?refcode=5fb69d9c62e4) and $5 for [Spaces](https://www.digitalocean.com/products/spaces/?refcode=5fb69d9c62e4) (their S3 storage) and be done. 


## If your website host already has backups, why do you need Tractorbeam?

Because your host’s backups are likely in the same data center—if not the same server—that your site lives on. What if something happens to that server or that data center? Your site and your backups might be compromised. It always makes good business sense to have off-site backups in different geographical locations from your site host.


## How we use Tractorbeam for our clients

Tractorbeam forms a part of our core business offering and powers each of our client’s backups, both onsite and off-site. Some clients are hosted [in our Kubernetes cloud](https://ten7.com/podcast/episode/kubernetes-our-next-gen-site-hosting) at[ DigitalOcean](https://www.digitalocean.com/), while others are at[ Pantheon](https://pantheon.io/) or even[ platform.sh](https://platform.sh/). All clients have at least two geographically redundant backups at all times. 

Our DigitalOcean clients get local backups and two off-site backups in two different geographic locations (AWS S3 on the east coast and Google Cloud all over the US). 

Our clients at Pantheon or platform.sh already have their own local backups, but we provide three geographically redundant backups at DigitalOcean, AWS and Google Cloud. 

**Would you like us to manage your site backups (or more)?** [Let’s talk](https://ten7.com/contact-us).


# Built with love by TEN7

Tractorbeam is built and supported by[ TEN7](https://ten7.com/). We create and care for Drupal-powered websites.


*   Ready to use Tractorbeam? [Get it from Github](https://github.com/ten7/tractorbeam)
*   Listen to episode 93 on [The TEN7 Podcast](https://ten7.com/podcast) with [@socketwench](https://twitter.com/socketwench) titled [Tractorbeam: Our open source multi-tier backup solution](https://ten7.com/podcast/episode/tractorbeam-our-open-source-multi-tier-backup-solution)
*   Need some help? [Create a new Github issue](https://github.com/ten7/tractorbeam/issues/new) for us to review.
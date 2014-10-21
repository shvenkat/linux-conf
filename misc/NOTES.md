# Upgrade log
* Create VM
  * Allocate new LVM storage
  * Configure VM with unique MAC, compare/adjust config against old VM
* Copy storage
  * Create new filesystem using entire LVM volume, no partitioning
    * Compare/adjust filesystem parameters using dumpe2fs/tune2fs
  * Restore filesystem from last wiki system snapshot using rsync
    > sudo /usr/local/bin/rsync \
    >   -az --delete -HSxAX --numeric-ids \
    >   --rsync-path='/scratch/shvenkat/opt/rsync-3.1.0dev-7cefbf1/bin/rsync --fake-super' \
    >   --exclude='lost+found/' \
    >   --log-file=/tmp/clone-wiki.$(date '+%F-%H%M').log \
    >   --rsh='ssh -o IdentityFile=/root/.ssh/id_rsa_backup -o PreferredAuthentications=publickey' \
    >   --protect-args \
    >   shvenkat@burns.amgen.com:/scratch/shvenkat/private/wiki-kvm-fedora16/wikifs/ \
    >   /mnt/
    > cd /mnt; sudo mklost+found
  * Create required ephemeral files
    > sudo dd if=/dev/zero of=/mnt/var/lib/varnish/varnish_storage.bin bs=1G count=1
    > sudo touch /var/log/mysqld.log; sudo chown <mysql>:<mysql> /var/log/mysqld.log
  * Install boot loader
    > sudo extlinux --install /mnt/boot/extlinux
* De-duplicate settings as needed
  * Update boot info in fstab, extlinux.conf
  * Update network config to change MAC, static IP and hostname
    * /etc/sysconfig/network-scripts/ifcfg-Wired_connection_1,
      /etc/sysconfig/network,
      /etc/sysconfig/networking/profiles/default/network
    * /etc/NetworkManager/NetworkManager.conf
    * /etc/varnish/mediawiki.vcl, /etc/sysconfig/varnish
    * /etc/nginx/conf.d/{default,ssl}.conf
    * /var/web/mediawiki/LocalSettings.d/LocalSettings-core.php
  * Update ssh host key
  * Update crontab
  * Update initramfs using dracut, check against old initramfs
    > dracut --add-drivers 'virtio_blk virtio_net virtio_balloon ...' \
    >   --force --verbose <initramfs.img> <kernel_ver>
* Boot VM, ensure all systems running on boot, check wiki
* Stop/disable wiki services: varnish, varnishlog, nginx, php-fpm, mysqld, rc-local, crontab
* Update OS using yum update
* Upgrade OS from Fedora 16 to 17 using the Fedora 17 install DVD, update to latest
  * Update initramfs to include virtio modules, update extlinux.conf, reboot
  * Boot log error for script tcsd
  * Install updates
    > yum list installed | grep i686 | cut -d' ' -f1 | xargs sudo yum remove
    > sudo yum upgrade
  * Update initramfs, extlinux.conf
  * Reboot, remove old kernel, initramfs
* Upgrade OS from Fedora 17 to 19 using FedUp
  > sudo yum --enablerepo=updates-testing install fedup
  > sudo fedup-cli --network 19
  * Check /var/log/fedup.log for errors
  * Add fedup boot entry to extlinux.conf
    * Did not find the correct one in grub.cfg, despite running grub2-mkconfig
    * Found correct options at https://bugzilla.redhat.com/show_bug.cgi?id=881764
    >   MENU LABEL System Upgrade (Fedora 17 -> 19)
    >   SAY Booting Fedup
    >   LINUX /boot/vmlinuz-fedup
    >   APPEND root=UUID=ec2a155c-e71c-45a7-95ac-d24fa17961c5 systemd.unit=system-upgrade.target rd.upgrade.debugshell ro
    >   INITRD /boot/initramfs-fedup.img
  * Reboot, select System Upgrade boot option
    * Hit escape key to switch from graphical to text screen during upgrade
    * Switch to VT2 for a login shell, debug issues and install extlinux/update extlinux.conf when complete
* Update locally installed software: initramfs, /usr/local (rsync, sphinx, nginx, pwauth), mysql tables
  > sudo mysql_upgrade
* Check package config files in /etc, merge/remove .rpmsave/.rpmnew versions
* Merge updates from old wiki since cloning
  * On old wiki:
    > cd /var/web/mediawiki/maintenance
    > sudo -u nginx php /var/web/mediawiki/maintenance/dumpBackup.php \
        --revrange --revstart=3052 --revend=3060 --uploads --conf \
        /var/web/mediawiki/LocalSettings.php < /dev/null 2>/dev/null > \
        /tmp/wiki_revid-3052-3059.xml
    > sudo rsync -aczi --delete -HSx --exclude="/cache/" --exclude="/tmp/" \
        --exclude="/lockdir/" --delete-excluded /var/web/mediawiki/images \
        shvenkat@tengu.amgen.com:/scratch/shvenkat/tmp/
  * On new wiki:
    > sudo -u nginx rsync -n -aczi --delete -HSx --exclude / \
        --exclude /README --exclude /.htaccess \
        --exclude="/cache/" --exclude="/tmp/" --exclude="/lockdir/" \
        shvenkat@tengu.amgen.com:/scratch/shvenkat/tmp/images/ \
        /var/web/mediawiki/images/
    > cd /var/web/mediawiki/maintenance
    > sudo -u nginx php /var/web/mediawiki/maintenance/importDump.php --conf \
        /var/web/mediawiki/LocalSettings.php --uploads --image-base-path \
        /var/web/mediawiki/images --debug /tmp/wiki_revid-3052-3059.xml
    > sudo php rebuildrecentchanges.php # or rebuildall.php

# TODO
  * Mediawiki $wgSquidServers = array('127.0.0.1', 'brows.amgen.com'); ?
    Varnish acl_purge { "localhost"; }
  * Cache more content? Parser cache and all other caches
  * Turn off Parser profiling (output as HTML comments and on edit pages visible footer)
  * Change beta message link to manual
  * Troubleshooting page
  * Renew SSL certif, get it signed by Amgen CA
  * Fix errors in nginx/error.log from PHP FastCGI
  * Remove non-functional button under advanced editor menu
  * Change GAU wiki to wiki
  * Turn off debugging/profiling in LocalSettings.php

# Refs:
  http://www.mediawiki.org/wiki/MediaWiki_1.22
  http://www.mediawiki.org/wiki/Release_notes/1.22
  http://www.mediawiki.org/wiki/Manual:Upgrading
  http://www.mediawiki.org/wiki/Extension:WikiEditor/Toolbar_customization
  http://www.mediawiki.org/wiki/Thread:Project:Support_desk/How_do_I_format_search_results_/_add_section_links_to_search_results_(the_same_way_Wikipedia_and_other_WikiMedia_sites_do)%3F

# backup_system_online

J'ai lu les préconisations du NAS OMV (https://openmediavault.readthedocs.io/en/5.x/new_user_guide/newuserguide.html#a-last-important-note-about-backing-up-your-os)
"Can it be assumed that those same software repositories and resources will be available on some future date, exactly as they were at the time of a current build? The answer is “No”. Distributions of a specific Linux version, complete with specific applications, fully patched and updated, can be built for a limited time."

Je me suis dit que ce ne serait pas pratique d'avoir à faire régulièrement chaque sauvegarde manuelle du système, puisqu'il faudrait la faire "hors ligne" et penser à la faire. Aussi je me suis demandé, s'il n'existait pas une solution pour cloner un système "en ligne" sous Linux (d'autant que je connais VMWare Convertor qui le fait sous Windows - https://wiki.maxcorp.org/realisation-dun-p2v-avec-vmware-vcenter-converter-standalone/) et je suis tombé sur cette procédure (https://stackoverflow.com/questions/37488629/how-to-use-dump-and-restore-to-clone-a-linux-os-drive). Je l'ai testé en l'état et apporté mes modifications car je n'arrivais pas à le faire fonctionner. De plus j'ai lu plusieurs posts qui dénoncent l'impossibilité de cloner un système en ligne sous Linux. Tout ce serait arrêté là, si mon travail n'avait pas finalement porté ses fruits... 

J'ai réussi à cloner un disque système en ligne sous Linux et booté sur le disque cloné pour avoir un système (qui semble) pleinement opérationnel :D

Pour le moment, ces tests, je les ai réalisé en environnement virtuel sous KVM/QEmu.

Voici le résultat :
```sh
# Solution qui marche
# Pour que GRUB ne fasse pas le lien avec SRC, on rend os-prober INexécutable 
#  (https://unix.stackexchange.com/questions/634150/hide-devices-in-chrooted-environment/634655#634655)
# Une fois chrooté un script intégré est exécuté et affiché sur stdout ;
#  il FAUT échapper chaque $ et \ (=> \$ et \\) pour qu'il soient appliqués à l'exécution après le chroot

apt -y install dump parted gawk acl
SRC=/dev/sda TGT=/dev/sdb
rm /part_SRC*
parted ${TGT} mklabel -s "msdos"
sfdisk -d ${SRC} > part_table
sfdisk --force ${TGT} < part_table
mkfs.ext4 -F ${TGT}1
mkswap -f ${TGT}5
mkdir -p /mnt${TGT}1
mount -t ext4 ${TGT}1 /mnt${TGT}1
 cd /mnt${TGT}1 ; dump -a0f - ${SRC}1 | restore -rf - ; cd
 echo ok
 # touch /part_SRC_$( blkid -s UUID |grep \
 #    $( df -h | grep ${SRC} | cut -f 1 -d " " ) | cut -f 2 -d '"' )
 # touch /mnt${TGT}1/part_TGT_$( blkid -s UUID | grep \
 #    $( df -h | grep ${TGT} | cut -f 1 -d " " ) | cut -f 2 -d '"' )
 for i in /dev /proc /sys /run /sys ; do \
    mount -B $i /mnt${TGT}1$i; done
  cat << EOF | o_blkid=$( blkid ) SRC=$SRC TGT=$TGT \
               chroot /mnt${TGT}1
   as=( \$( echo "\$o_blkid" | grep \${SRC} | sort | tr -d ' ' ) )
   ad=( \$( echo "\$o_blkid" | grep \${TGT} | sort | tr -d ' ' ) )
      for i in \${!as[@]} ; do sed -i "s/"\$( echo \${as[\$i]} \
      | cut -d '"' -f 2 )"/"\$( echo \${ad[\$i]} \
      | cut -d '"' -f 2 )"/g" /etc/fstab ; done
   echo RESUME=UUID=\$( blkid |grep -E "\${TGT}.+swap" \
      | cut -d '"' -f 2 ) > /etc/initramfs-tools/conf.d/resume
   os_prober_path=\$( which os-prober ) \
      && perms=\$( getfacl -e \$os_prober_path ) \
      && chmod a-x \$os_prober_path
    grub-install \${TGT}
    update-grub
   [[ \$os_prober_path ]] && echo "\$perms" \
      | setfacl -M- \$os_prober_path
#   gawk '/^(menuentry|submenu)/ { printf '\
#'     gensub( /[^\\x27]+\\x27([^\\x27]+)\\x27.+/ , "\\\\1" , "g" ) ; '\
#'     uuid = gensub( '\
#'     /.+'\
#'     ([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})'\
#'     .*/ '\
#'     , "\\\\1" , "g" ) ; '\
#'     if ( uuid != \$0 ) { printf " [" uuid "]" } printf "\\n" }' \
#      /boot/grub/grub.cfg
EOF
 for i in /dev /proc /sys /run /sys ; do umount -l /mnt${TGT}1$i; done
umount -l /mnt${TGT}1
rmdir /mnt${TGT}1
```
En réalité, j'ai tout de même vu quelques différences qui ralentissent (un peu) le démarrage et qui me font doûter de l'exactitude fonctionnelle du système cloné.

Pour un clone du disque système de OMV, au démarrage j'ai un message "mdadm: no arrays found in config file or automatically" répété des dizaines de fois, qui ralentit le démarrage et que j'ai essayé de résoudre à partir de ce lien (https://forum.openmediavault.org/index.php?thread/21351-mdadm-no-arrays-found-in-config-file-or-automatically/&pageNo=6) et d'autres sans succès. Aussi, j'ai remarqué que lorsque j'attache à nouveau le disque source à l'hôte et que je boote sur le clone, le problème est résolu. J'en ai déduit que le clone a encore des "trucs" qui sont liés au disque source. D'après mes connaissances, il y a deux choses qui permettent de référer à un disque : son nom de device (/dev/sda par ex) qui n'est pas garanti être permanent et les UUID qui eux sont uniques

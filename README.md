# WB-3 - CentOS
## Ejercicio 1

Decargar e instalar CentOS en virtualbox
http://ftp.rz.uni-frankfurt.de/pub/mirrors/centos/8.3.2011/isos/x86_64/CentOS-8.3.2011-x86_64-minimal.iso
Como he escogido la red NAT hay que crear un port forwarding en virtualbox para poder conectar a host desde ssh.
Instalación mínima.
Crear usuario root y esanz y passs enrique
Quitar DHCP
Particiones todas en ext4 
- /boot 1GB
- / 5GB 
- /home 2GB 
- Resto a /var


Cuando arranque, instalar esto, copiar la key y 

`yum -y install epel-release`

Vamos a crear un grupo de discos virtuales que actuan como uno solo.
> Pregunta 1 : apaga la maquina virtual, añade otro disco aparte de 5GB. A continuación enciende la VM, añade ese disco al VG y amplia home con +3GB. Cuantos GB quedan libres?
```
	fdisk /dev/sdb
	pvcreate /dev/sdb1
	vgdisplay
	vgextend cl /dev/sdb1
	lvdisplay
	pvscan
	lvextend -L +3G /dev/cl/home
	resize2fs /dev/mapper/cl-home
```
El sistema de ficheros en /dev/mapper/cl-home tiene ahora 1274880 bloques (de 4k).
```
  PV /dev/sda2   VG cl              lvm2 [<9,07 GiB / 0    free]
  PV /dev/sda3   VG cl              lvm2 [<10,00 GiB / 5,00 GiB free]
  PV /dev/sdb1   VG cl              lvm2 [<5,00 GiB / <5,00 GiB free]
  Total: 3 [<24,06 GiB] / in use: 3 [<24,06 GiB] / in no VG: 0 [0   ]
```
> Pregunta 2 : Busca en google como habilitar el trafico en el puerto 80 en el cortafuegos de CentOS.
```
	firewall-cmd --add-service=http --permanent
	firewall-cmd --reload
```
Y ahora debe aparecer el servicio http

`firewall-cmd --list-all`
	
> Pregunta 3 : encuentra una forma de filtrar el servicio cockpit por ip, por seguridad
  
`sudo firewall-cmd --permanent --zone=public --add-service=cockpit --add-source=<ip-address>/32 && sudo firewall-cmd --reload`

## SELINUX
SELINUX o Security Enchanced Linux añade una capa extra de seguridad en nuestro servidor. Por defecto viene instalado en todos los CentOS configurado en enforcing pero la mayoria de gente a la que tiene un problema lo desactiva.

Es mejor hacer una regla para ello, no desactivar todas las protecciones. En cockpit podremos ver las alarmas SELINUX.

Más info en -> https://es.wikipedia.org/wiki/SELinux

Hagamos una prueba
```
yum remove httpd
yum install nginx
systemctl enable --now nginx
```

Abre de nuevo un navegador y visita la ip de la máquina. Si ha funcionado deberemos ver el welcome page de nginx.

Ahora edita el archivo `/etc/nginx/nginx.conf` y en el scope de server, modifica el location /:
```
        location / {
          proxy_pass http://www.wordpress.com/;
          proxy_redirect off;
          proxy_buffering off;
          proxy_http_version 1.1;
          proxy_set_header Connection "Keep-Alive";
          proxy_set_header Proxy-Connection "Keep-Alive";
        }
```

Ahora probamos que la config está OK y reiniciamos:
```
nginx -t
service nginx reload
```
Prueba de nuevo a visitar la página web. Que ha ocurrido?

Abrimos el log de error de nginx y probamos a visitar de nuevo la página. Para más cómodidad podemos usar por ejemplo curl o wget para ayudarnos en el debug.
```
tail -f /var/log/nginx/error.log
wget -O/dev/null http://172.16.73.180/

```
Si estamos en un sistema con SELINUX y nos da un problema de permisos, es habitual que SELINUX esté bloqueando algo.
 
> Pregunta 4 : Parece que la capa de protección extra SELINUX está haciendo de las suyas. Abre el log con un tail -f para observar los cambios. Recarga la página en el navegador y encuentra el mensaje de error. A continuación busca una forma de solucionar el problema.

`tail -f /var/log/audit/audit.log /var/log/nginx/error.log`

## Filtro IP con firewalld
> Pregunta 5 : Busca una forma de proteger adecuadamente el acceso con ssh, permitiendo sólo el tráfico desde nuestra IP.



## Ejercicio 2

Raid 1 en servidor remoto ya instalado
Hoy vamos a aprender como hacer un raid 1 en un servidor remoto que fué ya instalado.
Aprovisiona un servidor cx11 con CentOS 8 y con un disco extra 20GB.
Una vez dentro, instala el software necesario para gestionar software raids en linux

`yum install mdadm`

Observemos ahora el disco duro con `fdisk -l`:
	
	Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
	Units: sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	
	
	Disk /dev/sda: 19.1 GiB, 20480786432 bytes, 40001536 sectors
	Units: sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disklabel type: gpt
	Disk identifier: 5484F84F-0903-4767-8833-EB0A3197CEBA
	
	Device      Start      End  Sectors Size Type
	/dev/sda1  135168 40001502 39866335  19G Linux filesystem
	/dev/sda14   2048     4095     2048   1M BIOS boot
	/dev/sda15   4096   135167   131072  64M EFI System
	
	Partition table entries are not in disk order.
  
Como podemos ver tenemos un disco duro sin particiones (sdb) y otro con un centos funcionando, sda.
Primero clonaremos las tabla de particiones de *sda* a *sdb* para dejarlo identico con el comando `sfdisk -d /dev/sda | sfdisk /dev/sdb`
Si observamos de nuevo con `fdisk -l` veremos que són iguales. Como vamos a montar un raid 1, primero cambiaremmos los tipos de todas las particiones de sdb a raid. Para ello usaremos el comando `gdisk /dev/sdb`:
	
	[root@centos02-master ~]# gdisk /dev/sdb
	GPT fdisk (gdisk) version 1.0.3
	
	Partition table scan:
	  MBR: protective
	  BSD: not present
	  APM: not present
	  GPT: present
	
	Found valid GPT with protective MBR; using GPT.
	
	Command (? for help): t
	Partition number (1-15): 1
	Current type is 'Linux RAID'
	Hex code or GUID (L to show codes, Enter = 8300): fd00
	Changed type of partition to 'Linux RAID'
	
	Command (? for help): t
	Partition number (1-15): 14
	Current type is 'Linux RAID'
	Hex code or GUID (L to show codes, Enter = 8300): fd00
	Changed type of partition to 'Linux RAID'

	Command (? for help): t
	Partition number (1-15): 15
	Current type is 'Linux RAID'
	Hex code or GUID (L to show codes, Enter = 8300): fd00
	Changed type of partition to 'Linux RAID'
	
	Command (? for help): w
	
	Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
	PARTITIONS!!
	
	Do you want to proceed? (Y/N): y
	OK; writing new GUID partition table (GPT) to /dev/sdb.
	The operation has completed successfully.
Ahora comprueba que el disco nuevo es todo tipo linux raid con `fdisk -l` :
	
	[root@centos02-master ~]# fdisk -l /dev/sdb
	Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
	Units: sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disklabel type: gpt
	Disk identifier: 5484F84F-0903-4767-8833-EB0A3197CEBA
	
	Device      Start      End  Sectors Size Type
	/dev/sdb1  135168 40001502 39866335  19G Linux RAID
	/dev/sdb14   2048     4095     2048   1M Linux RAID
	/dev/sdb15   4096   135167   131072  64M Linux RAID
	
	Partition table entries are not in disk order.
  
Ahora ya podemos inicializar nuestro nuevo raid, pero con solo un disco en cada dispositivo:
	
	
	mdadm --create /dev/md1 --level=1 --raid-devices=2 missing /dev/sdb1
	mdadm --create /dev/md15 --level=1 --raid-devices=2 missing /dev/sdb15
  
Comprobamos que se han arrancado los 2 dispositivos:
	
	[root@centos02-master ~]# cat /proc/mdstat 
	Personalities : [raid1] 
	md15 : active raid1 sdb15[1]
	      64512 blocks super 1.2 [2/1] [_U]
	      
	md1 : active raid1 sdb1[1]
	      19915712 blocks super 1.2 [2/1] [_U]
	      
	unused devices: <none>
  
Formateamos los nuevos dispositivos raid igual que el actual sistema:
	
	mkfs.ext4 /dev/md1
	mkfs.vfat /dev/md15
  
Montaremos el nuevo raid en un mountpoint temporal para así copiar todos los archivos del sistema.
	
	mount /dev/md1 /mnt/
	mkdir -p /mnt/boot/efi
	mount /dev/md15 /mnt/boot/efi/
  
Copiamos los archivos ...
	
	rsync -auxHAXSv --exclude=/dev/* --exclude=/proc/* --exclude=/sys/* --exclude=/tmp/* --exclude=/mnt/* /* /mnt
  
Dará unos errores de copiar /run pero los podemoos ignorar. También podriamos desactivar SELINUX si fuera un servidor en producción para quedarnos más tranquilos y entonces no daría el error.
Mountamos los dispositivos al nuevo mountpoint y luego hacemos chroot:
	
	mount --bind /proc /mnt/proc
	mount --bind /dev /mnt/dev
	mount --bind /sys /mnt/sys
	mount --bind /run /mnt/run
	chroot /mnt/
  
Miramos el UUID de los nuevos discos y lo apuntamos.
	
	blkid /dev/md1*
	
	/dev/md1: UUID="22f749e1-77a8-4ae3-8a25-86b57031e21b" BLOCK_SIZE="4096" TYPE="ext4"
	/dev/md15: SEC_TYPE="msdos" UUID="1958-DBE7" BLOCK_SIZE="512" TYPE="vfat"
	
Ahora editamos el archivo /etc/fstab y ponemos nuestros UUID nuevos:
	
	#
	# /etc/fstab
	# Created by anaconda on Sun Sep 27 04:34:49 2020
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
	#
	# After editing this file, run 'systemctl daemon-reload' to update systemd
	# units generated from this file.
	#
	UUID=af98a52e-8204-407b-a43e-f3ecf8894371 /                       ext4    defaults        1 1
	UUID=45BC-30AC          /boot/efi               vfat    defaults,uid=0,gid=0,umask=077,shortname=winnt 0 2
  
Escaneamos los dispositivos raid y los añadimos a la configuración del sistema:
	
	mdadm --detail --scan > /etc/mdadm.conf
  
Hacemos backkup del initram actual y generamos uno nuevo con los drivers de raid1:
	
	cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bck
	dracut --mdadmconf --fstab --add="mdraid" --filesystems "xfs ext4 ext3" --add-drivers="raid1" --force
  
Editamos la configuración de arranque de grub /etc/default/grub:
	
	GRUB_TIMEOUT=1
	GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
	GRUB_DEFAULT=saved
	GRUB_DISABLE_SUBMENU=true
	GRUB_TERMINAL_OUTPUT="console"
	GRUB_CMDLINE_LINUX="consoleblank=0 rd.auto rd.auto=1 systemd.show_status=true elevator=noop no_timer_check console=tty1 console=ttyS0,115200n8"
	GRUB_DISABLE_RECOVERY="true"
	GRUB_ENABLE_BLSCFG=true
	GRUB_PRELOAD_MODULES="mdraid1x"
  
Generamos la nueva configuración y la instalamos en ambos dispositivos:
	
	grub2-mkconfig -o /boot/grub2/grub.cfg
	grub2-install /dev/sda
	grub2-install /dev/sdb
  
Salimos del chroot y reiniciamos
	
	exit
	reboot
  
Entramos y comprobamos que estamos en el raid:
	
	[root@centos02-master ~]# df -h
	Filesystem      Size  Used Avail Use% Mounted on
	devtmpfs        955M     0  955M   0% /dev
	tmpfs           970M     0  970M   0% /dev/shm
	tmpfs           970M  8.5M  962M   1% /run
	tmpfs           970M     0  970M   0% /sys/fs/cgroup
	/dev/md1         19G  1.1G   17G   7% /
	/dev/md15        63M     0   63M   0% /boot/efi
	tmpfs           194M     0  194M   0% /run/user/0
  
Una vez dentro del raiz si ejecutamos el siguiente comando, vemos que únicamente tenemos 1 de los 2 discos activos del raid:
	
	cat /proc/mdstat
  
Entonces el siguiente paso es añadir el sda a nuestro raid. Para ello debemos cambiar el tipo de las particiones a fd00.
	
	echo -e "t\n1\nfd00\nt\n14\nfd00\nt\n15\nfd00\nw\nY\n" | gdisk /dev/sda
  
Añade las particiones al raid:
	
	mdadm --manage /dev/md1 --add /dev/sda1
	mdadm --manage /dev/md15 --add /dev/sda15
  
Una vez añadidas, comprobamos que se está realizando el recovery correctamente:
	
	[root@centos02-master ~]# cat /proc/mdstat 
	Personalities : [raid1] 
	md1 : active raid1 sda1[2] sdb1[1]
	      19915712 blocks super 1.2 [2/1] [_U]
	      [===>.................]  recovery = 16.5% (3305728/19915712) finish=1.3min speed=206608K/sec
	      
	md15 : active raid1 sda15[2] sdb15[1]
	      64512 blocks super 1.2 [2/1] [_U]
	        resync=DELAYED
	      
	unused devices: <none>
 
Finalmente, cuando termine el rebuild del raid, hacemos un reboot y comprobamos nuevamente el estado del raid.


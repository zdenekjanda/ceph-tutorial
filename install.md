# Instalace Ceph

## Hardware

### Cpu

Doporučujeme použít systém s jedním procesorem, odpadá konfigurace NUMA zón. Čím vyšší takt na jádro, tím lépe, klesá latence. Moderní single socket AMD systémy s 32-64 jader jsou nejlepší volba

### Paměť

Doporučujeme použít alespoň 4GB paměti na jedno OSD, 8GB optimálně. Komponenty MON a MGR si na menších clusterech vystačí s 32GB, to samé platí o MDS. Pokud uvažujeme o all in one systému, který má například 4 NVME disky, každý 4 OSD, a komponenty MON, MGR a MDS, nejméně 128GB, optimálně 192GB paměti

### Síť

Minimální doporučená rychlost sítě je 10Gbps. Pro NVME ceph je doporučena rychlejší síť 25/100Gbps, opět se sníží latence a zvýší výkon v IOPS. Doporučujeme použít zapojení sítě každého node Cephu v bond režimu dvěma kabely ke dvojici MLAG switchů, aby bylo dosaženo vysoké dostupnosti i při selhání switche

### Disky

#### HDD

Doporučujeme použít disky SAS, které mají konstantní rychlost otáček aspoň 7200 RPM v serverovém provedení pro 24/7 provoz

#### SSD/NVME

Doporučujeme použít disky SSD, které podporují o_direct a fsync v přiměřené rychlosti. Jedná se většinou o serverové SSD, vybavené kondenzátory. 

Standartní benchmark, který určí limitující rychlost SSD

    fio --name=/dev/sdX --ioengine=libaio --direct=1 --fsync=1 --readwrite=randwr --blocksize=4k --runtime=300

### Odkazy

Ceph manuál o hardware - https://docs.ceph.com/en/quincy/start/hardware-recommendations/

## Software

### Operační systém

Doporučujeme Ubuntu v aktuální verzi LTS, například 22.04 LTS

### Kernel

#. Na Ubuntu je vhodné použít HWE kernel, obsahuje novější verze ovladačů a bezpečnostní patche

    sudo apt install linux-image-generic-hwe-22.04

#. Pro optimální výkon a propustnost doporučujeme soubor kernelových nastavení v ``/etc/sysctl.conf``. Nastavení je třeba projít a zkontrolovat, že nebudou v konfliktu s ostatními službami, pokud jde o sdílený systém.

    fs.aio-max-nr = 1048576
    fs.file-max = 26234859
    kernel.core_uses_pid = 1
    kernel.msgmax = 65536
    kernel.msgmnb = 65536
    kernel.msgmni = 32000
    kernel.pid_max = 4194303
    kernel.sem = 32000 1024000000 500 32000
    kernel.shmall = 18446744073692774399
    kernel.shmmax = 18446744073692774399
    kernel.shmmni = 4096
    kernel.sysrq = 0
    net.bridge.bridge-nf-call-arptables = 0
    net.bridge.bridge-nf-call-ip6tables = 0
    net.bridge.bridge-nf-call-iptables = 0
    net.core.netdev_max_backlog = 65536
    net.core.rmem_max = 268435456
    net.core.somaxconn = 40000
    net.core.wmem_max = 268435456
    net.ipv4.conf.all.accept_redirects = 0
    net.ipv4.conf.all.accept_source_route = 0
    net.ipv4.conf.all.rp_filter = 0
    net.ipv4.conf.all.send_redirects = 0
    net.ipv4.conf.default.accept_source_route = 0
    net.ipv4.conf.default.rp_filter = 0
    net.ipv4.ip_forward = 0
    net.ipv4.ip_local_port_range = 10000 65000
    net.netfilter.nf_conntrack_max = 2621440
    net.netfilter.nf_conntrack_tcp_timeout_established = 1800
    vm.min_free_kbytes = 128000
    vm.swappiness = 0
    vm.zone_reclaim_mode = 0

#. Vypnutí powersave

Pro maximalizaci IOPS je nutné nastavit kernelový ovladač `Powersave` na maximální výkon procesoru, aby frekvence běhu všech jader fungovala vždy na plnou frekvenci a nedocházelo k poklesu IOPS způsobeném větší latencí.

# TBD

### Verze Ceph

Je doporučeno použít aktuální stable verzi. Balíčky pro Ubuntu jsou k dispozici v repository apt

#. Stáhnutí ``cephadm`` skriptu

        curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
        chmod +x cephadm

#. Přidání repozitáře

       ./cephadm add-repo --release quincy

#. Nainstalování balíčků

        ./cephadm install ceph-common


### Nastavení sítě

Doporučujeme ve všech sítích použít konfiguraci bond se dvojicí kabelů proti dvou switchům, a jednotlivé Ceph sítě oddělit pomocí VLAN. V tomto režimu budou odděleny ceph a klientská síť a obě zároveň mohou využívat agregovanou kapacitu bond adaptéru. Na všech sítích doporučujeme MTU 9000, když je ale konflikt se zbytkem infrastruktury tak je možné použít MTU 1500, rozdíl výkonu jsou asi 3%

#### Konfigurace sítě na Ubuntu

#. Doporučujeme nainstalovat klasické balíky pro management sítě, následující příklady jsou pro ``/etc/network/interfaces`` a ``ifupdown``

        sudo apt install ifupdown net-tools


#. Automatická konfigurace adaptérů v ``/etc/network/interfaces``

        auto bond0 bond0.1070 bond0.820 bond0.821 lo p1p1 p1p2
        allow-hotplug bond0 bond0.1070 bond0.820 bond0.821 lo p1p1 p1p2

#. Nastavení hw adaptérů, je vhodné použít nastavení ethtool offloading dle doporučení driverů konkrétního adaptéru

        iface p1p1 inet manual
            mtu 9000
            bond-master bond0
            post-up ethtool -G p1p1 rx 8192

        iface p1p2 inet manual
            mtu 9000
            bond-master bond0
            post-up ethtool -G p1p2 rx 8192

#. Nastavení bond adaptéru

        iface bond0 inet manual
            mtu 9000
            bond-mode 802.3ad
            bond-lacp-rate 1
            bond-min-links 1
            bond-miimon 100
            bond-xmit-hash-policy layer3+4
            bond-slaves p1p1 p1p2

#. Nastavení management vlan

        iface bond0.1070 inet static
            vlan-raw-device bond0
            address 10.14.0.71
            netmask 255.255.255.0
            gateway 10.14.0.1
            dns-nameservers 185.120.68.12 185.120.68.13

#. Nastavení oddělěných vlan pro ceph a klient sítě

        iface bond0.820 inet static
            vlan-raw-device bond0
            address 172.27.102.11
            netmask 255.255.252.0

        iface bond0.821 inet static
            vlan-raw-device bond0
            address 172.26.122.11
            netmask 255.255.255.0

#. Nastavení loopback adaptéru

        iface lo inet loopback

### Vypnutí Iptables

Pro omezení latence je vhodné úplně vypnout funkce iptables. Pakety IP tak nebudou procházet přes řetězce iptables, kromě snížené latence ušetříme jak CPU čas. Vypnutí iptables v Ubuntu 22.04 LTS provedeme pomocí vypnutí kernelových modulů pro funkci iptables.

# TBD

## Alternativy instalace

### Cephadm a Rook

Oficiální Ceph dokumentace doporučuje instalaci Ceph v kontejnerech pomocí Cephadm nebo Rook v Kubernetes. Ačkoliv toto technologické řešení vypadá lákavě, nedoporučujeme jej, protože přidává další vrstvy do softwarováho stacku, samotné kontejnery a iptables jednak zvýší latenci a druhak ztěžují debugging a kontrolu nad jednotlivými komponenty. V bare metal nasazení je proto nemůžeme doporučit

### Puppet Ceph

Puppet ceph používáme a doporučujeme, pokud na stávající infrastruktuře již Puppet existuje. Puppet modul doporučujeme používat na instalace balíků, konfiguraci komponent a management klíčů. Naopak nedoporučujeme jakékoliv operace s OSD, chybná konfigurace by mohla například zapříčinit neočekávou rekonfiguraci disků, kterou je vždy lepší dělat na produkčních systémech manuálně

Puppet modul: https://github.com/openstack/puppet-ceph

Pro jednotlivé role clusteru se volají jednotlivé puppet role a parametry nastavují v hiera

    include ceph
    include ceph::mon
    include ceph::osd

#. Ukázková konfigurace parametrů hiera pro klienty rbd a cluster v třídě ``ceph``

        "ceph::fsid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "ceph::osd_pool_default_size": 2,
        "ceph::mon_initial_members": "mon1,mon2,mon3",
        "ceph::mon_host": "172.27.102.1,172.27.102.2,172.27.102.3",
        "ceph::public_network": "172.27.100.0/22",
        "ceph::cluster_network": "172.26.122.0/24",
        "ceph::base_admin_key": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==",
        "ceph::base_release": "quincy",
        "ceph::rbd_cache": false,
        "ceph::rbd_cache_writethrough_until_flush": false,

#. Ukázková konfigurace parametrů hiera pro ``mon`` host v třídě ``ceph::mon``

        "ceph::mon_osd_nearfull_ratio": 0.9,
        "ceph::mon_osd_backfillfull_ratio": 0.93,
        "ceph::mon_osd_full_ratio": 0.96,
        "ceph::mon_sync_max_payload_size": 4096,
        "ceph::base_mon": true,
        "ceph::base_mon_key": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==
        "ceph::base_mgr_key": ; "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==",
        "ceph::base_osd_key": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==",
        "ceph::mon_max_pg_per_osd": 5000,
        "ceph::osd_max_pg_per_osd_hard_ratio": 10,
        "ceph::auth_insecure_global_id_reclaim": false

#. Ukázková konfigurace parametrů hiera pro ``osd`` host v třídě ``ceph::osd``

        "ceph::mon_osd_nearfull_ratio": 0.9,
        "ceph::mon_osd_backfillfull_ratio": 0.93,
        "ceph::mon_osd_full_ratio": 0.96,
        "ceph::osd_memory_target": true,
        "ceph::osd_memory_target::limit": 6442450944,
        "ceph::base_osd": true,
        "ceph::base_osd_key": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==",
        "ceph::config_debug::enabled": false,
        "ceph::mon_max_pg_per_osd": 5000,
        "ceph::osd_max_pg_per_osd_hard_ratio": 10,

### Ceph ansible

Ceph ansible doporučujeme, pokud na stávající infrastruktuře již Ansible existuje. Ansible modul doporučujeme používat na instalace balíků, konfiguraci komponent a management klíčů. Naopak nedoporučujeme jakékoliv operace s OSD, chybná konfigurace by mohla například zapříčinit neočekávou rekonfiguraci disků, kterou je vždy lepší dělat na produkčních systémech manuálně

Ansible modul: https://docs.ceph.com/ceph-ansible/

Příklady použití jsou uvedeny v dokumentaci modulu

## Manuální instalace

Manuální instalaci doporučujeme při nevyužití konfiguračního managementu, a kroky jako přidání nových OSD, které konfigurační managent doplňují. I při plném využití automatické instalace či nástrojů je vhodné si manuální instalaci projít, aby se správce seznámil se základy konfigurace nejnižší úrovně cephu a jeho komponent

## Instalace ``mon``

Monitor daemon ``mon`` potřebuje k provozu provést úvodní nastavení konfigurace, vytvoření klíčů a zavedení mapy. Konfigurační soubor potřebuje základní parametry ``fsid``, ``mon_initial_members`` a ``mon_host``

#. Generace fsid pomocí příkazu ``uuidgen``

      # uuidgen
      204274b9-b388-4b61-a825-3663c84d87b8

      fsid = 204274b9-b388-4b61-a825-3663c84d87b8

#. Parametr ``mon_initial_members`` nastavíme podle hostname všech monitorů

      # hostname
      mon1

      mon_initial_members = mon1,mon2,mon3

#. Parametr ``mon_host`` obsahuje ip adresy všech monitorů

      mon_host = 172.27.102.1,172.27.102.2,172.27.102.3

#. Vytvoření konfiguračního souboru ``/etc/ceph/ceph.conf``

      [global]
      fsid = 204274b9-b388-4b61-a825-3663c84d87b8
      osd_pool_default_pg_num = 64
      osd_pool_default_pgp_num = 64
      osd_pool_default_size = 2
      mon_osd_full_ratio = 0.96
      mon_osd_nearfull_ratio = 0.9
      mon_initial_members = mon1,mon2,mon3
      mon_host = 172.27.102.1,172.27.102.2,172.27.102.3
      cluster_network = 172.26.122.0/24
      public_network = 172.27.100.0/22
      auth_cluster_required = cephx
      auth_service_required = cephx
      auth_client_required = cephx
      auth_supported = cephx
      osd_pool_default_flag_hashpspool = true
      osd_op_threads = 6
      osd_op_num_threads_per_shard = 2
      osd_op_num_shards = 24
      filestore_fd_cache_size = 128
      filestore_fd_cache_shards = 32
      filestore_xattr_use_omap = true
      filestore_queue_max_ops = 100000
      filestore_queue_max_bytes = 1048576000
      ms_nocrc = true
      ms_dispatch_throttle_bytes = 0
      cephx_sign_messages = false
      cephx_require_signatures = false
      throttler_perf_counter = false
      mon_pg_warn_max_per_osd = 8192
      mon_compact_on_start = true
      mon_max_pg_per_osd = 5000
      osd_max_pg_per_osd_hard_ratio = 10
      mon_osd_backfillfull_ratio = 0.93
      auth_allow_insecure_global_id_reclaim = false
      auth_expose_insecure_global_id_reclaim = false

      [mon]
      mon_sync_max_payload_size = 4096


#. Vytvoření klíčenky pro cluster a vytvoření klíče pro monitory

	sudo ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'


#. Vytvoření klíčenky administrátora, uživatele  ``client.admin`` a přídání uživatele do klíčenky

	sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

#. Vytvoření klíčenky bootstrap-osd keyring, uživatele ``client.bootstrap-osd`` a přidání uživatele do klíčenky

	sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

#. Přidání vytvořench klíčů do ``ceph.mon.keyring``. ::

	sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
	sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring

#. Změna práv majitele souboru ``ceph.mon.keyring``. ::

	sudo chown ceph:ceph /tmp/ceph.mon.keyring

#. Vytvoření mapy monitorů pomocí hostname, ip adres monitorů a fsid do souboru ``/tmp/monmap``

      monmaptool --create --add mon1 172.27.102.1 --fsid 204274b9-b388-4b61-a825-3663c84d87b8 /tmp/monmap
      monmaptool --add mon2 172.27.102.2 --fsid 204274b9-b388-4b61-a825-3663c84d87b8 /tmp/monmap
      monmaptool --add mon3 172.27.102.3 --fsid 204274b9-b388-4b61-a825-3663c84d87b8 /tmp/monmap

#. Vytvoření adresáře pro monitor

	sudo -u ceph mkdir /var/lib/ceph/mon/ceph-mon1


#. Zavedení mapy a klíčenky do monitoru

	sudo -u ceph ceph-mon --mkfs -i mon1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring

#. Start monitoru pomocí systemd

	sudo systemctl start ceph-mon@mon1

#. Kontrola běhu monitoru

	sudo ceph -s

      cluster:
        id:     204274b9-b388-4b61-a825-3663c84d87b8
        health: HEALTH_OK

      services:
        mon: 1 daemons, quorum mon1
        mgr: mon1(active)
        osd: 0 osds: 0 up, 0 in

      data:
        pools:   0 pools, 0 pgs
        objects: 0 objects, 0 bytes
        usage:   0 kB used, 0 kB / 0 kB avail
        pgs:

#. Mapu a klíčenku zkopírujeme na ostatní monitory, vytvoříme adresáře, zavedeme mapu a klíčenku a spustíme daemony. Automaticky se připojí a pak je již uvidíme ve statusu cephu

	sudo ceph -s

      cluster:
        id:     204274b9-b388-4b61-a825-3663c84d87b8
        health: HEALTH_OK

      services:
        mon: 3 daemons, quorum mon1,mon2,mon3
        mgr: mon1(active)
        osd: 0 osds: 0 up, 0 in

      data:
        pools:   0 pools, 0 pgs
        objects: 0 objects, 0 bytes
        usage:   0 kB used, 0 kB / 0 kB avail
        pgs:

## Konfigurace ``mgr``

Na každém hostu, kde běží mon daemon, by měl běžet též mgr daemon pro dosažení vysoké dostupnosti. První mgr daemon který se připojí k monitorům se stane aktivním, a ostatní v režimu standby, k jejich funkci není třeba kvórum. Pokud aktivní mgr daemon nepošle v intervalu mon_mgr_beacon_grace beacon, bude nahrazen jiným v režimu standby. Pokud chceme otestovat failover, můžeme explicitně mgr odstavit.

X. Odstavení ``mgr`` daemonu

      ceph mgr fail mgr1

#. Vytvoření autentizačního klíče pro ``mgr`` daemon, jeho obsah umístit do ``/var/lib/ceph/mgr/ceph-mgr1/keyring``

    ceph auth get-or-create mgr.mgr1 mon 'allow profile mgr' osd 'allow *' mds 'allow *'

#. Start ``mgr``

	sudo systemctl start ceph-mon@mon1

#. Status mgr daemonu a jeho připojení ověříme pomocí ceph -s, který by měl obsahovat řádek ``mgr`` stavu

    mgr: mon2(active, since 3M), standbys: mon1, mon3

# Konfigurace `osd`

Pro vytvoření bloku pro daemony používáme LVM.

#. Vytvoření LVM PV

        pvcreate /dev/nvme0n1
        pvcreate /dev/nvme1n1
        pvcreate /dev/nvme2n1
        pvcreate /dev/nvme3n1
        pvcreate /dev/nvme4n1
        pvcreate /dev/nvme5n1

#. Vytvoření VG na každou PV

        vgcreate VG0
        vgcreate VG1
        vgcreate VG2
        vgcreate VG3
        vgcreate VG4
        vgcreate VG5

#. Zjištění, kolik máme Physical Extenses na dané VG

        vgdisplay VG0 | grep "Total PE"
        Total PE              66772

#. Pro standardní NVME disk doporučujeme 4 osd na jeden NVME disk, Total PE celočíselně vydělíme 4 a zjistíme ze jedna LV bude velká 16693 PEs

#. Vytvoření LVs pro OSD

        lvcreate -n vg0lv0 -l 16693 VG0
        lvcreate -n vg0lv1 -l 16693 VG0
        lvcreate -n vg0lv2 -l 16693 VG0
        lvcreate -n vg0lv3 -l 16693 VG0
        lvcreate -n vg1lv0 -l 16693 VG1
        lvcreate -n vg1lv1 -l 16693 VG1

... a tak dale pro vsechny budouci OSD, obdobny postup pouzijeme i pro pripadne oddelene WAL / BLOCK.DB.

#. Výsledek může vypadat cca takto

        nvme0n1      259:0    0 260.8G  0 disk 
        ├─VG0-vg0lv0 253:4    0  65.2G  0 lvm  
        ├─VG0-vg0lv1 253:5    0  65.2G  0 lvm  
        ├─VG0-vg0lv2 253:6    0  65.2G  0 lvm  
        └─VG0-vg0lv3 253:7    0  65.2G  0 lvm  
        nvme2n1      259:1    0 260.8G  0 disk 
        ├─VG2-vg2lv0 253:8    0  65.2G  0 lvm  
        ├─VG2-vg2lv1 253:9    0  65.2G  0 lvm  
        ├─VG2-vg2lv2 253:10   0  65.2G  0 lvm  
        └─VG2-vg2lv3 253:11   0  65.2G  0 lvm  

#. Alternativně, pokud používáme serverové disky NVME vyšších řad, které podporují namespaces, vytvoříme konfiguraci která namespaces využije. Dojde tak k lepší paralelizaci fronty a všeobecně optimálnější fragmentaci dat na disku.

# TBD

#. Nyní už můžeme vytvořit samotné OSD pomocí lvm batch pokud nepoužíváme oddělené WAL / BLOCK.DB:

        ceph-volume lvm batch /dev/VG*/vg*

#. Pokud toto máme oddělené tak musíme použít lvm prepare.
 - oddělení bdb / wal dává smysl na zařízení s řádově nižši latencí operace, musí to být 3x10^{0-3} při defaultním nastavení leveldb compactu.

        ceph-volume lvm prepare --bluestore --data $data --block.db $dblv --block.wal $wallv

#. A samozřejmě je nesmíme zapomenout aktivovat v případě prepare příkazu:
Neni třeba se bát --all, ono se to vyhne tomu je už aktivní.
POZOR activate rovnou nasadí váhy takže PGs zapeerují a hned udělají misplaced, nemusí být žádoucí pokud musí běžet balancer.

        ceph-volume lvm activate --all

#. Po přidání alespoň 1 OSD na 2 hostechprovedeme verifikaci umístění skupin PG, které by už měly peerovat pomocí příkazu


	ceph -w

#. Pro zobrazení stromu OSD použijeme

	ceph osd tree


	# id	weight	type name	up/down	reweight
	-1	2	root default
	-2	2		host osd1
	0	1			osd.0	up	1
	-3	1		host osd2
	1	1			osd.1	up	1






TBD :vypnout iptables
TBD :vypnout powersave
 ?nastavit scheduler







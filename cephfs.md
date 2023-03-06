# CephFS

``CephFS`` je ``POSIX`` kompatibilní souborový systém, který je vytvořený nad infrastrukturou objektového úložiště Ceph ``RADOS``. Je vysoce dostupný, škálovatelný a hodí se pro aplikaci sdíleného síťového úložiště souborů, při tradičních aplikacích jako sdílené adresáře a distribuovaná storage.

## Architektura

``CephFS`` používá pro uložení dat a metadata dva oddělené ``RADOS`` pooly. Přístup k souborům je koordinován přes metadata servery ``MDS``, které řídí přístupy a synchronizaci přístupu klientů k jednotlivým souborům. Klienti přistupují přímo k ``RADOS`` objektům, není zde tedy omezení v podobě úzkého hrdla serveru, který by soubory poskytoval. ``MDS`` servery zapisují změny do metadat do svých žurnálů na ``RADOS``, v aktivní konfiguraci si vzájemně metadata synchronizují a výsledná metadata pak zapisují do ``RADOS`` metadata poolu. Servery ``MDS`` jsou škálovatelné, výkon tak lze horizontálně zvyšovat

## Instalace MDS

Každý CephFS vyžaduje aspoň jedno ``MDS``. Instalace ``MDS`` je většinou automatizována pomocí Puppet nebo Ansible, následující příklad je manuální instalace pomocí samostatných příkazů

### Hardware pro MDS

``MDS`` využívá až 3 jádra, neškáluje s větším počtem jader. Pro horizontální škálování je tak vhodné nakonfigurovat více aktivních ``MDS``. ``MDS`` potřebuje minoimálně 8GB cache, pro systémy s více jak 1000 klienty pak až 64GB. Pro vysokou dostupnost je vhodné nakonfigurovat více aktivních ``MDS`` na více systémech, s výhodou a úsporou zdrojů můžeme ``MDS`` servery nainstalovat společně s ``MON`` a ``MGR``

### Instalace MDS

#. Vytvoření adresáře ``/var/lib/ceph/mds/ceph-${id}``, který se používá pro klíčenka ``MDS``

    mkdir -P /var/lib/ceph/mds/ceph-${id}

#. Vytvoření autentifikačního klíče pro ``CephX``

	sudo ceph auth get-or-create mds.${id} mon 'profile mds' mgr 'profile mds' mds 'allow *' osd 'allow *' > /var/lib/ceph/mds/ceph-${id}/keyring

#. Vytvoření MDS sekce v konfiguračním souboru ``/etc/ceph/ceph.conf``

    [mds]
    mds_data = /var/lib/ceph/mds/ceph-mds1
    keyring = /var/lib/ceph/mds/ceph-mds1/keyring
    mds_cache_memory_limit = 8589934592

#. Start ``MDS`` serveru

	sudo systemctl start ceph-mds@${id}

#. Status clusteru

	mds: ${id}:1 {0=${id}=up:active} 2 up:standby

#. Volitelně můžeme nastavit CephFS, kterému se daný ``MDS`` server připojí pomocí parameru ``mds-join-fs``

    $ ceph config set mds.${id} mds_join_fs ${fs}

## Vytvoření CephFS

### Vytvoření Poolů

CephFS potřebuje nejméně 2 RADOS pooly, jeden pro data a druhý
pro metadata. Metadata jsou kritická, doporučujeme použít nejméně 3-4 repliky a umístit metadata na nejrychlejší možné médium NVME/Optane, tak aby byla zajištěna co nejmenší latence pro klientské operace. Metadata pool obvykle obsahuje pouze několik GB dat, a menší počet PG, typicky 64 až 128 pro větší clustery

#. Vytvoření obou poolů

    ceph osd pool create cephfs_data
    ceph osd pool create cephfs_metadata

### Vytvoření filesystému

#. Vytvoření filesystému nad pooly ``cephfs`` a ``cephfs_data``

    ceph fs new cephfs cephfs_metadata cephfs_data

#. Kontrola nového filesystému

    ceph fs status
    name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]

#. Po vytvoření filesystému MDS servery automaticky vstoupí do aktivního stavu

    ceph mds stat
    cephfs-1/1/1 up {0=a=up:active}

#. Jakmile jsou MDS aktivní, je možné připojit filesystém CephFS ke klientům

## Připojení CephFS ke klientům

Pro připojení CephFS ke klientům se obvykle používá kernel driver, který zajistí kompatibilitu se standartními příkazy mount a fstab

### Připojení k CephFS pomocí příkazu ``mount`` 

#. Vytvoření připojení k CephFS

    mount -t ceph 172.27.102.1:3300,172.27.102.2:3300,172.27.102.3:3300:/ /mnt/cephfs -o name=cephfs,secret=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==,nocephx_sign_messages,noatime

#. Pro pernamentní připojení CephFS můžeme využít fstab:

    172.27.102.1:3300,172.27.102.2:3300,172.27.102.3:3300:/	/mnt/cephfs	ceph	x-systemd.automount,name=cephfs,secret=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==,nocephx_sign_messages,noatime	0	0

## Autentifikace klientů

V základu MDS nemá žádné restrikce pro restrikce klientů pomocí cesty ve filesystému. I když je klient připojen na mount podadresáře, není jeho přístup do nižších adresářů omezen

### Restrikce klientů do specifického adresáře

#. Pro povolení zápisu a čtení do specifického adresáře pro klienta jej specifikujeme při vytvoření klientského klíče

    ceph fs authorize cephfs client.client1 /storage rw

    client.client1
        key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==
        caps: [mds] allow rw path=/storage
        caps: [mon] allow r
        caps: [osd] allow rw tag cephfs data=cephfs

#. Pro omezení přístupu klienta do specifického adresáře ``/storage`` jej uvedeme při připojení cephfs

    mount -t ceph 172.27.102.1:3300,172.27.102.2:3300,172.27.102.3:3300:/storage /mnt/cephfs/storage -o name=cephfs,secret=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==,nocephx_sign_messages,noatime


### Kvóty klientů

CephFS umožnuje nastavení kvót omezující maximální velkost zapsaných dat a počtu souborů. Kvóty zajišťuje MDS ve spolupráci s klientem a nejsou tak vymahatelné na 100%, modifikovaný klient je tak může obejít, rovněž tak kontinuální zápis do otevřeného souboru, který MDS už nemůže na úrovni POSIX omezit do ukončení operace. Kvóty se nastavují pomocí extended atributů filesystému

#. Nastavení kvóty na maximální velikost a maximální počet souborů

    setfattr -n ceph.quota.max_bytes -v 100000000 /storage     # 100 MB
    setfattr -n ceph.quota.max_files -v 10000 /storage         # 10,000 souborů

#. Zobrazení kvóty

    getfattr -n ceph.quota.max_bytes /storage
    getfattr -n ceph.quota.max_files /storage

#. Odstranění kvóty

    setfattr -n ceph.quota.max_bytes -v 0 /storage
    setfattr -n ceph.quota.max_files -v 0 /storage
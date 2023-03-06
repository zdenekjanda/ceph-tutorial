# Monitoring

Monitoring Cephu se skládá z interní web služby Ceph Dashboard a volitelně externích služeb Prometheus a Grafana.

## Architektura

Ceph Dashboard je modulem daemon ceph-mgr. Ceph Dashboard umožňuje vizuální inspekci a správu jednotlivých služeb a zdrojů v Ceph clusteru pomocí webového přístupu, je tak užitečným doplňkem terminálových cli. Ceph Dashboard dále podporuje integraci externích nástrojů Prometheus a Grafana pro pokročilé ukládání a vizualizaci výkonových statistik. Ceph umožňuje automatickou instalaci Prometheus a Grafana do kontejnerů při použití cephadm orchestrátoru, případně Rook v Kubernetes. Pro naše účely předpokládáme manuální integraci těchto nástrojů, komponenty monitoringu nechceme primárně instalovat spolu s provozními komponenty hyperkonvergovaného cephu, tak aby byla zajištěna důsledná izolace. Instalace monitorovacích komponent je tak provedena na odděleném virtuálním serveru, na kterém tyto nástroje umístíme a taktéž připravíme architekturu pro centralizovaný monitoring, který takto může monitorovat libovolné množství nezávislých clusterů. Pro základní monitoring postačí virtuální server s 4GB RAM a 4VCPU, s 16GB místa na disku pro časovou retenci statistik

## Instalace Ceph Dashboard

Ceph Dashboard je modulem daemonu ceph-mgr. Pro spuštění modulu je třeba na všech serverech kde ceph-mgr doinstalovat balíkové závislosti Ceph Dashboard

#. Instalace balíku k ceph-mgr-dashboard

    sudo apt-get install ceph-mgr-dashboard

## Konfigurace Ceph Dashboard

Ceph Dashboard podporuje instalaci SSL. Pro naše použití předpokládáme externí SSL terminaci, tak abychom nebyli nuceni využívat self signed certifikáty, SSL autentifikaci tak můžeme vypnout

#. Vypnutí SSL autentifikace

    ceph config set mgr mgr/dashboard/ssl false

Ceph dashboard podporuje autentifikaci, a pro základní přístup je třeba vytvořit uživatele. Využijeme předvytvořené role, které Ceph dashboard používá. Heslo umístíme do souboru dashboard.pwd

#. Vytvoření hesla a přístupu pro uživatele admin

    ceph dashboard ac-user-create admin -i dashboard.pwd administrator

Nyní je Ceph Dashboard připravena k provozu, je možné povolit modul dashboard v ceph-mgr daemonu

#. Povolení Ceph Dashboard v daemonu ceph-mgr

    ceph mgr module enable dashboard

#. Ověření, že Ceph Dashboard běží a poslouchá na HTTP portu

    netstat -lpn | grep 8080

    tcp6       0      0 :::8080                 :::*                    LISTEN      3562877/ceph-mgr 

Ceph Dashboard je v provozu, a můžeme se přihlásit v prohlížeči a začít Ceph Dashboard používat

#. Otevření URL Ceph Dashboard v prohlížeči

    http://172.27.175.23:8080

Při změně provozních parametrů Ceph Dashboard je vždy nutné provést restart modulu pomocí disable/enable, tak aby se nové parametry aplikovaly

#. Restart modulu Ceph Dashboard

    ceph mgr module disable dashboard
    ceph mgr module enable dashboard

## Instalace Prometheus

Monitorovací software Prometheus nainstalujeme na oddělený virtuální server, který bude mít přístup na IP serverů v Ceph clusteru a provádět vzdálený sběr dat a monitoring. Pro instalaci se předpokládají distribuční balíky systému Debian/Ubuntu, a to konkrétně Ubuntu 22.04 LTS Jammy, který obsahuje již moderní verzi systému Prometheus 2.31, není tak nutné provádět manuální instalaci ze zdrojů.





.``RBD`` používá pro uložení blokových zařízení ``RADOS`` pool. Je možné použití více poolů pro více oddělených skupin blokových zařízení, například dle typů hardware. Pro uložení blokových zařízení doporučujeme použití nejvýkonnějších disků NVME či SSD, bloková zařízení jako virtuální servery potřebují v náročných aplikacích jako databáze nejvyšší možný výkon IOPS. Optimalizace IOPS pro ``RBD`` zařízení je nejčastějším důvodem pro výkonovou optimalizaci Ceph clusteru

## Instalace ``RBD``

Bloková zařízení ``RBD`` nepotřebují ke své funkci žádný  specifický komponent, potřebujeme pouze běžící Ceph cluster.

## Konfigurace ``RBD``

#. Vytvoření poolu pro ``RBD``

    ceph osd pool create rbd 128

#. Aplikační povolení ``RBD`` pro nový pool

    ceph osd pool application enable rbd rbd

#. Inicializace ``RBD`` poolu

    rbd pool init rbd

## Autentifikace ``RBD``

Ve výchozím nastavení používá příkaz ``rbd`` uživatele admin. Pro přístup klientů, například hypervisorů Openstack je vhodné vytvořit klientské účty

#. Vytvoření uživatele openstack který má rw přístup do poolu rbd a r přístup do poolu images

    ceph-authtool -C /etc/ceph/ceph.client.rbd.keyring -n client.rbd --cap osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rx pool=images' --cap mon 'allow r, allow command "osd blacklist"'

    ceph auth add client.rbd -i /etc/ceph/ceph.client.rbd.keyring

## Základní práce s blokovými zařízeními ``RBD`` (dále jen ``image``)

#. Vytvoření ``image`` myimage s velikostí 1GB v základním poolu rbd

    rbd create --size 1024 myimage

#. Vylistování ``image`` v základním poolu rbd

    rbd ls

#. Informace o ``image`` myimage

    rbd info myimage

#. Zvýšení velkosti ``image`` myimage na 2 GB

    rbd resize --size 2048 myimage

#. Snížení velkosti ``image`` myimage na 1 GB

    rbd resize --size 1048 myimage --allow-shrink

#. Odstranění ``image`` myimage

    rbd rm myimage

## Základní práce s ``RBD`` snapshoty

Snapshot je logická kopie read-only zdrojového RBD image v určitém čase, kdy byl snapshot vytvořen. Díky tomu můžeme vytvářet snapshoty images, které si zachovávají status v historii změn. Ceph podporuje vrstvení snapshotů, díky kterým můžeme klonovat nové images pomocí algoritmu Copy-On-Write, což znamená maximální úsporu času a místa na disku

#. Vytvoření snapshotu ``image``

    rbd snap create myimage@mysnap

#. Vylistování snapshotů ``image``

    rbd snap ls myimage

#. Rollback shapshotu, přepíše aktuální obsah ``image`` obsahem snapshotu

    rbd snap rollback myimage@mysnap

#. Smazání snapshotu ``image``

    rbd snap rm myimage@mysnap

#. Smazání všech snapshotů ``image``

    rbd snap purge myimage

## Vrstvení a klonování v ``RBD``

Vrstvení a klonování je jednoduchý proces. Ze zdrojové ``image`` se vytvoří snapshot, zapne se jeho ochrana, a poté vytvoříme jeho klon, který se opět stane zapisovatelným image. Výhoda je, že nové image používá algoritmus ``Copy-On-Write``, takže například nově naklované image operačního systému nebude zabírat žádné místo navíc

#. Vytvoření snapshotu ``image``

    rbd snap create myimage@mysnap

#. Ochrana snapshotu, po zapnutí ochrany jej nejde smazat

    rbd snap protect myimage@mysnap

#. Vypnutí ochrany snapshotu, aby jej šlo smazat. Pokud ze snapshotu vznikly klony, je třeba nejdříve klony smazat, či provést jejich sloučení pomocí flatten

    rbd snap unprotect myimage@mysnap

#. Klonování snapshotu. Vznikne nový ``image``, jehož velikost bude nulová a stávající obsah se bude odkazovat na zdrojový snapshot

    rbd clone myimage@mysnap myclone

#. Výpis klonů snapshotu

    rbd children myimage@mysnap

#. Sloučení klonů pomocí flatten. Při sloučení naroste velikost klonu, část která byla referencí původního snapshotu se přidá k stávajícímu ``image`` a to se oddělí od zdrojového snapshotu

    rbd flatten myclone

## Klientský přístup

Klienti, například hypervisory ``Openstack`` potřebují k připojení k ``RBD`` základní instalaci klientských knihoven, konfigurační soubor a klíč

#. Instalace klientských knihoven

    cephadm install ceph-common

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

#. Připojení blokového zařízení myimage na klientský systém

    rbd map --pool rbd myimage --id admin


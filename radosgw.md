# ``RadosGW``

``RadosGW`` je implementace objektové storage s HTTP REST pod ``RADOS``. ``RadosGW`` poskytuje API kompatibilní s S3 a Swift, nabízí tak řešení jako HTTP backend pro cloudové web aplikace či HTTP objektovou storage pro cloud Openstack

## Architektura

``RadosGW`` používá pro uložení blokových zařízení ``RADOS`` pooly. Konfigurace umožňuje použít více poolů pro jednotlivé objektové buckety, je tak možné vytvářet flexibilní objektovou storage pro různé druhy použití a úložišť v hyperkonvergovaném cephu, od HDD pro streamované video, po NVME pro miliony souborů obrázků a javascriptu/html. ``RadosGW`` je horizontálně škálovatelná, počet ``RadosGW`` daemonů lze tak neomezeně rozširovat. Provoz s HTTP protokolem dává možnosti load balancingu, cachování a vysoké dostupnosti pro nejnáročnější aplikace

## Instalace ``RadosGW``

``RadosGW`` potřebuje pro svůj provoz daemon radosgw, který slouží jako HTTP objektová gateway a zároveň manager metadat a objektových poolů. Práce s pooly je automatická, radosgw si je tak sám vytváří a spravuje. Níže je uveden proces manuální instalace, který lze dále automatizovat pomocí nástrojů konfiguračního managementu. V hyperkonvergovaném cephu při nižších výkonových požadavcích můžeme radosgw daemony nainstalovat společně s jinými komponenty, typicky mon a mgr, v nasazení web scale je ideální pro radosgw vyhradit samostatné virtuální či baremetal servery, při přísunu mnoho requestů či DDOS útoku může snadno dojít k přetížení CPU a v hyperkonvergované konfiguraci by to mohlo znamenat pád celého cephu v důsledku například timeoutů na mon daemonech.

## Instalace balíků ``RadosGW``

Předpokládá se, že na systému je již nainstalováno apt repository a základní společná konfigurace ``/etc/ceph/ceph.conf``

#. Instalace ``RadosGW`` daemonu

    apt-get install radosgw

#. Vytvoření konfiguračního adresáře

    mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.`hostname -s`

#. Vytvoření klíčenky a klíče pro ``RadosGW``

    ceph auth get-or-create client.radosgw.`hostname -s` osd 'allow rwx' mon 'allow rw' -o /var/lib/ceph/radosgw/ceph-radosgw.`hostname -s`/keyring

#. Změna uživatelských práv uživateli ``ceph``

    chown -R ceph:ceph /var/lib/ceph/radosgw

#. Vytvořední konfigurační sekce radosgw v ``/etc/ceph/ceph.conf``

Syntaxe

    [client.radosgw.<hostname>]
    host = <hostname>
    keyring = /var/lib/ceph/radosgw/ceph-radosgw.<hostname>/keyring
    log_file = /var/log/ceph/radosgw.log
    user = www-data
    rgw_frontends = beast port=80

Příklad

    [client.radosgw.ceph1]
    host = ceph1
    keyring = /var/lib/ceph/radosgw/ceph-radosgw.ceph1/keyring
    log_file = /var/log/ceph/radosgw.log
    user = www-data
    rgw_frontends = beast port=80

Volitelně lze použít další parametry pro optimalizaci provozu ``RadosGW``.

Při instalaci kde se předpokládá velké množství souborů v řádu milionů, je vhodné zapnout sharding indexů ``rgw_override_bucket_index_max_shards``, tak aby nebyl jednotný index přetížený

    rgw_override_bucket_index_max_shards = 16

Nedoporučujeme zapnout automatický resharding ``rgw_dynamic_resharding``, je lepší dělat manuálně na základě výkonových optimalizací 

    rgw_dynamic_resharding = false

#. Start ``RadosGW`` daemona

    systemctl enable ceph-radosgw.target
    systemctl enable ceph-radosgw@radosgw.`hostname -s`
    systemctl start ceph-radosgw@radosgw.`hostname -s`

#. Kontrola, že ``RadosGW`` naběhla a vytvořila automaticky svoje datové pooly

    ceph osd pool ls

    ..
    .rgw.root
    default.rgw.log
    default.rgw.control
    default.rgw.meta

## Použití ``RadosGW``

Ukázkové použití a test ``REST`` interface ``RadosGW``

#. Vytvoření s3 uživatele ``user1`` 

    radosgw-admin --uid testuser1 --display-name "Test User" user create

    {
    "user_id": "testuser1",
    "display_name": "Test User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "testuser1",
            "access_key": "UH91MYE8C1QWYR3QN7E9",
            "secret_key": "nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
    }

#. Instalace ``s3cmd``

    apt-get install s3cmd

#. Vytvoření bucketu ``testbucket``

    s3cmd --no-ssl --access_key=UH91MYE8C1QWYR3QN7E9 --secret_key=nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE --host=localhost --host-bucket= mb s3://testbucket

    Bucket 's3://testbucket/' created

#. Uložení souboru ``test.tst`` do bucketu ``testbucket``

    s3cmd --no-ssl --access_key=UH91MYE8C1QWYR3QN7E9 --secret_key=nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE --host=localhost --host-bucket= put test.tst s3://testbucket

    upload: 'test.tst' -> 's3://testbucket/test.tst'  [1 of 1]
    7 of 7   100% in    0s   112.91 B/s  done

#. Výpis souborů v bucketu ``testbucket``

    s3cmd --no-ssl --access_key=UH91MYE8C1QWYR3QN7E9 --secret_key=nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE --host=localhost --host-bucket= ls s3://testbucket

    2023-02-21 12:41         7   s3://testbucket/test.tst

#. Stažení souboru ``test.tst`` z bucketu ``testbucket``

    s3cmd --no-ssl --access_key=UH91MYE8C1QWYR3QN7E9 --secret_key=nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE --host=localhost --host-bucket= get s3://testbucket/test.tst

    download: 's3://testbucket/test.tst' -> './test.tst'  [1 of 1]
    7 of 7   100% in    0s   131.69 B/s  done

## Správa uživatelů

#. Vytvoření uživatele

    radosgw-admin --uid testuser1 --display-name "Test User" user create

    {
    "user_id": "testuser1",
    "display_name": "Test User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "testuser1",
            "access_key": "UH91MYE8C1QWYR3QN7E9",
            "secret_key": "nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
    }

#. Informace o uživateli

    radosgw-admin user info --uid=testuser1

#. Modifikace uživatele

    radosgw-admin user modify --uid=testuser1 --display-name="Test user 1"

#. Odstranění uživatele

    radosgw-admin user rm --uid=testuser1

#. Vytvoření klíče pro uživatele

Uživatelé mohou mít více klíču, nový klíč se přidá ke stávajcícím

    radosgw-admin key create --uid=testuser1 --key-type=s3 --access-key someaccesskey --secret-key somesecretkey

    {
    "user_id": "testuser1",
    "display_name": "Test User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "testuser1",
            "access_key": "UH91MYE8C1QWYR3QN7E9",
            "secret_key": "nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE"
        },
        {
            "user": "testuser1",
            "access_key": "someaccesskey",
            "secret_key": "somesecretkey"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
    }

#. Odstranění uživatelského klíče

    radosgw-admin key rm --uid=testuser1 --key-type=s3 --access-key=someaccesskey

    {
    "user_id": "testuser1",
    "display_name": "Test User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "testuser1",
            "access_key": "UH91MYE8C1QWYR3QN7E9",
            "secret_key": "nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
    }

## Uživatelské kvóty

#. Nastavení kvóty uživatele na maximálně 1024 objektů a 1Mbyte

    radosgw-admin quota set --quota-scope=user --uid=testuser1 --max-objects=1024 --max-size=1024B

Negativní hodnoty kvóty tuto vypnou

#. Povolení/zákaz kvóty uživatele

    radosgw-admin quota enable --quota-scope=user --uid=testuser1
    radosgw-admin quota disable --quota-scope=user --uid=testuser1

#. Nastavení kvóty na bucket

    radosgw-admin quota set --quota-scope=bucket --uid=testuser1 --max-objects=1024 --max-size=1024B

#. Povolení/zákaz kvóty na bucket

    radosgw-admin quota enable --quota-scope=bucket --uid=testuser1
    radosgw-admin quota disable --quota-scope=bucket --uid=testuser1

#. Zobrazení nastavení kvót

Zobrazení kvót je vidět v informacích o uživateli

    radosgw-admin user info --uid=testuser1

    }
    "user_id": "testuser1",
    "display_name": "Test User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "testuser1",
            "access_key": "UH91MYE8C1QWYR3QN7E9",
            "secret_key": "nT6OpBTNfAEYS0dTCVs4pF9y1JoFMqMH9nMqgfCE"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": 1024,
        "max_size_kb": 1,
        "max_objects": 1024
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
    }

#. Zobrazení zdrojů, které uživatel aktuálně využívá

    radosgw-admin user stats --uid=testuser1

    {
    "stats": {
        "size": 7,
        "size_actual": 4096,
        "size_kb": 1,
        "size_kb_actual": 4,
        "num_objects": 1
    },
    "last_stats_sync": "0.000000",
    "last_stats_update": "2023-02-21T12:42:54.812012Z"
    }

## Multitenance, uživatelské skupiny

``RadosGW`` podporuje multitenanci, skupiny uživatelů. Každý uživatel má nastaven svůj vlastní tenant, a zároveň můžeme více uživatelů sloučit do jednoho tenantu, typické použití jsou skupiny zákazníků

#. Vytvoření uživatele ``testuser2`` ve skupině ``company1``

    radosgw-admin --tenant company1 --uid testuser2 --display-name "Test User 2" user create

#. Přístup k datům přes ``S3`` v rámci skupiny company1

    http://localhost/tenant:bucket

## Troubleshooting

#. RadosGW nestartuje

Je třeba zjistit, proč ``RadosGW`` nestartuje i když nepíše logy, tak jak je očekáváno. Je nutné nastartovat ve foreground módu a zapnout maximální debug

Zjistit příkaz na start ``RadosGW`` přes ``systemctl``

    systemctl status ceph-radosgw@radosgw.ceph1
    ● ceph-radosgw@radosgw.ceph1.service - Ceph rados gateway
     Loaded: loaded (/lib/systemd/system/ceph-radosgw@.service; disabled; vendor preset: enabled)
     Active: active (running) since Tue 2023-02-21 13:24:53 CET; 5h 39min ago
    Main PID: 1146042 (radosgw)
     Memory: 88.8M
     CGroup: /system.slice/system-ceph\x2dradosgw.slice/ceph-radosgw@radosgw.ceph1.service
             └─1146042 /usr/bin/radosgw -f --cluster ceph --name client.radosgw.ceph1 --setuser ceph --setgroup ceph

Nastartovat ``RadosGW`` s přídavnými parametry --verbose --debug-rgw=20 -d

    /usr/bin/radosgw -f --cluster ceph --name client.radosgw.ceph1 --setuser ceph --setgroup ceph --verbose --debug-rgw=20 -d

#. Debug při provozu ``RadosGW``

Pro detailní debug HTTP requestů při provozu RadosGW je vhodné zapnout maximální debug a rovněž logovací soubor v ``/etc/ceph/ceph.conf``, který můžeme dále zpracovávat a monitorovat. Používat s rozumem, při velkém množství requestů je zdrojem latence a snadno zkonzumuje dostupnou diskovou kapacitu. Pozor taktéž na práva logovacího souboru ``/var/log/ceph/radosgw.log``, která musí být ceph:ceph, když radosgw v debug módu spouštíme manuálně pod uživatelem root, občas se stane že se práva změní a pak přemýšlíme proč radosgw neloguje...

    [client.radosgw.ceph1]
    debug rgw = 20
    log_file = /var/log/ceph/radosgw.log

## Load balancing

Pro horizontání škálování ``RadosGW`` a zároveň vysokou dostupnost služby je možné spustit více ``RadosGW`` daemonů. Potom je vhodné pro webový přístup aplikovat frontend loadbalancer, který zajistí rovnoměrné rozložení dotazů a caching, případně SSL terminaci a virtuální hosty

#. Jednoduchý loadbalancer s haproxy bez SSL terminace, konfigurace v ``/etc/haproxy/haproxy.cfg``

    global
      chroot  /var/lib/haproxy
      daemon  
      group  haproxy
      log  10.13.12.37 local0
      maxconn  4000
      pidfile  /var/run/haproxy.pid
      stats  socket /var/lib/haproxy/stats
      user  haproxy

    defaults
      log  global
      maxconn  8000
      option  redispatch
      retries  3
      stats  enable
      timeout  http-request 10s
      timeout  queue 1m
      timeout  connect 10s
      timeout  client 1m
      timeout  server 1m
      timeout  check 10s

    frontend radosgw
      bind 0.0.0.0:80 
      mode http
      default_backend radosgw

    backend radosgw
      mode http
      balance roundrobin
      option http-server-close
      option httpchk HEAD /
      server ceph1 172.27.102.1:80 check
      server ceph2 172.27.102.2:80 check
      server ceph3 172.27.102.3:80 check

## Optimalizace uložení dat

``RadosGW`` ukládá data v defaultním nastavení do několika poolů. Pooly jsou automaticky vytvořeny v Cephu podle defaultního nastavení PG skupin. Počet PG pro pooly s daty kde se očekává velké vytížení je vhodné optimalizovat na správný počet PG


    .rgw.root
    default.rgw.log
    default.rgw.control
    default.rgw.meta
    default.rgw.buckets.index
    default.rgw.buckets.data

Kontrolní data jsou ``.root``, ``.log``, ``.control`` a ``.meta``, obvykle nezabírají velký prostor a tak je můžeme nechat na rychlém médiu.

Samotná objektová data jsou uložena v ``buckets.data``, a indexy v ``buckets.index``. Opět můžeme indexy umístit na rychlé médium pro zvýšení výkonu například operací výpisu souboru v bucketu. Samotná data ``buckets.data`` potom lze umístit i na pomalejší médium, typicky HDD pro objemná data.

Pro ještě větší oddělení dat je možné nakonfigurovat vícero zón, které pracují odděleně, s vlastními daty, pooly i RadosGW daemony, viz multisite konfigurace v Ceph dokumentaci: https://docs.ceph.com/en/quincy/radosgw/multisite/
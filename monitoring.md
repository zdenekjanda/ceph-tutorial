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

![](ceph-dashboard.png)

Při změně provozních parametrů Ceph Dashboard je vždy nutné provést restart modulu pomocí disable/enable, tak aby se nové parametry aplikovaly

#. Restart modulu Ceph Dashboard

    ceph mgr module disable dashboard
    ceph mgr module enable dashboard


## Instalace Prometheus

Monitorovací software Prometheus je ideální instalovat na oddělený virtuální server, který bude mít přístup na IP serverů v Ceph clusteru a provádět vzdálený sběr dat a monitoring. Pro instalaci se předpokládají distribuční balíky systému Debian/Ubuntu, a to konkrétně Ubuntu 22.04 LTS Jammy, který obsahuje již moderní verzi systému Prometheus 2.31, není tak nutné provádět manuální instalaci ze zdrojů.

#. Instalace balíků prometheus

    sudo apt-get install prometheus

#. Ověření, že Prometheus běží a poslouchá na HTTP portu

    netstat -lpn | grep prometheus

    tcp6       0      0 :::9090                 :::*                    LISTEN      10649/prometheus

Prometheus je nainstalován. Nyní je možné navštívit webový UI

#. Otevření URL Prometheus v prohlížeči

    http://172.27.175.112:9090

![](prometheus-home.png)

##. Konfigurace Prometheus

Aby bylo možné zahájit sběr dat z Cephu, je nutné na všech ceph nodech, kde běží ceph-mgr, povolit sběr dat z prometheus modulu

#. Povolení modulu prometheus na všech nodech, kde běží ceph-mgr

    ceph mgr module enable prometheus

Daemon ceph-mgr automaticky pustí prometheus modul a exporter na portu 9283

#. Oveření Prometheus exporteru na všech nodech s ceph-mgr

    netstat -lpn | grep 9283

    tcp6       0      0 :::9283                 :::*                    LISTEN      3562877/ceph-mgr

Pro kompletní sběr všech informací je dále nutné na všechny ostatní Ceph servery nainstalovat i prometheus-node-exporter, který zajistí sběr systémových dat jako vytížení CPU či disků, které dále využijeme při analýze výkonu v grafech a které využívají i oficiální grafana šablony

#. Instalace  prometheus-node-exporter na všech serverech s komponenty Ceph

    sudo apt-get install prometheus-node-exporter

Nyní je Ceph cluster připraven na sběr dat a je možné nakonfigurovat Prometheus, aby tyto data sbíral. Konfiguraci provedeme přidáním setů pro modul ceph pomocí sekce scrape_cofigs v /etc/prometheus/prometheus.yaml. Je vhodné, aby fungovalo DNS a mohli jsme používat buď FQDN a nebo hostname, tak aby byla potom data v dashboardech čitelná. Konfigurační soubor můžeme též plnit pomocí service discovery a konfiguračního managementu pomocí tagů automaticky. 

#. Přidání serverů s ceph-mgr a modulem prometheus do /etc/prometheus/prometheus.yaml

    scrape_configs:

        - job_name: ceph
        # ceph-mgr module
            static_configs:
            - targets: ['ceph1:9283','ceph2:9283','ceph3:9283']

Rovněž přidáme novou sekci pro sběr systémových dat z prometheus-node-exporter ze všech serverů s Ceph

#. Přidání všech serverů v Ceph s prometheus-node-exporter pro sběr systémových statistik


    scrape_configs:

        - job_name: ceph-node-exporter
        # ceph s prometheus-node-exporter
            static_configs:
            - targets: ['ceph1:9100','ceph2:9100','ceph3:9100']

Po přidání všech serverů pro sběr dat je nutné provést restart prometheus

#. Restart daemonu prometheus

    systemctl restart prometheus

Prometheus nyní sbírá všechna potřebná data. Provedeme kontrolu přes dashboard v záložce Status/Targets, že se data načítají

![](prometheus-targets.png)

Sběr dat z Ceph clusteru je plnně funkční, nyní je třeba nainstalovat Grafanu pro vizualizaci dat

## Instalace Grafany

Grafovací software Grafana doporučujeme instalovat na stejný oddělený virtuální server jako Prometheus, je komplementem monitorovacího řešení pro Ceph. Pro instalaci Grafany využijeme repozitáře vývojářů, aktualizace jsou zde rapidní a otevřeme si tak cestu k bezproblémovému update.

#. Instalace repository a Grafany

    sudo apt-get install -y apt-transport-https software-properties-common wget
    sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
    echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
    sudo apt-get update
    sudo apt-get install grafana

Balík Grafana z neznámého důvodu nespouští grafana-server automaticky, je třeba výslovně vynutit

#. Automatický start Grafany po spuštění a okamžitý start

    sudo systemctl daemon-reload
    sudo systemctl enable grafana-server
    sudo systemctl start grafana-server

Pro správnou funkci všech předdefinovaných dashboardů je třeba nainstalovat Grafana pluginy

#. Instalace Grafana pluginů vonage-status-panel a grafana-piechart-panel

    grafana-cli plugins install vonage-status-panel
    grafana-cli plugins install grafana-piechart-panel

## Konfigurace Grafany

Pro přístup z Ceph Dashboard je třeba dále povolit volby pro anonymní přístup a embedding Grafany v /etc/grafana/grafana.ini

#. Povolení anonymního přístupu v /etc/grafana/grafana.ini

    [auth.anonymous]
    enabled = true
    org_name = Main Org.
    org_role = Viewer

#. Povolení embedded Grafany v /etc/grafana/grafana.ini

    [security]
    allow_embedding = true

Po dokončení konfiguračních změn je třeba provést restart grafana-server

#. Restart daemonu grafana-server

    systemctl restart grafana-server

Po dokončení konfigurace Grafana běží na portu 3000

#. Oveření funkce daemonu Grafany na portu 3000

    netstat -lpn | grep grafana

    tcp6       0      0 :::3000                 :::*                    LISTEN      11735/grafana

Grafana je nainstalována a nakonfigurována. Nyní je možné navštívit webový UI

#. Otevření URL Grafany v prohlížeči

    http://172.27.175.112:3000

![](grafana-home.png)

Nyní je třeba přidat datový zdroj Prometheus, aby Grafana mohla vizualizovat data. Datový zdroj přidáme v Settings/Data
 Sources, typ je Prometheus a adresa je lokální adresa Prometheus serveru http://172.27.175.112:9090. Datový zdroj je nutné pojmenovat Dashboard1, aby byla zachována kompatibita s Ceph Dashboard

 #. Přidání datového zdroje Prometheus na adrese http://172.27.175.112:9090 s názvem Dashboard1

 ![](grafana-datasource.png)

Grafana je připravena zobrazit vizualizaci dat z Cephu. Pro základní vizualizaci dat použijeme předpřipravené dashboardy od vývojářů Cephu, které se zobrazují z Ceph Dashboard. Zdrojové soubory dashboardů je možné stáhout z git repozitářů Cephu

#. Stažení Grafana dashboardů z git repozitářů Cephu

    https://github.com/ceph/ceph/tree/main/monitoring/ceph-mixin/dashboards_out

Jednotlivé JSON soubory je nutné importovat lokálně z browseru přes dashboard přes záložku Dashboards/Import. Po dokončení importu budou dashboards dostupné v hlavním seznamu Dashboards

#. Zobrazení seznamu importovaných dashboards

 ![](grafana-dashboards.png)

Po dokončení importu všech souborů je možné zobrazit veškeré dostupné grafy z diagnostiky Cephu

#. Zobrazení informací z dashboardu Ceph - Cluster

![](grafana-dashboard-cluster.png)

Instalace Grafany a vizualizace Ceph clusteru je nyní kompletní.




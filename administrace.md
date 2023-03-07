# Administrace

Zde si projdeme nejčastější údržbové akce kolem fungujícího clusteru.

## Crush mapy

Crush mapa je mapa clusteru, kde jsou jaké disky, kde jsou jaké stroje, kde jsou jaké crush rooty a co obsahují. Zobrazí se po zadání příkazu ``ceph osd crush tree --show-shadow``.

### Přidání nováho disku do crush mapy

Při složitějším rozložení crush ceph nedokáže odhadnout kam přidat nový disk a je tedy potřeba disk manuálně přidat tam kam sami chceme.

    ceph osd crush set osd.11 0.9 root=hdd host=ceph3-hdd

Zadává se jaké osd chceme přidat jako jak velké a jakou má mít cephu v crush mapě.

### Přidání nového hostu do crush mapy

Pokud ještě nemáme host crush bucket pro nové osd tak ho bude potřeba vytvořit:

    ceph osd crush add-bucket novyStroj host

Opět je možné specifikovat pod co se v crush mapě má zařadit pokud ne pod defaultní crush root.


## Recovery

``Recovery`` je souhrnné označení interních operací pomocí kterých se ceph snaží sám dostat do optimálního stavu.
Za optimální stav považujme stav kdy je plná redundance všech dat a všechny placement groupy jsou na OSD kam jsou namapovane ``CRUSH`` algoritmem.

### Řízení recovery

Při běžné operaci recovery běží ihned jakmile je ve frontě jakkákoli operace k provedení. Toto chování může být vhodné omezit či zakázat ve chvílích kdy se recovery nehodí provádět ihned. Jako příklad může posloužit snížená redundance z důvodu odstávky stroje nebo větší převažování vah disků kde by okamžitá recovery mohla způsobit přetížení a chceme ji pouštet až po nastavení různých parametrů pro bezproblémový průběh.

#### Zapínání a zakazování recovery

Pro řízení běhu recovery jsou nejvhodnější následující flagy:
 - norecover - zakáže veškeré recovery operace.
 - nobackfill - zakáže pouze obnovu redundance.
 - norebalance - zakáže pouze balancování dat dle CRUSH algorimtu.

flagy se nastavují standartním způsobem globálně pomocí ``ceph osd set XX`` nebo ``ceph osd unset XX``

#### Parametry pro řízení rychlosti recovery

V případě nutnosti zvyšit nebo snížit rychlost recovery můžeme použít následující parametry:
- osd_recovery_sleep - přidá nějaký čas mezi recovery operace, defaultně je pro ssd 0 a to může dělat problémy.
- osd_recovery_max_active - maximální počet paralelně aktivních recovery operací na osd.
- osd_max_backfills - maximální počet paralelních in/out recovery operací na osd.

obecně doporučuji mít v konfiguraci max active a backfills na 1, sleep na 0.1. V případě potřeby zvýšení rychlosti se dá injectnout příkaz bez nutnosti restartu:

    ceph tell osd.* injectargs '--osd_max_backfills=2'


## Servis hardware

Zde si ukážeme nejčastější údržbové akce kolem samotného HW.

### Údržba serveru

Nejjednodušší operace je odstávka celého serveru pro krátkou údržbu, například výměnu CPU nebo RAM. V případě krátké odstávky není potřeba odlévat data - krátkodobé snížení redundance se většinou vyplatí riskovat.

Klíčové je nemít nic degraded, to je pozná jednoduše podle toho, že v výstupu příkazu ``ceph -s`` není klíčové slovo degraded.

v takovém případě stačí nastavit clusteru aby nerecoveroval a je možné stroj vypnout.

    ceph osd set noout  # říká clusteru aby nepovažoval OSD za out a nechtěl rebalancovat data napříč clusterem.
    ceph osd set norecover  # říká clusteru aby nedělal žádné recovery operace, viz výše.

POZOR, vypnutí osd zapříčiní krátké zapeerování placement group - krátkodobé nedostupnosti IO nad některými daty. Je to krátké ale je, pokud chceme dosáhnout nejmenšího možného dopadu tak osd vypínáme po jednom tak že další nevypneme dokud nedopeerujou PGs z předchozího OSD (opět hledáme klíčové slovo peering v příkazu ceph -s).

### Výměna disku

Výměna disku bude asi nejčastější operace nad cephem v jeho dlouhodobém používání.

#### Nalezení vadného disku a čísla OSD

Pokud nalezneme vadný disk, tak pokud nemáme číslo samotného OSD tak ho musíme nějak zjistit. Nejsnažší cesta je pomocí příkazu ``ceph device ls`` přidaného v pozdějších verzích cephu odkud se dá najít host+jméno blockdevice+číslo OSD daemonu.

    ceph device ls | grep -e storage3-osd3:sdd
    HITACHI_HUHXXXXXXXXXXXXX_YYYYYYYY          storage3-osd3:sdd    osd.25

#### Odlití dat z vadného disku

Ve chvíli kdy disk umřel úplně a nemá jen read errory tak tento bod vynecháme. Data odlijeme tím že nastavíme jeho váhu na 0.

    ceph osd crush reweight osd.25 0

a počkáme než bude na osd 0 PGs.

    ceph osd df tree | grep -i -E "^164|^ID"
    ID   CLASS  WEIGHT      REWEIGHT  SIZE     RAW USE  DATA     OMAP      META     AVAIL    %USE   VAR   PGS  STATUS  TYPE NAME                  
    164    hdd    14.00052   1.00000   13 TiB  773 GiB  771 GiB       0 B  2.3 GiB   12 TiB   5.93  0.08    0      up          osd.164

Poté můžeme disk v klidu odebrat a fyzicky vyměnit.

#### Odstranění disku

Disk odstraníme tímto příkazem spuštěném na hostu kde daný disk žije (Viz výše). POZOR, zap je destruktivní! Ujistěte se že disk je již mrtvý, down, nemá žádná data a je možné ho odstranit.

    osd=XXX; ceph device ls-by-daemon osd.$osd ; ceph osd out $osd ; systemctl stop ceph-osd@$osd ; systemctl disable ceph-osd@$osd ; ceph osd purge $osd --yes-i-really-mean-it ; ceph-volume lvm zap --destroy --osd-id $osd

#### Přidání disku

Disk se přidává standartně pomocí ``prepare`` nebo ``batch`` příkazů popsaných v dřívějších školeních. Disk může být potřeba manuálně zařadit do crush mapy (viz výše).

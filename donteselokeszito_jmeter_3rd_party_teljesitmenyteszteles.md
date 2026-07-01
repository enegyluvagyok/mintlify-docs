# Döntéselőkészítő dokumentum  
## 3rd party / bérelt infrastruktúra JMeter alapú teljesítményteszteléshez

**Dokumentum célja:**  
A dokumentum célja annak előkészítése, hogy JMeter alapú teljesítményteszteléshez milyen külső, bérelt infrastruktúra használata indokolt ár, teljesítmény, rugalmasság, skálázhatóság és üzemeltetési komplexitás alapján.

**Vizsgált alternatívák:**

- Rackhost VPS
- AWS EC2
- Microsoft Azure Virtual Machines

**Elsődleges felhasználási cél:**  
JMeter controller + JMeter worker node-ok futtatása, elsősorban HTTP/API/webalkalmazás teljesítményteszteléshez, non-GUI módban.

---

## Vezetői összefoglaló

A vizsgált lehetőségek alapján a **Rackhost VPS alapú JMeter cluster** adja a legjobb fix havi ár-érték arányt, amennyiben a tesztelési igény rendszeres, előre tervezhető, és nem szükséges globális, több régiós terhelésgenerálás.

Az **AWS** és az **Azure** akkor indokolt, ha a tesztkörnyezetet csak időszakosan kell elindítani, ha a futtatás automatizált CI/CD folyamatba kerül, vagy ha fontos a több régióból történő terhelésgenerálás. Folyamatos, 0–24-ben fenntartott workerpark esetén a hyperscaler cloud szolgáltatások jellemzően drágábbak.

**Elsődleges javaslat:**  
Rackhost VPS alapú JMeter cluster:

- 1 db kisebb VPS controller node
- 2 db közepes vagy nagyobb VPS worker node
- non-GUI JMeter futtatás
- InfluxDB / Prometheus / Grafana monitoring
- JTL + HTML riport archiválás

**Ajánlott induló konfiguráció:**

| Szerep | Ajánlott gép | Darabszám |
|---|---:|---:|
| JMeter controller | VPS 4GB vagy VPS 8GB | 1 |
| JMeter worker | VPS 16GB vagy VPS 32GB | 2 |
| Monitoring / riport | controllerrel közösen vagy külön kis VPS-en | 0–1 |

---

## Technikai kiindulópont JMeterhez

JMeter esetén a teljesítményteszt generátor oldali erőforrásigénye főleg az alábbiaktól függ:

- párhuzamos threadek / virtuális felhasználók száma;
- sampler típusok;
- TLS/HTTPS terhelés;
- response body méret;
- assertionök mennyisége;
- használt listenerök;
- JMeter heap méret;
- hálózati késleltetés és sávszélesség;
- célrendszer földrajzi elhelyezkedése.

Éles teljesítményteszteléshez **JMeter GUI mód nem ajánlott**. A futtatás javasolt módja:

```bash
jmeter -n -t test-plan.jmx -l results.jtl -e -o ./report
```

Nagyobb teszteknél javasolt architektúra:

```diagram
controller: JMeter Controller
worker1: JMeter Worker 1
worker2: JMeter Worker 2
workerN: "JMeter Worker N"
monitoring: "Monitoring\n(Grafana / Prometheus / InfluxDB)"

controller -> worker1: RMI / SSH
controller -> worker2: RMI / SSH
controller -> workerN: RMI / SSH
controller -> monitoring: Backend Listener
```

---

## Terhelési szintek és erőforrásigény

### Terhelési szintek áttekintése

Az alábbi táblázat összefoglalja, hogy az egyes óránkénti kérésszám-szintekhez milyen JMeter generátor oldali erőforrásigény becsülhető. A threadszám becslés alapképlete:

```text
threadek ≈ TPS × (átlagos_válaszidő_s + think_time_s)
```

Példa: 278 TPS, 200 ms válaszidő, 1 s think time → 278 × 1,2 ≈ 334 thread.

Az alábbi értékek 100–500 ms átlagos válaszidőre és 0–1 s think time-ra vonatkoznak:

| Terhelési szint | Equivalent req/s | Becsült threadszám | Becsült JMeter heap | Javasolt worker |
|---|---:|---:|---:|---|
| 1 000 req/h | ~0,3 req/s | 1–10 | 512 MB – 1 GB | 1× VPS 4GB |
| 10 000 req/h | ~3 req/s | 5–50 | 1–2 GB | 1× VPS 8GB |
| 100 000 req/h | ~28 req/s | 50–300 | 2–4 GB | 1× VPS 16GB |
| 1 000 000 req/h | ~278 req/s | 300–1 500 | 8–16 GB | 2× VPS 32GB |
| 10 000 000 req/h | ~2 778 req/s | 2 000–8 000+ | több worker szükséges | 4–8× VPS 64GB vagy cloud burst |

### Javasolt Rackhost konfiguráció terhelési szintenként

| Terhelési szint | Controller | Worker csomag | Worker db | Becsült nettó havi díj |
|---|---|---|---:|---:|
| 1 000 req/h | VPS 4GB | VPS 4GB | 1 | 9 000 Ft |
| 10 000 req/h | VPS 4GB | VPS 8GB | 1 | 12 500 Ft |
| 100 000 req/h | VPS 8GB | VPS 16GB | 1–2 | 22 000 – 36 000 Ft |
| 1 000 000 req/h | VPS 8GB | VPS 32GB | 2 | 52 000 Ft |
| 10 000 000 req/h | VPS 16GB | VPS 64GB | 4–8 | 182 000 – 350 000 Ft |

> **Megjegyzés:** 10 000 000 req/h szinten a JMeter önmaga is szűk keresztmetszetté válhat. Ilyen extrém terhelésnél a cloud burst megközelítés (AWS/Azure több worker) vagy alternatív eszköz (pl. Gatling, k6) mérlegelése javasolt.

### Terhelési szint jellemzői és tipikus alkalmazási területek

| Terhelési szint | Jelleg | Tipikus alkalmazás |
|---|---|---|
| 1 000 req/h | minimális | fejlesztői smoke test, pipeline validáció |
| 10 000 req/h | alacsony | kisebb API validáció, integrációs teszt |
| 100 000 req/h | közepes | éles normál terhelési profil szimulációja |
| 1 000 000 req/h | magas | csúcsterhelési / stresszteszt |
| 10 000 000 req/h | extrém | kapacitásvizsgálat, skálázhatósági határok keresése |

### Kritikus figyelési pontok magas terhelésnél

100 000 req/h felett a generátor oldal monitorozása különösen fontos:

- **CPU saturáció:** JMeter worker CPU 80–90% felett a terhelésgenerálás torzulhat;
- **GC pause:** heap telítettség esetén Java GC szünetek keletkeznek, latencia spike-ok jelennek meg;
- **TCP port exhaustion:** ~28 000 nyitott TCP kapcsolat felett TIME_WAIT torlódás léphet fel;
- **File descriptor limit:** `ulimit -n` növelése szükséges lehet (javasolt: 65 536+);
- **Hálózati sávszélesség:** nagy response bodyknál a worker NIC sávszélessége korlátot jelenthet;
- **JMeter error rate emelkedése:** ha a worker nem bírja a tempót, mesterséges hibák és timeoutok jelennek meg.

Javasolt OS tuning nagyobb terhelésnél:

```bash
# TCP TIME_WAIT gyorsabb felszabadítása
echo "net.ipv4.tcp_tw_reuse=1" >> /etc/sysctl.conf
# Nyitható portok száma
echo "net.ipv4.ip_local_port_range=1024 65535" >> /etc/sysctl.conf
# File descriptor limit emelése
ulimit -n 65536
sysctl -p
```

---

## Rackhost VPS opciók

A Rackhost VPS tarifák a https://www.rackhost.hu/virtualis-szerver oldalról kerültek be a dokumentumba (lekérdezve: 2026. június 26.).

### Rackhost VPS csomagok

| Csomag | CPU mag | RAM | SSD | Adatforgalom | Nettó díj / hó | Bruttó díj / hó |
|---|---:|---:|---:|---:|---:|---:|
| VPS 1GB | 1 | 1 GB | 20 GB | 10 TB | 1 500 Ft | 1 905 Ft |
| VPS 2GB | 1 | 2 GB | 40 GB | 10 TB | 2 500 Ft | 3 175 Ft |
| VPS 4GB | 2 | 4 GB | 60 GB | 20 TB | 4 500 Ft | 5 715 Ft |
| VPS 8GB | 2 | 8 GB | 80 GB | 20 TB | 8 000 Ft | 10 160 Ft |
| VPS 12GB | 4 | 12 GB | 120 GB | 30 TB | 10 000 Ft | 12 700 Ft |
| VPS 16GB | 4 | 16 GB | 160 GB | 30 TB | 14 000 Ft | 17 780 Ft |
| VPS 24GB | 6 | 24 GB | 200 GB | 30 TB | 18 000 Ft | 22 860 Ft |
| VPS 32GB | 8 | 32 GB | 240 GB | 30 TB | 22 000 Ft | 27 940 Ft |
| VPS 48GB | 10 | 48 GB | 300 GB | 40 TB | 32 000 Ft | 40 640 Ft |
| VPS 64GB | 12 | 64 GB | 400 GB | 40 TB | 42 000 Ft | 53 340 Ft |
| VPS 80GB | 14 | 80 GB | 500 GB | 60 TB | 52 000 Ft | 66 040 Ft |
| VPS 96GB | 16 | 96 GB | 600 GB | 60 TB | 62 000 Ft | 78 740 Ft |

### Rackhost értékelés

**Előnyök:**

- kedvező havi fix díj;
- nagy adatforgalmi keret;
- egyszerű költségtervezés;
- magyar szolgáltató;
- alacsonyabb adminisztratív és üzemeltetési komplexitás;
- dedikált VPS erőforrások;
- 99,9%-os rendelkezésre állási vállalás a Rackhost VPS szolgáltatás leírása szerint.

**Hátrányok:**

- kisebb rugalmasság AWS/Azure környezethez képest;
- nincs natív autoscaling;
- nincs több régiós globális terhelésgenerálás;
- automatizálhatósága korlátozottabb lehet;
- nagy burst tesztekhez előre kell méretezni.

### Rackhost ajánlott JMeter konfigurációk

#### Belépő tesztkörnyezet

| Komponens | Csomag | Darabszám | Nettó havi díj |
|---|---|---:|---:|
| Controller | VPS 4GB | 1 | 4 500 Ft |
| Worker | VPS 8GB | 1 | 8 000 Ft |
| **Összesen** |  |  | **12 500 Ft + ÁFA / hó** |

**Felhasználás:** kisebb fejlesztői vagy validációs teljesítménytesztek.

#### Normál JMeter cluster

| Komponens | Csomag | Darabszám | Nettó havi díj |
|---|---|---:|---:|
| Controller | VPS 8GB | 1 | 8 000 Ft |
| Worker | VPS 16GB | 2 | 28 000 Ft |
| **Összesen** |  |  | **36 000 Ft + ÁFA / hó** |

**Felhasználás:** rendszeres API/web teljesítménytesztelés, közepes terheléssel.

#### Ajánlott ár-érték konfiguráció

| Komponens | Csomag | Darabszám | Nettó havi díj |
|---|---|---:|---:|
| Controller | VPS 8GB | 1 | 8 000 Ft |
| Worker | VPS 32GB | 2 | 44 000 Ft |
| **Összesen** |  |  | **52 000 Ft + ÁFA / hó** |

**Felhasználás:** rendszeres, stabil teljesítménytesztelés, nagyobb threadszámmal és jobb tartalékkal.

#### Nagyobb belső tesztpark

| Komponens | Csomag | Darabszám | Nettó havi díj |
|---|---|---:|---:|
| Controller | VPS 16GB | 1 | 14 000 Ft |
| Worker | VPS 32GB | 4 | 88 000 Ft |
| **Összesen** |  |  | **102 000 Ft + ÁFA / hó** |

**Felhasználás:** nagyobb kampánytesztek, több párhuzamos workerrel.

#### Elosztott 10 kisebb munkás konfiguráció

Abban az esetben, ha a terhelésgenerálást nem kevés, nagy memóriájú worker gépen, hanem 10 kisebb teljesítményű worker node-on kell elosztani, az alábbi konfiguráció javasolható. Minden JMeter JVM kisebb heapet kezel, ami csökkenti a GC nyomást, a 10 különböző forrás IP-cím reálisabb terhelést szimulál, és egy-egy node kiesése csak 10%-os kapacitásveszteséget jelent.

| Komponens | Csomag | Darabszám | Nettó havi díj |
|---|---|---:|---:|
| Controller | VPS 8GB | 1 | 8 000 Ft |
| Worker | VPS 4GB | 10 | 45 000 Ft |
| **Összesen** |  |  | **53 000 Ft + ÁFA / hó** |

**Felhasználás:** közepes terhelésgenerálás elosztott topológiával, ha a forrás IP-diverzitás vagy a GC stabilitás fontos szempont. Összehasonlítható ár-szinten a „normál 2 worker" konfigurációval, de az össz-threadszám eloszlása kedvezőbb.

Magasabb kapacitású, emelt workerű változat:

| Komponens | Csomag | Darabszám | Nettó havi díj |
|---|---|---:|---:|
| Controller | VPS 16GB | 1 | 14 000 Ft |
| Worker | VPS 8GB | 10 | 80 000 Ft |
| **Összesen** |  |  | **94 000 Ft + ÁFA / hó** |

**Felhasználás:** nagy threadszámú (~5 000+ párhuzamos felhasználó), elosztott terhelésgenerálás, ahol a teljesítményteszt forrásoldali stabilitása és a IP-diverzitás elsődleges szempont.

**Megjegyzés:** 10 worker esetén a JMeter controller több párhuzamos RMI kapcsolatot tart fenn, és az eredmény-aggregáció is erőforrásigényesebb. Ezért az elosztott konfigurációhoz legalább VPS 8GB controller ajánlott. A workerek OS-szintű tuningleírása megegyezik a nagy worker esetével (ulimit, tcp_tw_reuse, GC beállítás).

---

## AWS EC2 opciók

Az AWS EC2 előnye az óradíjas, rugalmasan indítható infrastruktúra. Az On-Demand modellnél nincs előleg és nincs hosszú távú elköteleződés, viszont folyamatos futás esetén a havi költség jelentős lehet.

### Vizsgált AWS instance típusok

A számítások 730 óra/hó becsléssel és kb. **310 HUF / USD** árfolyammal készültek.

| Instance | vCPU | RAM | On-Demand USD / óra | Becsült nettó HUF / hó | JMeter értékelés |
|---|---:|---:|---:|---:|---|
| t3.xlarge | 4 | 16 GiB | kb. 0,1664 USD | kb. 37 700 Ft | rövid, nem tartós CPU-heavy tesztekhez |
| m7i.xlarge | 4 | 16 GiB | kb. 0,2016 USD | kb. 45 600 Ft | stabilabb 4 vCPU worker |
| m7i.2xlarge | 8 | 32 GiB | kb. 0,4032 USD | kb. 91 300 Ft | erős általános worker |
| c7i.2xlarge | 8 | 16 GiB | kb. 0,3570 USD | kb. 80 800 Ft | CPU-orientált worker |

### AWS értékelés

**Előnyök:**

- gyorsan indítható és törölhető worker node-ok;
- jól automatizálható Terraformmal, Ansible-lel, CI/CD pipeline-ból;
- több régióból is indítható terhelés;
- On-Demand, Reserved, Savings Plan és Spot opciók;
- jó integráció monitoringgal és logolással.

**Hátrányok:**

- 0–24 futás esetén drágább, mint a Rackhost VPS;
- EBS disk, publikus IP, NAT, adatforgalom és monitoring külön költségtényező lehet;
- a burstable T-széria tartós CPU-terhelésre nem ideális;
- költségkontroll nélkül könnyen elszaladhat a számla.

### AWS ajánlott használati mód

AWS akkor javasolt, ha:

- a tesztelés kampányszerű;
- a worker node-ok csak a teszt idejére indulnak el;
- fontos az automatizálhatóság;
- több földrajzi régióból kell terhelni;
- a szervezet már AWS-t használ.

Példa alkalmi futtatásra:

| Konfiguráció | Futási idő | Becsült compute költség |
|---|---:|---:|
| 4 × m7i.2xlarge worker | 40 óra / hó | kb. 64,5 USD, azaz kb. 20 000 Ft |
| 4 × m7i.xlarge worker | 40 óra / hó | kb. 32,3 USD, azaz kb. 10 000 Ft |

A fenti értékek csak compute költséget jelentenek. Disk, IP, adatforgalom, logolás és monitoring külön számolandó.

---

## Microsoft Azure Virtual Machines opciók

Az Azure VM hasonló előnyt ad, mint az AWS: óradíjas, skálázható, automatizálható infrastruktúrát. Vállalati Microsoft / Entra ID / Azure környezetben könnyebben illeszthető lehet meglévő üzemeltetési és jogosultsági folyamatokhoz.

### Vizsgált Azure VM típusok

A számítások 730 óra/hó becsléssel és kb. **310 HUF / USD** árfolyammal készültek.

| VM típus | vCPU | RAM | On-Demand USD / óra | Becsült nettó HUF / hó | JMeter értékelés |
|---|---:|---:|---:|---:|---|
| D4as v5 | 4 | 16 GiB | kb. 0,172 USD | kb. 38 900 Ft | jó általános 4 vCPU worker |
| D8as v5 | 8 | 32 GiB | kb. 0,344 USD | kb. 77 800 Ft | ajánlott cloud worker |
| F8s v2 | 8 | 16 GiB | kb. 0,339 USD | kb. 76 700 Ft | CPU-orientált worker |

### Azure értékelés

**Előnyök:**

- jó illeszkedés Microsoft / Azure vállalati környezetbe;
- óradíjas használat;
- automatizálható Azure CLI-vel, Terraformmal, pipeline-ból;
- több régiós futtatás;
- Spot VM lehetőség olcsóbb, megszakítható tesztekhez.

**Hátrányok:**

- folyamatos futás esetén jellemzően drágább, mint Rackhost;
- managed disk, publikus IP, adatforgalom és monitoring külön díjtétel;
- a leállított, de nem deallocated VM tovább számlázható lehet;
- költségfigyelés nélkül nehezebben tervezhető, mint egy fix havi VPS.

### Azure ajánlott használati mód

Azure akkor javasolt, ha:

- a szervezet Microsoft/Azure orientált;
- a célrendszer Azure-ban fut;
- fontos az Entra ID és vállalati jogosultságkezelés;
- a tesztek nem állandóak, hanem időszakosan futnak;
- a node-ok automatikus indítása és törlése megoldható.

Példa alkalmi futtatásra:

| Konfiguráció | Futási idő | Becsült compute költség |
|---|---:|---:|
| 4 × D8as v5 worker | 40 óra / hó | kb. 55 USD, azaz kb. 17 000 Ft |
| 4 × D4as v5 worker | 40 óra / hó | kb. 27,5 USD, azaz kb. 8 500 Ft |

A fenti értékek csak compute költséget jelentenek. Disk, IP, adatforgalom, logolás és monitoring külön számolandó.

---

## Összehasonlító értékelés

### Fix havi működés

| Alternatíva | Erőforrás | Becsült havi költség | Értékelés |
|---|---|---:|---|
| Rackhost – 2 nagy worker | 2 × VPS 32GB + 1 × VPS 8GB | 52 000 Ft + ÁFA | legjobb ár-érték fix használatra |
| Rackhost – 10 kis worker | 10 × VPS 4GB + 1 × VPS 8GB | 53 000 Ft + ÁFA | hasonló ár, jobb IP-diverzitás, kisebb GC nyomás |
| Rackhost – 10 közepes worker | 10 × VPS 8GB + 1 × VPS 16GB | 94 000 Ft + ÁFA | nagy threadszám elosztottan, IP-diverzitással |
| AWS | 2 × m7i.2xlarge + 1 × m7i.xlarge | kb. 228 000 Ft + járulékos költségek | jó, de drága 0–24 használatra |
| Azure | 2 × D8as v5 + 1 × D4as v5 | kb. 194 500 Ft + járulékos költségek | jó, de drága 0–24 használatra |

### Alkalmi, havi 40 órás kampányteszt

| Alternatíva | Erőforrás | Futási idő | Becsült compute költség | Értékelés |
|---|---|---:|---:|---|
| AWS | 4 × m7i.2xlarge | 40 óra | kb. 20 000 Ft | jó rugalmas opció |
| Azure | 4 × D8as v5 | 40 óra | kb. 17 000 Ft | jó rugalmas opció |
| Rackhost | fix VPS cluster | egész hónap | 52 000 Ft + ÁFA | akkor jó, ha rendszeresen kell |

### Minőségi összehasonlítás

| Szempont | Rackhost VPS | AWS EC2 | Azure VM |
|---|---:|---:|---:|
| Fix havi ár-érték | 5/5 | 2/5 | 2/5 |
| Alkalmi futtatás | 3/5 | 5/5 | 5/5 |
| Automatizálhatóság | 3/5 | 5/5 | 5/5 |
| Több régiós terhelés | 1/5 | 5/5 | 5/5 |
| Költségtervezhetőség | 5/5 | 3/5 | 3/5 |
| Adatforgalmi keret | 5/5 | 2/5 | 2/5 |
| Vállalati cloud integráció | 2/5 | 5/5 | 5/5 |
| Üzemeltetési egyszerűség | 4/5 | 3/5 | 3/5 |

---

## Kockázatok és figyelendő pontok

### Terhelésgenerátor limitáció

A teljesítményteszt nem tekinthető hitelesnek, ha a JMeter workerök válnak szűk keresztmetszetté. Figyelni kell:

- CPU terhelés;
- heap usage;
- GC pause;
- hálózati throughput;
- open file limit;
- TCP port exhaustion;
- error rate;
- JMeter sampler latency;
- worker oldali throttling.

### Hálózati költségek

AWS és Azure esetén a kimenő adatforgalom külön költségtétel lehet. Nagy response bodykkal futó tesztek esetén ez releváns költségkockázat.

### Mérési torzítás

A szolgáltató és a célrendszer földrajzi távolsága befolyásolja:

- latency;
- throughput;
- TLS handshake idő;
- CDN útvonal;
- hálózati stabilitás.

Ha a célrendszer magyar vagy közép-európai felhasználókat szolgál ki, Rackhost vagy európai cloud régió indokolt.

### Spot instance / Spot VM kockázat

AWS Spot és Azure Spot gépek kedvező árúak, de megszakíthatók. Hivatalos elfogadási teljesítményteszthez csak akkor javasoltak, ha a worker kiesés kezelve van.

---

## Ajánlott implementációs modell

### Javasolt alap architektúra

```diagram
controller: "JMeter Controller\ntest plan · JTL gyűjtés\nHTML riport · backend listener"
worker: "JMeter Worker 1..N\nterhelésgenerálás · non-GUI\nfix JVM heap · OS tuning"
monitoring: "Monitoring\nGrafana · Prometheus / InfluxDB\nnode exporter · backend listener"

controller -> worker: orchestration / RMI
controller -> monitoring: backend listener
worker -> monitoring: metrikák
```

### Operációs rendszer

Javasolt:

- Ubuntu Server LTS
- Debian stable
- Rocky Linux / AlmaLinux, ha RHEL-kompatibilitás kell

### JMeter futtatási szabályok

- GUI mód kizárása éles mérésnél;
- listenerök minimalizálása;
- JTL írás csak szükséges mezőkkel;
- JVM heap fixálása;
- worker node-ok CPU/RAM/network monitorozása;
- előzetes kalibrációs teszt;
- célrendszer és generátor oldali metrikák együttes értelmezése.

Példa JVM beállítás:

```bash
export JVM_ARGS="-Xms4g -Xmx4g -XX:+UseG1GC"
```

Nagyobb worker esetén:

```bash
export JVM_ARGS="-Xms8g -Xmx8g -XX:+UseG1GC"
```

---

## Végső döntési javaslat

### Elsődleges választás

**Rackhost VPS alapú JMeter cluster.**

Ajánlott konfiguráció:

| Komponens | Csomag | Darabszám |
|---|---|---:|
| Controller | VPS 8GB | 1 |
| Worker | VPS 32GB | 2 |

Becsült havi költség:

```text
1 × 8 000 Ft + 2 × 22 000 Ft = 52 000 Ft + ÁFA / hó
```

**Indoklás:**

- legjobb fix havi ár-érték arány;
- nagy adatforgalmi keret;
- egyszerű költségtervezés;
- elegendő CPU és RAM JMeter worker feladatra;
- alkalmas rendszeres teljesítménytesztelési feladatokra;
- nem keletkezik nehezen kontrollálható cloud egress költség.

### Másodlagos választás

**AWS vagy Azure óradíjas workerpark.**

Akkor javasolt, ha:

- a tesztelés nem folyamatos;
- a node-ok csak a teszt idejére indulnak;
- fontos a pipeline alapú automatizálás;
- több régióból kell terhelést generálni;
- a szervezet cloud governance szempontból AWS-t vagy Azure-t preferál.

### Nem javasolt elsődlegesnek

**0–24-ben futó AWS/Azure On-Demand JMeter workerpark.**

Indok:

- azonos vagy hasonló erőforrás mellett többszörös havi költség;
- külön disk/IP/egress/monitoring díjak;
- költségtervezése bonyolultabb;
- fix havi használatnál a Rackhost kedvezőbb.

---

## Döntési mátrix

| Döntési helyzet | Javasolt megoldás |
|---|---|
| Rendszeres havi teljesítményteszt | Rackhost VPS |
| Fix költségkeret fontos | Rackhost VPS |
| Magyar / EU célrendszer terhelése | Rackhost VPS vagy EU cloud régió |
| Forrás IP-diverzitás fontos | Rackhost 10 × VPS 4GB worker |
| GC-stabilitás, kisebb heap/JVM kell | Rackhost 10 × VPS 4GB worker |
| Nagy threadszám elosztottan, IP-diverzitással | Rackhost 10 × VPS 8GB worker |
| Ritka, kampányszerű teszt | AWS vagy Azure óradíjas VM |
| CI/CD pipeline-ból indított teszt | AWS vagy Azure |
| Több régiós terhelés | AWS vagy Azure |
| Nagyon olcsó, megszakítható burst | AWS Spot vagy Azure Spot |
| Hivatalos elfogadási mérés | Rackhost fix VPS vagy On-Demand cloud, nem Spot |

---

## Összegzés

A jelenlegi adatok alapján a **Rackhost VPS 8GB controller + 2 × VPS 32GB worker** konfiguráció a legjobb elsődleges választás JMeter alapú teljesítményteszteléshez.

Ez a megoldás megfelelő egyensúlyt ad:

- ár;
- erőforrás;
- stabilitás;
- egyszerű üzemeltetés;
- nagy adatforgalmi keret;
- havi költségtervezhetőség.

Az AWS és Azure megoldások alternatívaként szerepeljenek a döntésben, elsősorban időszakos, automatizált vagy több régiós terhelésgenerálási igény esetére.

---

## Források és ellenőrzési pontok

A dokumentum készítésekor figyelembe vett források és adatok:

1. Rackhost VPS tarifák  
   https://www.rackhost.hu/virtualis-szerver  
   (lekérdezve: 2026. június 26.)

2. AWS EC2 On-Demand pricing  
   https://aws.amazon.com/ec2/pricing/on-demand/

3. AWS EC2 általános pricing modell  
   https://aws.amazon.com/ec2/pricing/

4. Azure Linux Virtual Machines pricing  
   https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/

5. AWS instance ár- és specifikációs összehasonlító adatok  
   https://instances.vantage.sh/

6. Azure VM ár- és specifikációs összehasonlító adatok  
   https://instances.vantage.sh/azure

7. USD/HUF árfolyam-ellenőrzés  
   https://wise.com/gb/currency-converter/usd-to-huf-rate  
   https://www.xe.com/currencyconverter/convert/?Amount=1&From=USD&To=HUF

---

## Megjegyzés

A cloud árak régió, operációs rendszer, disk típus, adatforgalom, publikus IP, monitoring és szerződéses konstrukció szerint változhatnak. Végleges beszerzési döntés előtt szükséges:

- AWS Pricing Calculator ellenőrzés;
- Azure Pricing Calculator ellenőrzés;
- Rackhost aktuális ajánlat visszaigazolása;
- várható havi tesztórák becslése;
- várható adatforgalom becslése;
- célrendszer földrajzi és hálózati elhelyezkedésének vizsgálata.

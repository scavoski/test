# Software-Defined Storage (SDS) u Cloud Infrastrukturi

**Autor:** Računarski fakultet, Beograd
**Verzija:** 1.0

---

## Sadržaj

1. [Uvod u SDS](#1-uvod-u-sds)
2. [SDS Storage Modeli](#2-sds-storage-modeli)
3. [Ceph - Open Source SDS](#3-ceph---open-source-sds)
4. [VMware vSAN - Enterprise SDS](#4-vmware-vsan---enterprise-sds)
5. [Ceph vs vSAN Poređenje](#5-ceph-vs-vsan-poređenje)
6. [SDS u Cloud Infrastrukturi](#6-sds-u-cloud-infrastrukturi)
7. [Sizing i Best Practices](#7-sizing-i-best-practices)

---

## 1. Uvod u SDS

### Šta je Software-Defined Storage?

**Software-Defined Storage (SDS)** je arhitektura skladištenja podataka u kojoj softver, a ne hardver, upravlja svim funkcijama storage sistema — uključujući provisioning, replikaciju, deduplication, snapshot-e i disaster recovery.

Tradicionalni storage sistemi (SAN/NAS) koriste specijalizovani hardver sa ugrađenim kontrolerima. SDS taj pristup transformiše:

| Aspekt | Tradicionalni Storage | SDS |
|--------|----------------------|-----|
| Hardver | Specijalizovani uređaji | Commodity x86 serveri |
| Upravljanje | Hardverski kontroler | Softverski sloj |
| Skaliranje | Scale-up (veći uređaj) | Scale-out (više čvorova) |
| Vendor lock-in | Visok | Nizak |
| Cena | Visoka (CAPEX) | Niža, fleksibilnija |

### Zašto SDS u Cloud okruženju?

SDS je prirodan izbor za cloud infrastrukturu jer omogućava:

- **Elastičnost** — dodavanje kapaciteta bez downtime-a
- **Automatizaciju** — API-driven provisioning
- **Multi-tenancy** — izolacija podataka između korisnika
- **Hardware independence** — izbor vendor-a po potrebi

### Control Plane vs Data Plane

Ključni arhitekturni koncept u SDS sistemima je razdvajanje **Control Plane** i **Data Plane**.

```mermaid
flowchart TB
    subgraph CP["Control Plane"]
        CM[Cluster Manager]
        META[Metadata Service]
        MON[Health Monitoring]
        POLICY[Policy Engine]
    end

    subgraph DP["Data Plane"]
        SD1[Storage Daemon 1]
        SD2[Storage Daemon 2]
        SD3[Storage Daemon 3]
    end

    subgraph DISK["Physical Storage"]
        D1[(SSD)]
        D2[(HDD)]
        D3[(NVMe)]
    end

    CM --> SD1
    CM --> SD2
    CM --> SD3

    SD1 --> D1
    SD2 --> D2
    SD3 --> D3

    MON -.-> SD1
    MON -.-> SD2
    MON -.-> SD3
```

**Control Plane** je odgovoran za:
- Cluster membership i quorum
- Metadata management
- Placement decisions (gde idu podaci)
- Health monitoring i alerting

**Data Plane** obavlja:
- Read/Write operacije
- Replication između čvorova
- Erasure coding
- Data recovery

### SDS Slojevita Arhitektura

```mermaid
flowchart TD
    subgraph APPS["Applications / Clients"]
        VM[Virtual Machines]
        K8S[Kubernetes]
        DB[Databases]
    end

    subgraph INTERFACE["Storage Interface Layer"]
        BLOCK[Block API<br/>iSCSI / RBD / NVMe-oF]
        FILE[File API<br/>NFS / SMB / CephFS]
        OBJ[Object API<br/>S3 / Swift]
    end

    subgraph CONTROL["Control Plane"]
        CMGR[Cluster Manager]
        PLACE[Placement Algorithm]
        HEALTH[Health Monitor]
    end

    subgraph DATA["Data Plane"]
        REPL[Replication Engine]
        EC[Erasure Coding]
        CACHE[Caching Layer]
    end

    subgraph PHYS["Physical Infrastructure"]
        NODE1[Node 1<br/>CPU / RAM / NIC]
        NODE2[Node 2<br/>CPU / RAM / NIC]
        NODE3[Node 3<br/>CPU / RAM / NIC]
    end

    APPS --> INTERFACE
    INTERFACE --> CONTROL
    CONTROL --> DATA
    DATA --> PHYS
```

---

## 2. SDS Storage Modeli

SDS sistemi mogu da izlože storage na tri načina iz istog fizičkog pool-a.

```mermaid
flowchart LR
    subgraph POOL["Storage Pool"]
        SSD1[(SSD)]
        SSD2[(SSD)]
        HDD1[(HDD)]
        HDD2[(HDD)]
    end

    POOL --> BLOCK["Block Storage<br/>/dev/rbd0"]
    POOL --> FILE["File Storage<br/>/mnt/cephfs"]
    POOL --> OBJECT["Object Storage<br/>s3://bucket"]

    BLOCK --> VM[VM Disk]
    BLOCK --> DATABASE[Database]

    FILE --> SHARE[File Share]
    FILE --> HOME[Home Dirs]

    OBJECT --> BACKUP[Backup]
    OBJECT --> ARCHIVE[Archive]
```

### Block Storage

Block storage prezentuje storage kao raw disk device. Host vidi LUN/volume kao lokalni disk.

**Protokoli:** iSCSI, Fibre Channel, NVMe-oF, RBD (Ceph)

**Use cases:**
- Virtual Machine disks
- Databases (MySQL, PostgreSQL, Oracle)
- Transakcioni sistemi

**Karakteristike:**
- Niska latencija
- Visok IOPS
- Host upravlja filesystem-om

### File Storage

File storage izlaže hijerarhijski filesystem preko mreže.

**Protokoli:** NFS, SMB/CIFS, CephFS

**Use cases:**
- Shared home directories
- Kolaboracija na dokumentima
- Application data

**Karakteristike:**
- Poznata struktura (folderi/fajlovi)
- File locking podrška
- Lako deljenje

### Object Storage

Object storage čuva podatke kao objekte sa metadata i unique ID-em.

**Protokoli:** S3 API, Swift API, REST/HTTP

**Use cases:**
- Backup i arhiviranje
- Big Data / Analytics
- Cloud-native aplikacije
- Media storage

**Karakteristike:**
- Ogromna skalabilnost (petabytes+)
- Rich metadata
- Eventual consistency (tipično)
- HTTP pristup

---

## 3. Ceph - Open Source SDS

**Ceph** je open-source, distribuirani storage sistem koji implementira Block, File i Object storage u jednoj platformi. Koriste ga CERN, Deutsche Telekom, Bloomberg i mnogi cloud provajderi.

### 3.1 Ceph Arhitektura

```mermaid
flowchart TD
    subgraph CLIENTS["Clients"]
        RBD[RBD Client<br/>Block]
        CEPHFS[CephFS Client<br/>File]
        RGW[RGW Client<br/>Object/S3]
    end

    subgraph CONTROL["Control Plane"]
        MON1[MON 1]
        MON2[MON 2]
        MON3[MON 3]
        MGR[MGR<br/>Dashboard]
    end

    subgraph MDS_LAYER["Metadata (CephFS only)"]
        MDS1[MDS 1]
        MDS2[MDS 2]
    end

    subgraph DATA["Data Plane - OSD Nodes"]
        subgraph NODE1["Node 1"]
            OSD1[OSD.0]
            OSD2[OSD.1]
            OSD3[OSD.2]
        end
        subgraph NODE2["Node 2"]
            OSD4[OSD.3]
            OSD5[OSD.4]
            OSD6[OSD.5]
        end
        subgraph NODE3["Node 3"]
            OSD7[OSD.6]
            OSD8[OSD.7]
            OSD9[OSD.8]
        end
    end

    CLIENTS --> CONTROL
    CEPHFS --> MDS_LAYER
    CONTROL --> DATA
    MDS_LAYER --> DATA
```

#### Ceph Servisi

| Servis | Funkcija | Minimum |
|--------|----------|---------|
| **MON** (Monitor) | Cluster state, quorum, CRUSH map | 3 (odd number) |
| **MGR** (Manager) | Monitoring, dashboard, alerts | 2 (HA) |
| **OSD** (Object Storage Daemon) | Čuva podatke, replication | 3+ |
| **MDS** (Metadata Server) | Metadata za CephFS | 2 (ako koristiš CephFS) |
| **RGW** (RADOS Gateway) | S3/Swift API | 2+ (load balanced) |

#### OSD - Srce Ceph sistema

Svaki disk u Ceph clusteru ima svoj OSD daemon:

```
Node 1
├── OSD.0 ← SSD 1
├── OSD.1 ← SSD 2
└── OSD.2 ← HDD 1

Node 2
├── OSD.3 ← SSD 1
├── OSD.4 ← HDD 1
└── OSD.5 ← HDD 2
```

OSD je odgovoran za:
- Čuvanje objekata (data + metadata)
- Replication ka drugim OSD-ovima
- Recovery kada OSD otkaže
- Scrubbing (provera integriteta)

### 3.2 CRUSH Algoritam

**CRUSH** (Controlled Replication Under Scalable Hashing) je algoritam koji određuje gde se podaci fizički čuvaju — bez centralnog metadata servera.

#### Zašto je CRUSH revolucionaran?

Tradicionalni pristup:
```
File → Metadata Server → "File je na Disk 47"
```
Problem: Metadata server je bottleneck i single point of failure.

CRUSH pristup:
```
File → hash(file_id) + CRUSH rules → Izračunaj lokaciju
```
Svaki klijent može izračunati gde su podaci!

```mermaid
sequenceDiagram
    participant Client
    participant CRUSH as CRUSH Algorithm
    participant MON as Monitor
    participant OSD1 as OSD.3 (Primary)
    participant OSD2 as OSD.7 (Replica)
    participant OSD3 as OSD.11 (Replica)

    Client->>MON: Get CRUSH Map
    MON-->>Client: CRUSH Map + Cluster State

    Client->>CRUSH: hash("photo.jpg")
    CRUSH-->>Client: PG 2.4a → OSD.3, OSD.7, OSD.11

    Client->>OSD1: Write photo.jpg
    OSD1->>OSD2: Replicate
    OSD1->>OSD3: Replicate
    OSD2-->>OSD1: ACK
    OSD3-->>OSD1: ACK
    OSD1-->>Client: Write Complete
```

#### CRUSH Hijerarhija (Failure Domains)

CRUSH koristi hijerarhijsku mapu klastera za pametno raspoređivanje replika:

```mermaid
flowchart TD
    ROOT[Root: default]

    ROOT --> DC1[Datacenter: DC1]
    ROOT --> DC2[Datacenter: DC2]

    DC1 --> RACK1[Rack: rack1]
    DC1 --> RACK2[Rack: rack2]

    DC2 --> RACK3[Rack: rack3]
    DC2 --> RACK4[Rack: rack4]

    RACK1 --> HOST1[Host: node01]
    RACK1 --> HOST2[Host: node02]

    RACK2 --> HOST3[Host: node03]
    RACK2 --> HOST4[Host: node04]

    HOST1 --> OSD0[OSD.0]
    HOST1 --> OSD1[OSD.1]
    HOST2 --> OSD2[OSD.2]
    HOST2 --> OSD3[OSD.3]
```

**CRUSH rule primer:**
```
Replike moraju biti na različitim RACK-ovima
```
Rezultat:
- Replica 1 → rack1/node01/OSD.0
- Replica 2 → rack2/node03/OSD.4
- Replica 3 → rack3/node05/OSD.8

Ako ceo rack izgubi struju — podaci su i dalje dostupni!

### 3.3 Data Protection

#### Replication

Najjednostavniji način zaštite podataka.

```
replication_factor = 3
```

```mermaid
sequenceDiagram
    participant Client
    participant Primary as Primary OSD
    participant Replica1 as Replica OSD 1
    participant Replica2 as Replica OSD 2

    Client->>Primary: Write Block X

    par Parallel Replication
        Primary->>Replica1: Replicate Block X
        Primary->>Replica2: Replicate Block X
    end

    Replica1-->>Primary: ACK
    Replica2-->>Primary: ACK
    Primary-->>Client: Write Success
```

**Karakteristike:**
- Overhead: 3x storage za replication=3
- Performanse: Brži recovery
- Jednostavnost: Lako razumljiv

#### Erasure Coding

Napredniji način koji koristi manje prostora.

```
k = 4 (data chunks)
m = 2 (parity chunks)
```

```
Original: [D1] [D2] [D3] [D4]
Encoded:  [D1] [D2] [D3] [D4] [P1] [P2]
```

Može izgubiti bilo koja 2 chunk-a i rekonstruisati podatke.

**Karakteristike:**
- Overhead: 1.5x storage za k=4, m=2
- Performanse: Sporiji recovery (računanje)
- Kompleksnost: CPU intensive

| Metoda | Overhead | Recovery Speed | CPU Usage |
|--------|----------|----------------|-----------|
| Replication (3x) | 200% | Brz | Nizak |
| EC (4+2) | 50% | Srednji | Visok |
| EC (8+3) | 37.5% | Spor | Vrlo visok |

### 3.4 Failure Handling

```mermaid
flowchart TD
    FAIL[OSD.5 Failure Detected]

    FAIL --> MON_DETECT[MON detektuje timeout]
    MON_DETECT --> MARK_DOWN[Mark OSD.5 as DOWN]
    MARK_DOWN --> UPDATE_MAP[Update CRUSH Map]

    UPDATE_MAP --> PG_CHECK{Affected PGs?}

    PG_CHECK -->|Da| RECOVERY[Start Recovery]
    PG_CHECK -->|Ne| DONE[No action needed]

    RECOVERY --> FIND_DATA[Find replica on OSD.2]
    FIND_DATA --> NEW_REPLICA[Create new replica on OSD.9]
    NEW_REPLICA --> REBALANCE[Rebalance complete]

    REBALANCE --> HEALTHY[Cluster HEALTH_OK]
```

**Recovery proces:**

1. MON detektuje da OSD ne odgovara (timeout)
2. OSD se markira kao `DOWN`
3. CRUSH mapa se ažurira
4. Podaci koji su bili na tom OSD-u se rekonstruišu
5. Nove replike se kreiraju na drugim OSD-ovima

### 3.5 Ceph Praktični Primeri

#### Cluster Status

```bash
# Provera zdravlja klastera
ceph status

# Primer output-a
  cluster:
    id:     a1b2c3d4-5678-90ab-cdef-1234567890ab
    health: HEALTH_OK
  services:
    mon: 3 daemons, quorum node1,node2,node3
    mgr: node1(active), standbys: node2
    osd: 12 osds: 12 up, 12 in
  data:
    pools:   3 pools, 256 pgs
    objects: 10.5k objects, 40 GiB
    usage:   120 GiB used, 1.8 TiB avail
```

#### OSD Tree

```bash
# Prikaz OSD hijerarhije
ceph osd tree

# Output
ID  CLASS  WEIGHT   TYPE NAME        STATUS
-1         12.00000 root default
-3          4.00000     host node1
 0    ssd   1.00000         osd.0       up
 1    ssd   1.00000         osd.1       up
 2    hdd   2.00000         osd.2       up
-5          4.00000     host node2
 3    ssd   1.00000         osd.3       up
 4    hdd   2.00000         osd.4       up
 5    hdd   1.00000         osd.5       up
```

#### Pool Management

```bash
# Kreiranje pool-a sa replication=3
ceph osd pool create mypool 128 128 replicated
ceph osd pool set mypool size 3
ceph osd pool set mypool min_size 2

# Kreiranje EC pool-a
ceph osd pool create ec-pool 128 128 erasure

# Prikaz pool-ova
ceph osd pool ls detail
```

#### RBD (Block Storage)

```bash
# Kreiranje RBD image-a
rbd create mypool/myvolume --size 100G

# Mapiranje na host
rbd map mypool/myvolume
# Output: /dev/rbd0

# Formatiranje i mount
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt/ceph-block

# Snapshot
rbd snap create mypool/myvolume@snapshot1
rbd snap ls mypool/myvolume
```

#### CephFS (File Storage)

```bash
# Mount CephFS
mount -t ceph mon1,mon2,mon3:/ /mnt/cephfs \
  -o name=admin,secret=AQBxxxxxx

# Ili korišćenjem ceph-fuse
ceph-fuse /mnt/cephfs
```

#### S3 API (Object Storage)

```bash
# Kreiranje S3 korisnika
radosgw-admin user create --uid=myuser --display-name="My User"

# AWS CLI sa Ceph RGW
aws --endpoint-url=http://rgw.example.com:7480 \
    s3 mb s3://mybucket

aws --endpoint-url=http://rgw.example.com:7480 \
    s3 cp myfile.txt s3://mybucket/
```

---

## 4. VMware vSAN - Enterprise SDS

**VMware vSAN** je enterprise SDS rešenje integrisano sa vSphere platformom. Koristi lokalne diskove ESXi hostova za kreiranje distribuiranog storage-a.

### 4.1 vSAN Arhitektura

```mermaid
flowchart TD
    subgraph VCENTER["vCenter Server"]
        SPBM[Storage Policy<br/>Based Management]
        VSAN_SVC[vSAN Service]
    end

    subgraph CLUSTER["vSAN Cluster"]
        subgraph HOST1["ESXi Host 1"]
            VM1[VM]
            VM2[VM]
            AGENT1[vSAN Agent]
            subgraph DG1["Disk Group 1"]
                CACHE1[(Cache SSD)]
                CAP1[(Capacity SSD)]
                CAP2[(Capacity SSD)]
            end
        end

        subgraph HOST2["ESXi Host 2"]
            VM3[VM]
            AGENT2[vSAN Agent]
            subgraph DG2["Disk Group 2"]
                CACHE2[(Cache SSD)]
                CAP3[(Capacity SSD)]
                CAP4[(Capacity SSD)]
            end
        end

        subgraph HOST3["ESXi Host 3"]
            VM4[VM]
            AGENT3[vSAN Agent]
            subgraph DG3["Disk Group 3"]
                CACHE3[(Cache SSD)]
                CAP5[(Capacity SSD)]
                CAP6[(Capacity SSD)]
            end
        end
    end

    subgraph NET["vSAN Network"]
        VMKNIC[vmknic vSAN<br/>10/25/100 GbE]
    end

    VCENTER --> CLUSTER
    HOST1 <--> NET
    HOST2 <--> NET
    HOST3 <--> NET
```

#### vSAN Komponente

| Komponenta | Opis |
|------------|------|
| **vSAN Cluster** | Grupa ESXi hostova koji dele storage |
| **Disk Group** | Cache disk + Capacity diskovi na jednom hostu |
| **vSAN Datastore** | Distribuirani datastore vidljiv svim hostovima |
| **vSAN Agent** | Proces na svakom ESXi koji upravlja lokalnim diskovima |
| **CLOM** (Cluster Level Object Manager) | Upravlja placement-om i health-om objekata |

#### Disk Groups

```
ESXi Host
├── Disk Group 1
│   ├── Cache Tier: 1x NVMe SSD (400GB)
│   └── Capacity Tier: 4x SSD (1.92TB each)
│
└── Disk Group 2
    ├── Cache Tier: 1x NVMe SSD (400GB)
    └── Capacity Tier: 4x SSD (1.92TB each)
```

**Cache Tier:**
- 70% za Read Cache
- 30% za Write Buffer
- Mora biti flash (SSD/NVMe)

**Capacity Tier:**
- Gde se podaci trajno čuvaju
- Može biti SSD ili HDD (all-flash vs hybrid)

### 4.2 Storage Policy Based Management (SPBM)

vSAN koristi polise za definisanje storage karakteristika.

```mermaid
flowchart LR
    VM[Virtual Machine]
    POLICY[Storage Policy]
    VSAN[vSAN Datastore]

    VM --> POLICY
    POLICY --> VSAN

    subgraph POLICY_RULES["Policy Rules"]
        FTT[Failures to Tolerate = 1]
        RAID[RAID Type = RAID-1]
        STRIPE[Stripe Width = 1]
    end

    POLICY --> POLICY_RULES
```

#### Ključni Policy Parametri

| Parametar | Opis | Vrednosti |
|-----------|------|-----------|
| **FTT** (Failures to Tolerate) | Koliko host/disk failure-a može preživeti | 0, 1, 2, 3 |
| **RAID Type** | Metoda zaštite | RAID-1 (mirror), RAID-5, RAID-6 |
| **Stripe Width** | Broj capacity diskova za striping | 1-12 |
| **Object Space Reservation** | Thin vs Thick provisioning | 0-100% |
| **Force Provisioning** | Ignoriši ako nema resursa | Yes/No |

#### FTT i Kapacitet

| FTT | RAID Type | Hosts Required | Capacity Overhead |
|-----|-----------|----------------|-------------------|
| 1 | RAID-1 | 3 | 2x |
| 1 | RAID-5 | 4 | 1.33x |
| 2 | RAID-1 | 5 | 3x |
| 2 | RAID-6 | 6 | 1.5x |

### 4.3 vSAN Object Structure

VM u vSAN-u se sastoji od više objekata:

```mermaid
flowchart TD
    VM[Virtual Machine]

    VM --> VMHOME[VM Home Object<br/>VMX, logs, snapshots]
    VM --> VMDK1[VMDK Object 1<br/>OS Disk]
    VM --> VMDK2[VMDK Object 2<br/>Data Disk]
    VM --> SWAP[Swap Object]

    subgraph VMDK1_STRUCT["VMDK1 with FTT=1, RAID-1"]
        COMP1[Component<br/>Host 1]
        COMP2[Component<br/>Host 2]
        WITNESS1[Witness<br/>Host 3]
    end

    VMDK1 --> VMDK1_STRUCT
```

#### Components i Witnesses

- **Component** — deo objekta koji čuva podatke
- **Witness** — mali metadata objekt za quorum (ne čuva podatke)

Za FTT=1 sa RAID-1:
- 2 data componenta (mirrored)
- 1 witness (za tie-breaking)

### 4.4 vSAN ESA vs OSA

VMware je uveo novu arhitekturu optimizovanu za NVMe:

```mermaid
flowchart LR
    subgraph OSA["Original Storage Architecture (OSA)"]
        DG_OSA[Disk Group]
        CACHE_OSA[(Cache SSD)]
        CAP_OSA[(Capacity)]

        DG_OSA --> CACHE_OSA
        DG_OSA --> CAP_OSA
    end

    subgraph ESA["Express Storage Architecture (ESA)"]
        POOL_ESA[Storage Pool]
        NVME1[(NVMe 1)]
        NVME2[(NVMe 2)]
        NVME3[(NVMe 3)]

        POOL_ESA --> NVME1
        POOL_ESA --> NVME2
        POOL_ESA --> NVME3
    end
```

| Aspekt | OSA | ESA |
|--------|-----|-----|
| Disk Groups | Da (Cache + Capacity) | Ne (single tier) |
| Cache | Dedicated SSD | Softverski (RAM + NVMe) |
| Compression | Per-cluster | Per-object, real-time |
| RAID-5/6 | Standardno | Poboljšano (nested) |
| Minimum hosts | 3 | 3 |
| Diskovi | SSD + HDD ili All-Flash | NVMe only |

**ESA prednosti:**
- Jednostavnija arhitektura
- Bolje performanse sa NVMe
- Efikasnija kompresija
- Manji CPU overhead

### 4.5 vSAN Stretched Cluster

Za disaster recovery, vSAN podržava stretched cluster preko dva sajta:

```mermaid
flowchart TD
    subgraph SITE1["Site 1 - Primary"]
        H1[Host 1]
        H2[Host 2]
    end

    subgraph SITE2["Site 2 - Secondary"]
        H3[Host 3]
        H4[Host 4]
    end

    subgraph WITNESS_SITE["Witness Site"]
        WIT[Witness Host]
    end

    SITE1 <-->|"< 5ms RTT"| SITE2
    SITE1 <--> WIT
    SITE2 <--> WIT
```

**Zahtevi:**
- Latencija između sajtova: < 5ms RTT
- Witness host na trećoj lokaciji
- Simetrična konfiguracija

### 4.6 vSAN Praktični Primeri

#### PowerCLI - Cluster Info

```powershell
# Povezivanje na vCenter
Connect-VIServer -Server vcenter.example.com

# vSAN Cluster konfiguracija
Get-Cluster "Production-Cluster" | Get-VsanClusterConfiguration

# Output:
# Cluster                  : Production-Cluster
# VsanEnabled              : True
# VsanDiskClaimMode        : Manual
# StretchedClusterEnabled  : False
# SpaceEfficiencyEnabled   : True
```

#### Disk Management

```powershell
# Prikaz vSAN diskova
Get-Cluster "Production-Cluster" | Get-VsanDisk

# Prikaz disk grupa
Get-Cluster "Production-Cluster" | Get-VsanDiskGroup

# Output:
# Name           DiskGroupType  Host          CacheDisk     CapacityDisks
# ----           -------------  ----          ---------     -------------
# DiskGroup-1    AllFlash       esxi-01       NVMe-Cache    {SSD1, SSD2}
# DiskGroup-2    AllFlash       esxi-01       NVMe-Cache    {SSD3, SSD4}
```

#### Storage Policy

```powershell
# Kreiranje storage polise
New-SpbmStoragePolicy -Name "FTT2-RAID1" -Description "2 failures, mirroring" `
  -AnyOfRuleSets (
    New-SpbmRuleSet -AllOfRules @(
      New-SpbmRule -Capability (Get-SpbmCapability -Name "VSAN.hostFailuresToTolerate") -Value 2,
      New-SpbmRule -Capability (Get-SpbmCapability -Name "VSAN.replicaPreference") -Value "RAID-1 (Mirroring)"
    )
  )

# Primena polise na VM
Get-VM "ImportantVM" | Set-SpbmEntityConfiguration -StoragePolicy "FTT2-RAID1"
```

#### Health Check

```powershell
# vSAN Health status
Get-VsanHealthSummary -Cluster "Production-Cluster"

# Detaljni health check
Get-VsanHealthTest -Cluster "Production-Cluster" |
    Where-Object { $_.TestHealth -ne "green" }
```

#### Capacity Monitoring

```powershell
# Kapacitet klastera
Get-VsanSpaceUsage -Cluster "Production-Cluster"

# Output:
# Cluster            : Production-Cluster
# TotalCapacityGB    : 15360
# FreeCapacityGB     : 8542
# UsedCapacityGB     : 6818
# DedupRatio         : 1.8
# CompressionRatio   : 2.1
```

#### esxcli komande (na ESXi hostu)

```bash
# vSAN status
esxcli vsan cluster get

# vSAN network info
esxcli vsan network list

# Disk info
esxcli vsan storage list
```

---

## 5. Ceph vs vSAN Poređenje

### Tabela Poređenja

| Aspekt | Ceph | vSAN |
|--------|------|------|
| **Licenca** | Open-source (LGPL) | Komercijalna (per-CPU) |
| **Vendor** | Community + Red Hat | VMware (Broadcom) |
| **Storage Types** | Block + File + Object | Block (+ vSAN File Services) |
| **Hypervisor** | Bilo koji (KVM, Xen, VMware) | Samo VMware ESXi |
| **Minimum Nodes** | 3 | 3 |
| **Max Nodes** | 1000+ | 64 |
| **Placement Algorithm** | CRUSH | CLOM |
| **Kubernetes Native** | Da (Rook-Ceph) | Kroz CSI driver |
| **Management** | CLI + Dashboard | vCenter GUI |
| **Learning Curve** | Strma | Umerena |
| **Support** | Community / Red Hat | VMware |

### Decision Tree

```mermaid
flowchart TD
    START[Izbor SDS Platforme]

    START --> Q1{Koristiš VMware<br/>vSphere?}

    Q1 -->|Da| Q2{Budget za<br/>licenciranje?}
    Q1 -->|Ne| Q3{Potreban Object<br/>Storage S3?}

    Q2 -->|Da| VSAN[vSAN]
    Q2 -->|Ne| Q4{OpenStack ili<br/>Kubernetes?}

    Q3 -->|Da| CEPH[Ceph]
    Q3 -->|Ne| Q5{Enterprise<br/>support?}

    Q4 -->|OpenStack| CEPH
    Q4 -->|Kubernetes| CEPH

    Q5 -->|Da| RHCS[Red Hat Ceph Storage]
    Q5 -->|Ne| CEPH

    VSAN --> VSAN_USE["Use Case:<br/>VM storage<br/>VDI<br/>Enterprise apps"]

    CEPH --> CEPH_USE["Use Case:<br/>Cloud infrastructure<br/>Kubernetes PV<br/>Object storage"]

    RHCS --> RHCS_USE["Use Case:<br/>Enterprise Ceph<br/>sa supportom"]
```

### Kada koristiti koji sistem?

#### Izaberi Ceph kada:
- Gradiš OpenStack ili Kubernetes infrastrukturu
- Treba ti S3-compatible object storage
- Želiš izbjeći vendor lock-in
- Imaš Linux expertise u timu
- Budget je ograničen
- Trebaš > 64 čvora

#### Izaberi vSAN kada:
- Već koristiš VMware vSphere
- Želiš integrisan management (vCenter)
- Prioritet je jednostavnost
- Imaš VMware licensing agreement
- Fokus je na VM workloads
- Potreban ti je enterprise support

---

## 6. SDS u Cloud Infrastrukturi

### 6.1 Private Cloud Deployment

#### OpenStack + Ceph

```mermaid
flowchart TB
    subgraph OPENSTACK["OpenStack"]
        NOVA[Nova<br/>Compute]
        CINDER[Cinder<br/>Block Storage]
        GLANCE[Glance<br/>Images]
        MANILA[Manila<br/>File Shares]
        SWIFT[Swift API<br/>Object]
    end

    subgraph CEPH_CLUSTER["Ceph Cluster"]
        RBD[RBD<br/>Pools]
        CEPHFS[CephFS]
        RGW[RADOS GW]
        RADOS[(RADOS)]
    end

    NOVA -->|ephemeral disks| RBD
    CINDER -->|volumes| RBD
    GLANCE -->|images| RBD
    MANILA -->|shares| CEPHFS
    SWIFT -->|S3/Swift| RGW

    RBD --> RADOS
    CEPHFS --> RADOS
    RGW --> RADOS
```

**Integracija:**

```ini
# /etc/cinder/cinder.conf
[DEFAULT]
enabled_backends = ceph

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
```

#### VMware Cloud Foundation + vSAN

```mermaid
flowchart TB
    subgraph VCF["VMware Cloud Foundation"]
        SDDC[SDDC Manager]
        VC[vCenter]
        NSX[NSX-T]
        ARIA[Aria Suite]
    end

    subgraph WORKLOAD["Workload Domains"]
        WD1[Management Domain]
        WD2[Workload Domain 1]
        WD3[Workload Domain 2]
    end

    subgraph VSAN_LAYER["vSAN Layer"]
        VS1[vSAN Cluster 1]
        VS2[vSAN Cluster 2]
    end

    VCF --> WORKLOAD
    WD1 --> VS1
    WD2 --> VS1
    WD3 --> VS2
```

### 6.2 Kubernetes Integracija

```mermaid
flowchart LR
    subgraph K8S["Kubernetes Cluster"]
        POD[Pod]
        PVC[PersistentVolumeClaim]
        PV[PersistentVolume]
        SC[StorageClass]
        CSI[CSI Driver]
    end

    subgraph SDS["SDS Backend"]
        CEPH_RBD[Ceph RBD]
        VSAN_DS[vSAN Datastore]
    end

    POD --> PVC
    PVC --> PV
    PV --> SC
    SC --> CSI

    CSI -->|rook-ceph| CEPH_RBD
    CSI -->|vsphere-csi| VSAN_DS
```

#### Rook-Ceph Operator

Rook je Kubernetes operator koji automatizuje deployment Ceph-a:

```yaml
# StorageClass za Ceph RBD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```yaml
# PVC primer
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 50Gi
```

#### vSphere CSI Driver

```yaml
# StorageClass za vSAN
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsan-default
provisioner: csi.vsphere.vmware.com
parameters:
  storagepolicyname: "vSAN Default Storage Policy"
  datastoreurl: "ds:///vmfs/volumes/vsan:xxxxx/"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### 6.3 Hybrid / Multi-Cloud

#### Stretched Ceph Cluster

```mermaid
flowchart TD
    subgraph SITE_A["Site A - Primary"]
        MON_A[MON]
        OSD_A1[OSD]
        OSD_A2[OSD]
        OSD_A3[OSD]
    end

    subgraph SITE_B["Site B - Secondary"]
        MON_B[MON]
        OSD_B1[OSD]
        OSD_B2[OSD]
        OSD_B3[OSD]
    end

    subgraph ARBITER["Arbiter Site"]
        MON_C[MON<br/>Arbiter]
    end

    SITE_A <-->|Sync Replication| SITE_B
    SITE_A <--> ARBITER
    SITE_B <--> ARBITER
```

**CRUSH rule za stretch cluster:**

```
rule stretch_rule {
    id 1
    type replicated
    min_size 2
    max_size 4
    step take site_a
    step chooseleaf firstn 2 type host
    step emit
    step take site_b
    step chooseleaf firstn 2 type host
    step emit
}
```

---

## 7. Sizing i Best Practices

### 7.1 Kapacitet Kalkulacija

#### Raw vs Usable Capacity

```
Raw Capacity = Broj diskova × Kapacitet diska

Usable Capacity = Raw Capacity / Protection Overhead
```

**Primer za Ceph (replication=3):**
```
12 nodes × 10 disks × 4TB = 480TB raw
480TB / 3 = 160TB usable
```

**Primer za vSAN (FTT=1, RAID-1):**
```
8 hosts × 6 disks × 1.92TB = 92TB raw
92TB / 2 = 46TB usable
```

#### Ceph Sizing Formula

```
Usable = Raw × (1 - system_overhead) / replication_factor

system_overhead ≈ 0.1 (10% za metadata, journals, etc.)
```

Za erasure coding (k+m):
```
Usable = Raw × (1 - overhead) × k / (k + m)

Primer EC 4+2:
Usable = 480TB × 0.9 × 4/6 = 288TB
```

### 7.2 Network Requirements

| Workload | Minimum | Recommended |
|----------|---------|-------------|
| Small cluster (< 10 nodes) | 10 GbE | 25 GbE |
| Medium cluster (10-50) | 25 GbE | 100 GbE |
| Large cluster (50+) | 100 GbE | 100 GbE + RDMA |

**Dual network setup:**

```
Client Network:  Za frontend (VM, aplikacije)
Cluster Network: Za backend (replication, recovery)
```

```mermaid
flowchart LR
    subgraph HOST["Storage Node"]
        NIC1[NIC 1<br/>10.0.1.x<br/>Client]
        NIC2[NIC 2<br/>10.0.2.x<br/>Cluster]
        OSD[OSD Daemons]
    end

    CLIENT[Client VMs] --> NIC1
    NIC1 --> OSD
    OSD --> NIC2
    NIC2 --> OTHER[Other OSDs]
```

### 7.3 Minimum Node Count

| Scenario | Ceph | vSAN |
|----------|------|------|
| Lab/Test | 1 (single-node) | 1 (vSAN Express) |
| Minimum Production | 3 | 3 |
| Recommended Production | 5+ | 4+ |
| Stretched Cluster | 5 (2+2+1 arbiter) | 5 (2+2+1 witness) |

### 7.4 Best Practices

#### Ceph

1. **Koristi odvojene mreže** za client i cluster traffic
2. **SSD za journals/WAL/DB** ako koristiš HDD za capacity
3. **Minimum 3 MON-a** u produkciji (odd number za quorum)
4. **Planiraj za recovery** — ostavi 10-20% kapaciteta slobodno
5. **CRUSH rules** — postavi failure domains (rack awareness)
6. **Monitoring** — koristi Prometheus + Grafana

```bash
# Provera zdravlja pre maintenance-a
ceph health detail
ceph osd df tree
ceph pg stat
```

#### vSAN

1. **All-Flash za produkciju** — hybrid samo za test
2. **vSAN Network** — dedicated vmknic, nikad shared sa vMotion
3. **Disk Groups** — max 5 po hostu, 7 capacity diskova po grupi
4. **FTT=1 minimum** za produkciju
5. **Slack Space** — ostavi 25-30% slobodno za rebalancing
6. **Health Checks** — redovno proveravaj

```powershell
# Weekly health check
Get-VsanHealthSummary -Cluster "Prod" | Format-List
Test-VsanClusterHealth -Cluster "Prod"
```

### 7.5 Monitoring Checklist

| Metrika | Warning | Critical |
|---------|---------|----------|
| Capacity Used | > 70% | > 85% |
| OSD/Host Down | 1 | 2+ |
| Latency (read) | > 10ms | > 50ms |
| Latency (write) | > 20ms | > 100ms |
| Recovery Speed | < 100 MB/s | < 50 MB/s |
| Network Errors | > 0.1% | > 1% |

---

## Zaključak

**Software-Defined Storage** je fundamentalna tehnologija za moderne cloud infrastrukture. Bilo da birate **Ceph** za open-source fleksibilnost ili **vSAN** za VMware integraciju, ključni koncepti ostaju isti:

1. **Razdvajanje Control i Data Plane** omogućava skalabilnost
2. **Distributed placement algoritmi** (CRUSH, CLOM) eliminišu single points of failure
3. **Policy-based management** pojednostavljuje operacije
4. **Erasure coding** optimizuje iskorišćenje kapaciteta

Izbor platforme zavisi od vaših specifičnih potreba — postojeće infrastrukture, budgeta, tima i workload-a.

---

*Dokument pripremljen za predavanje na Računarskom fakultetu, Beograd*

# Cloud Storage Virtualization

**Autor:** Računarski fakultet, Beograd
**Verzija:** 1.0

---

## Sadržaj

1. [Uvod u Storage Virtualization](#1-uvod-u-storage-virtualization)
2. [Tipovi Storage Virtualizacije](#2-tipovi-storage-virtualizacije)
3. [Arhitektura i Komponente](#3-arhitektura-i-komponente)
4. [Storage Virtualization vs SDS](#4-storage-virtualization-vs-sds)
5. [Cloud Storage Virtualization u Praksi](#5-cloud-storage-virtualization-u-praksi)
6. [Primeri i Tehnologije](#6-primeri-i-tehnologije)
7. [Zaključak i Best Practices](#7-zaključak-i-best-practices)

---

## 1. Uvod u Storage Virtualization

### Šta je Storage Virtualization?

**Storage Virtualization** je tehnologija koja kreira logički (apstraktni) sloj između fizičkog storage-a i aplikacija koje ga koriste. Ovaj sloj "sakriva" kompleksnost fizičke infrastrukture i prezentuje je kao jedinstven, jednostavan resurs.

Zamislite ovo kao **virtualnog bibliotekara** — umesto da sami tražite knjige po različitim policama, vi samo kažete šta vam treba, a bibliotekar zna gde se sve nalazi i donosi vam traženo.

```mermaid
flowchart TB
    subgraph APPS["Aplikacije"]
        APP1[Web Server]
        APP2[Database]
        APP3[File Server]
    end

    subgraph VIRT["Virtualization Layer"]
        VL[("Storage Virtualization<br/>Apstrakcija")]
    end

    subgraph PHYSICAL["Fizički Storage"]
        SAN1[(SAN Array 1<br/>EMC)]
        SAN2[(SAN Array 2<br/>NetApp)]
        NAS1[(NAS<br/>QNAP)]
        DAS1[(DAS<br/>Local Disk)]
    end

    APP1 --> VL
    APP2 --> VL
    APP3 --> VL

    VL --> SAN1
    VL --> SAN2
    VL --> NAS1
    VL --> DAS1

    style VL fill:#e1f5fe
```

> **Objašnjenje dijagrama:**
>
> Dijagram prikazuje tri sloja storage virtualizacije:
> - **Gornji sloj (Aplikacije)** — Web server, baza podataka i file server — svi vide samo jedan "virtualni disk"
> - **Srednji sloj (Virtualization Layer)** — apstrakcija koja skriva kompleksnost
> - **Donji sloj (Fizički Storage)** — različiti uređaji od različitih proizvođača (EMC, NetApp, QNAP, lokalni diskovi)
>
> *Ključna poenta:* Aplikacije ne znaju (i ne moraju da znaju) da postoje četiri različita storage sistema ispod.

### Zašto Storage Virtualization?

| Problem bez virtualizacije | Rešenje sa virtualizacijom |
|---------------------------|---------------------------|
| Svaki storage array se upravlja posebno | Jedinstvena upravljačka konzola |
| Migracija podataka zahteva downtime | Live migration bez prekida |
| Kapacitet "zarobljen" u silosima | Pooling — svi resursi dostupni svima |
| Vendor lock-in | Heterogena podrška |
| Kompleksno kapacitet planiranje | Thin provisioning i auto-tiering |

### Osnovni koncepti

**1. Abstraction (Apstrakcija)**
Fizički storage se "sakriva" iza logičkog interfejsa. Aplikacija vidi samo LUN ili volume, ne zna da li je to SSD, HDD, SAN ili cloud.

**2. Pooling (Agregacija)**
Više fizičkih uređaja se kombinuje u jedan logički pool kapaciteta.

**3. Provisioning**
Iz pool-a se "secka" kapacitet prema potrebama — thin ili thick provisioning.

**4. Data Mobility**
Podaci se mogu pomerati između fizičkih uređaja bez uticaja na aplikacije.

---

## 2. Tipovi Storage Virtualizacije

Storage virtualizacija se može implementirati na različitim nivoima arhitekture. Svaki pristup ima svoje prednosti i mane.

### 2.1 Host-Based Virtualization

Virtualizacija se dešava na nivou servera (hosta) — operativni sistem ili hypervisor upravlja apstrakcijom.

```mermaid
flowchart TB
    subgraph HOST["Server / Host"]
        OS[Operating System]
        LVM[Volume Manager<br/>LVM / Windows Storage Spaces]
        HBA1[HBA Port 1]
        HBA2[HBA Port 2]
    end

    subgraph STORAGE["Storage Arrays"]
        ARR1[(Array 1)]
        ARR2[(Array 2)]
    end

    OS --> LVM
    LVM --> HBA1
    LVM --> HBA2
    HBA1 --> ARR1
    HBA2 --> ARR2

    style LVM fill:#c8e6c9
```

**Primeri:**
- **Linux LVM** (Logical Volume Manager)
- **Windows Storage Spaces**
- **VMware VMFS** (na ESXi hostu)
- **Veritas Volume Manager**

**Prednosti:**
- Nema dodatnog hardvera
- Jednostavna implementacija
- Dobra za manje sredine

**Mane:**
- Svaki host ima svoju "sliku" storage-a
- Nema centralizovanog upravljanja
- CPU overhead na hostu

---

### 2.2 Network-Based (In-Band) Virtualization

Virtualizacioni uređaj se nalazi "u putu" između hosta i storage-a. Sav I/O prolazi kroz njega.

```mermaid
flowchart LR
    subgraph HOSTS["Hosts"]
        H1[Server 1]
        H2[Server 2]
        H3[Server 3]
    end

    subgraph FABRIC["SAN Fabric"]
        SW1[FC Switch]
        VA[/"Virtualization<br/>Appliance<br/>(In-Band)"/]
        SW2[FC Switch]
    end

    subgraph STORAGE["Storage"]
        S1[(Array 1)]
        S2[(Array 2)]
    end

    H1 --> SW1
    H2 --> SW1
    H3 --> SW1
    SW1 --> VA
    VA --> SW2
    SW2 --> S1
    SW2 --> S2

    style VA fill:#ffecb3
```

**Primeri:**
- **IBM SAN Volume Controller (SVC)**
- **EMC VPLEX**
- **DataCore SANsymphony**

**Prednosti:**
- Centralizovano upravljanje
- Transparentno za hostove
- Napredne funkcije (thin provisioning, snapshots, replication)

**Mane:**
- Potencijalni single point of failure
- Dodatna latencija
- Kompleksnost i cena

---

### 2.3 Array-Based (Out-of-Band) Virtualization

Virtualizacija je ugrađena u sam storage array. Metadata se čuva odvojeno od data path-a.

```mermaid
flowchart TB
    subgraph HOSTS["Hosts"]
        H1[Server 1]
        H2[Server 2]
    end

    subgraph ARRAY["Intelligent Storage Array"]
        CTRL["Controller<br/>(Virtualization Engine)"]
        CACHE[Cache]
        subgraph BACKEND["Backend Storage"]
            D1[(Disk Group 1)]
            D2[(Disk Group 2)]
            D3[(Disk Group 3)]
        end
    end

    H1 --> CTRL
    H2 --> CTRL
    CTRL --> CACHE
    CACHE --> D1
    CACHE --> D2
    CACHE --> D3

    style CTRL fill:#e1bee7
```

**Primeri:**
- **NetApp ONTAP** (FlexVol, FlexGroup)
- **Dell EMC PowerStore**
- **Pure Storage FlashArray**
- **HPE Nimble / Primera**

**Prednosti:**
- Optimizovano za performanse
- Nema dodatne komponente u data path-u
- Napredne array funkcije (dedupe, compression)

**Mane:**
- Vendor lock-in
- Virtualizacija ograničena na jedan array (ili familiju)

---

### 2.4 Cloud-Based Storage Virtualization

U cloud okruženju, storage virtualizacija je fundamentalni koncept. Cloud provider apstrahuje kompletan fizički sloj.

```mermaid
flowchart TB
    subgraph TENANT["Tenant / Korisnik"]
        VM1[VM 1]
        VM2[VM 2]
        K8S[Kubernetes Cluster]
    end

    subgraph CLOUD["Cloud Provider Layer"]
        API[Storage API]
        ORCH[Orchestration]
        subgraph SERVICES["Storage Services"]
            BLOCK[Block Storage<br/>EBS / Managed Disk]
            FILE[File Storage<br/>EFS / Azure Files]
            OBJ[Object Storage<br/>S3 / Blob]
        end
    end

    subgraph INFRA["Cloud Infrastructure (Skriveno)"]
        POOL1[(Storage Pool 1)]
        POOL2[(Storage Pool 2)]
        POOL3[(Storage Pool 3)]
    end

    VM1 --> API
    VM2 --> API
    K8S --> API
    API --> ORCH
    ORCH --> BLOCK
    ORCH --> FILE
    ORCH --> OBJ
    BLOCK --> POOL1
    FILE --> POOL2
    OBJ --> POOL3

    style API fill:#bbdefb
    style INFRA fill:#f5f5f5
```

**Primeri:**
- **AWS:** EBS, EFS, S3
- **Azure:** Managed Disks, Azure Files, Blob Storage
- **Oracle Cloud:** Block Volume, File Storage, Object Storage
- **Google Cloud:** Persistent Disk, Filestore, Cloud Storage

**Prednosti:**
- Potpuna apstrakcija — korisnik ne zna ništa o fizičkom sloju
- Elastičnost — kapacitet na zahtev
- Pay-as-you-go model
- Globalna dostupnost i replikacija

**Mane:**
- Zavisnost od cloud providera
- Latencija za on-premises aplikacije
- Egress costs (cena za izlazni saobraćaj)

---

## 3. Arhitektura i Komponente

### 3.1 Ključne komponente Storage Virtualizacije

```mermaid
flowchart TB
    subgraph MGMT["Management Plane"]
        UI[Admin UI / CLI]
        API_M[Management API]
        POLICY[Policy Engine]
        REPORT[Reporting & Analytics]
    end

    subgraph CONTROL["Control Plane"]
        META[Metadata Store]
        MAP[LUN Mapping Engine]
        DISCOVER[Discovery Service]
        HA[HA / Failover Logic]
    end

    subgraph DATA["Data Plane"]
        IO[I/O Processing]
        CACHE_D[Caching]
        THIN[Thin Provisioning]
        SNAP[Snapshot Engine]
        REPL[Replication Engine]
    end

    subgraph BACKEND["Backend Connectivity"]
        FC[Fibre Channel]
        ISCSI[iSCSI]
        NVME[NVMe-oF]
        CLOUD_C[Cloud Connector]
    end

    UI --> API_M
    API_M --> POLICY
    POLICY --> META
    META --> MAP
    MAP --> IO
    IO --> CACHE_D
    CACHE_D --> FC
    CACHE_D --> ISCSI
    CACHE_D --> NVME
    CACHE_D --> CLOUD_C

    DISCOVER -.-> BACKEND
    HA -.-> CONTROL
```

> **Objašnjenje komponenti:**
>
> **Management Plane** — interfejs za administratore:
> - *Admin UI/CLI* — grafički ili komandni interfejs
> - *Management API* — REST/SOAP API za automatizaciju
> - *Policy Engine* — definisanje pravila (tiering, retention)
> - *Reporting* — izveštaji o kapacitetu, performansama
>
> **Control Plane** — "mozak" sistema:
> - *Metadata Store* — gde se čuvaju informacije o mapiranju
> - *LUN Mapping Engine* — prevodi virtualne LUN-ove u fizičke
> - *Discovery Service* — otkriva nove storage uređaje
> - *HA/Failover* — osigurava visoku dostupnost
>
> **Data Plane** — obrada podataka:
> - *I/O Processing* — čitanje i pisanje
> - *Caching* — ubrzanje pristupa
> - *Thin Provisioning* — "lenja" alokacija
> - *Snapshot/Replication* — zaštita podataka
>
> **Backend Connectivity** — veza ka fizičkom storage-u

---

### 3.2 Metadata — Srce Virtualizacije

Metadata je ključna komponenta koja omogućava mapiranje između virtualnih i fizičkih resursa.

```mermaid
flowchart LR
    subgraph VIRTUAL["Virtualni Pogled"]
        VLUN1[Virtual LUN 1<br/>100 GB]
        VLUN2[Virtual LUN 2<br/>500 GB]
        VLUN3[Virtual LUN 3<br/>200 GB]
    end

    subgraph METADATA["Metadata Store"]
        direction TB
        MAP1["VLUN1 → Array1:LUN5 (50GB)<br/>        + Array2:LUN3 (50GB)"]
        MAP2["VLUN2 → Array1:LUN7 (500GB)"]
        MAP3["VLUN3 → Array2:LUN1 (200GB)"]
    end

    subgraph PHYSICAL["Fizički Storage"]
        A1L5[(Array1:LUN5)]
        A1L7[(Array1:LUN7)]
        A2L1[(Array2:LUN1)]
        A2L3[(Array2:LUN3)]
    end

    VLUN1 --> MAP1
    VLUN2 --> MAP2
    VLUN3 --> MAP3

    MAP1 --> A1L5
    MAP1 --> A2L3
    MAP2 --> A1L7
    MAP3 --> A2L1
```

> **Objašnjenje:**
>
> - **VLUN1 (100 GB)** se sastoji od dva fizička LUN-a sa različitih array-a — ovo je **striping/spanning**
> - **VLUN2 (500 GB)** je mapiran 1:1 na jedan fizički LUN
> - **VLUN3 (200 GB)** je takođe 1:1 mapiranje
>
> Metadata čuva ove relacije i omogućava transparentnu migraciju, proširenje i zaštitu.

---

### 3.3 Thin Provisioning — Ključna Funkcija

Thin provisioning je jedna od najvažnijih funkcija storage virtualizacije.

```mermaid
flowchart TB
    subgraph THICK["Thick Provisioning (Tradicionalno)"]
        T_ALLOC["Alocirano: 1 TB"]
        T_USED["Iskorišćeno: 200 GB"]
        T_WASTE["Neiskorišćeno: 800 GB<br/>(ZAUZETO ali prazno)"]
    end

    subgraph THIN["Thin Provisioning"]
        TH_ALLOC["Alocirano: 1 TB<br/>(Virtualno)"]
        TH_USED["Iskorišćeno: 200 GB<br/>(Fizički zauzeto)"]
        TH_FREE["800 GB dostupno<br/>drugim VM-ovima!"]
    end

    style T_WASTE fill:#ffcdd2
    style TH_FREE fill:#c8e6c9
```

| Aspekt | Thick Provisioning | Thin Provisioning |
|--------|-------------------|-------------------|
| Alokacija | Unapred, ceo kapacitet | Po potrebi, inkrementalno |
| Iskorišćenje | Često < 50% | Može biti > 100% (overcommit) |
| Performanse | Konzistentne | Može varirati pri ekspanziji |
| Rizik | Nema rizika od prepunjavanja | Potreban monitoring |
| Use case | Kritične baze podataka | Opšta namena, dev/test |

---

## 4. Storage Virtualization vs SDS

Ovo je ključno poglavlje koje objašnjava razlike između **Storage Virtualization** i **Software-Defined Storage (SDS)**.

### 4.1 Fundamentalna Razlika

```mermaid
flowchart TB
    subgraph SV["STORAGE VIRTUALIZATION"]
        SV_DESC["Apstrakcija POSTOJEĆEG<br/>storage hardvera"]
        SV_HW["Koristi tradicionalne<br/>SAN/NAS uređaje"]
        SV_LAYER["Dodaje virtualizacioni<br/>sloj IZNAD hardvera"]
    end

    subgraph SDS["SOFTWARE-DEFINED STORAGE"]
        SDS_DESC["Softver ZAMENJUJE<br/>storage kontroler"]
        SDS_HW["Koristi commodity<br/>x86 servere"]
        SDS_LAYER["Storage logika je<br/>KOMPLETNO u softveru"]
    end

    SV_DESC --> SV_HW
    SV_HW --> SV_LAYER

    SDS_DESC --> SDS_HW
    SDS_HW --> SDS_LAYER

    style SV fill:#e3f2fd
    style SDS fill:#fff3e0
```

### 4.2 Detaljna Komparacija

| Kriterijum | Storage Virtualization | Software-Defined Storage |
|------------|----------------------|--------------------------|
| **Definicija** | Apstrakcija i agregacija postojećeg storage hardvera | Storage sistem gde softver kontroliše sve funkcije |
| **Hardver** | Koristi postojeće SAN/NAS array-e | Commodity x86 serveri, bilo koji diskovi |
| **Kontroler** | Fizički array kontroler + virtualizacioni sloj | Softverski kontroler (nema dedicirani hardver) |
| **Vendor Lock-in** | Srednji (virtualizacija može biti vendor-specific) | Nizak (softver radi na bilo kom hardveru) |
| **Skaliranje** | Ograničeno kapacitetom backend array-a | Horizontalno, dodavanjem čvorova |
| **Cena** | Visoka (enterprise array + virtualizacija) | Niža (commodity hardver + softver) |
| **Kompleksnost** | Srednja do visoka | Srednja (ali zahteva ekspertizu) |
| **Performanse** | Zavise od backend array-a | Zavise od softvera i hardvera |
| **Primeri** | IBM SVC, EMC VPLEX, VMware vVol | Ceph, vSAN, MinIO, GlusterFS |

---

### 4.3 Arhitekturna Razlika

```mermaid
flowchart TB
    subgraph LEFT["Storage Virtualization"]
        direction TB
        L_APP[Aplikacije]
        L_VIRT[Virtualization Layer<br/>IBM SVC / VPLEX]
        L_ARR1[(Enterprise Array 1<br/>EMC VMAX)]
        L_ARR2[(Enterprise Array 2<br/>NetApp FAS)]

        L_APP --> L_VIRT
        L_VIRT --> L_ARR1
        L_VIRT --> L_ARR2
    end

    subgraph RIGHT["Software-Defined Storage"]
        direction TB
        R_APP[Aplikacije]
        R_SDS[SDS Software<br/>Ceph / vSAN]
        R_N1[x86 Server 1<br/>+ Local Disks]
        R_N2[x86 Server 2<br/>+ Local Disks]
        R_N3[x86 Server 3<br/>+ Local Disks]

        R_APP --> R_SDS
        R_SDS --> R_N1
        R_SDS --> R_N2
        R_SDS --> R_N3
    end

    style L_VIRT fill:#bbdefb
    style R_SDS fill:#ffe0b2
    style L_ARR1 fill:#e1bee7
    style L_ARR2 fill:#e1bee7
```

> **Objašnjenje razlike:**
>
> **Leva strana (Storage Virtualization):**
> - Virtualizacioni sloj (IBM SVC ili VPLEX) stoji IZMEĐU aplikacija i POSTOJEĆIH enterprise array-a
> - Array-i (EMC VMAX, NetApp) su i dalje "pametni" — imaju svoje kontrolere i softver
> - Virtualizacija AGREGIRA i APSTRAHUJE te uređaje
>
> **Desna strana (SDS):**
> - SDS softver (Ceph, vSAN) JE storage kontroler — nema "pametnog" array-a ispod
> - Serveri su obični x86 mašine sa lokalnim diskovima (JBOD)
> - SVA inteligencija je u softveru

---

### 4.4 Kada koristiti šta?

```mermaid
flowchart TD
    START([Izbor Storage Arhitekture]) --> Q1{Imate postojeću<br/>SAN/NAS investiciju?}

    Q1 -->|DA| Q2{Želite zadržati<br/>postojeći hardver?}
    Q1 -->|NE| Q3{Budget za<br/>enterprise hardver?}

    Q2 -->|DA| SV_REC[/"Storage Virtualization<br/>(IBM SVC, VPLEX, vVol)"/]
    Q2 -->|NE| SDS_REC

    Q3 -->|DA| ENT[/"Enterprise Array<br/>(NetApp, Pure, EMC)"/]
    Q3 -->|NE| SDS_REC[/"Software-Defined Storage<br/>(Ceph, vSAN, MinIO)"/]

    style SV_REC fill:#bbdefb
    style SDS_REC fill:#ffe0b2
    style ENT fill:#e1bee7
```

#### Preporuke:

**Izaberite Storage Virtualization kada:**
- Imate značajne investicije u postojeće SAN/NAS sisteme
- Trebate heterogenu podršku (više vendora)
- Želite live migration između array-a
- Imate tim obučen za tradicionalni SAN management
- Regulatorne zahteve rešavate enterprise support-om

**Izaberite SDS kada:**
- Gradite novu infrastrukturu "from scratch"
- Budget je ograničen (commodity hardver)
- Trebate masivno skaliranje (petabajti)
- Preferirate open-source ili cloud-native pristup
- Imate DevOps kulturu i automatizaciju

---

### 4.5 Hibridni Pristup

U praksi, mnoge organizacije koriste **kombinaciju** oba pristupa:

```mermaid
flowchart TB
    subgraph TIER1["Tier 1: Mission Critical"]
        MC_APP[SAP / Oracle DB]
        MC_VIRT[Storage Virtualization<br/>EMC VPLEX]
        MC_ARR[(Enterprise Flash Array)]
    end

    subgraph TIER2["Tier 2: Business Applications"]
        BA_APP[Web Apps / DevOps]
        BA_SDS[SDS Layer<br/>VMware vSAN]
        BA_HW[Commodity Servers]
    end

    subgraph TIER3["Tier 3: Archive / Backup"]
        AR_APP[Backup / Archive]
        AR_SDS[SDS Layer<br/>Ceph Object]
        AR_HW[High-Density JBOD]
    end

    MC_APP --> MC_VIRT --> MC_ARR
    BA_APP --> BA_SDS --> BA_HW
    AR_APP --> AR_SDS --> AR_HW

    style MC_VIRT fill:#bbdefb
    style BA_SDS fill:#ffe0b2
    style AR_SDS fill:#ffe0b2
```

> **Objašnjenje hibridnog modela:**
>
> - **Tier 1 (Mission Critical)** — SAP, Oracle — koristi Storage Virtualization sa enterprise flash array-ima za maksimalne performanse i podršku
> - **Tier 2 (Business Apps)** — Web aplikacije, DevOps — koristi SDS (vSAN) na commodity serverima za balans cene i performansi
> - **Tier 3 (Archive)** — Backup i arhiva — koristi SDS (Ceph Object) na JBOD-ovima za minimalne troškove po TB

---

## 5. Cloud Storage Virtualization u Praksi

### 5.1 Public Cloud Storage Virtualization

Svi veliki cloud provajderi implementiraju storage virtualizaciju "ispod haube":

```mermaid
flowchart TB
    subgraph AWS["Amazon Web Services"]
        EBS[Amazon EBS]
        EFS[Amazon EFS]
        S3[Amazon S3]
        EBS_INT["Block Virtualization"]
        EFS_INT["File Virtualization"]
        S3_INT["Object Virtualization"]

        EBS --> EBS_INT
        EFS --> EFS_INT
        S3 --> S3_INT
    end

    subgraph AZURE["Microsoft Azure"]
        MD[Managed Disks]
        AF[Azure Files]
        BLOB[Blob Storage]
        MD_INT["Block Virtualization"]
        AF_INT["File Virtualization"]
        BLOB_INT["Object Virtualization"]

        MD --> MD_INT
        AF --> AF_INT
        BLOB --> BLOB_INT
    end

    subgraph OCI["Oracle Cloud"]
        BV[Block Volume]
        FSS[File Storage]
        OS[Object Storage]
        BV_INT["Block Virtualization"]
        FSS_INT["File Virtualization"]
        OS_INT["Object Virtualization"]

        BV --> BV_INT
        FSS --> FSS_INT
        OS --> OS_INT
    end
```

### 5.2 Storage Virtualization na DDC Kragujevac

U kontekstu Državnog Data Centra u Kragujevcu, storage virtualizacija igra ključnu ulogu:

```mermaid
flowchart TB
    subgraph USERS["Korisnici DDC-a"]
        GOV[Državne Institucije]
        ENT[Enterprise Korisnici]
        CERN_U[CERN WLCG]
    end

    subgraph DDC["DDC Kragujevac"]
        subgraph OCI_SR["Oracle Cloud Jovanovac"]
            OCI_BV[OCI Block Volume]
            OCI_FS[OCI File Storage]
            OCI_OS[OCI Object Storage]
        end

        subgraph COLO["Colocation Zone"]
            CUST_SAN[(Customer SAN)]
            CUST_NAS[(Customer NAS)]
        end

        subgraph HPC["Supercomputing"]
            LUSTRE[(Lustre Parallel FS)]
            GPFS[(IBM Spectrum Scale)]
        end
    end

    GOV --> OCI_BV
    GOV --> OCI_FS
    ENT --> CUST_SAN
    ENT --> OCI_OS
    CERN_U --> LUSTRE
    CERN_U --> GPFS
```

> **DDC Kragujevac kombinuje:**
> - **Oracle Cloud storage servise** — potpuno virtualizovani, managed servisi
> - **Colocation storage** — korisnici donose svoj hardver koji može koristiti virtualizaciju
> - **HPC storage** — specijalizovani parallel file sistemi za CERN workloads

---

### 5.3 Private Cloud — VMware vSphere + vSAN

Primer kako storage virtualizacija i SDS rade zajedno u VMware okruženju:

```mermaid
flowchart TB
    subgraph VSPHERE["VMware vSphere Cluster"]
        VC[vCenter Server]

        subgraph HOST1["ESXi Host 1"]
            VM1_1[VM 1]
            VM1_2[VM 2]
            VSAN1[vSAN Agent]
        end

        subgraph HOST2["ESXi Host 2"]
            VM2_1[VM 3]
            VM2_2[VM 4]
            VSAN2[vSAN Agent]
        end

        subgraph HOST3["ESXi Host 3"]
            VM3_1[VM 5]
            VM3_2[VM 6]
            VSAN3[vSAN Agent]
        end
    end

    subgraph VSAN_DS["vSAN Datastore (SDS)"]
        VSAN_POOL[(Distributed<br/>Storage Pool)]
    end

    subgraph EXT["External Storage (Virtualized)"]
        VPLEX[EMC VPLEX]
        ARR1[(Array 1)]
        ARR2[(Array 2)]
    end

    VC --> HOST1
    VC --> HOST2
    VC --> HOST3

    VSAN1 --> VSAN_POOL
    VSAN2 --> VSAN_POOL
    VSAN3 --> VSAN_POOL

    VM1_1 -.-> VPLEX
    VPLEX --> ARR1
    VPLEX --> ARR2
```

> **Ovaj dijagram prikazuje:**
> - **vSAN** kao SDS rešenje — lokalni diskovi u ESXi hostovima formiraju distribuirani datastore
> - **VPLEX** kao Storage Virtualization — agregira eksterne array-e i prezentuje ih VM-ovima
> - VM-ovi mogu koristiti OBA tipa storage-a istovremeno

---

## 6. Primeri i Tehnologije

### 6.1 Storage Virtualization Tehnologije

| Proizvod | Vendor | Tip | Opis |
|----------|--------|-----|------|
| **SAN Volume Controller (SVC)** | IBM | In-Band | Enterprise SAN virtualizacija |
| **VPLEX** | Dell EMC | In-Band | Active-active metro clustering |
| **SANsymphony** | DataCore | Software | x86-based SAN virtualizacija |
| **vVols** | VMware | Protocol | Per-VM storage management |
| **ONTAP** | NetApp | Array-based | FlexVol/FlexGroup virtualizacija |

### 6.2 SDS Tehnologije

| Proizvod | Vendor/Community | Model | Opis |
|----------|-----------------|-------|------|
| **Ceph** | Open Source | Block/File/Object | Distribuirani storage, CRUSH algorithm |
| **vSAN** | VMware | Block | HCI storage za vSphere |
| **MinIO** | Open Source | Object | S3-compatible, cloud-native |
| **GlusterFS** | Red Hat | File | Scale-out NAS |
| **Longhorn** | Rancher/SUSE | Block | Kubernetes-native storage |

### 6.3 Primer: IBM SVC Konfiguracija (Storage Virtualization)

```
# Kreiranje MDisk grupe (pool fizičkih diskova)
svctask mkmdsgrp -name pool_tier1 -ext 256

# Dodavanje eksternog storage-a u pool
svctask mkmdisk -mdiskgrp pool_tier1 -array "IBM_2145:mdisk0"
svctask mkmdisk -mdiskgrp pool_tier1 -array "NetApp_FAS:lun0"

# Kreiranje virtualnog volumea
svctask mkvdisk -name oracle_data -mdiskgrp pool_tier1 -size 500 -unit gb

# Mapiranje na host
svctask mkvdiskhostmap -host ora_server1 -vdisk oracle_data
```

### 6.4 Primer: Ceph Konfiguracija (SDS)

```bash
# Kreiranje Ceph clustera (SDS)
cephadm bootstrap --mon-ip 192.168.1.10

# Dodavanje OSD-ova (commodity serveri sa lokalnim diskovima)
ceph orch daemon add osd node1:/dev/sdb
ceph orch daemon add osd node2:/dev/sdb
ceph orch daemon add osd node3:/dev/sdb

# Kreiranje RBD pool-a (block storage)
ceph osd pool create rbd_pool 128

# Kreiranje block device-a
rbd create --size 500G rbd_pool/oracle_data
```

> **Uočite razliku:**
> - **SVC** agregira POSTOJEĆE storage (IBM_2145, NetApp_FAS)
> - **Ceph** koristi LOKALNE DISKOVE u serverima (/dev/sdb)

---

## 7. Zaključak i Best Practices

### 7.1 Ključne Razlike — Sažetak

```mermaid
flowchart LR
    subgraph SV["Storage Virtualization"]
        SV1[Apstrakcija]
        SV2[Agregacija postojećeg HW]
        SV3[Heterogena podrška]
        SV4[Enterprise focus]
    end

    subgraph SDS["Software-Defined Storage"]
        SDS1[Zamena kontrolera]
        SDS2[Commodity hardver]
        SDS3[Scale-out arhitektura]
        SDS4[Cloud-native focus]
    end

    SV1 ~~~ SDS1
    SV2 ~~~ SDS2
    SV3 ~~~ SDS3
    SV4 ~~~ SDS4
```

### 7.2 Best Practices

#### Za Storage Virtualization:

1. **Planirajte metadata replikaciju** — Metadata je kritičan; uvek imajte redundantan metadata store
2. **Testirajte failover scenarije** — Virtualizacioni sloj je SPOF (Single Point of Failure)
3. **Monitoring latencije** — In-band virtualizacija dodaje latenciju
4. **Firmware kompatibilnost** — Održavajte kompatibilnost između virtualizacionog sloja i backend array-a
5. **Capacity planning** — Pratite STVARNI (fizički) kapacitet, ne samo virtualni

#### Za SDS:

1. **Network je kritičan** — SDS zavisi od mreže; koristite 25/100 GbE ili RDMA
2. **Failure domains** — Distribuirajte podatke preko rack-ova, ne samo servera
3. **Izbegavajte overcommit** — Thin provisioning je koristan, ali monitoring je obavezan
4. **Hardware homogenost** — Mešanje različitog hardvera otežava troubleshooting
5. **Testirajte recovery** — SDS sistemi su kompleksni; redovno testirajte oporavak

### 7.3 Odluka: Virtuelizacija ili SDS?

| Ako vam je prioritet... | Preporuka |
|------------------------|-----------|
| Zaštita postojeće investicije | Storage Virtualization |
| Minimalna cena po TB | SDS |
| Enterprise podrška | Storage Virtualization (IBM, EMC) |
| Open source / flexibility | SDS (Ceph, MinIO) |
| Maksimalne performanse | Zavisi od workload-a |
| Kubernetes/Cloud-native | SDS |
| Metro clustering / DR | Storage Virtualization (VPLEX) |

---

### 7.4 Budućnost: Konvergencija

Granica između Storage Virtualization i SDS se sve više zamagljuje:

- **NetApp ONTAP** — počeo kao array, sada radi i na commodity hardveru (ONTAP Select)
- **VMware vSAN** — SDS koji se integriše sa storage virtualizacijom (vVols)
- **Pure Storage** — flash array sa SDS karakteristikama (Portworx akvizicija)

```mermaid
flowchart LR
    PAST["Prošlost:<br/>Jasna podela"] --> PRESENT["Sadašnjost:<br/>Hibridni modeli"]
    PRESENT --> FUTURE["Budućnost:<br/>Unified Storage Platform"]

    style PAST fill:#ffcdd2
    style PRESENT fill:#fff9c4
    style FUTURE fill:#c8e6c9
```

---

## Reference

1. IBM SAN Volume Controller Documentation
2. VMware vSAN Design Guide
3. Ceph Documentation — https://docs.ceph.com
4. Dell EMC VPLEX Architecture Guide
5. NetApp ONTAP 9 Documentation
6. "Software-Defined Storage for Dummies" — Wiley

---

**Kraj dokumenta**

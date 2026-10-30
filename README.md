# Margin Ledger – Dashboard Profitabilitas E-commerce (Power BI)

> **Di mana margin tercipta, dan di mana bocor?**
> Dashboard Power BI untuk direksi non-teknis: membaca profitabilitas sebuah retailer online Eropa (2024–2025, EUR) dari penjualan kotor sampai contribution margin.

**Status:** 🚧 Work in progress. Tahap 1–2 (business understanding, profiling, desain model) selesai. Lihat [Roadmap](#roadmap).

---

## 1. Konteks bisnis

| | |
|---|---|
| **Domain** | E-commerce / retail online Eropa |
| **Periode data** | Januari 2024 – Desember 2025 |
| **Mata uang** | EUR |
| **Audiens** | Direksi dan manajemen non-teknis |
| **Keputusan yang didukung** | Produk, promo, channel, market, dan segmen pelanggan mana yang layak ditambah investasinya, diperbaiki, atau dihentikan |

### Pertanyaan bisnis per halaman

| Halaman | Pertanyaan |
|---|---|
| Overview | Di mana margin tercipta, dan di mana bocor? |
| Products | Produk mana yang menghasilkan margin? |
| Returns | Berapa biaya retur? |
| Promotions | Apakah diskon balik modal? |
| Customers | Pelanggan mana yang layak dipertahankan? |
| Channels & markets | Channel dan market mana yang paling menguntungkan? |
| Fulfillment | Berapa biaya pengiriman, dan apakah tiba tepat waktu? |
| Inventory | Berapa penjualan yang hilang karena stok kosong? |

---

## 2. KPI utama

| KPI | Definisi |
|---|---|
| **Gross Sales** | Fulfilled quantity × harga list (sebelum diskon dan refund) |
| **Net Sales** | Gross Sales − Discount − Refund |
| **Contribution Revenue** | Net Sales + Shipping Revenue |
| **Contribution Margin** | Contribution Revenue − product cost − semua biaya variabel (outbound shipping, return shipping, fulfillment, payment fee, marketing, return processing) |
| **Margin %** | Contribution Margin ÷ Contribution Revenue |
| **Orders** | Jumlah `OrderID` unik |
| **Repeat Share** | Net Sales dari order pelanggan berulang ÷ total Net Sales |

> **Definisi first vs repeat:** first order = `CohortAgeKey = 0` (bulan ke-0 sejak pembelian pertama); repeat = `CohortAgeKey > 0`.

---

## 3. Sumber data

File: `data/ecommerce_profitability_challenge_data.xlsx` (10 sheet data + 1 sheet Data Dictionary).

| Tabel | Baris | Grain |
|---|---:|---|
| `Fact_OrderLine` | 9.109 | 1 baris = 1 order line (`LineID` unik) |
| `Dim_Date` | 1.096 | 1 baris = 1 hari |
| `Dim_Product` | 60 | 1 produk |
| `Dim_Customer` | 2.200 | 1 pelanggan |
| `Dim_Geography` | 24 | 1 kota |
| `Dim_Promotion` | 10 | 1 promo |
| `Dim_Fulfillment` | 6 | 1 kombinasi fulfillment center dan carrier |
| `Dim_SalesChannel` | 5 | 1 channel dan device |
| `Dim_ReturnReason` | 7 | 1 alasan retur |
| `Dim_CohortAge` | 25 | 1 bulan sejak pembelian pertama (0–24) |

> Nama asli di file adalah `FactOrderLine`, `DimDate`, dst. Di model Power BI saya ubah menjadi `Fact_` / `Dim_` agar konsisten dengan standar penamaan.

### Hasil data profiling

- **Integritas key:** 0 orphan. Seluruh foreign key fact punya pasangan di dimension (termasuk 3 key tanggal tambahan).
- **Duplikat:** 0 baris duplikat; `LineID` unik.
- **Null yang wajar (bukan error):**
  - `ReturnDateKey` kosong untuk line yang tidak diretur (8.063 baris)
  - `DeliveryDateKey`, `ActualDeliveryDays`, `OnTimeFlag` kosong untuk line yang tidak terkirim (142 baris)
  - `LoyaltyTier` kosong untuk pelanggan tanpa tier (1.166 baris) → diganti "No tier" di Power Query
  - `DiscountType` kosong untuk promo "No promotion" (1 baris)
- **Identitas akuntansi terjaga:** `ContributionRevenue = NetSales + ShippingRevenue` (selisih maksimum 0).
- **Contribution Margin negatif:** ada di 2.168 line. Ini bukan error, tapi bahan analisis ("produk atau promo yang merugi").

### Angka acuan untuk rekonsiliasi (dihitung langsung dari file sumber)

| Metrik | Nilai |
|---|---:|
| Gross Sales | €789.191 |
| Discount | €77.513 |
| Refund | €71.257 |
| Net Sales | €640.422 |
| Shipping Revenue | €16.785 |
| Contribution Revenue | €657.207 |
| Product Cost | €386.537 |
| Contribution Margin | €127.652 |
| Margin % | 19,4% |
| Margin % – first order | 13,4% |
| Margin % – repeat order | 22,9% |
| Lost Sales Value | €36.715 |
| Return Loss | €102.898 |
| Order lines / Orders / Customers | 9.109 / 5.873 / 2.200 |

Angka ini menjadi tolok ukur: nilai di dashboard Power BI harus sama persis.

---

## 4. Semantic model (star schema)

![Diagram star schema](model/model_diagram.svg)

### Relationship

| Dari (Dimension) | Ke (Fact) | Cardinality | Filter | Status |
|---|---|---|---|---|
| `Dim_Date[DateKey]` | `OrderDateKey` | 1:* | Single | **Aktif** |
| `Dim_Date[DateKey]` | `ShipDateKey` | 1:* | Single | Inactive |
| `Dim_Date[DateKey]` | `DeliveryDateKey` | 1:* | Single | Inactive |
| `Dim_Date[DateKey]` | `ReturnDateKey` | 1:* | Single | Inactive |
| `Dim_Product[ProductKey]` | `ProductKey` | 1:* | Single | Aktif |
| `Dim_Customer[CustomerKey]` | `CustomerKey` | 1:* | Single | Aktif |
| `Dim_Geography[GeographyKey]` | `GeographyKey` | 1:* | Single | Aktif |
| `Dim_Promotion[PromotionKey]` | `PromotionKey` | 1:* | Single | Aktif |
| `Dim_Fulfillment[FulfillmentKey]` | `FulfillmentKey` | 1:* | Single | Aktif |
| `Dim_SalesChannel[SalesChannelKey]` | `SalesChannelKey` | 1:* | Single | Aktif |
| `Dim_ReturnReason[ReturnReasonKey]` | `ReturnReasonKey` | 1:* | Single | Aktif |
| `Dim_CohortAge[CohortAgeKey]` | `CohortAgeKey` | 1:* | Single | Aktif |

### Keputusan desain

1. **Star schema murni.** Semua dimension terhubung langsung ke fact; tanpa many-to-many dan tanpa bi-directional filter.
2. **Role-playing date.** Hanya `OrderDateKey` yang aktif. Tanggal kirim, terima, dan retur memakai relationship inactive yang diaktifkan per measure dengan `USERELATIONSHIP`. *Trade-off:* model sedikit lebih rumit, tapi analisis fulfillment dan retur bisa dibaca menurut tanggal yang tepat.
3. **Tidak ada relationship dari atribut `Dim_Customer`** (`HomeGeographyKey`, `FirstPurchaseDateKey`) ke dimension lain, untuk menghindari jalur filter ganda yang ambigu.
4. **Satu Date table khusus** (`Dim_Date`, mark as date table); *Auto date/time* dimatikan.
5. **Tipe uang** memakai Fixed decimal agar tidak ada selisih pembulatan.
6. **Semua measure** disimpan di tabel `_Measures`, dikelompokkan dengan display folder.
7. **Kolom key dan kolom teknis disembunyikan** dari Report view.

### Mode data

Import dari file Excel (parameter `FilePath` di Power Query).

---

## 5. Desain dashboard

- Satu halaman = satu pertanyaan bisnis; alur dari ringkasan (KPI) ke detail (tren, breakdown, tabel).
- Judul visual berupa insight/pertanyaan dengan bahasa bercerita.
- Gaya editorial: latar krem, aksen navy dan ochre; merah/hijau hanya untuk varians.
- Tema: [`theme/margin_ledger_theme.json`](theme/margin_ledger_theme.json) *(akan ditambahkan di Tahap 4)*.

---

## 6. Struktur repo

```
margin-ledger-powerbi/
├─ README.md
├─ data/        sumber data (xlsx)
├─ model/       diagram model, data dictionary
├─ dax/         daftar measure + penjelasan
├─ theme/       theme JSON Power BI
├─ screenshots/ tangkapan layar tiap halaman
└─ MarginLedger.pbix
```

---

## Roadmap

- [x] **Tahap 1** – Business understanding, data profiling, desain model
- [ ] **Tahap 2** – Power Query, relationship, Date table
- [ ] **Tahap 3** – DAX measures (`_Measures`)
- [ ] **Tahap 4** – Theme dan halaman Overview
- [ ] **Tahap 5** – Halaman Products, Returns, Promotions, Customers, Channels & markets, Fulfillment, Inventory
- [ ] **Tahap 6** – QA, performance, publish, case study

## Teknologi

Power BI Desktop · Power Query (M) · DAX · Star schema

## Penulis

**[Nama Anda]** · [LinkedIn](#) · [Portofolio](#)

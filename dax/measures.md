# Daftar Measure DAX – Margin Ledger

Semua measure disimpan di tabel `_Measures` dan dikelompokkan dengan display folder. Tiap measure menghitung ulang dari baris `Fact_OrderLine` yang lolos filter aktif (tahun, region, channel, kategori), kecuali disebutkan lain.

Angka acuan (total, tanpa filter) berasal dari data sumber dan dipakai untuk rekonsiliasi.

---

## 1 Sales & Margin

| Measure | Format | Acuan |
|---|---|---:|
| Gross Sales | € 0 | €789.191 |
| Discounts | € 0 | €77.513 |
| Refunds | € 0 | €71.257 |
| Net Sales | € 0 | €640.422 |
| Contribution Revenue | € 0 | €657.207 |
| Contribution Margin | € 0 | €127.652 |
| Margin % | 0,0% | 19,4% |

```dax
Gross Sales = SUM ( Fact_OrderLine[GrossSales] )
Discounts = SUM ( Fact_OrderLine[DiscountAmount] )
Refunds = SUM ( Fact_OrderLine[RefundAmount] )
Net Sales = SUM ( Fact_OrderLine[NetSales] )
Contribution Revenue = SUM ( Fact_OrderLine[ContributionRevenue] )
Contribution Margin = SUM ( Fact_OrderLine[ContributionMargin] )

-- Margin % = margin dibagi contribution revenue (bukan net sales).
Margin % = DIVIDE ( [Contribution Margin], [Contribution Revenue] )
```

## 2 Volume

| Measure | Acuan |
|---|---:|
| Orders | 5.873 |
| Customers | 2.200 |
| Order Lines | 9.109 |

```dax
Orders = DISTINCTCOUNT ( Fact_OrderLine[OrderID] )
Customers = DISTINCTCOUNT ( Fact_OrderLine[CustomerKey] )
Order Lines = COUNTROWS ( Fact_OrderLine )
```

> `Customers` tidak bisa dijumlahkan antar tahun: 1.130 (2024) + 1.650 (2025) = 2.780, tetapi total unik 2.200, karena sebagian pelanggan membeli di kedua tahun.

## 3 Customer Cohort

First order = `CohortAgeKey = 0`; repeat = `CohortAgeKey > 0`.

```dax
Net Sales - First Order  = CALCULATE ( [Net Sales], Fact_OrderLine[CohortAgeKey] = 0 )
Net Sales - Repeat Order = CALCULATE ( [Net Sales], Fact_OrderLine[CohortAgeKey] > 0 )

Repeat Share = DIVIDE ( [Net Sales - Repeat Order], [Net Sales] )

-- Filter cohort dipasang di pembilang DAN penyebut.
Margin % - First Order =
DIVIDE (
    CALCULATE ( [Contribution Margin],  Fact_OrderLine[CohortAgeKey] = 0 ),
    CALCULATE ( [Contribution Revenue], Fact_OrderLine[CohortAgeKey] = 0 )
)

Margin % - Repeat Order =
DIVIDE (
    CALCULATE ( [Contribution Margin],  Fact_OrderLine[CohortAgeKey] > 0 ),
    CALCULATE ( [Contribution Revenue], Fact_OrderLine[CohortAgeKey] > 0 )
)

Margin % - All Orders = [Margin %]

Average Order Value = DIVIDE ( [Net Sales], [Orders] )
```

Acuan: Margin % first order 13,4%; repeat 22,9%; Average Order Value €109,05.

## 4 Growth (YoY)

Kartu KPI selalu membandingkan tahun terbaru vs tahun sebelumnya, apa pun pilihan slicer Year. `REMOVEFILTERS ( Dim_Date )` membuang pilihan tahun dari slicer, lalu tahun dikunci secara eksplisit.

```dax
-- Konstan. Trade-off: ubah angka ini bila ada data 2026.
Latest Year = 2025

Net Sales YoY % =
VAR LY   = [Latest Year]
VAR Curr = CALCULATE ( [Net Sales], REMOVEFILTERS ( Dim_Date ), Dim_Date[Year] = LY )
VAR Prev = CALCULATE ( [Net Sales], REMOVEFILTERS ( Dim_Date ), Dim_Date[Year] = LY - 1 )
RETURN DIVIDE ( Curr - Prev, Prev )
```

`Gross Sales YoY %`, `Contribution Margin YoY %`, dan `Orders YoY %` memakai pola yang sama dengan mengganti `[Net Sales]`. Format custom: `+0.0%;-0.0%;0.0%`.

```dax
-- Selisih persentase dalam POIN, bukan persen.
Margin % YoY (pts) =
VAR LY   = [Latest Year]
VAR Curr = CALCULATE ( [Margin %], REMOVEFILTERS ( Dim_Date ), Dim_Date[Year] = LY )
VAR Prev = CALCULATE ( [Margin %], REMOVEFILTERS ( Dim_Date ), Dim_Date[Year] = LY - 1 )
RETURN ( Curr - Prev ) * 100
```

`Repeat Share YoY (pts)` sama, dengan `[Repeat Share]`. Format custom: `+0.0" pts";-0.0" pts";0.0" pts"`.

| Measure | Acuan |
|---|---:|
| Gross Sales YoY % | +68,9% |
| Net Sales YoY % | +71,0% |
| Contribution Margin YoY % | +103,7% |
| Orders YoY % | +67,4% |
| Margin % YoY (pts) | +3,3 pts |
| Repeat Share YoY (pts) | +23,0 pts |

## 5 Costs

```dax
Product Cost = SUM ( Fact_OrderLine[ProductCost] )

-- Delivery = ongkir keluar + biaya fulfillment
Delivery Cost =
SUM ( Fact_OrderLine[OutboundShippingCost] ) + SUM ( Fact_OrderLine[FulfillmentCost] )

Payment Fees = SUM ( Fact_OrderLine[PaymentFee] )
Marketing Cost = SUM ( Fact_OrderLine[AllocatedMarketingCost] )

-- Return costs = ongkir balik + biaya proses retur
Return Costs =
SUM ( Fact_OrderLine[ReturnShippingCost] ) + SUM ( Fact_OrderLine[ReturnProcessingCost] )

Shipping Charged = SUM ( Fact_OrderLine[ShippingRevenue] )

-- Termasuk product cost.
Variable Costs =
[Product Cost] + [Delivery Cost] + [Payment Fees] + [Marketing Cost] + [Return Costs]
```

| Measure | Acuan |
|---|---:|
| Product Cost | €386.537 |
| Delivery Cost | €75.143 |
| Payment Fees | €26.601 |
| Marketing Cost | €29.880 |
| Return Costs | €11.393 |
| Shipping Charged | €16.785 |
| Variable Costs | €529.555 |

## 6 Waterfall

Visual: stacked bar horizontal dengan lapisan `Waterfall Base` (transparan) dan 4 seri warna. Sumbu Y memakai tabel mandiri `Calc_Waterfall[Step]` (tanpa relationship, sort by `Order`).

```dax
Waterfall Value =
VAR Step = SELECTEDVALUE ( Calc_Waterfall[Step] )
RETURN
    SWITCH (
        Step,
        "Gross sales",          [Gross Sales],
        "Discounts",            -[Discounts],
        "Refunds",              -[Refunds],
        "Net sales",            [Net Sales],
        "Shipping charged",     [Shipping Charged],
        "Contribution revenue", [Contribution Revenue],
        "Product cost",         -[Product Cost],
        "Delivery",             -[Delivery Cost],
        "Payment fees",         -[Payment Fees],
        "Marketing",            -[Marketing Cost],
        "Return costs",         -[Return Costs],
        "Contribution margin",  [Contribution Margin]
    )

Waterfall Length = ABS ( [Waterfall Value] )

Waterfall Base =
VAR Step = SELECTEDVALUE ( Calc_Waterfall[Step] )
RETURN
    SWITCH (
        Step,
        "Gross sales",          0,
        "Discounts",            [Gross Sales] - [Discounts],
        "Refunds",              [Net Sales],
        "Net sales",            0,
        "Shipping charged",     [Net Sales],
        "Contribution revenue", 0,
        "Product cost",         [Contribution Revenue] - [Product Cost],
        "Delivery",             [Contribution Revenue] - [Product Cost] - [Delivery Cost],
        "Payment fees",         [Contribution Revenue] - [Product Cost] - [Delivery Cost] - [Payment Fees],
        "Marketing",            [Contribution Revenue] - [Product Cost] - [Delivery Cost] - [Payment Fees] - [Marketing Cost],
        "Return costs",         [Contribution Margin],
        "Contribution margin",  0
    )

WF Total =
VAR Step = SELECTEDVALUE ( Calc_Waterfall[Step] )
RETURN IF ( Step IN { "Gross sales", "Net sales", "Contribution revenue" }, [Waterfall Length] )

WF Decrease =
VAR Step = SELECTEDVALUE ( Calc_Waterfall[Step] )
RETURN
    IF (
        Step IN { "Discounts", "Refunds", "Product cost", "Delivery",
                  "Payment fees", "Marketing", "Return costs" },
        [Waterfall Length]
    )

WF Increase =
VAR Step = SELECTEDVALUE ( Calc_Waterfall[Step] )
RETURN IF ( Step = "Shipping charged", [Waterfall Length] )

WF Margin =
VAR Step = SELECTEDVALUE ( Calc_Waterfall[Step] )
RETURN IF ( Step = "Contribution margin", [Waterfall Length] )
```

Rantai: `Waterfall Value` → `Waterfall Length` → 4 seri warna. `Waterfall Value` dan `Waterfall Length` disembunyikan dari Report view tetapi tidak boleh dihapus.

## 7 Focus Areas

```dax
Loss-Making Products =
COUNTROWS ( FILTER ( VALUES ( Dim_Product[ProductKey] ), [Contribution Margin] < 0 ) )

Product Cost % of Revenue = DIVIDE ( [Product Cost], [Contribution Revenue] )

Return Loss = SUM ( Fact_OrderLine[ReturnLoss] )
Return Loss % of Gross Sales = DIVIDE ( [Return Loss], [Gross Sales] )

Margin % - Promoted Lines =
DIVIDE (
    CALCULATE ( [Contribution Margin],  Fact_OrderLine[IsPromoted] = 1 ),
    CALCULATE ( [Contribution Revenue], Fact_OrderLine[IsPromoted] = 1 )
)

Margin % - Full Price Lines =
DIVIDE (
    CALCULATE ( [Contribution Margin],  Fact_OrderLine[IsPromoted] = 0 ),
    CALCULATE ( [Contribution Revenue], Fact_OrderLine[IsPromoted] = 0 )
)

Promo Margin Gap (pts) = ( [Margin % - Promoted Lines] - [Margin % - Full Price Lines] ) * 100

Margin per Customer = DIVIDE ( [Contribution Margin], [Customers] )

Margin % - Owned Commerce =
DIVIDE (
    CALCULATE ( [Contribution Margin],  Dim_SalesChannel[ChannelGroup] = "Owned commerce" ),
    CALCULATE ( [Contribution Revenue], Dim_SalesChannel[ChannelGroup] = "Owned commerce" )
)

Margin % - Marketplace & Social =
DIVIDE (
    CALCULATE ( [Contribution Margin],
        Dim_SalesChannel[ChannelGroup] IN { "Marketplace", "Social commerce" } ),
    CALCULATE ( [Contribution Revenue],
        Dim_SalesChannel[ChannelGroup] IN { "Marketplace", "Social commerce" } )
)

-- OnTimeFlag kosong = belum terkirim (142 baris); COUNT mengabaikannya.
On-Time Rate = DIVIDE ( SUM ( Fact_OrderLine[OnTimeFlag] ), COUNT ( Fact_OrderLine[OnTimeFlag] ) )
Average Delivery Days = AVERAGE ( Fact_OrderLine[ActualDeliveryDays] )

Lost Sales = SUM ( Fact_OrderLine[LostSalesValue] )

-- Asumsi penyebut: sales before returns + lost sales (cocok dengan 4,9%).
Lost Sales % of Demand =
DIVIDE ( [Lost Sales], SUM ( Fact_OrderLine[SalesBeforeReturns] ) + [Lost Sales] )
```

| Measure | Acuan |
|---|---:|
| Loss-Making Products | 3 |
| Product Cost % of Revenue | 58,8% |
| Return Loss / % of Gross Sales | €102.898 / 13,0% |
| Margin % promoted / full price / gap | 16,3% / 24,9% / −8,6 pts |
| Margin per Customer | €58,02 |
| Margin % Owned Commerce / Marketplace & Social | 22,0% / 12,2% |
| On-Time Rate / Average Delivery Days | 92,8% / 3,3 |
| Lost Sales / % of Demand | €36.715 / 4,9% |

## 9 QA

```dax
-- Harus selalu 0. Bukti bahwa semua biaya terhitung di waterfall.
Margin Check = [Contribution Revenue] - [Variable Costs] - [Contribution Margin]
```

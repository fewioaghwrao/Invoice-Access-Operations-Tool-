# Invoice Access Operations Tool - 詳細設計書

## 1. 文書概要

### 1.1 目的

本書は、Invoice Management System に追加する Access 月次照合用Export機能、および Microsoft Access 側のAPI取得・CSV取込・月次照合処理について、実装レベルの詳細設計を定義する。

Invoice側は以下の3種類のデータをCSVとして生成し、1つのZIPファイルとして返却する。

```text
invoices_YYYYMM.csv
payments_YYYYMM.csv
allocations_YYYYMM.csv
```

Access側はAdminログインで取得したJWTを用いてZIPを取得し、3CSVをImport Tableへ取り込み、請求・入金・入金割当を再集計して照合する。

Phase 1では、対象年月に応じた基本集計と `R_MonthlyCheck` による1ページの月次確認レポートまでを実装範囲とする。

---

# 2. 基本方針

責務は以下の4層に分離する。

```text
AccessExportEndpoints
    │
    │ HTTP Request / Validation
    ▼
IAccessExportService
    │
    ▼
AccessExportService
    │
    │ DB → DTO
    ▼
AccessExportDataDto
    │
    ▼
AccessExportCsvBuilder
    │
    │ DTO → CSV
    ▼
AccessExportZipBuilder
    │
    │ 3 CSV → ZIP
    ▼
Results.File
```

各責務は以下とする。

| クラス | 責務 |
|---|---|
| AccessExportEndpoints | HTTP受付、入力値検証、Response生成 |
| AccessExportService | DBから対象データ取得 |
| AccessExportCsvBuilder | DTOからCSV生成 |
| AccessExportZipBuilder | 3CSVをZIP化 |
| AccessExportDataDto | Export対象データ一式を保持 |

AccessExportServiceではCSV生成、ZIP生成、照合判定を行わない。

---

# 3. ファイル構成

## 3.1 Application

```text
InvoiceSystem.Application
│
├─ Dtos
│  └─ AccessExport
│     ├─ AccessInvoiceExportRow.cs
│     ├─ AccessPaymentExportRow.cs
│     ├─ AccessAllocationExportRow.cs
│     └─ AccessExportDataDto.cs
│
├─ Queries
│  └─ AccessExport
│     └─ AccessExportQuery.cs
│
└─ Services
   └─ AccessExport
      └─ IAccessExportService.cs
```

## 3.2 Infrastructure

```text
InvoiceSystem.Infrastructure
│
├─ Services
│  └─ AccessExport
│     └─ AccessExportService.cs
│
└─ InfrastructureServices.cs
```

## 3.3 API

```text
InvoiceSystem.Api
│
├─ Endpoints
│  └─ AccessExportEndpoints.cs
│
├─ Utils
│  ├─ AccessExportCsvBuilder.cs
│  └─ AccessExportZipBuilder.cs
│
└─ Program.cs
```

---

# 4. DTO詳細設計

## 4.1 AccessInvoiceExportRow

### Namespace

```csharp
InvoiceSystem.Application.Dtos.AccessExport
```

### 定義

```csharp
public sealed record AccessInvoiceExportRow(
    long InvoiceId,
    long MemberId,
    string InvoiceNumber,
    DateTime InvoiceDate,
    DateTime DueDate,
    decimal TotalAmount,
    string StatusCode,
    string StatusName,
    bool IsOverdue,
    bool IsClosed,
    string MemberName);
```

### 項目

| 項目 | 元データ |
|---|---|
| InvoiceId | Invoice.Id |
| MemberId | Invoice.MemberId |
| InvoiceNumber | Invoice.InvoiceNumber |
| InvoiceDate | Invoice.InvoiceDate |
| DueDate | Invoice.DueDate |
| TotalAmount | Invoice.TotalAmount |
| StatusCode | Invoice.Status.Code |
| StatusName | Invoice.Status.Name |
| IsOverdue | Invoice.Status.IsOverdue |
| IsClosed | Invoice.Status.IsClosed |
| MemberName | Invoice.Member.Name |

---

# 5. AccessPaymentExportRow

### Namespace

```csharp
InvoiceSystem.Application.Dtos.AccessExport
```

### 定義

```csharp
public sealed record AccessPaymentExportRow(
    long PaymentId,
    long MemberId,
    DateTime PaymentDate,
    decimal Amount,
    string? PayerName,
    string? Method);
```

### 項目

| 項目 | 元データ |
|---|---|
| PaymentId | Payment.Id |
| MemberId | Payment.MemberId |
| PaymentDate | Payment.PaymentDate |
| Amount | Payment.Amount |
| PayerName | Payment.PayerName |
| Method | Payment.Method |

---

# 6. AccessAllocationExportRow

### Namespace

```csharp
InvoiceSystem.Application.Dtos.AccessExport
```

### 定義

```csharp
public sealed record AccessAllocationExportRow(
    long AllocationId,
    long PaymentId,
    long InvoiceId,
    decimal Amount);
```

### 項目

| 項目 | 元データ |
|---|---|
| AllocationId | PaymentAllocation.Id |
| PaymentId | PaymentAllocation.PaymentId |
| InvoiceId | PaymentAllocation.InvoiceId |
| Amount | PaymentAllocation.Amount |

---

# 7. AccessExportDataDto

3種類のExportデータを1単位として保持する。

```csharp
public sealed record AccessExportDataDto(
    int Year,
    int Month,
    IReadOnlyList<AccessInvoiceExportRow> Invoices,
    IReadOnlyList<AccessPaymentExportRow> Payments,
    IReadOnlyList<AccessAllocationExportRow> Allocations);
```

EndpointとZIP Builderの間では本DTOを受け渡す。

---

# 8. Query詳細設計

## 8.1 AccessExportQuery

### Namespace

```csharp
InvoiceSystem.Application.Queries.AccessExport
```

### 定義

```csharp
public sealed class AccessExportQuery
{
    public int Year { get; init; }
    public int Month { get; init; }
}
```

HTTP RequestをApplication層へ直接渡さず、EndpointでQueryへ変換する。

---

# 9. Service Interface

## 9.1 IAccessExportService

### Namespace

```csharp
InvoiceSystem.Application.Services.AccessExport
```

### 定義

```csharp
public interface IAccessExportService
{
    Task<AccessExportDataDto> ExportAsync(
        AccessExportQuery query);
}
```

### 責務

以下のみを担当する。

```text
対象期間計算
↓
Invoice取得
↓
Payment取得
↓
PaymentAllocation取得
↓
DTO作成
```

CSVやZIPは生成しない。

---

# 10. AccessExportService

## 10.1 Namespace

```csharp
InvoiceSystem.Infrastructure.Services.AccessExport
```

## 10.2 フィールド

```csharp
private readonly AppDbContext _db;
```

## 10.3 Constructor

```csharp
public AccessExportService(AppDbContext db)
{
    _db = db;
}
```

---

# 11. 対象期間算出

指定されたYear / Monthから翌月1日を生成する。

例：

```text
Year  = 2026
Month = 9
```

の場合、

```text
NextMonthStart = 2026-10-01 00:00:00 UTC
```

とする。

実装例：

```csharp
var nextMonthStart = new DateTime(
    query.Year,
    query.Month,
    1,
    0,
    0,
    0,
    DateTimeKind.Utc)
    .AddMonths(1);
```

抽出条件は、

```text
Date < NextMonthStart
```

とする。

月末日の23:59:59等は使用しない。

---

# 12. Invoice取得処理

## 12.1 抽出条件

```text
Invoice.InvoiceDate < NextMonthStart
```

StatusやMember.IsActiveでは除外しない。

## 12.2 Query

```csharp
var invoices = await _db.Invoices
    .AsNoTracking()
    .Where(i => i.InvoiceDate < nextMonthStart)
    .OrderBy(i => i.Id)
    .Select(i => new AccessInvoiceExportRow(
        i.Id,
        i.MemberId,
        i.InvoiceNumber,
        i.InvoiceDate,
        i.DueDate,
        i.TotalAmount,
        i.Status.Code,
        i.Status.Name,
        i.Status.IsOverdue,
        i.Status.IsClosed,
        i.Member.Name))
    .ToListAsync();
```

### 補足

`Include()` は使用しない。

Projection内で必要な関連情報を参照し、EF Coreに必要なJOINを生成させる。

---

# 13. Payment取得処理

## 13.1 抽出条件

```text
Payment.PaymentDate < NextMonthStart
```

## 13.2 Query

```csharp
var payments = await _db.Payments
    .AsNoTracking()
    .Where(p => p.PaymentDate < nextMonthStart)
    .OrderBy(p => p.Id)
    .Select(p => new AccessPaymentExportRow(
        p.Id,
        p.MemberId,
        p.PaymentDate,
        p.Amount,
        p.PayerName,
        p.Method))
    .ToListAsync();
```

---

# 14. PaymentAllocation取得処理

## 14.1 抽出条件

以下の両方を満たすこと。

```text
Allocation.Invoice.InvoiceDate < NextMonthStart

AND

Allocation.Payment.PaymentDate < NextMonthStart
```

## 14.2 Query

```csharp
var allocations = await _db.PaymentAllocations
    .AsNoTracking()
    .Where(a =>
        a.Invoice.InvoiceDate < nextMonthStart &&
        a.Payment.PaymentDate < nextMonthStart)
    .OrderBy(a => a.Id)
    .Select(a => new AccessAllocationExportRow(
        a.Id,
        a.PaymentId,
        a.InvoiceId,
        a.Amount))
    .ToListAsync();
```

これにより、出力されたInvoice・Paymentに紐づくAllocationだけが対象となる。

---

# 15. Service戻り値

```csharp
return new AccessExportDataDto(
    query.Year,
    query.Month,
    invoices,
    payments,
    allocations);
```

---

# 16. CSV Builder

## 16.1 クラス

```csharp
public static class AccessExportCsvBuilder
```

## 16.2 メソッド

```csharp
public static string BuildInvoices(
    IReadOnlyList<AccessInvoiceExportRow> rows);

public static string BuildPayments(
    IReadOnlyList<AccessPaymentExportRow> rows);

public static string BuildAllocations(
    IReadOnlyList<AccessAllocationExportRow> rows);
```

---

# 17. invoices CSV生成

## 17.1 Header

```text
InvoiceId,MemberId,InvoiceNumber,InvoiceDate,DueDate,TotalAmount,StatusCode,StatusName,IsOverdue,IsClosed,MemberName
```

## 17.2 日付

```csharp
row.InvoiceDate.ToString("yyyy-MM-dd")
```

```csharp
row.DueDate.ToString("yyyy-MM-dd")
```

## 17.3 金額

CultureInfo.InvariantCultureを使用する。

```csharp
row.TotalAmount.ToString(
    "0.00",
    CultureInfo.InvariantCulture)
```

---

# 18. payments CSV生成

Header：

```text
PaymentId,MemberId,PaymentDate,Amount,PayerName,Method
```

日付：

```csharp
row.PaymentDate.ToString("yyyy-MM-dd")
```

金額：

```csharp
row.Amount.ToString(
    "0.00",
    CultureInfo.InvariantCulture)
```

nullのPayerName / Methodは空文字として出力する。

---

# 19. allocations CSV生成

Header：

```text
AllocationId,PaymentId,InvoiceId,Amount
```

金額：

```csharp
row.Amount.ToString(
    "0.00",
    CultureInfo.InvariantCulture)
```

---

# 20. CSV Escape

共通privateメソッドを定義する。

```csharp
private static string Escape(string? value)
```

nullの場合：

```text
""
```

を返す。

文字列に以下のいずれかが含まれる場合、

```text
,
"
\r
\n
```

CSV全体を `"` で囲む。

文字列内部の、

```text
"
```

は、

```text
""
```

へ変換する。

概念実装：

```csharp
private static string Escape(string? value)
{
    if (string.IsNullOrEmpty(value))
        return string.Empty;

    var escaped = value.Replace("\"", "\"\"");

    if (escaped.Contains(',') ||
        escaped.Contains('"') ||
        escaped.Contains('\r') ||
        escaped.Contains('\n'))
    {
        return $"\"{escaped}\"";
    }

    return escaped;
}
```

---

# 21. CSV改行コード

CSVはCRLFで生成する。

```csharp
const string NewLine = "\r\n";
```

`StringBuilder.AppendLine()` のOS依存動作には依存せず、必要に応じて明示的にCRLFを付与する。

---

# 22. BOM

CSV文字列自体にはBOMを付けない。

ZIP Entryへ書き込む際に、

```csharp
Encoding.UTF8.GetPreamble()
```

を先頭へ書き込む。

各CSVファイルそれぞれにBOMを付与する。

---

# 23. ZIP Builder

## 23.1 クラス

```csharp
public static class AccessExportZipBuilder
```

## 23.2 Public Method

```csharp
public static byte[] Build(
    AccessExportDataDto data)
```

---

# 24. ZIP生成フロー

```text
AccessExportDataDto
        │
        ├─ BuildInvoices()
        ├─ BuildPayments()
        └─ BuildAllocations()
        │
        ▼
3 CSV string
        │
        ▼
MemoryStream
        │
        ▼
ZipArchive
        │
        ├─ invoices_YYYYMM.csv
        ├─ payments_YYYYMM.csv
        └─ allocations_YYYYMM.csv
        │
        ▼
byte[]
```

---

# 25. ZIP内ファイル名

```csharp
var ym = $"{data.Year}{data.Month:00}";
```

生成するファイル：

```text
invoices_{ym}.csv
payments_{ym}.csv
allocations_{ym}.csv
```

例：

```text
invoices_202609.csv
payments_202609.csv
allocations_202609.csv
```

---

# 26. ZIP生成処理

概念実装：

```csharp
using var stream = new MemoryStream();

using (var archive = new ZipArchive(
    stream,
    ZipArchiveMode.Create,
    leaveOpen: true))
{
    WriteEntry(
        archive,
        $"invoices_{ym}.csv",
        invoicesCsv);

    WriteEntry(
        archive,
        $"payments_{ym}.csv",
        paymentsCsv);

    WriteEntry(
        archive,
        $"allocations_{ym}.csv",
        allocationsCsv);
}

return stream.ToArray();
```

---

# 27. ZIP Entry書込

privateメソッド：

```csharp
private static void WriteEntry(
    ZipArchive archive,
    string fileName,
    string content)
```

処理：

```text
ZIP Entry作成
↓
Entry Stream取得
↓
UTF-8 BOM書込
↓
CSV本体書込
```

---

# 28. Endpoint

## 28.1 Route

```http
GET /api/admin/access-export
```

## 28.2 Query Parameter

```text
year
month
```

例：

```http
GET /api/admin/access-export?year=2026&month=9
```

---

# 29. Authorization

Route Group単位で、

```csharp
.RequireAuthorization("AdminOnly")
```

を設定する。

```csharp
var group = app
    .MapGroup("/api/admin/access-export")
    .WithTags("Access Export")
    .RequireAuthorization("AdminOnly");
```

個別Endpointでの重複指定は行わない。

---

# 30. AccessExportRequest

Endpoint内部クラスとする。

```csharp
public sealed class AccessExportRequest
{
    public int? Year { get; set; }
    public int? Month { get; set; }
}
```

---

# 31. 入力Validation

以下の順にValidationする。

## Year未指定

```csharp
if (request.Year is null)
    return Results.BadRequest("year is required.");
```

## Month未指定

```csharp
if (request.Month is null)
    return Results.BadRequest("month is required.");
```

## Month範囲外

```csharp
if (request.Month is < 1 or > 12)
    return Results.BadRequest(
        "month must be between 1 and 12.");
```

## Year範囲外

```csharp
if (request.Year is < 1 or > 9999)
    return Results.BadRequest(
        "year is invalid.");
```

---

# 32. Query変換

Validation完了後、

```csharp
var query = new AccessExportQuery
{
    Year = request.Year.Value,
    Month = request.Month.Value
};
```

を生成する。

---

# 33. Service呼出

```csharp
var data = await service.ExportAsync(query);
```

---

# 34. ZIP生成

```csharp
var bytes =
    AccessExportZipBuilder.Build(data);
```

---

# 35. HTTP Response

ファイル名：

```csharp
var fileName =
    $"invoice-access-{query.Year}{query.Month:00}.zip";
```

Response：

```csharp
return Results.File(
    bytes,
    "application/zip",
    fileName);
```

---

# 36. DI登録

InfrastructureServices.csへ追加する。

using：

```csharp
using InvoiceSystem.Application.Services.AccessExport;
using InvoiceSystem.Infrastructure.Services.AccessExport;
```

登録：

```csharp
services.AddScoped<
    IAccessExportService,
    AccessExportService>();
```

配置：

```csharp
// 売上一覧
services.AddScoped<ISalesService, SalesService>();

// Access月次照合用Export
services.AddScoped<
    IAccessExportService,
    AccessExportService>();
```

---

# 37. Endpoint登録

Program.csへ追加する。

```csharp
app.MapAccessExportEndpoints();
```

配置例：

```csharp
app.MapSalesEndpoints();
app.MapAccessExportEndpoints();
app.MapAdminEndpoints();
```

---

# 38. 空データ

0件の場合もCSVを生成する。

例えばPaymentが0件の場合、

```csv
PaymentId,MemberId,PaymentDate,Amount,PayerName,Method
```

だけを持つCSVをZIPへ格納する。

3ファイルのいずれも省略しない。

---

# 39. CANCELLED

Invoice.Status.Codeが、

```text
CANCELLED
```

でもCSVへ出力する。

AccessExportServiceでは除外しない。

Access側の照合Queryで通常集計から除外する。

---

# 40. Member.IsActive

Member.IsActiveはExport条件に使用しない。

退会済みMemberの過去Invoice / Paymentも照合対象として保持する。

---

# 41. Access側ステータス再計算

Invoice側Exportでは再計算しない。

Access側で、

```text
AllocatedAmount =
SUM(PaymentAllocation.Amount)
GROUP BY InvoiceId
```

を実行する。

---

# 42. Access側入金状態

| 条件 | 結果 |
|---|---|
| Allocation = 0 | UNPAID |
| 0 < Allocation < TotalAmount | PARTIAL |
| Allocation = TotalAmount | PAID |
| Allocation > TotalAmount | OVER_ALLOCATED |

---

# 43. 期限超過

Access側で、

```text
DueDate < 対象月末
AND
未決済
```

の場合に期限超過判定を行う。

Invoice.Status.IsOverdueとの比較に使用する。

---

# 44. CANCELLED照合

StatusCodeが、

```text
CANCELLED
```

の場合、通常のUNPAID / PARTIAL / PAID比較から除外する。

ただし、

```text
AllocationAmount > 0
```

の場合、

```text
CANCELLED_WITH_ALLOCATION
```

として確認対象にする。

---

# 45. Payment照合

PaymentIdごとに、

```text
SUM(Allocation.Amount)
```

を計算する。

| 条件 | 判定 |
|---|---|
| 0 | UNALLOCATED |
| Payment.Amountより少ない | PARTIAL |
| Payment.Amountと一致 | ALLOCATED |
| Payment.Amountより多い | OVER_ALLOCATED |

---

# 46. ログ

Phase 1では既存SerilogのHTTP Request Loggingを利用する。

追加業務ログを実装する場合は以下を出力する。

```text
AccessExport
Year
Month
InvoiceCount
PaymentCount
AllocationCount
```

MemberName、PayerName等の個人情報はログに出力しない。

---

# 47. Exception

DB取得中、CSV生成中、ZIP生成中に予期しないExceptionが発生した場合は、部分的なZIPを返却しない。

既存APIのグローバルな例外処理方針に従う。

Phase 1ではEndpoint内部で広範囲な、

```csharp
catch (Exception)
```

を追加しない。

---

# 48. PaymentAllocation時点制約

PaymentAllocationには作成日時が存在しない。

そのため指定年月の過去時点状態を後日完全再現することはできない。

本Exportは、

```text
Export実行時点の現在データ
+
指定年月以前のInvoice / Payment
```

を使用した照合データ生成とする。

---

# 49. テスト方針

最低限以下をテストする。

## Service

### Test 1

```text
指定月以前のInvoiceが出力される
```

### Test 2

```text
翌月以降のInvoiceが出力されない
```

### Test 3

```text
指定月以前のPaymentが出力される
```

### Test 4

```text
翌月以降のPaymentが出力されない
```

### Test 5

```text
対象Invoice / Paymentに対応するAllocationが出力される
```

### Test 6

```text
CANCELLED Invoiceも出力される
```

---

# 50. CSV Builderテスト

以下を確認する。

```text
Headerが正しい
日付がyyyy-MM-dd
decimalがInvariantCulture
NULLが空文字
カンマを含む文字列がQuoteされる
" が "" にEscapeされる
改行を含む文字列がQuoteされる
0件でもHeaderが生成される
```

---

# 51. ZIP Builderテスト

以下を確認する。

```text
ZIPが開ける
3CSVが必ず存在する
ファイル名がYYYYMM形式
各CSVがUTF-8 BOM付き
各CSV Headerが正しい
```

---

# 52. Endpoint Integration Test

## 正常系

```http
GET /api/admin/access-export?year=2026&month=9
```

期待：

```text
200 OK
Content-Type: application/zip
```

ファイル名：

```text
invoice-access-202609.zip
```

---

# 53. Endpoint異常系

以下を確認する。

```text
yearなし → 400
monthなし → 400
month=0 → 400
month=13 → 400
未認証 → 401
一般Member → 403
Admin → 200
```

---

# 54. 実装順序

## 54.1 Invoice側

```text
1. AccessInvoiceExportRow
2. AccessPaymentExportRow
3. AccessAllocationExportRow
4. AccessExportDataDto
5. AccessExportQuery
6. IAccessExportService
7. AccessExportService
8. InfrastructureServices DI
9. AccessExportCsvBuilder
10. AccessExportZipBuilder
11. AccessExportEndpoints
12. Program.cs Endpoint登録
13. build
14. Unit Test
15. Integration Test
16. Swagger / 実API確認
```

## 54.2 Access側

```text
1. Import Table作成
2. 照合Query作成
3. F_CheckResult作成
4. F_MainMenu作成
5. F_CsvImport作成
6. F_Login作成
7. JWTログイン連携
8. ZIP取得処理
9. ZIP展開処理
10. UTF-8 CSV自動取込
11. TargetYear / TargetMonth保持
12. Q_TargetMonthInvoices
13. Q_TargetMonthPayments
14. Q_MonthlySummary動的集計
15. R_MonthlyCheck
16. VPS APIとのE2E動作確認
```

---

# 55. 実装完了確認

以下をすべて満たした場合、Phase 1実装完了とする。

## 55.1 Invoice側

```text
dotnet build 成功
    ↓
GET /api/admin/access-export
?year=YYYY
&month=MM
    ↓
HTTP 200
    ↓
invoice-access-YYYYMM.zip
    ↓
invoices_YYYYMM.csv
payments_YYYYMM.csv
allocations_YYYYMM.csv
    ↓
各CSVがUTF-8 BOM付き
```

## 55.2 Access側

```text
F_Login
    ↓
POST /auth/login
    ↓
JWT取得
    ↓
F_CsvImport
    ↓
TargetYear / TargetMonth
    ↓
Invoiceから取得・取込
    ↓
ZIP取得・展開
    ↓
3CSV UTF-8取込
    ↓
Invoice / Payment / Allocation照合
    ↓
Q_TargetMonthInvoices
Q_TargetMonthPayments
    ↓
Q_MonthlySummary
    ↓
R_MonthlyCheck
```

## 55.3 E2E

本番相当のVPS APIにHTTPS接続し、以下を確認する。

```text
Adminログイン成功
Access Export API成功
ZIP取得成功
CSV取込成功
対象月抽出成功
月次サマリー成功
月次確認レポート表示成功
```

2027年2月の動作確認例：

```text
Import Snapshot
Invoice    : 17件
Payment    : 16件
Allocation : 16件

Target Month
InvoiceCount  : 4件
InvoiceAmount : 450000
PaymentCount  : 5件
PaymentAmount : 400000
```

上記の基本集計が `Q_MonthlySummary` と `R_MonthlyCheck` で一致することを確認する。

---

# 56. Phase 1 実装対象外

以下は今回実装しない。

```text
AccessからInvoiceへの更新
DB直接接続
月次締めロック
過去時点完全復元
PaymentAllocation履歴
Excel出力
移行CSV変換
Exportファイルのサーバー保存
```

---

# 57. 最終クラス・処理関係

## 57.1 Invoice側

```text
AccessExportEndpoints
        │
        ▼
AccessExportQuery
        │
        ▼
IAccessExportService
        ▲
        │ implements
AccessExportService
        │
        ▼
AppDbContext
        │
        ├─ Invoices
        ├─ Payments
        └─ PaymentAllocations
        │
        ▼
AccessExportDataDto
        │
        ▼
AccessExportCsvBuilder
        │
        ▼
AccessExportZipBuilder
        │
        ▼
byte[]
        │
        ▼
Results.File
```

## 57.2 Access側

```text
F_Login
    │
    ├─ ApiBaseUrl
    ├─ Email
    └─ Password
    │
    ▼
POST /auth/login
    │
    ▼
TempVars
    ├─ AccessToken
    └─ ApiBaseUrl
    │
    ▼
F_CsvImport
    ├─ TargetYear
    └─ TargetMonth
    │
    ▼
TempVars
    ├─ TargetYear
    └─ TargetMonth
    │
    ▼
GET /api/admin/access-export
    │
    ▼
ZIP
    │
    ▼
3CSV
    │
    ▼
Import Tables
    │
    ├─ 照合Query
    ├─ Q_TargetMonthInvoices
    └─ Q_TargetMonthPayments
    │
    ▼
Q_MonthlySummary
    │
    ▼
R_MonthlyCheck
```

本構成により、既存Sales Exportとは独立したAccess月次照合用データ出力機能と、Access側の月次照合EUCを一連の処理として構成する。

---



---

# 58. F_Login 詳細設計

## 58.1 コントロール

```text
txtApiBaseUrl
txtEmail
txtPassword
btnLogin
```

## 58.2 ログイン処理

`btnLogin_Click` では以下を行う。

```text
入力値取得
    ↓
必須チェック
    ↓
ApiBaseUrl末尾 / 除去
    ↓
JSON生成
    ↓
POST /auth/login
    ↓
HTTP Status確認
    ↓
Response JSONからJWT抽出
    ↓
TempVars更新
    ↓
Password欄クリア
    ↓
F_MainMenu表示
```

Request：

```json
{
  "email": "...",
  "password": "..."
}
```

Header：

```http
Content-Type: application/json
Accept: application/json
```

HTTP 200以外の場合は、`Status` と `ResponseText` を表示して処理を終了する。

## 58.3 JSON文字列処理

メールアドレスとパスワードはJSON文字列として送信するため、最低限以下をEscapeする。

```text
\      → \\
"      → \"
CR/LF  → \n
```

## 58.4 JWT抽出

レスポンス差異に備え、以下のキーを順番に確認する。

```text
accessToken
token
access_token
```

現行Invoice APIの `LoginResponse` では `Token` が返却される。

## 58.5 TempVars

ログイン成功時：

```vb
TempVars.Add "AccessToken", token
TempVars.Add "ApiBaseUrl", baseUrl
```

既存値がある場合は削除してから再設定する。

パスワードはTempVarsへ保存しない。

---

# 59. F_CsvImport 詳細設計

## 59.1 コントロール

```text
txtInvoiceCsv
txtPaymentCsv
txtAllocationCsv

txtTargetYear
txtTargetMonth

btnDownloadFromInvoice
btnImport
btnBack
lblApiStatus
```

`btnDownloadFromInvoice` の表示名は「Invoiceから取得・取込」とする。

## 59.2 対象年月

入力値を検証する。

```text
TargetYear  : 2000～2100
TargetMonth : 1～12
```

検証後、対象年月をTempVarsへ保存する。

```vb
TempVars.Add "TargetYear", targetYear
TempVars.Add "TargetMonth", targetMonth
```

---

# 60. Access Export API呼出 詳細

Request URL：

```text
{ApiBaseUrl}/api/admin/access-export?year={year}&month={month}
```

Header：

```http
Authorization: Bearer {AccessToken}
Accept: application/zip
```

Response：

```text
HTTP 200
Content-Type: application/zip
```

保存先：

```text
%TEMP%\InvoiceAccessOperations\invoice-access-YYYYMM.zip
```

保存には `ADODB.Stream` のBinary Modeを利用する。

HTTP 401 / 403 は認証・権限エラーとして利用者へ通知する。

---

# 61. ZIP展開 詳細

ZIP展開にはPowerShellを利用する。

```powershell
Expand-Archive -LiteralPath '<zip>' -DestinationPath '<dir>' -Force
```

展開先：

```text
%TEMP%\InvoiceAccessOperations\extract-YYYYMM
```

展開前に同名ディレクトリが存在する場合は削除して再作成する。

展開後、以下3ファイルの存在を確認する。

```text
invoices_YYYYMM.csv
payments_YYYYMM.csv
allocations_YYYYMM.csv
```

1つでも存在しない場合はエラーとする。

---

# 62. CSV取込 詳細

## 62.1 既存データ削除

```text
T_AllocationImport
    ↓
T_PaymentImport
    ↓
T_InvoiceImport
```

の順に削除する。

## 62.2 UTF-8取込

```vb
DoCmd.TransferText _
    TransferType:=acImportDelim, _
    TableName:="T_InvoiceImport", _
    FileName:=CStr(Me.txtInvoiceCsv.Value), _
    HasFieldNames:=True, _
    CodePage:=65001
```

Payment / Allocationも同様に `CodePage:=65001` を指定する。

これはUTF-8 BOM付きCSVの先頭ヘッダーを正しく認識するために必要である。

## 62.3 取込完了表示

以下の件数を `DCount` で表示する。

```text
Invoice
Payment
Allocation
```

---

# 63. Q_TargetMonthInvoices 詳細

目的：

> 累積スナップショットから、対象年月に発生したInvoiceだけを基本集計用に抽出する。

SQL：

```sql
SELECT *
FROM T_InvoiceImport
WHERE
    InvoiceDate >= DateSerial(
        TempVars!TargetYear,
        TempVars!TargetMonth,
        1
    )
    AND
    InvoiceDate < DateSerial(
        TempVars!TargetYear,
        TempVars!TargetMonth + 1,
        1
    );
```

`DateSerial` に翌月を指定することで、12月から翌年1月への繰上げもAccess側に任せる。

---

# 64. Q_TargetMonthPayments 詳細

目的：

> 累積スナップショットから、対象年月に発生したPaymentだけを基本集計用に抽出する。

SQL：

```sql
SELECT *
FROM T_PaymentImport
WHERE
    PaymentDate >= DateSerial(
        TempVars!TargetYear,
        TempVars!TargetMonth,
        1
    )
    AND
    PaymentDate < DateSerial(
        TempVars!TargetYear,
        TempVars!TargetMonth + 1,
        1
    );
```

---

# 65. Q_MonthlySummary 詳細

1レコードの月次サマリーを返す。

SQL：

```sql
SELECT
    DCount("*", "Q_TargetMonthInvoices") AS InvoiceCount,
    Nz(
        DSum(
            "TotalAmount",
            "Q_TargetMonthInvoices",
            "StatusCode <> 'CANCELLED'"
        ),
        0
    ) AS InvoiceAmount,
    DCount("*", "Q_TargetMonthPayments") AS PaymentCount,
    Nz(
        DSum(
            "Amount",
            "Q_TargetMonthPayments"
        ),
        0
    ) AS PaymentAmount,
    DCount("*", "Q_UnpaidInvoices") AS UnpaidCount,
    DCount("*", "Q_PartiallyPaidInvoices") AS PartiallyPaidCount,
    DCount("*", "Q_OverAllocatedInvoices") AS OverAllocatedInvoiceCount,
    DCount("*", "Q_UnallocatedPayments") AS UnallocatedPaymentCount,
    DCount("*", "Q_PaymentOverAllocation") AS PaymentOverAllocationCount,
    DCount("*", "Q_StatusMismatch") AS StatusMismatchCount,
    DCount("*", "Q_CancelledWithAllocation") AS CancelledWithAllocationCount,
    DCount("*", "Q_PastDueMismatch") AS PastDueMismatchCount,
    DCount("*", "Q_OrphanAllocations") AS OrphanAllocationCount
FROM
    T_InvoiceImport
WHERE
    InvoiceId = DMin("InvoiceId", "T_InvoiceImport");
```

基本4項目は対象月Queryを参照する。

照合件数は取込スナップショット全体のQueryを参照する。

---

# 66. R_MonthlyCheck 詳細

## 66.1 RecordSource

```text
Q_MonthlySummary
```

## 66.2 対象年月表示

テキストボックスのControlSource：

```text
="対象年月：" & [TempVars]![TargetYear] & "年" & Format([TempVars]![TargetMonth],"00") & "月"
```

表示例：

```text
対象年月：2027年02月
```

## 66.3 フィールド対応

| 表示 | ControlSource |
|---|---|
| 請求件数 | InvoiceCount |
| 請求金額 | InvoiceAmount |
| 入金件数 | PaymentCount |
| 入金金額 | PaymentAmount |
| 未入金 | UnpaidCount |
| 一部入金 | PartiallyPaidCount |
| 過剰割当 | OverAllocatedInvoiceCount |
| 未割当・一部未割当 | UnallocatedPaymentCount |
| 入金割当超過 | PaymentOverAllocationCount |
| ステータス不整合 | StatusMismatchCount |
| キャンセル請求に割当 | CancelledWithAllocationCount |
| 期限超過状態不整合 | PastDueMismatchCount |
| 参照不整合 | OrphanAllocationCount |

1ページ内で、

```text
基本集計
請求チェック
入金チェック
その他の整合性チェック
```

の4ブロックに分ける。

---

# 67. F_CheckResult 詳細

`Q_CheckResult` をRecordSourceとする。

主な表示項目：

```text
CheckType
TargetType
TargetId
InvoiceNumber
TargetName
BaseAmount
AllocatedAmount
DifferenceAmount
StatusCode
CheckMessage
```

ヘッダーの `cboCheckType` で以下を絞り込む。

```text
すべて
未入金
一部入金
過剰割当
未割当入金
入金割当超過
```

`すべて` の場合は `FilterOn = False` とする。

---

# 68. F_MainMenu 詳細

Access起動時に `F_MainMenu` を表示する。

ボタン：

```text
CSV取込
チェック結果を開く
月次サマリーを開く
月次確認レポート
終了
```

月次確認レポートは印刷プレビューで開く。

```vb
DoCmd.OpenReport "R_MonthlyCheck", acViewPreview
```

---

# 69. E2E動作確認

本番相当構成：

```text
Microsoft Access
    ↓ HTTPS
nginx
    ↓
ASP.NET Core API
    ↓
PostgreSQL
```

2027年2月の確認結果：

```text
API / Import Snapshot
Invoice    : 17
Payment    : 16
Allocation : 16

Q_TargetMonthInvoices
4 records

Q_TargetMonthPayments
5 records

Q_MonthlySummary
InvoiceCount  = 4
InvoiceAmount = 450000
PaymentCount  = 5
PaymentAmount = 400000

R_MonthlyCheck
請求件数 = 4
請求金額 = 450000
入金件数 = 5
入金金額 = 400000
```

対象月Query、月次サマリー、レポートの基本集計が一致することを確認する。

---

# 70. Phase 1 最終状態

Phase 1の最終処理は以下とする。

```text
Invoice Management System
        ↓
Admin JWT認証
        ↓
Access Export API
        ↓
ZIP / 3CSV
        ↓
Microsoft Access
        ↓
Import Table
        ↓
再集計・照合
        ↓
F_CheckResult
        ↓
Q_MonthlySummary
        ↓
R_MonthlyCheck
```

AccessはInvoice Management SystemのCRUDを代替せず、正データを変更しない月次照合・確認用EUCとして運用する。
# 71. Phase 2：ODBC参照専用連携 詳細設計（2026年10月）

## 71.1 PostgreSQL View定義

対象：既存データが存在するローカルDB `invoice_postgres`（5432側）。先行の空DB `invoice-system-postgres`（5433側）とは異なる。PostgreSQL内では`public`スキーマの既存テーブルを読む。Viewは実データを複製しない。

```sql
CREATE SCHEMA IF NOT EXISTS access_reporting;
REVOKE ALL ON SCHEMA access_reporting FROM PUBLIC;

CREATE OR REPLACE VIEW access_reporting.v_invoice_read AS
SELECT
    i."Id" AS "InvoiceId", i."MemberId", i."InvoiceNumber",
    i."InvoiceDate", i."DueDate", i."TotalAmount", i."StatusId",
    s."Code" AS "StatusCode", s."Name" AS "StatusName",
    s."IsOverdue", s."IsClosed", m."Name" AS "MemberName"
FROM public."Invoices" AS i
LEFT JOIN public."InvoiceStatuses" AS s ON i."StatusId" = s."Id"
LEFT JOIN public."Members" AS m ON i."MemberId" = m."Id";

CREATE OR REPLACE VIEW access_reporting.v_payment_read AS
SELECT p."Id" AS "PaymentId", p."MemberId", p."PaymentDate",
       p."Amount", p."PayerName", p."Method"
FROM public."Payments" AS p;

CREATE OR REPLACE VIEW access_reporting.v_allocation_read AS
SELECT a."Id" AS "AllocationId", a."PaymentId", a."InvoiceId", a."Amount"
FROM public."PaymentAllocations" AS a;
```

既存Viewの`CREATE OR REPLACE VIEW`では、元列の順序・名前・型を維持し、追加列を末尾に配置する。今回 `v_invoice_read` の7列の後ろにStatus/Memberの5列を追加した。

## 71.2 DBロール・権限

```sql
-- 未作成の場合のみDB管理者が実行
CREATE ROLE access_readonly LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION;
GRANT CONNECT ON DATABASE invoicesystem TO access_readonly;
GRANT USAGE ON SCHEMA access_reporting TO access_readonly;
GRANT SELECT ON access_reporting.v_invoice_read,
                access_reporting.v_payment_read,
                access_reporting.v_allocation_read TO access_readonly;
```

パスワードは`psql`の `\password access_readonly` で対話的に設定し、SQL・Markdownには記載しない。既存ロールがある環境で`CREATE ROLE`を再実行しないこと。確認には `has_table_privilege` でViewのSELECTと書込権限を検査する。また、基底テーブルへのSELECT権限の有無も確認する。

## 71.3 Accessリンクテーブル

| View | Access側 | 識別子 |
|---|---|---|
| `v_invoice_read` | `L_Invoice` | `InvoiceId` |
| `v_payment_read` | `L_Payment` | `PaymentId` |
| `v_allocation_read` | `L_Allocation` | `AllocationId` |

既設の `T_InvoiceImport` / `T_PaymentImport` / `T_AllocationImport` は維持する。ODBCリンクはAccessの「外部データ」から追加・再リンクする。PostgreSQL Viewに列を追加した後は、Access側でリンクを更新し列構成の反映を確認する。DSNのパスワード保存は行わない方針とする。

## 71.4 Q_ODBC_InvoiceAllocationSummary

```sql
SELECT
    I.InvoiceId, I.MemberId, I.InvoiceNumber,
    I.InvoiceDate, I.DueDate, I.TotalAmount,
    I.StatusCode, I.StatusName, I.IsOverdue, I.IsClosed, I.MemberName,
    Nz(Sum(A.Amount), 0) AS AllocatedAmount,
    I.TotalAmount - Nz(Sum(A.Amount), 0) AS RemainingAmount
FROM L_Invoice AS I
LEFT JOIN L_Allocation AS A ON I.InvoiceId = A.InvoiceId
GROUP BY
    I.InvoiceId, I.MemberId, I.InvoiceNumber,
    I.InvoiceDate, I.DueDate, I.TotalAmount,
    I.StatusCode, I.StatusName, I.IsOverdue, I.IsClosed, I.MemberName;
```

表示列は既設 `Q_InvoiceAllocationSummary` の13項目と対応する。Accessの集計処理では割当なしを `Nz(Sum(...),0)` で0として扱う。ODBCデータの日時・型の解釈とCSV取込後の日付表現の差に注意する。

## 71.5 Q_ODBC_ValidationSummary

```sql
SELECT
    Count(*) AS InvoiceCount,
    Sum(TotalAmount) AS TotalInvoiceAmount,
    Sum(AllocatedAmount) AS TotalAllocatedAmount,
    Sum(RemainingAmount) AS TotalRemainingAmount,
    Sum(IIf(RemainingAmount > 0, 1, 0)) AS UnpaidInvoiceCount,
    Sum(IIf(RemainingAmount < 0, 1, 0)) AS OverAllocatedCount
FROM Q_ODBC_InvoiceAllocationSummary;
```

UnpaidInvoiceCountは残額が正の請求件数（未入金・一部入金を含む）。TotalRemainingAmountは負の残額も含めた正味合計である。

検証用PostgreSQL集計SQL：

```sql
WITH InvoiceSummary AS (
  SELECT i."InvoiceId", i."TotalAmount" AS TotalAmount,
         COALESCE(SUM(a."Amount"), 0) AS AllocatedAmount,
         i."TotalAmount" - COALESCE(SUM(a."Amount"), 0) AS RemainingAmount
  FROM access_reporting.v_invoice_read AS i
  LEFT JOIN access_reporting.v_allocation_read AS a
    ON i."InvoiceId" = a."InvoiceId"
  GROUP BY i."InvoiceId", i."TotalAmount"
)
SELECT COUNT(*) AS "InvoiceCount",
       SUM(TotalAmount) AS "TotalInvoiceAmount",
       SUM(AllocatedAmount) AS "TotalAllocatedAmount",
       SUM(RemainingAmount) AS "TotalRemainingAmount",
       COUNT(*) FILTER (WHERE RemainingAmount > 0) AS "UnpaidInvoiceCount",
       COUNT(*) FILTER (WHERE RemainingAmount < 0) AS "OverAllocatedCount"
FROM InvoiceSummary;
```

## 71.6 Q_ODBC_CsvComparison

```sql
SELECT
    C.InvoiceId,
    C.InvoiceNumber,
    C.TotalAmount AS CsvTotal,
    O.TotalAmount AS OdbcTotal,
    C.AllocatedAmount AS CsvAllocated,
    O.AllocatedAmount AS OdbcAllocated,
    C.RemainingAmount AS CsvRemaining,
    O.RemainingAmount AS OdbcRemaining,
    IIf(
      O.InvoiceId Is Null, "ODBC側なし",
      IIf(C.TotalAmount = O.TotalAmount
          AND C.AllocatedAmount = O.AllocatedAmount
          AND C.RemainingAmount = O.RemainingAmount,
          "一致", "差異あり")
    ) AS CheckResult
FROM Q_InvoiceAllocationSummary AS C
LEFT JOIN Q_ODBC_InvoiceAllocationSummary AS O ON C.InvoiceId = O.InvoiceId
ORDER BY C.InvoiceId;
```

このクエリは**CSV取込側のInvoiceIdを母集団としたLEFT JOIN**であり、ODBC側のみに存在するレコードは比較しない。また、表示される`InvoiceNumber`は判定式に含まれない。両側のデータ時点・対象範囲が異なる場合は差異理由を別途確認する。

## 71.7 確認済み結果と証跡

| 検証 | 結果 | 証跡 |
|---|---|---|
| ODBC請求集計13列 | 全61件を表示 | `images/13-odbc-invoice-allocation.png` |
| Accessの6項目集計 | 61 / 6,310,000 / 4,486,000 / 1,824,000 / 26 / 1 | `images/14-odbc-validation-summary.png` |
| PostgreSQL同一集計 | 上記6項目すべて一致 | `images/15-postgresql-validation-summary.png` |
| 既設CSV17件との比較 | 金額3項目を17件すべて「一致」、差異0 | `images/16-odbc-csv-comparison.png` |
| 比較元CSVの請求照合 | 17件を確認 | `images/17-csv-invoice-allocation.png` |

付帯確認：ODBC接続先5432の元データ件数はInvoice 61、Payment 108、Allocation 58。5433側は別DBであり、ODBCの実データ照合には使用しない。ロールでのView参照成功は確認したが、本書の証跡に**5432側でのUPDATE拒否の実操作ログは含まれない**。

## 71.8 制約

- Phase 2はローカルODBC/SQL検証として扱う。VPS上のDBにODBCで接続した証跡ではない。
- 時点再現なし。特にAllocationの作成日時がないため、過去月時点の全状態を完全再現しない。
- CSV 17件の金額3項目一致は、その17件に限る部分集合の検証結果であり、61件すべてや全13列の同値性を証明しない。
- 認証情報を保存したACCDBの公開は行わない。リンクテーブル内に保存済みのパスワードがないか公開前に確認する。

詳細・環境設定は [`04_odbc_readonly.md`](04_odbc_readonly.md) に記載する。

---
# 72. Phase 3：異常系・DB権限試験 詳細手順（2026年10月）

## 72.1 テスト環境の分離

リポジトリの `access/InvoiceAccessOperations.accdb` は変更しない。Accessを終了してから検証用コピーを作成し、そのコピーを起動する。

```powershell
Copy-Item ".\access\InvoiceAccessOperations.accdb" "$env:TEMP\InvoiceAccessOperations-Phase3-Test.accdb"
Invoke-Item "$env:TEMP\InvoiceAccessOperations-Phase3-Test.accdb"
```

SQL更新は `T_PaymentImport` / `T_InvoiceImport` のみで、`L_*` リンクテーブルには更新SQLを発行しない。再取込はテストデータを上書きするため実施しない。

## 72.2 PostgreSQL権限・拒否検証

psqlで `invoice_postgres` / `invoicesystem` に接続し、`access_readonly` の権限を確認する。

```sql
SELECT
    has_table_privilege('access_readonly','access_reporting.v_invoice_read','SELECT') AS view_select,
    has_table_privilege('access_readonly','access_reporting.v_invoice_read','UPDATE') AS view_update,
    has_table_privilege('access_readonly','public."Invoices"','SELECT') AS base_select,
    has_table_privilege('access_readonly','public."Invoices"','UPDATE') AS base_update,
    has_table_privilege('access_readonly','public."Invoices"','DELETE') AS base_delete;
```

結果：`t / f / f / f / f`。

```sql
BEGIN;
SET LOCAL ROLE access_readonly;
SELECT current_user;
UPDATE public."Invoices" SET "TotalAmount" = 0 WHERE FALSE;
ROLLBACK;
```

`current_user` = `access_readonly`、`ERROR: permission denied for table Invoices` を確認。

```sql
BEGIN;
SET LOCAL ROLE access_readonly;
DELETE FROM public."Invoices" WHERE FALSE;
ROLLBACK;
```

こちらも同じ権限エラーを確認。

```sql
BEGIN;
SET LOCAL ROLE access_readonly;
SELECT COUNT(*) AS invoice_count FROM access_reporting.v_invoice_read;
ROLLBACK;
```

SELECTは61件で成功。`WHERE FALSE` と `ROLLBACK` を併用しているため元DBのデータ変更は残さない。ロール切替の検証であり、ODBC接続元のログイン試験ではない。

## 72.3 Q_PaymentOverAllocationの異常系

判定SQLは `AllocatedAmount > PaymentAmount`。元データ（PaymentId=1）は `Amount=120000`、`AllocatedAmount=120000`。

```sql
UPDATE T_PaymentImport
SET [Amount] = 119000
WHERE [PaymentId] = 1 AND [Amount] = 120000;
```

1,000円超過の1件を検出した後、境界値を検証する。

```sql
UPDATE T_PaymentImport
SET [Amount] = 119999
WHERE [PaymentId] = 1 AND [Amount] = 119000;
```

1円超過の1件（`UnallocatedAmount=-1`）を確認。

```sql
UPDATE T_PaymentImport
SET [Amount] = 120000
WHERE [PaymentId] = 1 AND [Amount] = 119999;
```

## 72.4 Q_StatusMismatchの3パターン

`CalculatedStatus` は割当金額と請求金額の比較で `UNPAID` / `PARTIAL` / `PAID` / `OVER_ALLOCATED` を算出する。`StatusCode IN ('UNPAID','PARTIAL','PAID')` のレコードのみ登録値との差を検出する。

```sql
-- 全額入金済みなのに未入金
UPDATE T_InvoiceImport SET StatusCode='UNPAID'
WHERE InvoiceId=1 AND StatusCode='PAID';
-- 復元
UPDATE T_InvoiceImport SET StatusCode='PAID'
WHERE InvoiceId=1 AND StatusCode='UNPAID';
```

```sql
-- 未入金なのに入金済み
UPDATE T_InvoiceImport SET StatusCode='PAID'
WHERE InvoiceId=10 AND StatusCode='UNPAID';
-- 復元
UPDATE T_InvoiceImport SET StatusCode='UNPAID'
WHERE InvoiceId=10 AND StatusCode='PAID';
```

```sql
-- 一部入金なのに全額入金済み
UPDATE T_InvoiceImport SET StatusCode='PAID'
WHERE InvoiceId=12 AND StatusCode='PARTIAL';
-- 復元
UPDATE T_InvoiceImport SET StatusCode='PARTIAL'
WHERE InvoiceId=12 AND StatusCode='PAID';
```

3ケースすべて1件を検出。`StatusName` は元のままとし、**今回はStatusCodeと計算ステータスの比較に限定**する。StatusCode/StatusName間の整合性まで検証したわけではない。

## 72.5 Q_CancelledWithAllocation

```sql
UPDATE T_InvoiceImport SET StatusCode='CANCELLED'
WHERE InvoiceId=12 AND StatusCode='PARTIAL';
```

InvoiceId=12について、`StatusCode='CANCELLED'` / `AllocatedAmount=30000` の1件を検出。

```sql
UPDATE T_InvoiceImport SET StatusCode='PARTIAL'
WHERE InvoiceId=12 AND StatusCode='CANCELLED';
```

## 72.6 Q_PastDueMismatch

現行クエリの判定式は次のとおり。

```sql
IIf(Q.DueDate < #2026/10/01#
    AND Q.AllocatedAmount < Q.TotalAmount, True, False)
```

変更前から5件の不一致がある。InvoiceId=10（DueDate=2027/01/25、AllocatedAmount=0、IsOverdue=False）を対象に、検証用ACCDBでフラグのみ変更する。

```sql
UPDATE T_InvoiceImport SET IsOverdue=True
WHERE InvoiceId=10 AND IsOverdue=False;
```

6件への増加を確認。テスト後、次を実行して開発者により5件へ戻ったことが確認されている。

```sql
UPDATE T_InvoiceImport SET IsOverdue=False
WHERE InvoiceId=10 AND IsOverdue=True;
```

この試験は固定基準日の不一致判定の動作確認であり、2026年11月以降の請求を2026/10/01時点で「期限超過」と扱う業務妥当性の試験ではない。

## 72.7 完了判定と証跡

権限拒否・各検出が期待どおりで、Accessテストデータは復元済み（開発者報告）。正式ACCDBのQuery/VBAやDB元テーブルのデータは本テストでは変更しない。テスト用ACCDBの修正分は公開版へマージしない。

詳細な一覧と証跡の対応は [`05_phase3_validation.md`](05_phase3_validation.md) を参照する。

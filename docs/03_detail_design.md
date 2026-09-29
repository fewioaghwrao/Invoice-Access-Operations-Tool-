# Invoice Access Operations Tool - 詳細設計書

## 1. 文書概要

### 1.1 目的

本書は、Invoice Management System に追加する Access 月次照合用Export機能について、実装レベルの詳細設計を定義する。

本機能は以下の3種類のデータをCSVとして生成し、1つのZIPファイルとして返却する。

```text
invoices_YYYYMM.csv
payments_YYYYMM.csv
allocations_YYYYMM.csv
```

生成したデータは Microsoft Access 側で取り込み、請求・入金・入金割当の照合に使用する。

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

---

# 55. 実装完了確認

以下をすべて満たした場合、Invoice側Access Export機能の実装完了とする。

```text
dotnet build 成功

↓

GET
/api/admin/access-export
?year=2026
&month=9

↓

HTTP 200

↓

invoice-access-202609.zip

↓

invoices_202609.csv
payments_202609.csv
allocations_202609.csv

↓

各CSVがUTF-8 BOM付き

↓

Accessで取込可能

↓

Invoice / Payment / Allocationの
参照関係を確認可能
```

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

# 57. 最終クラス関係

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
        │
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

本構成により、既存Sales Exportとは独立したAccess月次照合用データ出力機能として実装する。
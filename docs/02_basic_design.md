# Invoice Access Operations Tool - 基本設計書

## 1. 文書概要

### 1.1 目的

本書は、Invoice Management System と Microsoft Access 月次照合ツールを連携するための基本設計を定義する。

対象範囲は以下とする。

- AccessからInvoice APIへの管理者ログイン
- JWTを利用したAccess Export API呼出
- Access Export API
- 対象データ抽出条件
- ZIPファイル構成
- invoices CSV
- payments CSV
- allocations CSV
- Access側ZIP展開・CSV取込
- 対象年月の保持と月次抽出
- ステータス照合ルール
- 月次サマリー
- 月次確認レポート
- エラー処理
- Invoice側とAccess側の責務境界

---

# 2. システム構成

```text
Microsoft Access
│
├─ F_Login
│    │ POST /auth/login
│    ▼
│  JWT / ApiBaseUrl
│    │ TempVars
│    ▼
├─ F_CsvImport
│    │ TargetYear / TargetMonth
│    │ GET /api/admin/access-export
│    ▼
│
├──────────── HTTPS ────────────┐
│                               ▼
│                     nginx / ConoHa VPS
│                               │
│                               ▼
│                    Invoice Management System
│                    ASP.NET Core API
│                               │
│                               ▼
│                          PostgreSQL
│
│                    AccessExportDataDto
│                               │
│                               ▼
│                    AccessExportCsvBuilder
│                               │
│                               ▼
│                    AccessExportZipBuilder
│                               │
└──────── invoice-access-YYYYMM.zip
                │
                ▼
        PowerShell Expand-Archive
                │
                ├─ invoices_YYYYMM.csv
                ├─ payments_YYYYMM.csv
                └─ allocations_YYYYMM.csv
                │
                ▼
        T_InvoiceImport
        T_PaymentImport
        T_AllocationImport
                │
                ├─ 照合Query
                ├─ 対象月Query
                ├─ Q_MonthlySummary
                └─ R_MonthlyCheck
```

Invoice Management System を正データの管理主体とする。

AccessからInvoice Management SystemのDBに対する登録・更新・削除は行わない。

通常はAPI自動連携を使用し、APIを利用できない場合は3CSVを手動指定して同じImport Tableへ取り込む。

---

# 3. 既存Sales Exportとの責務分離

既存のSales Exportは、請求単位で以下の情報を生成する。

```text
Invoice
 ↓
PaymentAllocation集計
 ↓
PaidAmount
RemainingAmount
LastPaidAt
 ↓
売上一覧CSV
```

Access Exportではこれを流用しない。

Access側でPaymentAllocationの元データから再集計・照合できるよう、以下の3種類のデータを分離して出力する。

```text
Invoice
Payment
PaymentAllocation
```

役割を以下のように分離する。

| 機能 | 目的 |
|---|---|
| Sales Export | 人が売上・入金状況を一覧確認する |
| Access Export | Accessが元データを再計算して照合する |

Access Exportでは `PaidAmount`、`RemainingAmount` 等の派生値をCSVには持たせない。

---

# 4. API設計

## 4.1 Endpoint

```http
GET /api/admin/access-export?year={year}&month={month}
```

例：

```http
GET /api/admin/access-export?year=2026&month=9
```

## 4.2 認可

管理者のみ実行可能とする。

Invoice Management Systemの既存Admin認可方式を利用する。

## 4.3 Request

| パラメータ | 必須 | 型 | 内容 |
|---|:---:|---|---|
| year | ○ | int | 対象年 |
| month | ○ | int | 対象月（1～12） |

## 4.4 Response

正常時：

```text
HTTP 200
Content-Type: application/zip
```

ダウンロードファイル名：

```text
invoice-access-YYYYMM.zip
```

例：

```text
invoice-access-202609.zip
```

## 4.5 HTTPステータス

| Status | 内容 |
|---:|---|
| 200 | ZIP生成成功 |
| 400 | year / month 不正 |
| 401 | 未認証 |
| 403 | 権限不足 |
| 500 | CSV生成・ZIP生成等の予期しないエラー |

---

# 5. 対象年月・基準日時

## 5.1 対象期間

対象年月から以下を算出する。

例：

```text
year  = 2026
month = 9
```

の場合：

```text
TargetMonthStart = 2026-09-01
NextMonthStart   = 2026-10-01
```

抽出条件では原則として、

```text
日時 < NextMonthStart
```

を使用する。

月末日の23:59:59等を生成しないことで、時刻精度に依存した境界条件を避ける。

---

# 6. 出力対象データ

## 6.1 基本方針

Access Exportは「対象月に新規作成されたデータだけ」を出力する機能ではない。

対象月末までの照合に必要な累積データを出力する。

例えば、

```text
8月請求
 ↓
9月入金
 ↓
9月末照合
```

の場合、8月請求も9月のAccess Exportに含める。

これによりAccess側で請求・入金・割当の関係を再計算できるようにする。

---

# 7. Invoice抽出条件

以下を満たすInvoiceを対象とする。

```text
InvoiceDate < NextMonthStart
```

2026年9月の場合：

```text
InvoiceDate < 2026-10-01
```

したがって9月以前に発生した請求を対象とする。

## 7.1 CANCELLED

`CANCELLED` のInvoiceもCSVには出力する。

CSV生成時点で除外しない。

理由は、Access側で「存在しているがキャンセルされた請求」であることを確認可能にするためである。

通常の請求金額・未入金集計ではAccess側が除外する。

## 7.2 Member.IsActive

Memberの `IsActive` はInvoice Exportの除外条件に使用しない。

退会・無効化されたMemberであっても、過去のInvoiceやPaymentの照合に必要なためである。

---

# 8. Payment抽出条件

以下を満たすPaymentを対象とする。

```text
PaymentDate < NextMonthStart
```

2026年9月の場合：

```text
PaymentDate < 2026-10-01
```

9月以前に発生した入金を対象とする。

Memberの現在の有効・無効状態によってPaymentを除外しない。

---

# 9. PaymentAllocation抽出条件

以下の両方を満たすPaymentAllocationを対象とする。

```text
InvoiceId が今回出力対象のInvoiceに存在する

AND

PaymentId が今回出力対象のPaymentに存在する
```

概念上は以下となる。

```text
Export対象Invoice
       ↑
       │ InvoiceId
PaymentAllocation
       │ PaymentId
       ↓
Export対象Payment
```

これにより3CSV内で参照関係が成立するデータセットを生成する。

---

# 10. PaymentAllocationの時点管理に関する制約

現在のPaymentAllocationは以下の情報を保持する。

```text
Id
PaymentId
InvoiceId
Amount
```

割当作成日時を保持していない。

そのため、例えば以下の場合、

```text
PaymentDate
2026-08-20

PaymentAllocation登録
2026-10-05
```

11月に2026年9月Exportを再実行した際、

```text
2026-09-30時点で
PaymentAllocationが存在していたか
```

を判定できない。

したがってPhase 1のAccess Exportは、

> 過去時点を完全再現する会計スナップショット

ではなく、

> Export実行時点の現在データを、指定年月を基準として照合する業務支援機能

と位置付ける。

厳密な過去時点再現はPhase 1対象外とする。

---

# 11. ZIP構成

ZIP内部は以下の3ファイルで固定する。

```text
invoice-access-202609.zip
│
├─ invoices_202609.csv
├─ payments_202609.csv
└─ allocations_202609.csv
```

サブディレクトリは作成しない。

Access側ではZIP展開後の3ファイルを1つの取込セットとして扱う。

---

# 12. CSV共通仕様

| 項目 | 仕様 |
|---|---|
| 文字コード | UTF-8 BOM付き |
| 区切り文字 | `,` |
| 改行 | CRLF |
| ヘッダー | 必須 |
| NULL | 空文字 |
| 日付 | `yyyy-MM-dd` |
| decimal | 桁区切りなし |
| 金額小数 | 最大2桁 |
| bool | `true` / `false` |
| 文字列 | 必要時 `"` で囲む |
| 並び順 | 主キー昇順 |

文字列内に以下が含まれる場合はCSVエスケープを行う。

```text
,
"
CR
LF
```

文字列中の `"` は `""` に変換する。

---

# 13. invoices CSV

## 13.1 ファイル名

```text
invoices_YYYYMM.csv
```

## 13.2 項目

| No | CSV列 | 元データ | 型 | 必須 |
|---:|---|---|---|:---:|
| 1 | InvoiceId | Invoice.Id | long | ○ |
| 2 | MemberId | Invoice.MemberId | long | ○ |
| 3 | InvoiceNumber | Invoice.InvoiceNumber | string | ○ |
| 4 | InvoiceDate | Invoice.InvoiceDate | date | ○ |
| 5 | DueDate | Invoice.DueDate | date | ○ |
| 6 | TotalAmount | Invoice.TotalAmount | decimal | ○ |
| 7 | StatusCode | Invoice.Status.Code | string | ○ |
| 8 | StatusName | Invoice.Status.Name | string | ○ |
| 9 | IsOverdue | Invoice.Status.IsOverdue | bool | ○ |
| 10 | IsClosed | Invoice.Status.IsClosed | bool | ○ |
| 11 | MemberName | Invoice.Member.Name | string | ○ |

## 13.3 出力例

```csv
InvoiceId,MemberId,InvoiceNumber,InvoiceDate,DueDate,TotalAmount,StatusCode,StatusName,IsOverdue,IsClosed,MemberName
1,1001,INV-202609-001,2026-09-01,2026-09-30,50000.00,UNPAID,未入金,false,false,株式会社AAA
```

---

# 14. payments CSV

## 14.1 ファイル名

```text
payments_YYYYMM.csv
```

## 14.2 項目

| No | CSV列 | 元データ | 型 | 必須 |
|---:|---|---|---|:---:|
| 1 | PaymentId | Payment.Id | long | ○ |
| 2 | MemberId | Payment.MemberId | long | ○ |
| 3 | PaymentDate | Payment.PaymentDate | date | ○ |
| 4 | Amount | Payment.Amount | decimal | ○ |
| 5 | PayerName | Payment.PayerName | string |  |
| 6 | Method | Payment.Method | string |  |

## 14.3 出力例

```csv
PaymentId,MemberId,PaymentDate,Amount,PayerName,Method
100,1001,2026-09-15,30000.00,カブシキガイシャAAA,振込
```

---

# 15. allocations CSV

## 15.1 ファイル名

```text
allocations_YYYYMM.csv
```

## 15.2 項目

| No | CSV列 | 元データ | 型 | 必須 |
|---:|---|---|---|:---:|
| 1 | AllocationId | PaymentAllocation.Id | long | ○ |
| 2 | PaymentId | PaymentAllocation.PaymentId | long | ○ |
| 3 | InvoiceId | PaymentAllocation.InvoiceId | long | ○ |
| 4 | Amount | PaymentAllocation.Amount | decimal | ○ |

## 15.3 出力例

```csv
AllocationId,PaymentId,InvoiceId,Amount
5001,100,1,30000.00
```

---

# 16. Access側の請求入金状態計算

AccessではInvoiceId単位でAllocation.Amountを集計する。

```text
AllocatedAmount =
    SUM(PaymentAllocation.Amount)
    GROUP BY InvoiceId
```

以下の状態を算出する。

| 条件 | Access計算状態 |
|---|---|
| AllocatedAmount = 0 | UNPAID |
| 0 < AllocatedAmount < TotalAmount | PARTIAL |
| AllocatedAmount = TotalAmount | PAID |
| AllocatedAmount > TotalAmount | OVER_ALLOCATED |

`OVER_ALLOCATED` はInvoice本体のInvoiceStatusではなく、Access照合ツール独自の異常判定とする。

---

# 17. 期限超過判定

入金状態と期限超過状態は別軸として計算する。

Access側では以下を算出する。

```text
IsPastDueCalculated =
    DueDate < 対象月末
    AND
    Access計算状態 <> PAID
```

ただし `CANCELLED` は対象外とする。

これにより、

```text
入金状態      = PARTIAL
期限超過      = true
```

のような状態も表現できる。

---

# 18. Invoice本体ステータスとの照合

Invoice CSVにはInvoice本体が保持する以下を出力する。

```text
StatusCode
StatusName
IsOverdue
IsClosed
```

Accessではこれらと、PaymentAllocationから再計算した状態を比較する。

---

# 19. ステータス照合ルール

## 19.1 PAID

Invoice本体：

```text
StatusCode = PAID
```

Access計算：

```text
AllocatedAmount = TotalAmount
```

であれば正常とする。

以下の場合は要確認とする。

```text
StatusCode = PAID
かつ
AllocatedAmount < TotalAmount
```

`AllocatedAmount > TotalAmount` の場合はステータス不整合とは別に「過剰割当」として扱う。

---

## 19.2 PARTIAL

Invoice本体：

```text
StatusCode = PARTIAL
```

Access計算：

```text
0 < AllocatedAmount < TotalAmount
```

であれば正常とする。

それ以外は要確認とする。

---

## 19.3 UNPAID

Invoice本体：

```text
StatusCode = UNPAID
```

Access計算：

```text
AllocatedAmount = 0
```

であれば入金状態としては整合する。

ただし対象月末時点で、

```text
DueDate < 対象月末
```

の場合は、

```text
期限超過状態との整合確認
```

を別途行う。

---

## 19.4 OVERDUE

Invoice本体：

```text
StatusCode = OVERDUE
```

の場合、少なくとも以下を確認する。

```text
DueDate < 対象月末
AND
Access計算状態 <> PAID
```

これを満たさない場合は要確認とする。

`OVERDUE` は単純な入金状態ではなく期限状態を含むため、Allocation額だけで `UNPAID` と同一判定しない。

---

## 19.5 CANCELLED

Invoice本体：

```text
StatusCode = CANCELLED
```

の場合、

```text
通常の未入金
一部入金
PAID
OVERDUE
```

とのステータス比較対象外とする。

ただしPaymentAllocationが存在する場合、

```text
キャンセル済み請求に割当あり
```

として確認対象にできるものとする。

---

# 20. ステータス照合結果

Access側では少なくとも以下の結果を持つ。

```text
OK

STATUS_MISMATCH

OVER_ALLOCATED

CANCELLED_WITH_ALLOCATION

PAST_DUE_MISMATCH
```

画面表示名称例：

| コード | 表示 |
|---|---|
| OK | 正常 |
| STATUS_MISMATCH | ステータス不整合 |
| OVER_ALLOCATED | 過剰割当 |
| CANCELLED_WITH_ALLOCATION | キャンセル請求に割当あり |
| PAST_DUE_MISMATCH | 期限超過状態不整合 |

---

# 21. Payment側照合

PaymentId単位でAllocation.Amountを合計する。

```text
AllocatedAmount =
    SUM(PaymentAllocation.Amount)
    GROUP BY PaymentId
```

判定は以下とする。

| 条件 | 判定 |
|---|---|
| AllocatedAmount = 0 | 未割当 |
| 0 < AllocatedAmount < Payment.Amount | 一部未割当 |
| AllocatedAmount = Payment.Amount | 割当完了 |
| AllocatedAmount > Payment.Amount | 割当額超過 |

以下を算出する。

```text
UnallocatedAmount =
    Payment.Amount - AllocatedAmount
```

入金額より割当額が少ない状態は必ずしもデータ異常とは限らないため、「未割当」として確認対象とする。

入金額より割当額が多い場合は不整合とする。

---

# 22. Access側参照整合性チェック

以下をチェックする。

```text
Allocation.InvoiceId
    ↓
Invoice.InvoiceId
```

存在しない場合：

```text
ORPHAN_INVOICE
```

また、

```text
Allocation.PaymentId
    ↓
Payment.PaymentId
```

が存在しない場合：

```text
ORPHAN_PAYMENT
```

とする。

通常はInvoice側のExport処理で発生しない想定だが、CSV破損・手修正等を検知する目的でAccess側でもチェックする。

---

# 23. Invoice側クラス構成案

既存SalesServiceとは分離する。

```text
Application
│
├─ Dtos
│  └─ AccessExport
│     ├─ AccessInvoiceExportRow.cs
│     ├─ AccessPaymentExportRow.cs
│     └─ AccessAllocationExportRow.cs
│
└─ Services
   └─ AccessExport
      └─ IAccessExportService.cs

Infrastructure
└─ Services
   └─ AccessExport
      └─ AccessExportService.cs
```

Endpointは既存プロジェクトのEndpoint構成に合わせて追加する。

概念上は以下とする。

```text
AccessExportEndpoint
        ↓
IAccessExportService
        ↓
AccessExportService
        ↓
AppDbContext
```

---

# 24. Access Export関連クラスの責務

Access Exportは責務を分離して実装する。

| クラス | 責務 |
|---|---|
| `AccessExportEndpoints` | HTTP受付、year/month validation、Service呼出、ZIP Response生成 |
| `IAccessExportService` | Application層のExport Service契約 |
| `AccessExportService` | 対象期間算出、Invoice / Payment / Allocation取得、DTO生成 |
| `AccessExportCsvBuilder` | DTOから3CSV文字列を生成 |
| `AccessExportZipBuilder` | 3CSVをUTF-8 BOM付きでZIPへ格納 |
| `AccessExportDataDto` | Export対象データ一式を保持 |

`AccessExportService`ではCSV生成、ZIP生成、Access側の照合判定は行わない。

```text
AccessExportEndpoints
        ↓
IAccessExportService
        ↓
AccessExportService
        ↓
AccessExportDataDto
        ↓
AccessExportCsvBuilder
        ↓
AccessExportZipBuilder
        ↓
application/zip
```

---

# 25. CSV生成責務

CSV生成処理では以下のみ行う。

```text
Entity / DTO
    ↓
文字列変換
    ↓
CSV Escape
    ↓
UTF-8 BOM
    ↓
CSV byte[]
```

以下は行わない。

```text
PaidAmount計算
RemainingAmount計算
ステータス再計算
月次集計
不整合判定
```

これらはAccess側の責務とする。

---

# 26. ZIP生成処理

処理イメージ：

```text
AccessExportService
        │
        ├─ invoicesCsv
        ├─ paymentsCsv
        └─ allocationsCsv
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

サーバー上への恒久的なZIP保存は行わない。

原則としてメモリ上で生成しHTTP Responseとして返却する。

---

# 27. データ整合性

3CSVは1回のAccess Export要求によって生成する。

```text
1 Request
   ↓
Invoice取得
Payment取得
Allocation取得
   ↓
1 ZIP
```

利用者が個別に3CSVを別タイミングでダウンロードする方式は採用しない。

これにより、異なるタイミングに生成されたCSVが混在する可能性を低減する。

なお、厳密に同一DBスナップショットを保証する必要が生じた場合は、トランザクション分離レベルを含む整合性方式を別途検討する。

Phase 1では単一Export処理内で3データを取得することを基本とする。

---

# 28. ログ

Phase 1では既存のSerilog HTTP Request Loggingを利用する。

Access Export専用の業務ログを追加する場合は、以下を候補とする。

```text
対象年月
実行日時
Invoice出力件数
Payment出力件数
Allocation出力件数
処理結果
```

CSV内容そのもの、MemberName、PayerName、JWT、パスワード等はログへ出力しない。

現行実装では、専用業務ログの追加をPhase 1完成条件には含めない。

---

# 29. セキュリティ

Access Export APIはAdmin専用とする。

Access側は `POST /auth/login` で取得したJWTをBearer Tokenとして送信する。

```http
Authorization: Bearer {token}
```

Access側の認証情報保持方針：

| 項目 | 保持方法 |
|---|---|
| JWT | `TempVars("AccessToken")` |
| API Base URL | `TempVars("ApiBaseUrl")` |
| 対象年 | `TempVars("TargetYear")` |
| 対象月 | `TempVars("TargetMonth")` |
| パスワード | 保持しない |

ログイン成功後はパスワード入力欄をクリアする。

CSVにはMemberNameやPayerName等の業務情報を含むため、一般ユーザーからAccess Exportを実行できないこと。

以下はCSVへ出力しない。

```text
Member.Email
Member.PasswordHash
Member.Address
Member.Phone
EmailVerificationToken
```

Access照合に不要な個人情報・認証情報は連携対象外とする。

---

# 30. エラー処理

以下の場合、ZIPを返却しない。

```text
year不正
month不正
DBアクセス失敗
CSV生成失敗
ZIP生成失敗
```

処理途中で生成されたデータを部分的にレスポンスしない。

3CSVすべて生成できた場合のみ正常終了とする。

---

# 31. 空データの場合

対象となるデータが0件の場合でも、CSV自体は生成する。

例：

```csv
PaymentId,MemberId,PaymentDate,Amount,PayerName,Method
```

ヘッダーのみのCSVとする。

これによりAccess側では、

```text
ファイル不存在
```

と

```text
対象データ0件
```

を区別できる。

---

# 32. Access取込単位

Access側ではZIP展開後の3CSVを1セットとして取り込む。

API自動連携：

```text
F_Login
        ↓
Admin JWT取得
        ↓
F_CsvImport
        ↓
対象年月指定
        ↓
Access Export API
        ↓
invoice-access-YYYYMM.zip
        ↓
PowerShell Expand-Archive
        ↓
3CSV存在確認
        ↓
既存Import Tableクリア
        ↓
UTF-8 CSV取込
        ↓
件数確認
        ↓
照合Query / 月次Query
```

3CSVのうち1ファイルでも存在しない場合は取込完了扱いにしない。

Import Tableの削除順は参照関係を考慮し、

```text
T_AllocationImport
    ↓
T_PaymentImport
    ↓
T_InvoiceImport
```

とする。

CSV取込時は `CodePage:=65001` を指定する。

手動取込も同じImport Tableと照合Queryを利用する。

---

# 33. Phase 1 処理シーケンス

```text
利用者
 │
 │ API URL / Admin credentials
 ▼
F_Login
 │
 │ POST /auth/login
 ▼
Invoice API
 │
 │ JWT
 ▼
F_Login
 │
 │ AccessToken / ApiBaseUrl → TempVars
 ▼
F_MainMenu
 │
 ▼
F_CsvImport
 │
 │ TargetYear / TargetMonth
 │ TargetYear / TargetMonth → TempVars
 ▼
GET /api/admin/access-export
 │
 ▼
AccessExportEndpoints
 │
 │ year/month validation
 ▼
AccessExportService
 │
 ├─ Invoice取得
 ├─ Payment取得
 └─ Allocation取得
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
invoice-access-YYYYMM.zip
 │
 ▼
Access VBA
 │
 ├─ ZIP保存
 ├─ ZIP展開
 ├─ 3CSV存在確認
 ├─ Import Tableクリア
 └─ TransferText(CodePage=65001)
 │
 ▼
Q_TargetMonthInvoices / Q_TargetMonthPayments
 │
 ▼
Q_MonthlySummary
 │
 ▼
R_MonthlyCheck
```

---

# 34. Phase 1 完成条件

Invoice側について以下を満たすこと。

```text
Adminユーザー
    ↓
GET /api/admin/access-export
    ↓
対象Invoice / Payment / Allocation取得
    ↓
3CSV生成
    ↓
ZIP生成
    ↓
HTTP 200 application/zip
```

ZIPには必ず以下を含む。

```text
invoices_YYYYMM.csv
payments_YYYYMM.csv
allocations_YYYYMM.csv
```

Access側について以下を満たすこと。

```text
F_LoginでAdminログイン
    ↓
JWT取得
    ↓
F_CsvImportで対象年月指定
    ↓
ZIP取得・展開・3CSV自動取込
    ↓
未入金 / 一部入金 / 入金済
過剰割当
未割当入金 / 入金割当超過
ステータス不整合
キャンセル請求への割当
期限超過不整合
参照不整合
    ↓
F_CheckResult
    ↓
対象月基本集計
    ↓
Q_MonthlySummary
    ↓
R_MonthlyCheck
```

本番相当のVPS APIへHTTPS接続し、ログインから月次レポートまでE2Eで動作確認できること。

---

# 35. Phase 1 対象外

以下は本基本設計の対象外とする。

```text
AccessからInvoice APIへの更新
AccessからInvoice DBへの直接接続
月次締めロック
会計仕訳
過去任意日時の完全な状態復元
PaymentAllocation履歴管理
Excel自動生成
旧システム移行CSV変換
```

これらが必要となった場合は後続Phaseとして設計する。

---

# 36. 設計上の最終方針

Phase 1では以下を基本方針とする。

```text
Invoice Management System
    = 正データ管理

Access Export
    = 生データ連携

Microsoft Access
    = 再集計・照合・確認
```

Invoice本体で計算済み結果だけをAccessへ渡すのではなく、

```text
Invoice
Payment
PaymentAllocation
```

をそれぞれ出力し、Access側で再計算する。

これによりAccessを単なるCSVビューアではなく、

> Invoice Management Systemのデータ整合性を確認する周辺業務・月次照合ツール

として位置付ける。

---

# 37. Accessログイン設計

## 37.1 F_Login

入力項目：

```text
txtApiBaseUrl
txtEmail
txtPassword
```

実行ボタン：

```text
btnLogin
```

ログイン時に以下を実行する。

```text
入力チェック
    ↓
POST {ApiBaseUrl}/auth/login
    ↓
HTTP 200確認
    ↓
JWT抽出
    ↓
TempVarsへ保存
    ↓
パスワード欄クリア
    ↓
F_MainMenu表示
```

HTTP 200以外の場合は、HTTP StatusとResponse本文を表示して終了する。

## 37.2 TempVars

```text
AccessToken
ApiBaseUrl
TargetYear
TargetMonth
```

`AccessToken` と `ApiBaseUrl` はログイン成功時に設定する。

`TargetYear` と `TargetMonth` はAPI取得時の対象年月として設定する。

---

# 38. F_CsvImport設計

F_CsvImportは以下の2方式を提供する。

```text
1. API自動取得・取込
2. 3CSV手動取込
```

API自動取得では以下を実行する。

```text
TargetYear / TargetMonth入力
    ↓
TempVars更新
    ↓
GET /api/admin/access-export
    ↓
ZIP保存
    ↓
ZIP展開
    ↓
3CSVパス自動設定
    ↓
ImportCsvFiles
```

ZIP保存先はWindowsの一時領域配下とする。

```text
%TEMP%\InvoiceAccessOperations
```

ZIP展開にはPowerShell `Expand-Archive` を利用する。

---

# 39. 対象月抽出設計

Access Exportの3CSVは対象月末照合に必要な累積データを含む。

そのため、月次の基本集計だけは対象年月で明示的に抽出する。

## Q_TargetMonthInvoices

```text
InvoiceDate >= 対象月1日
AND
InvoiceDate < 翌月1日
```

## Q_TargetMonthPayments

```text
PaymentDate >= 対象月1日
AND
PaymentDate < 翌月1日
```

対象年月は `TempVars("TargetYear")` / `TempVars("TargetMonth")` を参照する。

---

# 40. Q_MonthlySummary設計

`Q_MonthlySummary` は1レコードを返す。

基本集計：

| 項目 | 参照元 |
|---|---|
| InvoiceCount | Q_TargetMonthInvoices |
| InvoiceAmount | Q_TargetMonthInvoices（CANCELLED除外） |
| PaymentCount | Q_TargetMonthPayments |
| PaymentAmount | Q_TargetMonthPayments |

照合件数：

```text
Q_UnpaidInvoices
Q_PartiallyPaidInvoices
Q_OverAllocatedInvoices
Q_UnallocatedPayments
Q_PaymentOverAllocation
Q_StatusMismatch
Q_CancelledWithAllocation
Q_PastDueMismatch
Q_OrphanAllocations
```

基本集計は「対象月に発生したデータ」、照合件数は「取込スナップショット全体の確認対象」という役割分担とする。

---

# 41. R_MonthlyCheck設計

`R_MonthlyCheck` のレコードソースは `Q_MonthlySummary` とする。

レポート上部に対象年月を表示する。

```text
対象年月：YYYY年MM月
```

表示ブロック：

```text
基本集計
請求チェック
入金チェック
その他の整合性チェック
```

1ページで月次確認結果を把握できるレイアウトとする。

---

# 42. F_MainMenu設計

F_MainMenuは以下への入口を提供する。

```text
CSV取込
チェック結果
月次サマリー
月次確認レポート
終了
```

Access起動時の表示フォームとして使用する。

---

# 43. 動作確認構成

本番相当確認では以下の経路を使用する。

```text
Microsoft Access
    ↓ HTTPS
nginx
    ↓
ASP.NET Core API
    ↓
PostgreSQL
```

確認項目：

```text
Adminログイン
JWT取得
Access Export API
ZIP取得
ZIP展開
3CSV取込
対象月抽出
照合Query
Q_MonthlySummary
R_MonthlyCheck
```

---

# 44. 最終責務分担

```text
Invoice Management System
    = 正データ管理・Export用元データ提供

Access Export API
    = 対象年月基準の元データ抽出・ZIP/CSV提供

Microsoft Access
    = 取込・再集計・不整合確認・月次レポート
```

Access側で派生値を再計算することにより、既存Sales Exportと責務を分離し、月次照合用EUCとして独立させる。

# Invoice Access Operations Tool - 要件定義書

## 1. 文書概要

### 1.1 目的

本書は、Invoice Management System から出力した請求・入金・入金割当データを Microsoft Access に取り込み、月次の請求・入金状況を照合する業務支援ツールの要件を定義する。

本ツールは Invoice Management System 本体の請求・入金機能を代替するものではない。

Invoice Management System を正データの管理主体とし、本ツールは以下を目的とする。

- CSVデータの取込
- 請求額と入金割当額の照合
- 未入金・一部入金の抽出
- 過剰割当の検出
- 入金額と割当額の照合
- 月次集計
- 確認結果の一覧表示
- 月次確認レポートの出力

---

# 2. システム構成

```text
Invoice Management System
        │
        │ CSV Export
        ▼
┌───────────────────────┐
│ invoices_YYYYMM.csv   │
│ payments_YYYYMM.csv   │
│ allocations_YYYYMM.csv│
└───────────┬───────────┘
            │
            ▼
 Microsoft Access
            │
            ├─ CSV取込
            ├─ 請求照合
            ├─ 入金照合
            ├─ 不整合チェック
            ├─ 月次集計
            └─ 確認レポート
```

AccessからInvoice Management SystemのDBを直接更新しない。

CSVを介した一方向連携とする。

---

# 3. 対象範囲

## 3.1 Phase 1 対象

Phase 1では以下を実装対象とする。

1. 請求CSV取込
2. 入金CSV取込
3. 入金割当CSV取込
4. CSV基本形式チェック
5. 請求額と割当額の照合
6. 入金額と割当額の照合
7. 未入金抽出
8. 一部入金抽出
9. 過剰割当抽出
10. 未割当入金抽出
11. 入金額超過割当の検出
12. 月次集計
13. 確認結果一覧表示
14. 月次確認レポート表示

## 3.2 Phase 1 対象外

以下はPhase 1では実装しない。

- AccessからInvoice本体へのデータ登録
- AccessからInvoice本体へのデータ更新
- 請求書作成
- 入金登録
- 入金割当登録
- 請求ステータス更新
- 月次締め処理
- Invoice DBへの直接接続
- Excel帳票の高度な自動生成
- 旧システムからの移行データ変換
- 過去時点のデータ状態復元
- 複数ユーザーによる同時更新

---

# 4. データ連携方式

## 4.1 基本方針

Invoice Management Systemから以下の3ファイルを出力する。

```text
invoices_YYYYMM.csv
payments_YYYYMM.csv
allocations_YYYYMM.csv
```

例：

```text
invoices_202609.csv
payments_202609.csv
allocations_202609.csv
```

3ファイルは同一の基準日時で作成されたデータセットとして扱う。

Access側では3ファイルを同一取込単位として処理する。

---

# 5. 月次データの考え方

## 5.1 スナップショット方式

Phase 1では単純な「対象月に発生したデータのみ」の出力とはしない。

月末時点の照合に必要な関連データを含んだスナップショットとしてCSVを出力する。

理由は、対象月以前に登録された請求・入金が対象月にも未決済または割当対象として残る可能性があるためである。

例えば以下のデータが存在する。

```text
8月請求
    ↓
9月入金
    ↓
9月にPaymentAllocation
```

この場合、9月分だけの請求CSVを出力すると8月請求が存在しないため、正しい照合ができない。

そのためPhase 1では、対象月末時点で照合に必要な請求・入金・割当データを一式出力する。

---

## 5.2 基準日

対象年月に対し、月末日を基準日とする。

例：

```text
対象年月：2026年9月

基準日：
2026-09-30
```

Access側の月次集計は、この対象年月を基準として実施する。

---

## 5.3 制約事項

現在のPaymentAllocationは、以下を主なデータとして保持する。

```text
Id
PaymentId
InvoiceId
Amount
```

Phase 1ではPaymentAllocation自体の割当日時を利用しない。

そのため、過去の任意日時におけるPaymentAllocationの状態を完全に再現する機能は対象外とする。

本ツールは、CSV出力時点のデータ状態を照合する業務支援ツールとする。

---

# 6. CSV共通仕様

| 項目 | 仕様 |
|---|---|
| 文字コード | UTF-8 BOM付き |
| 区切り文字 | カンマ `,` |
| 改行 | CRLF |
| ヘッダー | 1行目必須 |
| 文字列囲み | 必要に応じて `"` |
| NULL | 空文字 |
| 日付 | `yyyy-MM-dd` |
| 金額 | 小数点形式、桁区切りなし |
| ID | 整数 |
| 金額精度 | 最大 decimal(18,2) 相当 |
| 並び順 | 原則ID昇順 |

例：

```csv
InvoiceId,MemberId,InvoiceNumber,InvoiceDate,DueDate,TotalAmount,StatusCode,StatusName,MemberName
1,1001,INV-202609-001,2026-09-01,2026-09-30,50000.00,UNPAID,未入金,株式会社AAA
```

---

# 7. invoices CSV仕様

## 7.1 ファイル名

```text
invoices_YYYYMM.csv
```

## 7.2 項目定義

| No | 項目名 | 必須 | データ型 | 内容 |
|---:|---|:---:|---|---|
| 1 | InvoiceId | ○ | 64bit整数 | 請求ID |
| 2 | MemberId | ○ | 64bit整数 | 会員ID |
| 3 | InvoiceNumber | ○ | 文字列 | 請求番号 |
| 4 | InvoiceDate | ○ | 日付 | 請求日 |
| 5 | DueDate | ○ | 日付 | 支払期限 |
| 6 | TotalAmount | ○ | decimal(18,2) | 請求金額 |
| 7 | StatusCode | ○ | 文字列 | Invoice側の現在ステータスコード |
| 8 | StatusName | ○ | 文字列 | ステータス表示名称 |
| 9 | MemberName | ○ | 文字列 | 確認画面表示用の会員名 |

## 7.3 使用目的

`InvoiceId` はPaymentAllocationとのJOINキーとして使用する。

`MemberId` は会員単位の確認および将来の整合性チェックに使用する。

`InvoiceNumber` と `MemberName` は利用者が確認画面および帳票上で対象を識別するために使用する。

`TotalAmount` はPaymentAllocationの割当額合計と比較する基準金額とする。

---

# 8. payments CSV仕様

## 8.1 ファイル名

```text
payments_YYYYMM.csv
```

## 8.2 項目定義

| No | 項目名 | 必須 | データ型 | 内容 |
|---:|---|:---:|---|---|
| 1 | PaymentId | ○ | 64bit整数 | 入金ID |
| 2 | MemberId | ○ | 64bit整数 | 会員ID |
| 3 | PaymentDate | ○ | 日付 | 入金日 |
| 4 | Amount | ○ | decimal(18,2) | 入金額 |
| 5 | PayerName |  | 文字列 | 振込人・入金者名称 |
| 6 | Method |  | 文字列 | 入金方法 |

## 8.3 使用目的

`PaymentId` はPaymentAllocationとのJOINキーとして使用する。

`Amount` はPaymentAllocationの割当額合計と比較する。

`MemberId` は会員単位の照合補助情報として使用する。

`PayerName` および `Method` は不整合発生時の確認補助情報として利用する。

---

# 9. allocations CSV仕様

## 9.1 ファイル名

```text
allocations_YYYYMM.csv
```

## 9.2 項目定義

| No | 項目名 | 必須 | データ型 | 内容 |
|---:|---|:---:|---|---|
| 1 | AllocationId | ○ | 64bit整数 | 入金割当ID |
| 2 | PaymentId | ○ | 64bit整数 | 入金ID |
| 3 | InvoiceId | ○ | 64bit整数 | 請求ID |
| 4 | Amount | ○ | decimal(18,2) | 割当額 |

## 9.3 使用目的

PaymentAllocationを請求と入金を結び付ける正データとして扱う。

```text
Payment
    1
    │
    N
PaymentAllocation
    N
    │
    1
Invoice
```

請求額に対する入金済額は、InvoiceId単位のPaymentAllocation.Amount合計から算出する。

入金額に対する割当済額は、PaymentId単位のPaymentAllocation.Amount合計から算出する。

---

# 10. CSVに含めない派生項目

Phase 1のAccess向けCSVでは、以下のような算出可能な値は原則として必須項目にしない。

```text
PaidAmount
RemainingAmount
AllocatedAmount
UnallocatedAmount
```

これらはAccess側でPaymentAllocationから再計算する。

例えば、

```text
PaidAmount
    = SUM(PaymentAllocation.Amount)
      GROUP BY InvoiceId

RemainingAmount
    = Invoice.TotalAmount - PaidAmount

AllocatedAmount
    = SUM(PaymentAllocation.Amount)
      GROUP BY PaymentId

UnallocatedAmount
    = Payment.Amount - AllocatedAmount
```

とする。

Invoice本体が算出した値をそのままAccessへ渡すのではなく、Access側で元データから再計算することで、照合ツールとしての独立性を確保する。

---

# 11. Access取込テーブル

CSVは以下のAccessテーブルへ取り込む。

```text
T_InvoiceImport
T_PaymentImport
T_AllocationImport
```

対応関係は以下とする。

```text
invoices_YYYYMM.csv
    ↓
T_InvoiceImport

payments_YYYYMM.csv
    ↓
T_PaymentImport

allocations_YYYYMM.csv
    ↓
T_AllocationImport
```

CSVを再取込する場合、Phase 1では対象取込テーブルをクリアした後に再取込する方式とする。

CSV取込履歴の世代管理はPhase 1対象外とする。

---

# 12. 請求照合要件

InvoiceId単位でPaymentAllocation.Amountを集計する。

```text
AllocatedAmount =
    SUM(T_AllocationImport.Amount)
    GROUP BY InvoiceId
```

Invoice.TotalAmountと比較し、以下の状態を判定する。

| 条件 | 判定 |
|---|---|
| AllocatedAmount = 0 | 未入金 |
| 0 < AllocatedAmount < TotalAmount | 一部入金 |
| AllocatedAmount = TotalAmount | 入金済 |
| AllocatedAmount > TotalAmount | 過剰割当 |

差額は以下で算出する。

```text
RemainingAmount =
    TotalAmount - AllocatedAmount
```

---

# 13. 入金照合要件

PaymentId単位でPaymentAllocation.Amountを集計する。

```text
AllocatedAmount =
    SUM(T_AllocationImport.Amount)
    GROUP BY PaymentId
```

Payment.Amountと比較する。

| 条件 | 判定 |
|---|---|
| AllocatedAmount = 0 | 未割当 |
| 0 < AllocatedAmount < Payment.Amount | 一部未割当 |
| AllocatedAmount = Payment.Amount | 割当完了 |
| AllocatedAmount > Payment.Amount | 割当額超過 |

未割当金額は以下で算出する。

```text
UnallocatedAmount =
    Payment.Amount - AllocatedAmount
```

なお、入金額より割当額が少ない状態は、業務上許容される場合があるため、直ちにデータ異常とは判定しない。

「未割当入金」として確認対象とする。

一方、

```text
AllocatedAmount > Payment.Amount
```

はデータ不整合として扱う。

---

# 14. 参照整合性チェック

Accessへの取込後、PaymentAllocationについて以下を確認する。

### Invoice存在チェック

```text
T_AllocationImport.InvoiceId
    ↓
T_InvoiceImport.InvoiceId
```

対応するInvoiceが存在しない場合はエラーとする。

### Payment存在チェック

```text
T_AllocationImport.PaymentId
    ↓
T_PaymentImport.PaymentId
```

対応するPaymentが存在しない場合はエラーとする。

---

# 15. 月次集計要件

指定した対象年月について以下を集計する。

| 項目 | 内容 |
|---|---|
| 請求件数 | 対象月に発生した請求件数 |
| 請求金額 | 対象月のTotalAmount合計 |
| 入金件数 | 対象月に発生した入金件数 |
| 入金金額 | 対象月のPayment.Amount合計 |
| 未入金件数 | 未入金と判定された請求件数 |
| 一部入金件数 | 一部入金と判定された請求件数 |
| 過剰割当件数 | 請求額より割当額が多い件数 |
| 未割当入金件数 | 入金額の全額が割当されていない件数 |
| 割当額超過件数 | 入金額より割当額が多い件数 |

月次売上や会計上の正式な締め金額を確定する機能ではなく、月次確認用の集計とする。

---

# 16. Accessクエリ

Phase 1では以下のクエリを作成する。

```text
Q_InvoiceAllocationSummary
    請求単位の割当額集計

Q_PaymentAllocationSummary
    入金単位の割当額集計

Q_UnpaidInvoices
    未入金請求

Q_PartiallyPaidInvoices
    一部入金請求

Q_OverAllocatedInvoices
    請求額超過割当

Q_UnallocatedPayments
    未割当・一部未割当入金

Q_PaymentOverAllocation
    入金額超過割当

Q_OrphanAllocations
    存在しない請求・入金への割当

Q_MonthlySummary
    月次集計
```

---

# 17. 画面要件

Phase 1では以下の3画面を作成する。

```text
F_MainMenu

F_CsvImport

F_CheckResult
```

## F_MainMenu

以下の機能への入口を提供する。

```text
CSV取込

月次チェック

チェック結果

月次レポート
```

## F_CsvImport

以下のCSVを指定して取り込めること。

```text
請求CSV
入金CSV
入金割当CSV
```

取込完了時に、それぞれの取込件数を表示する。

## F_CheckResult

以下の種類で結果を絞り込めること。

```text
すべて
未入金
一部入金
過剰割当
未割当入金
割当額超過
参照不整合
```

---

# 18. レポート要件

以下のAccessレポートを作成する。

```text
R_MonthlyCheck
```

レポートには少なくとも以下を表示する。

```text
対象年月
CSV取込件数
請求件数
請求金額
入金件数
入金金額
未入金件数
一部入金件数
過剰割当件数
未割当入金件数
割当額超過件数
```

---

# 19. 非機能要件

## 19.1 データ安全性

AccessからInvoice Management SystemのDBを変更しない。

本ツールは参照・照合用途とする。

## 19.2 再実行性

同じCSVを使用して再取込・再照合できること。

## 19.3 トレーサビリティ

不整合結果から以下のIDを確認できること。

```text
InvoiceId
PaymentId
AllocationId
```

## 19.4 可搬性

CSVファイルの保存先を固定パスに依存させず、利用者がファイル選択できること。

---

# 20. Phase 1 CSVインターフェース確定版

Invoice Management SystemからAccessへ渡すCSVは以下の3種類とする。

## invoices_YYYYMM.csv

```text
InvoiceId
MemberId
InvoiceNumber
InvoiceDate
DueDate
TotalAmount
StatusCode
StatusName
MemberName
```

## payments_YYYYMM.csv

```text
PaymentId
MemberId
PaymentDate
Amount
PayerName
Method
```

## allocations_YYYYMM.csv

```text
AllocationId
PaymentId
InvoiceId
Amount
```

Phase 1ではこの3CSVをAccessとの正式なデータ連携インターフェースとする。

---

# 21. 完成条件

Phase 1は以下を満たした時点で完成とする。

```text
Invoice Management System
        ↓
3種類のCSVを出力
        ↓
Accessへ取込
        ↓
InvoiceId / PaymentIdでPaymentAllocationを集計
        ↓
請求額・入金額と照合
        ↓
未入金
一部入金
過剰割当
未割当入金
割当額超過
参照不整合
        ↓
一覧表示
        ↓
月次確認レポート
```

Access側からInvoice Management Systemへの更新は行わない。

本ツールはInvoice Management Systemに対する月次照合・確認用の業務支援ツールとして位置付ける。
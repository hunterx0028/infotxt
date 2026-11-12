# 線上授信審件：徵審表資料來源說明

「徵審表」的完整名稱是「授信業務額度與徵信審核表」。點下去會開啟 `IASQRY/IASQRY0157.aspx`，這是一張 RDLC 報表（`Reports/Rdlc/rpt_IASQRY0157.rdlc`）。資料來自一支主要 SQL，再補上另外 5 個來源的資料，最後由程式把各欄位組成報表上要顯示的文字。

> 程式包：`integrate_account_clean_code`。以下行號以該程式包內的檔案為準。

## 一、從列表到報表

```
ModifyCreditActStatus.aspx:1239 「徵審表」連結
   → GetUrl(SeqNo, BranchId, Account, IDNO, FileID, 2)（:438）
   → 對話框 vfn == "2"（:30-32）
   → IASQRY/IASQRY0157.aspx?Pk_BKNo=SeqNo&BranchId=...&Account=...&IDNO=...
   → IASQRY0157.aspx.cs Page_Load：取資料、加工
   → ReportViewer 套用 rpt_IASQRY0157.rdlc 顯示（:397-403）
```

## 二、主要資料：`OCADAO.GetCreditLimtCredifInfoBySales`（`OCADAO.cs:1262`）

使用 `connstr` 連線，以 `SeqNo + BranchId + Account` 只查這一筆申請：

| 資料表 | 關聯方式 | 提供的內容 |
|---|---|---|
| `PC_OLAccount`（主檔） | `SeqNo` | 申請日、開戶日、契約開戶日、狀態、預計額度 `ExpectAmt`、退票 `Refund`、申請項目 `Kind`、申請種類 `Type`、申請營業員 |
| `Content` | 代碼對照 | 申請項目、申請種類的中文名稱 |
| `StkCusData` | 分公司 + 帳號 | 客戶姓名 |
| `PC_RPA` | left join `SeqNo` | 交易量 `TradeAmt`、庫存 `InventoryAmt`、`FileID1`~`FileID3`、本人各項現有額度（信用、不限用途、款項借貸、證券借貸）、關聯戶額度（`...R`）、交易次數 `TradeCount` |
| `AcctInfoByID` | **inner join** 分公司 + 帳號，`Type='1'` | 證券帳戶開戶日 `ODate`，用來算投資經驗和開戶滿三個月 |
| `PC_CreditInfoBySales` | left join `OnlineSeqNo` | 聯徵結果 `CType_4_Flag`/`CType_4_Des`，以及營業員、經理人、督導、處長、總經理、後台經辦、後台主管的工號與簽核日期。姓名由 `fn_GetEmpName` 轉換 |

注意 `AcctInfoByID` 是 inner join：這個帳號在歸戶檔沒有 `Type='1'` 的資料時，整張報表會是空的。

## 三、另外補充的 5 個來源

| 項目 | 程式 | 連線 / 資料 |
|---|---|---|
| 違約記錄、聯徵查詢日期、已開信用/證券借貸/款項借貸/不限用途戶數 | `B27_JointCreditBySalesDAO.GetCreaditB27forCRM(SeqNo, "5")`（`IASQRY0157.aspx.cs:201`） | `CRMconnstr2`，執行預存程序 `sp_GetB27Data` |
| 聯徵異常說明、高風險說明 | `B27_JointCreditBySalesDAO.GetB27CreditAccount(SeqNo)`（`:387`） | `CRMconnstr2`，`Workflow.dbo.CreditAccount` |
| 關聯戶額度加總 | `IASQRY015DAO.GetSumMoney_CRM(SeqNo)`（`:240`） | `CRMWorkflowconnstr`，`AffiliatedAccountInfo` 最新一天的資料 |
| 契約開戶日（調整額度時） | 先查 `OCADAO.GetContractOpenDateFromOCA`（`:347`） | `connstr`，`PC_OLAccount` 狀態 3 的那筆 |
| 契約開戶日（上一步查不到時） | `R6DBDAO.GetContractOpenDateFromR6DB`（`:356`） | `R6DBconnstr`，依申請項目查 `CD_CLIENT`/`SB_CLIENT`/`BP_CLIENT` |

## 四、程式加工的欄位（`IASQRY0157.aspx.cs:86-395`）

SQL 取回的多半是原始值，報表上的勾選框文字是程式組出來的：

| 報表項目 | 規則 |
|---|---|
| 開設受託買賣帳戶滿三個月 | 用證券開戶日比對：狀態 3 比授信開戶日，其他狀態比 `PC_RPA` 的異動日（`ChkStkOpenMonth`） |
| 2. 退票記錄 | `Refund`：`Y`→是，`N`→否，空→不適用 |
| 7. 聯合徵信記錄 | `CType_4_Flag`：`0`→正常，其他→異常並附說明 |
| 投資經驗 | 證券開戶日到 `PC_RPA` 異動日的天數：≥1825 天為 5 年以上；730 天以上為 2~5 年；365 天以上為 1~2 年；大於 0 為 1 年以下；其他為新開戶 |
| 交易頻率 | `TradeCount`：≥201 每日、30~200 每週、6~29 每季、2~5 半年、其他一年以上 |
| 3. 違約記錄、查詢日期、已開戶數 | 來自 `sp_GetB27Data` |
| 申請額度（Apply1~3） | 依申請項目 `Kind`，把 `ExpectAmt` 填進對應的額度欄位：1 填融資/融券、2 填借貸款項、3 填有價證券借貸、4 填不限用途 |
| 總授信額度（本人） | 本次申請額度 + 其他三項現有額度 |
| 總授信額度（含關聯戶） | 本人額度 + 關聯戶額度（`PC_RPA` 的 `...R` + `AffiliatedAccountInfo` 加總） |
| 審核人員欄 | 姓名 + 簽核日期 |
| 申請日、簽約日 | 都填該次作業完成日 `OpenDate` |
| 核准日、開戶日、有效日 | `GatDateVal` 依申請項目和種類決定：新開或續約用完成日，調整額度用原契約開戶日 |
| 8. 聯徵異常說明、9. 高風險說明 | 來自 `Workflow.dbo.CreditAccount` |

## 五、需要注意的地方

1. **開報表會寫資料庫。** 申請種類不是新開或續約，而且契約開戶日要從 R6DB 查到時，`IASQRY0157.aspx.cs:369` 會呼叫 `UpdateContractOpenDate`，把契約開戶日寫回資料庫，使用者只是打開報表也會發生。
2. **`PC_RPA` 影響很多欄位。** 投資經驗、交易頻率、現有額度、關聯戶額度、開戶滿三個月都來自 `PC_RPA`。這筆申請在 `PC_RPA` 還沒有資料時，這些欄位會是空的或顯示預設值。
3. **既有的小錯誤：** `IASQRY0157.aspx.cs:77-84` 新增 `B27AbnormalNote`、`AmountNote` 兩個欄位時，設定型別的對象寫成 `col12`。因為型別預設就是字串，目前不影響結果。

如果要看報表版面上每個欄位的位置，可以再讀 `rpt_IASQRY0157.rdlc`。


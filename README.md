# Cypress Customer Master Distribution

*Technical Specification*

> YTG_PIR_CYPRESS_DEBMAS — sends DEBMAS IDocs for customers whose business partner identification numbers changed, driven by change pointers

| | |
|---|---|
| **Object type** | Executable program (report) |
| **Program name** | `YTG_PIR_CYPRESS_DEBMAS` |
| **Package** | `YTGPI` |
| **Transaction** | None — SE38 / SA38, or as a background job step |
| **Receiver** | Cypress (external system, via ALE / IDoc) |
| **Platform** | SAP S/4HANA 2021, on-premise |
| **Language version** | ABAP 7.56, Unicode |
| **Author** | T. Parianos |
| **Version / date** | 1.0 · 2026-09-24 |
| **Status** | Draft — for review |
| **Related objects** | BDCP · BUT0ID · BUT000 · CVI_CUST_LINK · KNVV · ESINDX |

---

## 1. Purpose and scope

The report distributes customer master data to Cypress whenever an identification number of the underlying business partner changes. It uses the standard ALE change pointer mechanism, but reads the pointers of the **business partner** change document object and sends a **customer** IDoc, bridging the two through the CVI link.

- **Read** — unprocessed change pointers of change document object `BUPA_BUP`, restricted to table `BUT0ID` (BP identification numbers).
- **Map** — each business partner to its customer number through `CVI_CUST_LINK`, only if the customer is extended to sales organisation `GR02` .
- **Send** — one `DEBMAS` master IDoc per customer through `MASTERIDOC_CREATE_REQ_DEBMAS`.
- **Flag** — all read `BUT0ID` pointers as processed, including those for which no customer was found.

**Out of scope:** changes to BP fields outside `BUT0ID`, customers not extended to sales organisation `GR02`, the Cypress side of the interface (port, partner profile processing, inbound handling), IDoc reprocessing and monitoring (`BD87`, `WE02`), and the standard `BD21` / `RBDMIDOC` processing of the same message type.

## 2. Object inventory and prerequisites

The program is a single object in package `YTGPI`. It depends on the following customizing, which must exist in every target system.

**Program**

| Object | Type | Description | Role |
|---|---|---|---|
| `YTG_PIR_CYPRESS_DEBMAS` | PROG | Program YTG_PIR_CYPRESS_DEBMAS | Container, local class `LCL_CLASS` |

**ALE customizing**

| Transaction | Setting | Value | Purpose |
|---|---|---|---|
| `WE81` | Message type | `YGR_DEBMAS` | Change pointer message type (default of `P_CPMT`) |
| `BD50` | Change pointers active | `YGR_DEBMAS` | Pointers are written for this message type |
| `BD52` | Change document fields | `BUPA_BUP` / `BUT0ID` / `IDNUMBER` (+ `KEY` if needed) | Which field changes create a pointer |
| `BD61` | Change pointers globally active | Active | General switch |
| `BD64` | Distribution model | `DEBMAS` to the Cypress logical system | Receiver determination when `P_LOGSYS` is empty |
| `WE20` | Partner profile | Outbound `DEBMAS` for the Cypress logical system | Port and IDoc type for the outbound IDoc |

> [!WARNING]
>
> **Message type name differs in the source header**
>
> The program header lists the prerequisites under message type `ZDEBMAS_ID`, while the selection screen default is `YGR_DEBMAS`. The customizing must match the value actually passed in `P_CPMT`. Align the header comment with the real message type.



**Standard objects used**

| Object | Type | Use |
|---|---|---|
| `CHANGE_POINTERS_READ` | Function module | Reads unprocessed pointers from `BDCP` / `BDCPS` |
| `MASTERIDOC_CREATE_REQ_DEBMAS` | Function module | Builds and dispatches the `DEBMAS` IDocs |
| `CHANGE_POINTERS_STATUS_WRITE` | Function module | Flags pointers as processed |
| `ENQUEUE_ESINDX` | Function module (lock object `ESINDX`) | Prevents parallel runs |
| `BUT000`, `CVI_CUST_LINK`, `KNVV` | Tables | BP → customer mapping |

## 3. Selection screen

**Block B01** *(frame title `TEXT-001`)*
- Change pointer message type — `YGR_DEBMAS`
- IDoc message type — `DEBMAS`
- Logical system — *(empty)*
- ☑ Test run

| Field | Kind | Reference | Text | Default | Meaning |
|---|---|---|---|---|---|
| `P_CPMT` | Parameter, obligatory | `EDI_MESTYP` | Pointer message type | `YGR_DEBMAS` | Message type under which the change pointers are read and flagged |
| `P_OUTMT` | Parameter, obligatory | `EDI_MESTYP` | IDoc message type | `DEBMAS` | Message type of the outbound IDoc |
| `P_LOGSYS` | Parameter | `LOGSYS` | Logical system | — | Empty = receivers from the distribution model; filled = send to this logical system directly |
| `P_TEST` | Checkbox | — | Test run | `X` | No IDoc, no pointer flagging |

> [!NOTE]
>
> **Default is test mode**
>
> `P_TEST` is ticked by default. A job variant for productive use must untick it explicitly, otherwise the job runs indefinitely without sending anything.



## 4. Architecture

`START-OF-SELECTION` instantiates the local class `LCL_CLASS` and calls its single public method `PROCESS_POINTERS`. All logic is contained in that method.

[![](https://mermaid.ink/img/pako:eNptk9FymkAUhl_lzN54gymaxEamTQeBGIIiBTSppcMgrIYGdp0FprHG-870Lfsk3d2qsZ1ywcAu_7f_Oedni1KaYaShZUG_pY8JqyE0IxYR4Jf-OUIjmj6BFdiu-fBuwd5c-9bINqE1n7cUeMIbeA_Bp7ZvebYZoS_Qbl-_RGhJGc5XBAqu1QAzRpkCmGQReoF5h0NbHqMrlpSQV5AUDCfZBlhDSE5WLY45ni94MOAC41Z3h1bsTWw3tPwg9i3dlHbo4itOaxhMPT3mNwXKagX1Zo3Biw1vHJ7QBpJmcJqD8RpCfeDqY4sX0BpMQ9U2W0BJsTkRGPtyCCUYCryspf-u8O9SaMia0RRXFc6AN46sMKxpTmrMKljShmStf1FgikpoUQjHWV7VORHWvUpWsmS0BMOcDO5kKw9KUyotrhQuVRV-_fwBxsyOjWkQxiPbdeSK485mEjNzJv5QFDX01e6pBUuCbkT5rt6Rs6uTRYGlqqKsxpny6sqZuq5_or6R6uE2QiGuajGsD1LoxaEV8C7vDh8O-YcQoYcIScWtiBCHapA2VU1L0Z2aQsXToEjAsWd8dVkkq5Mz96hFkZCnPc7muLEe8AzY5sSIDZ6D0OJp-Bib1oBvSGRKyzKvK5BgkhR_DdWWnLv_ZCoI9XAaxPe-HVqSw5XwJxqvkxVZPaHdSZojaJPx2A7hfuI7J_uO3B8dm2CbNIWEZAcgpDwpArzXIAWtWJ4hrWYNVhBvV5mIV7QVvAjVj7jEEdL4I8NZ89xOaUFZhCKy49J1QuaUlgc1o83qEWnLpKj4W7POkhqbeSJ-vOMq44PAzBAukNbtSQbStugZaeed7pnav-x1Or2rfqfb55sbpF2oZ90Ltde96qm9t_3zTn-noO_yUPXs6u3l7jcp9lHs?type=png)](https://mermaid.live/edit#pako:eNptk9FymkAUhl_lzN54gymaxEamTQeBGIIiBTSppcMgrIYGdp0FprHG-870Lfsk3d2qsZ1y4bC7_N_-55zfLUpphpGGIhaRZUG_pY8JqyE0xRr4o3-O0IimT2AFtms-vFuwN9e-NbJNaM3nLQWe8AbeQ_Cp7VuebUboC7Tb1y8RWlKG8xWBgms1wIxRpgAmWYReYN7h0JbH6IolJeQVJAXDSbYB1hCSk1WLY473Cx4MuMC41d2hFXsT2w0tP4h9SzelHbr4itMaBlNPj_mPAmW1gnqzxuDFhjcOT2gDSTM4zcF4DaE-cPWxxQtoDaahapstoKTYnAiMfTmEEgwFXtbSf1f4dyk0ZM1oiqsKZ8AbR1YY1jQnNWYVLGlDsta_KDBFJbQohOMsr-qcCOteJStZMlqCYU4Gd7KVB6UplRZXCpeqCr9-_gBjZsfGNAjjke06csdxZzOJmTkTfyiKGvpq99SCJUE3onxX78jZ1cmiwFJVUVbjTHl15Uxd1z9R30j1cBuhEFe1GNYHKfTi0Ap4l3eHD4f8Q4jQQ4Sk4lZEiEM1SJuqpqXoTk2h4mlQJODYM767LJLVyZ171KJIyNMeZ3PcWA94BmxzYsQGz0Fo8TR8jE1rwA8kMqVlmdcVSDBJir-GakvO3X8yFYR6OA3ie98OLcnhSvgTjdfJiqye0O4kzRG0yXhsh3A_8Z2Tc0eej45NsE2aQkKyAxBSnhQB3muQglYsz5BWswYriLerTMQSbQUvQvUjLnGENP7KcNY8t1NaUBahiOy4dJ2QOaXlQc1os3pE2jIpKr5q1llSYzNPxB_vuMv4IDAzhAukdbuSgbQtekbaead7pvYve51O76rf6fZ7Ctog7UI9616ove5VT-297Z93-jsFfZeXqmdXby93vwEK21OA))

*The test run flag decides the path. Only the productive path creates IDocs and changes the pointer status.*

## 5. Change pointer read

```abap
CALL FUNCTION 'CHANGE_POINTERS_READ'
  EXPORTING
    change_document_object_class = 'BUPA_BUP'
    message_type                 = iv_cpmt
    read_not_processed_pointers  = abap_true
  TABLES
    change_pointers              = lt_cp
  ...

DELETE lt_cp WHERE tabname <> 'BUT0ID'.
```

All unprocessed pointers of change document object `BUPA_BUP` for the given message type are read, with no date or time restriction. Pointers of any other table of the object are removed from the working table immediately afterwards. A read error ends the program with the message returned by the function module. If no `BUT0ID` pointer remains, the program ends with a success message and nothing is sent or flagged.

## 6. BP → customer mapping

The object key `CDOBJID` of a `BUPA_BUP` pointer is the business partner number. The distinct partner numbers are collected in a sorted table and mapped to customers in one read:

```abap
SELECT b~partner, l~customer
  FROM but000 AS b
  INNER JOIN cvi_cust_link AS l ON l~partner_guid = b~partner_guid
  INNER JOIN knvv          AS v ON v~kunnr        = l~customer
  FOR ALL ENTRIES IN @lt_partner
  WHERE b~partner = @lt_partner-table_line
    AND v~vkorg   = 'GR02'
  INTO TABLE @DATA(lt_map).
```

| Source | Join / filter | Purpose |
|---|---|---|
| `BUT000` | `PARTNER` from the pointer | Resolves the partner GUID |
| `CVI_CUST_LINK` | `PARTNER_GUID` | CVI link BP → customer |
| `KNVV` | `KUNNR = CUSTOMER`, `VKORG = 'GR02'` | Restricts to customers extended to sales organisation `GR02` |

For every partner the program writes one list line — either the customer found, or *no customer in sales org GR02 - pointer skipped*. The found customers are collected in a `BDIKNA1KEY` table, sorted and reduced to distinct `KUNNR`, so a customer is sent once even if several of its pointers were read.

## 7. IDoc creation

The receiver parameters are derived from `P_LOGSYS`:

| `P_LOGSYS` | `RCVPRN` | `RCVPRT` | Effect |
|---|---|---|---|
| Empty | empty | empty | Receivers determined from the distribution model (`BD64`) |
| Filled | `P_LOGSYS` | `LS` | IDoc sent to that logical system only |

```abap
CALL FUNCTION 'MASTERIDOC_CREATE_REQ_DEBMAS'
  EXPORTING
    rcvprn       = lv_rcvprn
    rcvprt       = lv_rcvprt
    message_type = iv_outmt
  IMPORTING
    created_comm_idocs   = lv_comm_idocs
    created_master_idocs = lv_master_idocs
  TABLES
    kna1key      = lt_kna1key.
```

The call is skipped if no customer was mapped. The function module builds the full customer master IDoc for every key in `KNA1KEY` and dispatches it according to the partner profile; it commits internally.

## 8. Pointer status

```abap
lt_ident = VALUE #( FOR cp IN lt_cp ( cpident = cp-cpident ) ).

CALL FUNCTION 'CHANGE_POINTERS_STATUS_WRITE'
  EXPORTING
    message_type           = iv_cpmt
  TABLES
    change_pointers_idents = lt_ident.

COMMIT WORK.
```

Every `BUT0ID` pointer read in this run is flagged as processed — also those whose partner has no customer in `GR02`. This is deliberate: without it, those pointers would be read again on every run, indefinitely.

> [!IMPORTANT]
>
> **Pointers are flagged regardless of the IDoc result**
>
> The status is written after `MASTERIDOC_CREATE_REQ_DEBMAS` without checking `CREATED_MASTER_IDOCS`. If no IDoc is created — distribution model missing, filter not met, wrong `P_LOGSYS` — the pointers are still flagged and the change is not sent again. See §12.



## 9. Program structure

The report is a single source. All logic sits in the local class `LCL_CLASS`; `START-OF-SELECTION` only instantiates it and calls one method.

| Unit | Kind | Signature | Task |
|---|---|---|---|
| `LCL_CLASS` | Local class, `FINAL` | — | Container for the processing logic |
| `PROCESS_POINTERS` | Public method | `IMPORTING iv_cpmt TYPE edi_mestyp, iv_outmt TYPE edi_mestyp, iv_logsys TYPE logsys, iv_test TYPE abap_bool` | Lock, read, map, send, flag, list output |

### Event blocks

| Event | Calls |
|---|---|
| `START-OF-SELECTION` | `NEW lcl_class( )->process_pointers( … )` with `P_CPMT`, `P_OUTMT`, `P_LOGSYS`, `P_TEST` |

## 10. Locking and commit

A parallel run is prevented with the generic lock object `ESINDX`, keyed on the program name:

```abap
CALL FUNCTION 'ENQUEUE_ESINDX'
  EXPORTING
    relid = 'ZZ'
    srtfd = CONV indx_srtfd( sy-repid )
  EXCEPTIONS
    foreign_lock   = 1
    system_failure = 2
    OTHERS         = 3.
```

If the lock cannot be set the program ends with *Program is already running*. There is no explicit `DEQUEUE_ESINDX`.

There are two commits in a productive run: the internal commit of `MASTERIDOC_CREATE_REQ_DEBMAS`, and the explicit `COMMIT WORK` after the pointer status update. IDoc creation and pointer flagging are therefore **not atomic** — a dump between the two leaves IDocs sent and pointers still unprocessed, so the same customers are sent again on the next run. For master data distribution this is the safe direction.

> [!WARNING]
>
> **Lock duration**
>
> The lock is set with the default scope and is released at the first `COMMIT WORK`, i.e. inside `MASTERIDOC_CREATE_REQ_DEBMAS`. It therefore protects the read and mapping phase, not the pointer flagging. A second run starting in that window would read the same, not yet flagged pointers. For a single scheduled job this is harmless; to be strict, pass `_SCOPE = '1'` and dequeue explicitly at the end.



## 11. Output and messages

Output is a classic list (`NO STANDARD PAGE HEADING`). One line is written per business partner:

```
BP 1000012345 -> customer 1000012345
BP 1000012399 no customer in sales org GR02 - pointer skipped
```

The run closes with the counters:

| Mode | Output |
|---|---|
| Test run | `Test mode: <n> customer(s) would be sent, <m> pointer(s) would be flagged.` |
| Productive | `<n> master IDoc(s), <m> communication IDoc(s) created` |
| Productive | `<n> change pointer(s) flagged as processed` |

| Message | Type | Situation |
|---|---|---|
| Program is already running | E | Lock `ESINDX` held by another session |
| *Message of `CHANGE_POINTERS_READ`* | E | Pointer read failed |
| No unprocessed change pointers found | S | No `BUT0ID` pointer for the message type |

## 12. Design decisions and risks

### No check of the IDoc result

Pointers are flagged even when `CREATED_MASTER_IDOCS` is zero while customers were passed. A misconfigured distribution model or partner profile silently consumes the changes. Recommended: if `lt_kna1key` is not empty and `lv_master_idocs = 0`, stop before `CHANGE_POINTERS_STATUS_WRITE` and issue an error.

### Pointers of other tables are never flagged

`DELETE lt_cp WHERE tabname <> 'BUT0ID'` removes non-`BUT0ID` pointers from the run but does not flag them. If `BD52` for the message type ever includes fields of another `BUPA_BUP` table, those pointers accumulate in `BDCP`/`BDCPS` and are read — and discarded — on every run. Keep `BD52` restricted to `BUT0ID`, or flag the discarded pointers as well.

### Hard-coded sales organisation

`VKORG = 'GR02'` is a literal. Customers extended only to other sales organisations are never sent, and their pointers are flagged. If the scope may change, move the sales organisation to the selection screen or a TVARVC entry.

### Mapping only for customers with a sales area

The inner join on `KNVV` excludes customers that exist in `CVI_CUST_LINK` but have no sales area in `GR02` yet. A BP whose identification number changes *before* the sales area extension is not sent, and its pointer is flagged; the customer only reaches Cypress with the next ID change or a manual `BD12` send.

### Full read without date limit

`CHANGE_POINTERS_READ` is called without a date interval. Volume is bounded only by how many pointers are unprocessed; after a long outage the first run may be large. Processed pointers should be reorganised regularly with `RBDCPCLR`.

### No authorization check

The program performs no `AUTHORITY-CHECK`. Anyone who can start it can trigger customer master distribution. In practice it should run only as a background job under a technical user; restrict `S_PROGRAM` or assign an authorization group to the program if dialog start must be prevented.

### Minor source findings

- The header comment names message type `ZDEBMAS_ID`; the default is `YGR_DEBMAS` (§2).
- `REPORT … NO STANDARD PAGE HEADING..` carries a double period.
- The empty `FORMS` banner at the end of the source can be removed.
- `LCL_CLASS` and `GO_HELPER` are generic names; `LCL_DEBMAS_SENDER` would state the purpose.

## 13. Test and operation

1. **Check customizing.** `BD61` active, `BD50` active for `YGR_DEBMAS`, `BD52` fields on `BUT0ID`, `BD64` model and `WE20` partner profile for `DEBMAS` to the Cypress logical system.
2. **Create a pointer.** Change an identification number of a BP that has a customer in `GR02` (`BP`, tab *Identification*), and one of a BP without.
3. **Test run.** Execute with `P_TEST` ticked. Both BPs appear in the list, one mapped and one skipped; the counters show one customer and two pointers.
4. **Productive run.** Untick `P_TEST` and execute. Check the counters, then the IDoc in `WE02` / `BD87` (status 03 or 12 expected).
5. **Re-run.** Execute again — *No unprocessed change pointers found* confirms the flagging.
6. **Schedule.** Create a variant with `P_TEST` unticked and schedule the report as a periodic job (`SM36`) under the technical ALE user.

> [!TIP]
> **Transport**
>
> `R3TR PROG YTG_PIR_CYPRESS_DEBMAS`, including text symbol `001` and the selection texts. The message type (`WE81`), change pointer activation (`BD50`), and change document fields (`BD52`) are customizing and need a separate customizing transport; the distribution model (`BD64`) and partner profile (`WE20`) are maintained per system.





---

*YTG_PIR_CYPRESS_DEBMAS · Technical specification v1.0 · SAP S/4HANA 2021 on-premise · 2026-09-24*

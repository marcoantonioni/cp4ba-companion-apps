# RPA scripts

## 1. Bot Script Architecture and Analysis

To illustrate script execution in production, let us analyze the anatomy of a modular WAL (Workplace Application Language) automation script [`MyBot1.wal`](MyBot1.wal).

### 1.1 Script Structure and Lifecycle

The script follows structured automation engineering standards with variable handling, subroutine decomposition, formatted timestamp generation, Windows Event logging, and central error management:

```
+-----------------------------------------------------------------------------+
|                             Bot1 Execution Flow                             |
|                                                                             |
|  [Main Entry]                                                               |
|     |                                                                       |
|     +--> Sub Init           (Validate required input / get Windows user)    |
|     +--> Sub BuildDateTime  (Extract date parts, pad, format timestamp)     |
|     +--> Sub LogRun         (Log start event to console & Windows Log)      |
|     +--> Sub Wait           (Perform deterministic pause / business delay)  |
|     +--> Sub LogFinish      (Log end event to console & Windows Log)        |
|     +--> Sub Cleanup        (Release resources and gracefully exit)         |
|                                                                             |
|  [Exception Handler: ErrorHandler] -> Stop execution on failure             |
+-----------------------------------------------------------------------------+
```

### 1.2 Key Technical Capabilities Demonstrated in Script

1. **Parameter Passing & Fallback**:
   ```wal
   defVar --name localInput --type String --value DefaultTestValue --parameter
   ...
   textNullOrEmpty --text "${localInput}" isNullOrEmpty=value
   if --left "${isNullOrEmpty}" --operator "Equal_To" --right True
       setVar --name "${localInput}" --value "No input found"
   endIf
   ```
   The script declares `localInput` as an external entry parameter marked `--required`. If called without explicit payload, fallback handling prevents null dereferencing.

2. **Date/Time Arithmetic & Zero-Padding**:
   Extracts distinct date/time integers (`Years`, `Months`, `Days`, `Hours`, `Minutes`, `Seconds`), applies `padText --paddingchar 0`, and constructs a normalized ISO-like string `YYYY-MM-DD HH:MM:SS`.

3. **Operating System Level Event Log Integration**:
   ```wal
   logMessage --message "Bot1 run at ${dateTimeOfRun} with param value [ ${localInput} ] using userId [ ${theUserId} ]" --type "Info" --logonwindows --eventid 987
   ```
   Emits structured diagnostic events directly into Windows Event Viewer using application-specific Event IDs (`987` for job initiation, `988` for job completion).


# RPA configuration steps

...to be completed
# UAT S07 brief 3 — validación contra el render

Solo los headings de **anexo a profundidad 4 y 5**: son los que el parser no reconocía y
ahora sí. El padre se lista para situarse en el documento.

**Qué se comprueba:** que el número de la izquierda es el que **imprime Word** junto a ese
título. Un número desplazado (que empiece en 2, o que salte uno) es el fallo que importa:
daría un ancla precisa pero incorrecta, peor que la gruesa que había antes.

Si algo no cuadra, **no forzarlo** — se revierte el enrutado de ese estilo.

---

## ECSS-E-ST-70-31C — 107 secciones nuevas

- **A.2.3** «Service requests and reports»
  - [ ] `A.2.3.1` — Telecommand acceptance
  - [ ] `A.2.3.2` — Telecommand execution started (S1 ST3 and ST4 Provider = TRUE)
  - [ ] `A.2.3.3` — Telecommand execution progress (S1 ST5 and ST6 Provider = TRUE)
  - [ ] `A.2.3.4` — Telecommand execution complete (S1 ST7 and ST8 Provider = TRUE)

- **A.3.3** «Service requests and reports»
  - [ ] `A.3.3.1` — Distributing On/Off commands (S2 ST1 Provider = TRUE)
  - [ ] `A.3.3.2` — Distributing register load commands (S2 ST2 Provider = TRUE)
  - [ ] `A.3.3.3` — Distributing CPDU commands (S2 ST3 Provider = TRUE)

- **A.4.3** «Service requests and reports»
  - [ ] `A.4.3.1` — Defining new housekeeping parameter report (S3 ST1 ST3 ST5 and ST6 Provider = TRUE)
  - [ ] `A.4.3.2` — Clearing housekeeping parameter report definitions (S3 ST1 ST3 ST5 and ST6 Provider = TRUE)
  - [ ] `A.4.3.3` — Controlling the generation of housekeeping parameter reports (S3 ST1 ST3 ST5 and ST6 Provider = TRUE)
  - [ ] `A.4.3.4` — Reporting housekeeping parameter report definitions (S3 ST9 and ST10 Provider = TRUE)
  - [ ] `A.4.3.5` — Reporting housekeeping parameter sampling-time offsets (S3 ST13 and ST15 Provider = TRUE)
  - [ ] `A.4.3.6` — Selecting housekeeping parameter report generation mode (S3 ST17 and ST19 Provider = TRUE)
  - [ ] `A.4.3.7` — Reporting unfiltered housekeeping parameters (S3 ST21 and ST23 Provider = TRUE)
  - [ ] `A.4.3.8` — Reporting housekeeping data (S3 HK Provider = TRUE)
  - [ ] `A.4.3.9` — Defining new diagnostic parameter report (S3 Diagnostic Provider = TRUE)
  - [ ] `A.4.3.10` — Clearing diagnostic parameter report definitions (S3 Diagnostic Provider = TRUE)
  - [ ] `A.4.3.11` — Controlling the generation of diagnostic parameter reports (S3 Diagnostic Provider = TRUE)
  - [ ] `A.4.3.12` — Reporting diagnostic parameter report definitions (S3 ST11 and ST12 Provider = TRUE)
  - [ ] `A.4.3.13` — Reporting diagnostic parameter sampling-time offsets (S3 ST14 and ST16 Provider = TRUE)
  - [ ] `A.4.3.14` — Selecting diagnostic parameter report generation mode (S3 ST18 and ST20 Provider = TRUE)
  - [ ] `A.4.3.15` — Reporting unfiltered diagnostic parameters (S3 ST22 and ST24 Provider = TRUE)
  - [ ] `A.4.3.16` — Reporting diagnostic data (S3 Diagnostic Provider = TRUE)

- **A.5.3** «Service requests and reports»
  - [ ] `A.5.3.1` — Reporting the parameter statistics results
  - [ ] `A.5.3.2` — Resetting the parameter statistics reporting
  - [ ] `A.5.3.3` — Selecting the parameter statistics reporting mode (S4 ST4 and ST5 Provider = TRUE)
  - [ ] `A.5.3.4` — Adding parameters to the parameter statistics list
  - [ ] `A.5.3.5` — Deleting parameters from the parameter statistics list (S4 ST6 and ST7 Provider = TRUE)
  - [ ] `A.5.3.6` — Reporting the parameter statistics list (S4 ST8 and ST9 Provider = TRUE)
  - [ ] `A.5.3.7` — Clearing the parameter statistics list (S4 ST6 and ST10 Provider = TRUE)

- **A.6.3** «Service requests and reports»
  - [ ] `A.6.3.1` — Reporting events
  - [ ] `A.6.3.2` — Controlling the generation of event reports

- **A.7.3** «Service requests and reports»
  - [ ] `A.7.3.1` — Loading data in memory using base plus offsets (S6 ST1 ST3 and ST4 Provider = TRUE)
  - [ ] `A.7.3.2` — Loading data in memory using absolute addresses (S6 ST2 ST5 and ST6 Provider = TRUE)
  - [ ] `A.7.3.3` — Dumping memory using base plus offsets (S6 ST1 ST3 and ST4 Provider = TRUE)
  - [ ] `A.7.3.4` — Dumping memory using absolute addresses (S6 ST2 ST5 and ST6 Provider = TRUE)
  - [ ] `A.7.3.5` — Checking memory using base plus offsets (S6 ST7 and ST8 Provider = TRUE)
  - [ ] `A.7.3.6` — Checking memory using absolute addresses (S6 ST9 and ST10 Provider = TRUE)

- **A.8.3** «Service requests and reports»
  - [ ] `A.8.3.1` — Perform function (S8 Provider = TRUE)

- **A.9.3** «Service requests and reports»
  - [ ] `A.9.3.1` — Controlling the time report generation rate (S9 ST1 Provider = TRUE)
  - [ ] `A.9.3.2` — Time reporting (S9 ST2 Provider = TRUE)

- **A.10.3** «Service requests and reports»
  - [ ] `A.10.3.1` — Controlling the release of telecommands
  - [ ] `A.10.3.2` — Resetting the command schedule
  - [ ] `A.10.3.3` — Inserting telecommands in the command schedule
  - [ ] `A.10.3.4` — Deleting telecommands from the command schedule
  - [ ] `A.10.3.5` — Time-shifting of telecommands in the command schedule
  - [ ] `A.10.3.6` — Reporting of the command schedule contents
  - [ ] `A.10.3.7` — Reporting of the status of the command schedule (S11 ST18 and ST19 Provider = TRUE)

- **A.10.3.4** «Deleting telecommands from the command schedule»
  - [ ] `A.10.3.4.1` — Deleting telecommands (S11 ST5 Provider = TRUE)
  - [ ] `A.10.3.4.2` — Deleting telecommands over a time period (S11 ST6 Provider = TRUE)

- **A.10.3.5** «Time-shifting of telecommands in the command schedule»
  - [ ] `A.10.3.5.1` — Time-shifting all telecommands (S11 ST15 Provider = TRUE)
  - [ ] `A.10.3.5.2` — Time-shifting selected telecommands (S11 ST15 Provider = TRUE and S11 ST7 Provider = TRUE)
  - [ ] `A.10.3.5.3` — Time-shifting selected telecommands over a time period (S11 ST15 Provider = TRUE and S11 ST7 Provider = TRUE and S11 ST8 Provider = TRUE)

- **A.10.3.6** «Reporting of the command schedule contents»
  - [ ] `A.10.3.6.1` — Detailed reporting of the command schedule (S11 ST16 and ST10 Provider = TRUE)
  - [ ] `A.10.3.6.2` — Detailed reporting of a subset of the command schedule (S11 ST16 and ST10 Provider = TRUE and S11 ST9 Provider = TRUE)
  - [ ] `A.10.3.6.3` — Summary reporting of the command schedule (S11 ST17 and ST13 Provider = TRUE)
  - [ ] `A.10.3.6.4` — Summary reporting of a subset of the command schedule (S11 ST17 and ST13 Provider = TRUE and S11 ST12 Provider = TRUE)
  - [ ] `A.10.3.6.5` — Detailed reporting of the command schedule over a time period (S11 ST16 and ST10 Provider = TRUE and S11 ST9 Provider = TRUE and S11 ST11 Provider = TRUE)
  - [ ] `A.10.3.6.6` — Summary reporting of the command schedule over a time period (S11 ST17 and ST13 Provider = TRUE and S11 ST12 Provider = TRUE and S11 ST14 Provider = TRUE)

- **A.11.3** «Service requests and reports»
  - [ ] `A.11.3.1` — Controlling the on-board monitoring
  - [ ] `A.11.3.2` — Changing the maximum reporting delay (S12 ST3 Provider = TRUE)
  - [ ] `A.11.3.3` — Clearing the monitoring list (S12 ST4 Provider = TRUE and S12 ST5 Provider = TRUE)
  - [ ] `A.11.3.4` — Adding parameters to the monitoring list (S12 ST5 Provider = TRUE)
  - [ ] `A.11.3.5` — Deleting parameters from the monitoring list (S12 ST5 Provider = TRUE and S12 ST6 Provider = TRUE)
  - [ ] `A.11.3.6` — Modifying the parameter checking information (S12 ST7 Provider = TRUE)
  - [ ] `A.11.3.7` — Reporting the current monitoring list contents (S12 ST8 and ST9 Provider = TRUE)
  - [ ] `A.11.3.8` — Reporting the current parameters out-of-limit list (S12 ST10 and ST11 Provider = TRUE)
  - [ ] `A.11.3.9` — Reporting the check transitions

- **A.12.3** «Service requests and reports»
  - [ ] `A.12.3.1` — Transferring the first part of a service data unit
  - [ ] `A.12.3.2` — Transferring an intermediate part of a service data unit
  - [ ] `A.12.3.3` — Transferring the last part of a service data unit
  - [ ] `A.12.3.4` — Re-transferring a part of a service data unit
  - [ ] `A.12.3.5` — Transfer abort initiated by the sending end
  - [ ] `A.12.3.6` — Acknowledging the successful reception up to a part
  - [ ] `A.12.3.7` — Notifying which parts have not been properly received
  - [ ] `A.12.3.8` — Transfer abort initiated by the receiving end

- **A.13.3** «Service requests and reports»
  - [ ] `A.13.3.1` — Controlling the forwarding of specified telemetry source packets
  - [ ] `A.13.3.2` — Reporting the list of enabled telemetry source packets (S14 ST3 and ST4 Provider = TRUE)
  - [ ] `A.13.3.3` — Controlling the forwarding of specified housekeeping packets (S14 ST5 and ST6 Provider = TRUE)
  - [ ] `A.13.3.4` — Reporting the list of enabled housekeeping packets (S14 ST7 and ST8 Provider = TRUE)
  - [ ] `A.13.3.5` — Controlling the forwarding of specified diagnostic packets (S14 ST9 and ST10 Provider = TRUE)
  - [ ] `A.13.3.6` — Reporting the list of enabled diagnostic packets (S14 ST11 and ST12 Provider = TRUE)
  - [ ] `A.13.3.7` — Controlling the forwarding of specified event report packets (S14 ST13 and ST14 Provider = TRUE)
  - [ ] `A.13.3.8` — Reporting the list of enabled event report packets (S14 ST15 and ST16 Provider = TRUE)

- **A.14.3** «Service requests and reports»
  - [ ] `A.14.3.1` — Controlling the storage in specified packet stores (S15 Packet Selection Provider = TRUE)
  - [ ] `A.14.3.2` — Modifying the definition of a storage selection criteria (S15 Packet Selection Provider = TRUE and S15 ST3 and ST4 Provider = TRUE)
  - [ ] `A.14.3.3` — Reporting a storage selection definition (S15 Packet Selection Provider = TRUE and S15 ST3 and ST4 Provider = TRUE and S15 ST5 and ST6 Provider = TRUE)
  - [ ] `A.14.3.4` — Downlinking the contents of a packet store for a specified packet range (S15 Storage and Retrieval Provider = TRUE and S15 ST7 Provider = TRUE)
  - [ ] `A.14.3.5` — Downlinking the contents of a packet store for a specified time period (S15 Storage and Retrieval Provider = TRUE and S15 Time Stamping = TRUE and S15 ST9 Provider = TRUE)
  - [ ] `A.14.3.6` — Deleting the contents of specified packet stores up to specified packets (S15 Storage and Retrieval Provider = TRUE and S15 ST10 Provider = TRUE)
  - [ ] `A.14.3.7` — Deleting the contents of specified packet stores up to a specified storage time (S15 Storage and Retrieval Provider = TRUE and S15 ST11 Provider = TRUE)
  - [ ] `A.14.3.8` — Reporting packet store catalogues (S15 Storage and Retrieval Provider = TRUE and S15 ST12 and ST13 Provider = TRUE)

- **A.16.3** «Service requests and reports»
  - [ ] `A.16.3.1` — Loading a procedure
  - [ ] `A.16.3.2` — Deleting a procedure
  - [ ] `A.16.3.3` — Starting a procedure
  - [ ] `A.16.3.4` — Stopping a procedure
  - [ ] `A.16.3.5` — Suspend a procedure
  - [ ] `A.16.3.6` — Resume a procedure
  - [ ] `A.16.3.7` — Abort a procedure
  - [ ] `A.16.3.8` — Communicate parameters to a procedure (S18 ST7 Provider = TRUE)
  - [ ] `A.16.3.9` — Reporting the list of on-board operations procedures (S18 ST8 and ST9 Provider = TRUE)
  - [ ] `A.16.3.10` — Reporting the list of active on-board operations procedures (S18 ST10 and ST11 Provider = TRUE)

- **A.17.3** «Service requests and reports»
  - [ ] `A.17.3.1` — Adding events to the detection list
  - [ ] `A.17.3.2` — Deleting events from the detection list (S19 ST2 Provider = TRUE)
  - [ ] `A.17.3.3` — Clearing the event detection list (S19 ST3 Provider= TRUE)
  - [ ] `A.17.3.4` — Controlling the actions associated with events
  - [ ] `A.17.3.5` — Reporting the event detection list (S19 ST6 and ST7 Provider = TRUE)

## ECSS-E-ST-70-41C — 33 secciones nuevas

- **C.2.1** «ST[01] request verification»
  - [ ] `C.2.1.1` — Acceptance and reporting
  - [ ] `C.2.1.2` — Execution reporting
  - [ ] `C.2.1.3` — Routing and reporting

- **C.2.2** «ST[02] device access»
  - [ ] `C.2.2.1` — Device access

- **C.2.3** «ST[03] housekeeping»
  - [ ] `C.2.3.1` — Housekeeping reporting
  - [ ] `C.2.3.2` — Diagnostic reporting
  - [ ] `C.2.3.3` — Parameter functional reporting configuration

- **C.2.4** «ST[04] parameter statistics reporting»
  - [ ] `C.2.4.1` — Parameter statistics reporting

- **C.2.5** «ST[05] event reporting»
  - [ ] `C.2.5.1` — Event reporting

- **C.2.6** «ST[06] memory management»
  - [ ] `C.2.6.1` — Raw data memory management
  - [ ] `C.2.6.2` — Structured data memory management
  - [ ] `C.2.6.3` — Common memory management
  - [ ] `C.2.6.4` — Memory configuration

- **C.2.8** «ST[08] function management»
  - [ ] `C.2.8.1` — Function management

- **C.2.9** «ST[09] time management»
  - [ ] `C.2.9.1` — Time reporting
  - [ ] `C.2.9.2` — Time reporting control

- **C.2.11** «ST[11] time-based scheduling»
  - [ ] `C.2.11.1` — Time-based scheduling

- **C.2.12** «ST[12] on-board monitoring»
  - [ ] `C.2.12.1` — Parameter monitoring
  - [ ] `C.2.12.2` — Functional monitoring

- **C.2.13** «ST[13] large packet transfer»
  - [ ] `C.2.13.1` — Large packet downlink
  - [ ] `C.2.13.2` — Large packet uplink

- **C.2.14** «ST[14] real-time forwarding control»
  - [ ] `C.2.14.1` — Real-time forwarding control

- **C.2.15** «ST[15] on-board storage and retrieval»
  - [ ] `C.2.15.1` — Storage and retrieval
  - [ ] `C.2.15.2` — Packet selection

- **C.2.17** «ST[17] test»
  - [ ] `C.2.17.1` — Test

- **C.2.18** «ST[18] on-board operations procedure»
  - [ ] `C.2.18.1` — OBCP management
  - [ ] `C.2.18.2` — OBCP engine management

- **C.2.19** «ST[19] event­action»
  - [ ] `C.2.19.1` — Event-action

- **C.2.20** «ST[20] Parameter management»
  - [ ] `C.2.20.1` — Parameter management

- **C.2.21** «ST[21] request sequencing»
  - [ ] `C.2.21.1` — Request sequencing

- **C.2.22** «ST[22] position-based scheduling»
  - [ ] `C.2.22.1` — Position-based scheduling

- **C.2.23** «ST[23] file management»
  - [ ] `C.2.23.1` — File handling
  - [ ] `C.2.23.2` — File copy

## ECSS-Q-HB-80-03A Rev.1 — 6 secciones nuevas

- **A.2.1** «Criticality classification of software products»
  - [ ] `A.2.1.1` — System-level supplier
  - [ ] `A.2.1.2` — Software-level supplier

- **A.2.2** «Criticality classification of software components»
  - [ ] `A.2.2.1` — System-level supplier
  - [ ] `A.2.2.2` — Software-level supplier

- **A.2.3** «Software dependability and safety analysis report»
  - [ ] `A.2.3.1` — System-level supplier
  - [ ] `A.2.3.2` — Software-level supplier

## ECSS-Q-HB-80-04A — 59 secciones nuevas

- **A.2.1** «Functionality»
  - [ ] `A.2.1.1` — completeness
  - [ ] `A.2.1.2` — correctness
  - [ ] `A.2.1.3` — efficiency

- **A.2.2** «Reliability»
  - [ ] `A.2.2.1` — reliability evidence

- **A.2.3** «Maintainability»
  - [ ] `A.2.3.1` — modularity
  - [ ] `A.2.3.2` — testability
  - [ ] `A.2.3.3` — complexity

- **A.2.4** «Reusability»
  - [ ] `A.2.4.1` — reusability documentation
  - [ ] `A.2.4.2` — portability
  - [ ] `A.2.4.3` — reuse modification

- **A.2.5** «Suitability for safety»
  - [ ] `A.2.5.1` — safety evidence

- **A.2.6** «Security»
  - [ ] `A.2.6.1` — security evidence

- **A.2.7** «Usability»
  - [ ] `A.2.7.1` — user documentation quality
  - [ ] `A.2.7.2` — user interface quality

- **A.2.8** «Software development effectiveness»
  - [ ] `A.2.8.1` — project development process level
  - [ ] `A.2.8.2` — project management effectiveness

- **A.3.3** «Detailed description of all metrics»
  - [ ] `A.3.3.1` — Requirement allocation
  - [ ] `A.3.3.2` — Requirement implementation coverage
  - [ ] `A.3.3.3` — Requirement completeness
  - [ ] `A.3.3.4` — V&V coverage
  - [ ] `A.3.3.5` — SPR/NCR trend analysis
  - [ ] `A.3.3.6` — Requirement clarity
  - [ ] `A.3.3.7` — Suitability of development documentation
  - [ ] `A.3.3.8` — Adherence to coding standards
  - [ ] `A.3.3.9` — CPU margin
  - [ ] `A.3.3.10` — Memory margin
  - [ ] `A.3.3.11` — Cyclomatic complexity (VG)
  - [ ] `A.3.3.12` — Nesting level
  - [ ] `A.3.3.13` — Lines of code (LOC)
  - [ ] `A.3.3.14` — Comment frequency
  - [ ] `A.3.3.15` — Requirement testability
  - [ ] `A.3.3.16` — Modular span of control
  - [ ] `A.3.3.17` — Modular coupling
  - [ ] `A.3.3.18` — Modular cohesion
  - [ ] `A.3.3.19` — Process reliability adequacy
  - [ ] `A.3.3.20` — Structural coverage
  - [ ] `A.3.3.21` — SPR/NCR status
  - [ ] `A.3.3.22` — Environmental software independence
  - [ ] `A.3.3.23` — System hardware independence
  - [ ] `A.3.3.24` — Reusability checklist
  - [ ] `A.3.3.25` — Reuse modification rate
  - [ ] `A.3.3.26` — Safety activities adequacy
  - [ ] `A.3.3.27` — Security checklist
  - [ ] `A.3.3.28` — User documentation clarity
  - [ ] `A.3.3.29` — User documentation completeness
  - [ ] `A.3.3.30` — User manual suitability
  - [ ] `A.3.3.31` — Adherence to MMI standards
  - [ ] `A.3.3.32` — Process assessment [ECSS-HB-Q-80-02]
  - [ ] `A.3.3.33` — Milestone tracking
  - [ ] `A.3.3.34` — Effort tracking
  - [ ] `A.3.3.35` — Code size stability
  - [ ] `A.3.3.36` — Requirement stability
  - [ ] `A.3.3.37` — RID/action status
  - [ ] `A.3.3.38` — V&V progress

- **A.4.2** «Detail description of the proposed OO metrics»
  - [ ] `A.4.2.1` — Number of children
  - [ ] `A.4.2.2` — Depth of inheritance tree
  - [ ] `A.4.2.3` — Parametric polymorphic factor
  - [ ] `A.4.2.4` — Coupling between objects
  - [ ] `A.4.2.5` — Lack of cohesion of methods

---

## Requisitos con texto completado (`require:level4`)

Comprobar que el texto sale entero, sin duplicar ni partir:

- [ ] **ECSS-E-ST-40-07C Rev.1** cláusula `5.3.9.3` · UID ECSS-E-ST-40-07_1630315

  ```
  The first IPublishField PublishField overloaded method shall allow publishing of a primitive type field, with the following arguments and behaviour:
  Arguments:
  “name” giving the field name;
  “description” giving the field description;
  “address” giving the pointer to the address where the value of the field is found supporting the following pointer types:
  Char8,
  Bool,
  Int8,
  Int16,
  Int32,
  Int64,
  UInt8,
  UInt16,
  UInt32,
  UInt64,
  Float32,
  Float64.
  “view” giving the fields view attribute as per ViewKind.h in [SMP_FILES];
  “state” given if the field is part of the simulation state when storing or restoring or not;
  “input” giving if the field is an input field or not;
  “output” giving if the field is an output field or not.
  Behaviour:
  If the name of the new field to be published is already used by another published field in the same parent object, it throws DuplicateName as per DuplicateName.h in [SMP_FILES];
  Otherwise, it creates and returns a new IField object.
  ```

- [ ] **ECSS-E-ST-70-01C** cláusula `5.4.2.2` (sin UID)

  ```
  The following OBCP execution states shall be supported:
  in store i.e. the OBCP is in the executable form, stored either on-board or on-ground.
  in engine i.e. the OBCP is in an on-board memory accessible by the OBCP engine:
  inactive i.e. the OBCP is idle;
  active i.e. the OBCP is under control of the OBCP engine:
  running i.e. the OBCP is executing;
  held i.e. the OBCP execution has been placed on hold.
  ```

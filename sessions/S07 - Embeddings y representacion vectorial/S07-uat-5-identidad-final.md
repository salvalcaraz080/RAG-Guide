# UAT S07 brief 5 — validación contra el PDF

Toca `corpus/numbering.py`, el módulo que S06 validó contra el render, así que la
validación es exhaustiva y el oráculo es el **PDF publicado** — no el render del DOCX, que
muestra las marcas de revisión sin aceptar y fue justo lo que produjo la errata del brief 4.

**Qué se comprueba:** que el número de la izquierda es el que imprime el PDF junto a ese
término. El caso crítico son los términos que **siguen a uno borrado**: ahí es donde el
arreglo del contador cambia el número, y donde un fallo daría una cita precisa e incorrecta.

---

## ECSS-E-HB-40-01A — 28 términos

- [ ] `3.2.1` — Burndown chart
- [ ] `3.2.2` — Capacity
- [ ] `3.2.3` — Continuous integration
- [ ] `3.2.4` — Cycle time
- [ ] `3.2.5` — Daily stand-up
- [ ] `3.2.6` — Definition of done (DoD)
- [ ] `3.2.7` — Definition of ready
- [ ] `3.2.8` — Epic
- [ ] `3.2.9` — Feature
- [ ] `3.2.10` — Increment
- [ ] `3.2.11` — Iteration
- [ ] `3.2.12` — Lead time
- [ ] `3.2.13` — Load factor
- [ ] `3.2.14` — Pair programming
- [ ] `3.2.15` — Peer review
- [ ] `3.2.16` — Product backlog
- [ ] `3.2.17` — Product Owner
- [ ] `3.2.18` — Product Backlog Refinement
- [ ] `3.2.19` — Sprint backlog
- [ ] `3.2.20` — Release plan
- [ ] `3.2.21` — Retrospective
- [ ] `3.2.22` — Sprint
- [ ] `3.2.23` — Sprint review
- [ ] `3.2.24` — Stakeholder
- [ ] `3.2.25` — Technical debt
- [ ] `3.2.26` — Test driven development
- [ ] `3.2.27` — User story
- [ ] `3.2.28` — Velocity

Result: OK

## ECSS-E-HB-40A — 2 términos

- [ ] `3.2.1` — software product:
- [ ] `3.2.2` — acceptance test

Result: OK

## ECSS-E-ST-10-02C Rev.1 — 4 términos

- [ ] `3.2.1` — model philosophy
- [ ] `3.2.2` — review­of­design
- [ ] `3.2.3` — Verification Control Board (VCB)
- [ ] `3.2.4` — verification level

Result: OK

## ECSS-E-ST-10-03C Rev.1 — 34 términos

- [ ] `3.2.1` — 24-hour equivalent noise exposure level
- [ ] `3.2.2` — abbreviated functional test (AFT)
- [ ] `3.2.3` — acceptance level
- [ ] `3.2.4` — acceptance margin
- [ ] `3.2.5` — crewed space segment element
- [ ] `3.2.6` — dwell time
- [ ] `3.2.7` — environmental tests
- [ ] `3.2.8` — full functional test (FFT)
- [ ] `3.2.9` — maximum expected acceleration
- [ ] `3.2.10` — maximum expected acoustic spectrum
- [ ] `3.2.11` — maximum expected shock
- [ ] `3.2.12` — maximum expected random vibration spectrum
- [ ] `3.2.13` — maximum predicted temperature
- [ ] `3.2.14` — minimum predicted temperature
- [ ] `3.2.15` — notching
- [ ] `3.2.16` — operational modes
- [ ] `3.2.17` — performance test
- [ ] `3.2.18` — polarity test
- [ ] `3.2.19` — qualification level
- [ ] `3.2.20` — qualification margin
- [ ] `3.2.21` — reduced functional test (RFT)
- [ ] `3.2.22` — residual life
- [ ] `3.2.23` — resolution
- [ ] `3.2.24` — resonance search
- [ ] `3.2.25` — reverberation time (T60)
- [ ] `3.2.26` — shock response spectrum (SRS)
- [ ] `3.2.27` — sign test
- [ ] `3.2.28` — temperature cycle
- [ ] `3.2.29` — temperature plateau
- [ ] `3.2.30` — test block
- [ ] `3.2.31` — test input tolerance
- [ ] `3.2.32` — thermal test at room pressure
- [ ] `3.2.33` — thermal test at mission pressure
- [ ] `3.2.34` — thermal vacuum

Result: OK

## ECSS-E-ST-10-06C — 11 términos

- [ ] `3.2.1` — constraint
- [ ] `3.2.2` — environment
- [ ] `3.2.3` — environment
- [ ] `3.2.4` — environment
- [ ] `3.2.5` — function
- [ ] `3.2.6` — functional analysis
- [ ] `3.2.7` — life cycle
- [ ] `3.2.8` — mission
- [ ] `3.2.9` — need
- [ ] `3.2.10` — specification
- [ ] `3.2.11` — technical requirements specification

Result: OK

## ECSS-E-ST-10C Rev.1 — 4 términos

- [ ] `3.2.1` — requirement traceability
- [ ] `3.2.2` — recurring product
- [ ] `3.2.3` — system engineering
- [ ] `3.2.4` — verification matrix

Result: OK

## ECSS-E-ST-40-07C Rev.1 — 55 términos

- [ ] `3.2.1` — aggregate
- [ ] `3.2.2` — association
- [ ] `3.2.3` — component
- [ ] `3.2.4` — composite
- [ ] `3.2.5` — composition
- [ ] `3.2.6` — configuration
- [ ] `3.2.7` — constructor
- [ ] `3.2.8` — consumer
- [ ] `3.2.9` — container
- [ ] `3.2.10` — contract
- [ ] `3.2.11` — data transfer
- [ ] `3.2.12` — entry point
- [ ] `3.2.13` — epoch time
- [ ] `3.2.14` — event
- [ ] `3.2.15` — event manager
- [ ] `3.2.16` — event sink
- [ ] `3.2.17` — event source
- [ ] `3.2.18` — exception
- [ ] `3.2.19` — field
- [ ] `3.2.20` — input field
- [ ] `3.2.21` — interface
- [ ] `3.2.22` — logger
- [ ] `3.2.23` — mission time
- [ ] `3.2.24` — model
- [ ] `3.2.25` — model implementation
- [ ] `3.2.26` — model instance
- [ ] `3.2.27` — output field
- [ ] `3.2.28` — operation
- [ ] `3.2.29` — package
- [ ] `3.2.30` — platform
- [ ] `3.2.31` — primitive type
- [ ] `3.2.32` — property
- [ ] `3.2.33` — provider
- [ ] `3.2.34` — reference
- [ ] `3.2.35` — resolver
- [ ] `3.2.36` — schedule
- [ ] `3.2.37` — scheduler
- [ ] `3.2.38` — service
- [ ] `3.2.39` — simple type
- [ ] `3.2.40` — simple field
- [ ] `3.2.41` — simulation environment
- [ ] `3.2.42` — simulation event
- [ ] `3.2.43` — simulation time
- [ ] `3.2.44` — simulator
- [ ] `3.2.45` — simulation
- [ ] `3.2.46` — simulation service
- [ ] `3.2.47` — source
- [ ] `3.2.48` — source component
- [ ] `3.2.49` — target
- [ ] `3.2.50` — target component
- [ ] `3.2.51` — simulation timeline
- [ ] `3.2.52` — time keeper
- [ ] `3.2.53` — value
- [ ] `3.2.54` — value type
- [ ] `3.2.55` — Zulu time

Result: OK

## ECSS-E-ST-40-08C — 37 términos

- [ ] `3.2.1` — assembly
- [ ] `3.2.2` — assembly instance
- [ ] `3.2.3` — child model instance
- [ ] `3.2.4` — component configuration
- [ ] `3.2.5` — dataflow link
- [ ] `3.2.6` — event link
- [ ] `3.2.7` — field link
- [ ] `3.2.8` — field automatic-propagation
- [ ] `3.2.9` — field propagation
- [ ] `3.2.10` — field scheduled-propagation
- [ ] `3.2.11` — field self-propagation
- [ ] `3.2.12` — field value
- [ ] `3.2.13` — global event
- [ ] `3.2.14` — global event subscription
- [ ] `3.2.15` — interface link
- [ ] `3.2.16` — link
- [ ] `3.2.17` — link base
- [ ] `3.2.18` — link client path
- [ ] `3.2.19` — link client model instance
- [ ] `3.2.20` — link owner path
- [ ] `3.2.21` — link owner model instance
- [ ] `3.2.22` — model fully qualified type name
- [ ] `3.2.23` — model instance connection
- [ ] `3.2.24` — model integrator
- [ ] `3.2.25` — model provider
- [ ] `3.2.26` — operation call
- [ ] `3.2.27` — parent model instance
- [ ] `3.2.28` — property value
- [ ] `3.2.29` — resolvable object
- [ ] `3.2.30` — resolvable object path
- [ ] `3.2.31` — sub-model instance
- [ ] `3.2.32` — simulator artefact
- [ ] `3.2.33` — simulator artefact loading
- [ ] `3.2.34` — simulator artefact application
- [ ] `3.2.35` — simulator artefact instantiation
- [ ] `3.2.36` — sub-assembly
- [ ] `3.2.37` — template argument

Result: OK

## ECSS-E-ST-40C Rev.1 — 51 términos

- [ ] `3.2.1` — acceptance test
- [ ] `3.2.2` — automatic code generation
- [ ] `3.2.3` — code coverage
- [ ] `3.2.4` — condition
- [ ] `3.2.5` — configurable code
- [ ] `3.2.6` — COTS, MOTS software
- [ ] `3.2.7` — critical software
- [ ] `3.2.8` — deactivated code
- [ ] `3.2.9` — decision
- [ ] `3.2.10` — decision coverage
- [ ] `3.2.11` — existing software
- [ ] `3.2.12` — integration testing
- [ ] `3.2.13` — logical model
- [ ] `3.2.14` — margin philosophy
- [ ] `3.2.15` — metric
- [ ] `3.2.16` — migration
- [ ] `3.2.17` — modified condition and decision coverage
- [ ] `3.2.18` — operational
- [ ] `3.2.19` — portability (a quality characteristic)
- [ ] `3.2.20` — processing unit
- [ ] `3.2.21` — quality characteristics (software)
- [ ] `3.2.22` — quality model (software)
- [ ] `3.2.23` — real-time
- [ ] `3.2.24` — regression testing (software)
- [ ] `3.2.25` — reusability
- [ ] `3.2.26` — robustness
- [ ] `3.2.27` — runtime error
- [ ] `3.2.28` — singular input
- [ ] `3.2.29` — software
- [ ] `3.2.30` — software component
- [ ] `3.2.31` — software intensive system
- [ ] `3.2.32` — software item
- [ ] `3.2.33` — software observability
- [ ] `3.2.34` — software product
- [ ] `3.2.35` — software problem
- [ ] `3.2.36` — software product assurance
- [ ] `3.2.37` — software unit
- [ ] `3.2.38` — statement coverage
- [ ] `3.2.39` — stress test
- [ ] `3.2.40` — test case
- [ ] `3.2.41` — test design
- [ ] `3.2.42` — test procedure
- [ ] `3.2.43` — test script
- [ ] `3.2.44` — threat
- [ ] `3.2.45` — unit test
- [ ] `3.2.46` — unreachable code
- [ ] `3.2.47` — usability (a quality characteristic)
- [ ] `3.2.48` — validation
- [ ] `3.2.49` — verification
- [ ] `3.2.50` — vulnerability
- [ ] `3.2.51` — walk-through

Result: OK

## ECSS-E-ST-70-01C — 10 términos

- [ ] `3.2.1` — automation
- [ ] `3.2.2` — on-board control procedure
- [ ] `3.2.3` — OBCP code
- [ ] `3.2.4` — OBCP engine
- [ ] `3.2.5` — OBCP language
- [ ] `3.2.6` — OBCP system
- [ ] `3.2.7` — OBCP step
- [ ] `3.2.8` — on-board software
- [ ] `3.2.9` — scheduling
- [ ] `3.2.10` — survival mode

Result: OK

## ECSS-E-ST-70-31C — 9 términos

- [ ] `3.2.1` — activity
- [ ] `3.2.2` — compound parameter
- [ ] `3.2.3` — event
- [ ] `3.2.4` — parameter
- [ ] `3.2.5` — reporting data
- [ ] `3.2.6` — resource
- [ ] `3.2.7` — space system model
- [ ] `3.2.8` — synthetic parameter
- [ ] `3.2.9` — system element

Result: OK

## ECSS-E-ST-70-41C — 27 términos

- [ ] `3.2.1` — acceptance notification
- [ ] `3.2.2` — acceptance verification report
- [ ] `3.2.3` — application process
- [ ] `3.2.4` — capability
- [ ] `3.2.5` — data report
- [ ] `3.2.6` — event report
- [ ] `3.2.7` — execution notification
- [ ] `3.2.8` — execution verification report
- [ ] `3.2.9` — instruction
- [ ] `3.2.10` — message
- [ ] `3.2.11` — notification
- [ ] `3.2.12` — object path
- [ ] `3.2.13` — on-board file system
- [ ] `3.2.14` — on-board memory
- [ ] `3.2.15` — on-board parameter
- [ ] `3.2.16` — report
- [ ] `3.2.17` — repository path
- [ ] `3.2.18` — request
- [ ] `3.2.19` — routing notification
- [ ] `3.2.20` — routing verification report
- [ ] `3.2.21` — service
- [ ] `3.2.22` — subservice
- [ ] `3.2.23` — subservice entity
- [ ] `3.2.24` — subservice provider
- [ ] `3.2.25` — subservice user
- [ ] `3.2.26` — transaction
- [ ] `3.2.27` — verification report

Result: OK

## ECSS-E-ST-80C — 50 términos

- [ ] `3.2.1` — access control
- [ ] `3.2.2` — accountability
- [ ] `3.2.3` — asset
- [ ] `3.2.4` — attack
- [ ] `3.2.5` — attack vector
- [ ] `3.2.6` — authentication
- [ ] `3.2.7` — authenticity
- [ ] `3.2.8` — authorisation
- [ ] `3.2.9` — certification
- [ ] `3.2.10` — compromise
- [ ] `3.2.11` — confidentiality
- [ ] `3.2.12` — cyber
- [ ] `3.2.13` — cyber disaster
- [ ] `3.2.14` — cyber incident
- [ ] `3.2.15` — cyber space
- [ ] `3.2.16` — cyber security
- [ ] `3.2.17` — data integrity
- [ ] `3.2.18` — encryption
- [ ] `3.2.19` — fuzzing
- [ ] `3.2.20` — incident
- [ ] `3.2.21` — incident handling
- [ ] `3.2.22` — information security
- [ ] `3.2.23` — information security management system
- [ ] `3.2.24` — information system
- [ ] `3.2.25` — integrity
- [ ] `3.2.26` — malware
- [ ] `3.2.27` — penetration testing
- [ ] `3.2.28` — protective marking
- [ ] `3.2.29` — residual risk
- [ ] `3.2.30` — risk treatment
- [ ] `3.2.31` — security audit
- [ ] `3.2.32` — security controls
- [ ] `3.2.33` — security incident
- [ ] `3.2.34` — security maintenance
- [ ] `3.2.35` — security monitoring
- [ ] `3.2.36` — security patch
- [ ] `3.2.37` — security policy
- [ ] `3.2.38` — security posture
- [ ] `3.2.39` — security risk
- [ ] `3.2.40` — security risk analysis
- [ ] `3.2.41` — security risk appetite
- [ ] `3.2.42` — security risk assessment
- [ ] `3.2.43` — security risk management
- [ ] `3.2.44` — security risk owner
- [ ] `3.2.45` — security sensitivity
- [ ] `3.2.46` — source code review
- [ ] `3.2.47` — threat
- [ ] `3.2.48` — threat source
- [ ] `3.2.49` — vulnerability
- [ ] `3.2.50` — vulnerability assessment

Result: OK

## ECSS-M-ST-10C Rev.1 — 3 términos

- [ ] `3.2.1` — discipline
- [ ] `3.2.2` — domain
- [ ] `3.2.3` — function

Result: OK

## ECSS-M-ST-40C Rev.1 — 14 términos

- [ ] `3.2.1` — change control
- [ ] `3.2.2` — class 1 change
- [ ] `3.2.3` — class 2 change
- [ ] `3.2.4` — configuration control board
- [ ] `3.2.5` — configuration definition document
- [ ] `3.2.6` — configured item
- [ ] `3.2.7` — departure
- [ ] `3.2.8` — information/documentation management
- [ ] `3.2.9` — information system
- [ ] `3.2.10` — metadata
- [ ] `3.2.11` — product item
- [ ] `3.2.12` — self-signed certificate
- [ ] `3.2.13` — technical data package
- [ ] `3.2.14` — technical description

Result: OK

## ECSS-M-ST-80C — 11 términos

- [ ] `3.2.1` — acceptance of (risk)
- [ ] `3.2.2` — (risk) communication
- [ ] `3.2.3` — (risk) index
- [ ] `3.2.4` — individual (risk)
- [ ] `3.2.5` — (risk) management process
- [ ] `3.2.6` — overall (risk)
- [ ] `3.2.7` — (risk) reduction
- [ ] `3.2.8` — resolved (risk)
- [ ] `3.2.9` — (risk) scenario
- [ ] `3.2.10` — (risk) trend
- [ ] `3.2.11` — unresolved (risk)

Result: OK

## ECSS-Q-HB-80-01A — 5 términos

- [ ] `3.2.1` — asset
- [ ] `3.2.2` — domain engineering
- [ ] `3.2.3` — operational software
- [ ] `3.2.4` — reuse
- [ ] `3.2.5` — reuse software

Result: OK

## ECSS-Q-HB-80-02 Part 1A — 47 términos

- [ ] `3.2.1` — accreditation
- [ ] `3.2.2` — assessment indicator
- [ ] `3.2.3` — assessment instrument
- [ ] `3.2.4` — assessment method
- [ ] `3.2.5` — assessment process
- [ ] `3.2.6` — assessment purpose
- [ ] `3.2.7` — assessment scope
- [ ] `3.2.8` — base practice
- [ ] `3.2.9` — capability dimension
- [ ] `3.2.10` — capability profile
- [ ] `3.2.11` — certification
- [ ] `3.2.12` — certified S4S assessor
- [ ] `3.2.13` — competent assessor
- [ ] `3.2.14` — conformant assessment model
- [ ] `3.2.15` — defined process
- [ ] `3.2.16` — generic practice
- [ ] `3.2.17` — generic practice indicator
- [ ] `3.2.18` — generic resource indicator
- [ ] `3.2.19` — generic work-product indicator
- [ ] `3.2.20` — implemented process
- [ ] `3.2.21` — indicator (assessment indicator)
- [ ] `3.2.22` — organizational unit
- [ ] `3.2.23` — process assessment
- [ ] `3.2.24` — process assessment model
- [ ] `3.2.25` — process attribute
- [ ] `3.2.26` — process attribute rating
- [ ] `3.2.27` — process capability
- [ ] `3.2.28` — process capability determination
- [ ] `3.2.29` — process capability level
- [ ] `3.2.30` — process capability level rating
- [ ] `3.2.31` — process category
- [ ] `3.2.32` — process context
- [ ] `3.2.33` — process dimension
- [ ] `3.2.34` — process improvement
- [ ] `3.2.35` — process improvement action
- [ ] `3.2.36` — process improvement project
- [ ] `3.2.37` — process performance
- [ ] `3.2.38` — process profile
- [ ] `3.2.39` — process purpose
- [ ] `3.2.40` — process reference model
- [ ] `3.2.41` — process tailoring
- [ ] `3.2.42` — provisional assessor
- [ ] `3.2.43` — reference method
- [ ] `3.2.44` — standard process
- [ ] `3.2.45` — tailored process
- [ ] `3.2.46` — target capability
- [ ] `3.2.47` — work product

Result: OK

## ECSS-Q-HB-80-04A — 6 términos

- [ ] `3.3.1` — base measure
- [ ] `3.3.2` — measure (noun)
- [ ] `3.3.3` — measure (verb)
- [ ] `3.3.4` — measurement
- [ ] `3.3.5` — metric
- [ ] `3.3.6` — quality model

Result: OK

## ECSS-Q-ST-20C Rev.2 — 4 términos

- [ ] `3.2.1` — acceptance authority media
- [ ] `3.2.2` — inspectability
- [ ] `3.2.3` — producibility
- [ ] `3.2.4` — testability

Result: OK

## ECSS-Q-ST-30-02C — 17 términos

- [ ] `3.2.1` — active redundancy
- [ ] `3.2.2` — area analysis
- [ ] `3.2.3` — criticality
- [ ] `3.2.4` — end effect
- [ ] `3.2.5` — failure cause
- [ ] `3.2.6` — failure effect
- [ ] `3.2.7` — failure propagation
- [ ] `3.2.8` — failure mode and effects analysis (FMEA)
- [ ] `3.2.9` — failure mode, effects and criticality analysis (FMECA)
- [ ] `3.2.10` — functional description
- [ ] `3.2.11` — functional FMEA
- [ ] `3.2.12` — functional FMECA
- [ ] `3.2.13` — hardware FMEA
- [ ] `3.2.14` — hardware FMECA
- [ ] `3.2.15` — hardwaresoftware interaction analysis
- [ ] `3.2.16` — process FMECA
- [ ] `3.2.17` — protection device

Result: OK

## ECSS-Q-ST-40C Rev.1 — 12 términos

- [ ] `3.2.1` — cause
- [ ] `3.2.2` — criticality
- [ ] `3.2.3` — fail safe
- [ ] `3.2.4` — hazard control
- [ ] `3.2.5` — hazardous command
- [ ] `3.2.6` — hazard reduction
- [ ] `3.2.7` — hazard scenario
- [ ] `3.2.8` — operator error
- [ ] `3.2.9` — safety approval authority
- [ ] `3.2.10` — safety audit
- [ ] `3.2.11` — safety risk
- [ ] `3.2.12` — safety status parameter

Result: OK

## ECSS-Q-ST-80C Rev.2 — 39 términos

- [ ] `3.2.1` — acceptance test
- [ ] `3.2.2` — automatic code generation
- [ ] `3.2.3` — code coverage
- [ ] `3.2.4` — competent assessor
- [ ] `3.2.5` — condition
- [ ] `3.2.6` — configurable code
- [ ] `3.2.7` — COTS, MOTS software
- [ ] `3.2.8` — critical software
- [ ] `3.2.9` — deactivated code
- [ ] `3.2.10` — existing software
- [ ] `3.2.11` — integration testing
- [ ] `3.2.12` — metric
- [ ] `3.2.13` — migration
- [ ] `3.2.14` — mission products
- [ ] `3.2.15` — operational
- [ ] `3.2.16` — portability (a quality characteristic)
- [ ] `3.2.17` — processing unit
- [ ] `3.2.18` — quality characteristics (software)
- [ ] `3.2.19` — quality model (software)
- [ ] `3.2.20` — real-time
- [ ] `3.2.21` — regression testing (software)
- [ ] `3.2.22` — reusability
- [ ] `3.2.23` — software
- [ ] `3.2.24` — software component
- [ ] `3.2.25` — software item
- [ ] `3.2.26` — software product
- [ ] `3.2.27` — software problem
- [ ] `3.2.28` — software product assurance
- [ ] `3.2.29` — software unit
- [ ] `3.2.30` — test case
- [ ] `3.2.31` — test procedure
- [ ] `3.2.32` — threat
- [ ] `3.2.33` — unit test
- [ ] `3.2.34` — unreachable code
- [ ] `3.2.35` — usability (a quality characteristic)
- [ ] `3.2.36` — validation
- [ ] `3.2.37` — verification
- [ ] `3.2.38` — vulnerability
- [ ] `3.2.39` — walk-through

Result: OK

---

## Requisitos con AIM atado (24 párrafos en 14 requisitos)

Comprobar que cada AIM queda con **su** requisito, y que dos AIM de requisitos distintos
ya no se funden.

> **⚠️ Los bloques de texto de esta sección salieron truncados a 95 caracteres por línea.**
> Es un defecto del **generador de este guion**, no de los datos: el chunk contiene el AIM
> completo. Verificado sobre el caso más largo (`5.3.8.1 · ECSS-E-ST-40_0860049`, 901 chars en
> 10 párrafos): los 10 aparecen íntegros en `cite_ref.text`, incluido el de 191 caracteres.
> Los `Result: KO` de abajo son sobre la truncación del guion.
>
> El listado completo y sin truncar está en **`S07-uat-5-aim-completo.md`**, para poder
> terminar la comprobación de que cada AIM va con su requisito.

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.1` · ECSS-E-ST-40_0860036 — 1 párrafo(s)

  ```
  AIM:	Reach the approval of the software requirements baseline by all stakeholders.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.2` · ECSS-E-ST-40_0860037 — 1 párrafo(s)

  ```
  AIM:	To review compliance of the technical specification (TS) with the requirements baseline, t
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.2` · ECSS-E-ST-40_0860038 — 1 párrafo(s)

  ```
  AIM:	To baseline the software requirements before the start of the architectural design.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.3` · ECSS-E-ST-40_0860039 — 2 párrafo(s)

  ```
  AIM:	—To review the design definition file, including software 
       architectural design, detailed design, code and user manuals;
  —	To review the design justification file, including the completeness of the software unit test
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.3` · ECSS-E-ST-40_0860040 — 1 párrafo(s)

  ```
  AIM:	To baseline the software detailed design before the start of the coding.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.4` · ECSS-E-ST-40_0860041 — 1 párrafo(s)

  ```
  AIM:	To ensure that verification and validation processes have been completed successfully.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.4.5` · ECSS-E-ST-40_0860042 — 1 párrafo(s)

  ```
  AIM:	To accept the software product in the intended operational environment.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.5.1` · ECSS-E-ST-40_0860043 — 1 párrafo(s)

  ```
  AIM:	To ensure that the software status is compatible with the commencement of test activities 
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.5.4` · ECSS-E-ST-40_0860731 — 1 párrafo(s)

  ```
  AIM:	To ensure that the software validation specification is complete and gives confidence that
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.7.1` · ECSS-E-ST-40_0860048 — 1 párrafo(s)

  ```
  AIM:	Define procedures that guarantee the consistency of the system interfaces.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.3.8.1` · ECSS-E-ST-40_0860049 — 10 párrafo(s)

  ```
  AIM:	This allows anticipating:
  —	Expected changes in the requirements baseline,
  —	Requirements on reprogramming of the system during operational use,
  —	Required budget for temporary copies of software images,
  —	Constraints on state transitions, especially when recovery from a faulty state is concerned,
  —	Constraints on physical processor (CPU) type and memory and expected processor capacity, wait
  —	Equipment, communication and performances aspects such as buses, protocols, acceptable errors
  —	Accuracy aspects, such as conversion to/from analogue signals, and accuracy of timing signals
  —	Budgets for OS kernel characterisation, such as context switch latency or deadlines for taski
  —	Mission and system operation characteristics and reference operational scenarios.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.8.2.2` · ECSS-E-ST-40_0860129 — 1 párrafo(s)

  ```
  AIM:	A coherent and consistent approach to project organization within each project.
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.8.3.5` · ECSS-E-ST-40_0860138 — 1 párrafo(s)

  ```
  AIM:	The goal of the complementary analysis is to assess that the non-covered code behaviour is
  ```
  Result: OK

- [ ] **ECSS-E-ST-40C Rev.1** `5.8.3.5` · ECSS-E-ST-40_0860141 — 1 párrafo(s)

  ```
  AIM:	use static analysis for the errors that are difficult to detect at run-time.
  ```
  Result: OK

---

## Regresión: nada vivo fuera del glosario cambió de número

Ninguno de estos comandos imprime el número de secciones — esa cifra la da `parse_all`. Lo que
cada uno debe dar:

```
uv run python -m corpus.diagnostics.diff_reparse           # changed: 0
uv run python -m corpus.diagnostics.audit_structural_styles  # defects: ['Normal']  (el caso conocido)
uv run python -m corpus.chunk_all --dry-run                # leaked: 0 · colliding_ids: 0 · chunks: 13876
```

Valores de referencia de `chunk_all --dry-run` tras el brief 5:

```
chunks 13.876 · unique_ids 13.876 · colliding_ids 0 · leaked 0
source_chars 5.236.527 = accounted_chars 5.236.527
by_anchor  requirement 7.922 · section 2.583 · table 1.481 · note 1.410 · glossary_term 480
```

**`source_chars` baja de 5.244.208 a 5.236.527, y la diferencia es exactamente 7.681** — los
chars del estilo `Definition1` medidos en el brief 2. Es el efecto esperado del enrutado: el
texto del término dejó de ser prosa de cuerpo y pasó a ser el **título** de su cláusula, y los
títulos no cuentan como contenido indexable. El desglose cuadra al carácter:

```
section      1.686.757 → 1.684.728   (-2.029)
glossary_term   99.501 →    91.379   (-8.122)
requirement  1.560.142 → 1.562.612   (+2.470)   los AIM, que antes eran prosa de sección
note           436.921 →   436.921   (      0)
table        1.460.887 → 1.460.887   (      0)
                                       ───────
                                        -7.681
```

**Sobre el número de secciones** (que lo imprime `parse_all`, no estos comandos): pasa de
**5.356 a 5.836**, y los términos de glosario son exactamente **480** — o sea lo único que se
añadió fue el glosario. Requisitos: 7.922, sin cambio.
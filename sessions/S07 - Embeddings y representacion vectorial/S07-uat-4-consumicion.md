# UAT S07 brief 4 — muestreo dirigido

No aleatorio: los dos defectos de esta sesión que sobrevivieron a la verificación
automática cayeron mirando sitios concretos. Cuatro comprobaciones, una por riesgo.

---

## 1 · Estructura anidada sobrevive al chunking

Contra el render: la sangría refleja el anidamiento real del documento y no se ha aplanado.

- [ ] `ECSS-E-ST-40C#5.3.4.3::s0` · autoridad **normative**

  ```
  AIM:	—To review the design definition file, including software 
       architectural design, detailed design, code and user manuals;
  —	To review the design justification file, including the completeness of the software unit testing, integration and validation with respect to the technical specification.
  AIM:	To baseline the software detailed design before the start of the coding.
  ```

  Resultado: el contenido de la clausula 5.3.4.3 es el siguiente:

  ```
  After completion of the design of software items, coding and testing,
integration and validation with respect to the technical specification, a
critical design review (CDR) shall take place.
AIM: —To review the design definition file, including software
 architectural design, detailed design, code and user manuals;
— To review the design justification file, including the
completeness of the software unit testing, integration and
validation with respect to the technical specification.
EXPECTED OUTPUT: Approved design definition file and design
justification file [DDF, DJF; CDR].

  ```

  Conclusion: parece que el parser no extrajo bien el contenido. Independientemente de la indentación.

## 2 · Glosario agrupado por término

Cada término es UN chunk con su definición y su NOTE de procedencia dentro. Contra la cláusula 3.2 de ECSS-E-ST-40C.

- [ ] `ECSS-E-ST-40C#3.2::g0` · autoridad **normative**

  ```
  acceptance test
  test of a system or functional unit usually performed by the customer on his premises after installation, with the participation of the supplier to ensure that the contractual requirements are met
  Adapted from ISO/IEC 2382-20:1990
  ```

- [ ] `ECSS-E-ST-40C#3.2::g1` · autoridad **normative**

  ```
  automatic code generation
  generation of source code with a tool from a model
  Code issued from a generative artificial intelligence is considererd as manually generated code.
  ```

- [ ] `ECSS-E-ST-40C#3.2::g2` · autoridad **normative**

  ```
  code coverage
  percentage of the software that has been executed (covered) by the test suite
  ```

Resultado: la cláusula 3.2 es un header de alto nivel con título "Terms specific to the present standard". Las 3 clausulas normativas hacen referencia a las subsecciones de la sección 3.2:
- 3.2.1 acceptance test
- 3.2.2 automatic code generation
- 3.2.3 code coverage

El contenido normativo en las dos primeras no es del todo correcto:
- 3.2.1 acceptance test:
  - normativo: test of a system or functional unit usually performed by the customer on his premises after installation, with the participation of the supplier to ensure that the contractual requirements are met
  - informativo: NOTE Adapted from ISO/IEC 2382-20:1990
- 3.2.2 automatic code generation
  - normativo: generation of source code with a tool from a model
  - informativo: NOTE Code issued from a generative artificial intelligence is considererd as manually generated code.

## 3 · NOTE en cláusula normativa, fuera del chunk normativo

Debe salir como chunk **informativo** propio, con la coordenada de su cláusula. Comprobar que su texto NO aparece también dentro del chunk de sección de esa cláusula.

- [ ] `ECSS-E-ST-40C#3.3::n0` · autoridad **informative**

  ```
  The abbreviated terms are common for the ECSS-E-ST-40 and ECSS-Q-ST-80 Standards.
  ```

- [ ] `ECSS-E-ST-40C#5.2.2.1::n0` · autoridad **informative**

  ```
  There are several pre-existing security requirements catalogues or controls that can be used (e.g., ISO 27002:2013, NIST SP 800-53, and ITSG-33).
  ```

Resultado: las notas informativas en este caso están bien clasificadas.

## 4 · EXPECTED OUTPUT dentro del requisito

Requisito y salida esperada bajo la misma coordenada, delimitados y verbatim.

- [ ] `ECSS-E-ST-40C#5.2.2.1::ECSS-E-ST-40_0860001` · autoridad **normative**

  ```
  The customer shall derive system requirements allocated to software from an analysis of the specific intended use of the system, from the results of the security analysis, from the results of the hardware/software co-engineering analysis, and from the results of the safety and dependability analysis.
  Expected output:
  The following outputs are expected:
  a.	Functions and performance system requirements allocated to software [RB, SSS; SRR];
  b.	Verification and validation product requirements [RB, SSS; SRR];
  c.	Software operations requirements [RB, SSS; SRR];
  d.	Software maintenance requirements [RB, SSS; SRR];
  e.	Requirements for in flight modification capabilities [RB, SSS; SRR];
  f.	Requirements for real-time [RB, SSS; SRR];
  g.	Requirements for security [RB, SSS; SRR];
  h.	Quality requirements [RB, SSS; SRR].
  ```

- [ ] `ECSS-E-ST-40C#5.2.2.2::ECSS-E-ST-40_0860002` · autoridad **normative**

  ```
  The customer shall specify all software observability requirements to monitor the software behaviour and to facilitate the system integration and failure investigation.
  Expected output:
  System and software observability requirements [RB, SSS; SRR].
  ```

Resultado: el contenido normativo en este caso están bien extraído.

---

## 5 · El reporte de cobertura cuadra

```
uv run python -m corpus.chunk_all --dry-run
```

Debe dar `leaked: 0` y `colliding_ids: 0`. Hoy:

```
chunks 13888 · source_chars 5.244.208 · accounted_chars 5.244.208 · leaked 0
```

Resultado: el esperado

---

## 6 · Conclusion extra

Al revisar el documento ECSS-E-ST-40C_Rev_1 me he percatado de una cosa. En muchas secciones aparecen varias numeraciones, por ejemplo, la seccion 3.2.5 aparece de la siguiente forma en el render: "3.2.5[3.2.4] configurable code". Además el contenido de esa sección viene con formato, uno en azul subrayado. El otro en naranja tachado.

La sensacion que da es que este documento, al tratarse de una revisión, contiene ambas versiones del contenido y numero de clausula. Al comparar contra el PDF descargado, esto no se da: las secciones tienen un unico numero de clausula y el contenido que aparece es coherente, no hay referencias a nuevas/viejas versiones del mismo contenido.

Creo que esto supone un problema importante a la hora de parsear el contenido, y abre de nuevo la decisión de si usar PDF.


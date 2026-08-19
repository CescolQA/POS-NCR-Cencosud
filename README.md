# POS NCR — Modernización de líneas de caja y etiquetado electrónico
### Cencosud Supermercados · Jumbo · Disco · VEA

> **Rol:** Analista Funcional QA — frente de Procesos Comerciales de Supermercados
> **Período:** 2023 – 2024
> **Alcance del programa:** +250 locales en tres cadenas
> **Stack funcional:** SGC · SAP/MDH · API Compras · BizTalk · Kafka · POS NCR · ASLS

---

## 1. El problema

Cencosud operaba sus líneas de caja sobre una plataforma de punto de venta
heredada y un sistema de impresión de flejes de escritorio. La compañía decidió
migrar a la solución **POS NCR** en más de 250 locales de Jumbo, Disco y VEA,
con un nuevo layout de hardware (interfaz touch, monitor de cliente,
terminales de autoservicio) y llevar el **etiquetado electrónico (ASLS)** a una
versión web.

El desafío no era instalar cajas. Era garantizar que **el precio que llega a la
caja sea el precio correcto**, en todos los locales, para todas las
regiones y para todas las variantes de cambio de precio que el negocio maneja —
sin cortar la operación de las salas.

Ahí es donde entra mi trabajo.

---

## 2. Mi rol

Fui **Analista Funcional QA del frente de Precios** dentro de un equipo núcleo
de tres personas en Procesos Comerciales de Supermercados, articulando con
múltiples departamentos que trabajaban en paralelo (desarrollo, integración,
QC, negocio, proveedor).

Mi responsabilidad, en una línea:

> Definir el enfoque de prueba de la migración de precios y sostener la
> visibilidad de calidad y riesgo desde el piloto hasta el rollout.

Concretamente:

- Relevé y documenté la **casuística funcional** del cambio de precios y la
  traduje en una estrategia de prueba ejecutable.
- Diseñé la **matriz de prueba de datos** y los niveles de prueba del frente.
- Coordiné lateralmente con desarrollo, negocio e integración; y **directamente
  con el equipo de QC**, con quien automatizamos pruebas y cargas contra el POS,
  incluyendo escenarios de masividad y estrés.
- Reporté avance, riesgos y criterios de pasaje semana a semana.

---

## 3. Arquitectura funcional del frente

El corazón del problema es una cadena de integración: el maestro de artículos y
precios viaja desde los sistemas comerciales hasta la caja, transformado en
archivos y publicado por mensajería.

SGC / MDH API Compras Integración Punto de venta
┌───────────────┐ ┌──────────────┐ ┌─────────────┐ ┌──────────────┐
│ Artículos │ ───▶ │ Generación │ ───▶ │ BizTalk │ ───▶ │ POS NCR │
│ Precios │ │ JSON / XML │ │ Kafka │ │ ASLS │
│ Bajas / DPC │ ◀─── │ ACK / Log │ ◀── │ │ │ (flejes) │
└───────────────┘ └──────────────┘ └─────────────┘ └──────────────┘


**Seis procesos** disparan movimiento sobre esa cadena: alta completa inicial,
novedades de artículos, novedades de precios, bajas, DPC y cambios
extraordinarios (push).

**Cuatro tipos de cambio de precio**, cada uno con un universo distinto de
artículos afectados:

| Tipo | Qué arrastra |
|---|---|
| Tradicional | Novedades del día: cotizaciones a futuro, modificaciones de maestro, ofertas y promociones, artículos en pricing, precio controlado |
| Tradicional Completo | El universo vigente completo para la empresa/región |
| Alta Completa | Todas las cotizaciones vigentes y a futuro de la empresa/región |
| Extraordinario | Artículos seleccionados manualmente por pantalla |

Multiplicado por regionalización, artículos pesables, tratamiento impositivo
(interno / nacional / importado) y locales ya migrados vs. no migrados, la
casuística se vuelve inmanejable si se ataca de frente.

---

## 4. Cómo abordé las pruebas

### 4.1 Fragmentación de la casuística en bloques

En lugar de una lista plana de casos de prueba, separé el alcance en **bloques
por punto de falla**, de modo que un error se localice en su capa en vez de
diluirse en el flujo completo:

- **Bloque 1 — Módulo de Compras (SGC/FOX):** la regla de negocio del cambio de precio.
- **Bloque 2 — API Compras:** generación de archivos, estructura, rutas, log, resguardo y ACK.
- **Bloque 3 — Integración:** el viaje SGC → BizTalk → Kafka → POS NCR.

### 4.2 Tres niveles de prueba

| Nivel | Pregunta que responde |
|---|---|
| **Prueba estructural** | ¿El archivo se genera correctamente y viaja al lugar correcto? |
| **Prueba de datos** | ¿El dato es correcto antes, durante y después del envío, y en su impacto final en el POS? |
| **Prueba de sistema (SAP–MDH)** | ¿Los escenarios de origen disparan efectivamente los procesos esperados? |

### 4.3 Matriz de prueba de datos: secuencia → orden → iteración

Diseñé una matriz de referencia que no lista casos, sino que **ordena la
ejecución**. Responde tres preguntas:

- **Secuencia:** ¿con qué proceso arranco y con cuál sigo?
- **Orden:** ¿qué campos y valores miro primero dentro de la estructura del mensaje?
- **Iteración:** ¿cómo repito la validación antes / durante / después de cada envío?

Sobre esa base se descompuso la estructura del mensaje campo por campo
(identificadores de paquete, registros de ítem, atributos dinámicos,
descripciones, unidades de medida) para que la validación fuera reproducible y
no dependiera de la memoria de quien la ejecutaba.

### 4.4 Criterios de entrada y salida

Cada ciclo semanal se abría solo si estaban dados los criterios de entrada
(documentación disponible, ambientes levantados, datos de prueba cargados,
casuística relevada, matriz de enfoque acordada con QC) y se cerraba contra
criterios de salida explícitos de cobertura para piloto y para rollout, con
fechas comprometidas de entrega a calidad y de pasaje a producción.

El estado se comunicaba semanalmente con un semáforo simple —ejecutado / en
curso / backlog— más las observaciones abiertas por tema. El objetivo no era
lucir un número, sino que **el frente entero viera el mismo riesgo al mismo
tiempo**.

---

## 5. Temas de negocio que atravesaron la prueba

No fue prueba de laboratorio. Los escenarios salieron de la operación real de sala:

- **Artículos pesables y balanza:** unidades, medidas y factor de rendimiento.
- **Trazabilidad de alta:** que un producto recién dado de alta llegue completo al POS.
- **Tratamiento impositivo:** impuesto interno, nacional, importado, proveedor.
- **Flags de cambio por campo:** qué modificación dispara qué envío.
- **Cobertura amplia de novedades en un mismo envío.**
- **Envíos masivos:** volumen y estrés, en conjunto con el equipo de QC.
- **Impresión de flejes (ASLS):** lotes, priorización de tipos de fleje,
  constantes de fórmula y su correlato con el precio publicado en góndola.

---

## 6. Coordinación

- **Lateral** con desarrollo, integración y negocio: seguimiento de incidencias,
  acompañamiento de dailies, gestión de cambios sobre el alcance acordado y
  escalamiento de errores con evidencia reproducible.
- **Directa con el equipo de QC** (departamento independiente): acordamos la
  matriz de enfoque, automatizamos pruebas y cargas contra el POS y ejecutamos
  los escenarios de masividad y estrés.
- **Documental:** especificaciones, diagramas de procesamiento e integración,
  descriptivo de datos, rutas, estructura de artículos y sets de prueba
  centralizados en Confluence, con el trabajo diario y los relevamientos en
  OneNote y el seguimiento operativo en Jira.

---

## 7. Qué me llevé de este proyecto

- Una casuística grande no se prueba: **se fragmenta hasta que cada bloque tiene
  un punto de falla identificable.**
- En integraciones, el bug rara vez está en la regla de negocio; está en el
  borde entre dos sistemas. Diseñar la prueba por capas es lo que lo revela.
- El artefacto más valioso no fue el set de casos, sino la **matriz de secuencia**:
  el orden de ejecución fue lo que hizo la prueba repetible por cualquiera del equipo.
- Un criterio de salida acordado por escrito antes de empezar evita la discusión
  de "¿esto está listo?" cuando ya no hay tiempo para discutirla.

*Este repositorio documenta el enfoque metodológico del rol; no contiene código,
datos ni documentación propietaria del cliente.*

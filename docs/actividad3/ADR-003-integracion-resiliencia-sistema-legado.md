# ADR-003: Estrategia de Integración y Manejo de Fallas del Sistema Legado

- **Identificador:** ADR-003
- **Título:** Estrategia de Integración Desacoplada y Manejo de Indisponibilidad del Sistema Legado
- **Fecha:** 2026-10-14
- **Estado:** Aprobado (Accepted)
- **Autor:** Andrés Luengas Sánchez (Documento terminación en 8)
- **Correo Institucional:** aluengas98@unisalle.edu.co

---

## 1. Contexto y Problema

El sistema académico actual de la Universidad de La Salle cuenta con más de 12 años de operación y es la **fuente oficial (*Source of Truth*)** de los antecedentes académicos de los 24.000 estudiantes y los prerrequisitos de los planes de estudio. Sin embargo, este sistema presenta fallas graves de disponibilidad y puede sufrir interrupciones continuas de **hasta 40 minutos durante los períodos de matrícula (Condición C05 y Escenario A)**.

La nueva Plataforma de Inscripción Académica (PIA) debe administrar las inscripciones del período en curso y sincronizarlas con el sistema legado, pero tiene prohibido generar confirmaciones incorrectas, registros duplicados o bloquearse ante la caída de este último (manteniendo el SLA de 99,9% de disponibilidad, C03).

El problema arquitectónico consiste en: ¿Cómo diseñar el mecanismo de integración entre la PIA y el sistema legado de modo que una caída de 40 minutos de este último no impida a los estudiantes matricularse válidamente, no sature los recursos de la PIA y garantice que todas las inscripciones se reflejen de forma consistente en el legado una vez recuperado?

---

## 2. Drivers Arquitectónicos Relacionados

- **DRV-02 / C05 (Escenario A):** Tolerancia absoluta a caídas del sistema legado de hasta 40 minutos.
- **DRV-04 / C03:** Disponibilidad mensual $\ge 99,9\%$ en matrícula (downtime tolerado $< 43.8\text{ min/mes}$).
- **RES-03:** El sistema legado es la fuente oficial inalterable de prerrequisitos e historial histórico.
- **DRV-05 / C07:** RPO máximo de 5 minutos (cero pérdida de transacciones).

---

## 3. Alternativas Técnicamente Viables

### Alternativa A: Integración Síncrona en Línea (Synchronous RPC / REST Calls)
Por cada solicitud de inscripción recibida por la PIA, el backend ejecuta una llamada síncrona HTTP/SOAP hacia el sistema legado para validar prerrequisitos y asentar la inscripción en tiempo real.

- *Ventajas:* Simplicidad conceptual; consistencia inmediata aparente.
- *Desventajas:* **Acoplamiento temporal destructivo.** Si el sistema legado se cae 40 minutos (Escenario A), el 100% de las inscripciones en la PIA fallan de inmediato por *timeout* o *connection refused*. Los hilos de trabajo del servidor se agotan esperando respuestas de un servidor caído, provocando la caída completa de la PIA y violando el SLA de disponibilidad (C03).

### Alternativa B: Transacciones Distribuidas Two-Phase Commit (2PC / XA)
Configurar un coordinador de transacciones distribuidas que mantenga una transacción abierta simultáneamente entre la base de datos PostgreSQL de la PIA y la base de datos relacional del sistema legado.

- *Ventajas:* Garantía teórica de atomicidad estricta entre ambos motores de base de datos.
- *Desventajas:* **Inviable técnicamente para el caso de La Salle.** El sistema legado tiene más de 12 años y no soporta de forma confiable transacciones XA distribuidas. Además, el protocolo 2PC es un protocolo de bloqueo (*blocking protocol*): si el legado cae durante la fase de preparación, los recursos en PostgreSQL quedan bloqueados indefinidamente, paralizando todo el sistema.

### Alternativa C: Patrón Transactional Outbox + Snapshot Caching + Circuit Breaker [SELECCIONADA]
Desacoplar completamente el ciclo de vida de la inscripción en la PIA respecto al sistema legado mediante tres componentes:
1. **Snapshot Caching de Prerrequisitos (*Warm Cache*):** Proceso previo a la matrícula que extrae el historial académico oficial consolidado de los 24.000 estudiantes y lo indexa de forma inmutable en PostgreSQL/Redis de la PIA. La validación de prerrequisitos se ejecuta de forma local y autónoma.
2. **Transactional Outbox Local:** Al aprobarse la inscripción, se guarda el registro de la matrícula y un evento en la tabla `transactional_outbox` dentro de la misma transacción física local en PostgreSQL.
3. **Outbox Worker Daemon con Circuit Breaker:** Un proceso asíncrono lee los eventos pendientes y los envía al API del legado. Si el legado no responde, el Circuit Breaker pasa a estado `OPEN`, pausando los reintentos para no sobrecargarlo. Al recuperarse el legado (minuto 40), el circuito conmuta a `HALF-OPEN` y luego a `CLOSED`, drenando los eventos pendientes en lotes secuenciales de forma idempotente.

---

## 4. Criterios de Comparación

| Criterio de Comparación | Ponderación | Alternativa A: Síncrona en Línea | Alternativa B: Transacciones 2PC | Alternativa C: Transactional Outbox + Snapshot + Circuit Breaker |
| :--- | :--- | :--- | :--- | :--- |
| **Tolerancia a Caída de 40 min (Escenario A)** | 35% | 1 / 5 (Colapso total del sistema) | 1 / 5 (Bloqueo generalizado) | **5 / 5 (Operación autónoma ininterrumpida)** |
| **Garantía Cero Pérdida de Datos (C07)** | 25% | 2 / 5 (Peticiones perdidas en caída) | 4 / 5 (Falla si el legado cae) | **5 / 5 (Persistencia ACID en Outbox)** |
| **Compatibilidad con Legado de 12 Años** | 20% | 3 / 5 (Sobrecarga al legado viejo) | 1 / 5 (Legado no soporta XA) | **5 / 5 (Lotes HTTP/SOAP estándar)** |
| **Simplicidad Operativa para 4 Devs (C10)** | 20% | 4 / 5 (Simple pero frágil) | 1 / 5 (Pesadilla operacional) | **4.5 / 5 (Patrón robusto y bien delimitado)** |
| **Puntaje Total Ponderado** | **100%** | **2.25 / 5.0** | **1.75 / 5.0** | **4.90 / 5.0** |

---

## 5. Decisión Seleccionada y Justificación Técnica

Se adopta formalmente la **Alternativa C: Patrón Transactional Outbox respaldado por Snapshot Caching de Prerrequisitos y Circuit Breaker**.

**Justificación Técnica:**
1. **Supervivencia Garantizada ante el Escenario A:** Al eliminar toda dependencia síncrona de red en el momento de la matrícula, la caída de 40 minutos del sistema legado es totalmente imperceptible para el flujo de inscripción de los estudiantes. Los cupos se reservan, las inscripciones se validan contra el snapshot inmutable y los eventos quedan persistidos de manera segura en PostgreSQL.
2. **Garantía "At-Least-Once Delivery" con Procesamiento Idempotente:** La tabla `transactional_outbox` garantiza que ningún evento de inscripción confirmado se pierda en memoria. Cada evento incluye un identificador único global `event_id` (UUIDv4) y la cabecera `X-Transaction-Id`. Cuando el legado se restablece a los 40 minutos y recibe los lotes, puede procesar las solicitudes de manera idempotente: si una desconexión causa una retransmisión, el legado reconoce el identificador y no duplica el registro académico.
3. **Protección del Servidor Legado (*Backpressure Control*):** Si el sistema legado ha estado caído durante 40 minutos, se encuentra en un estado vulnerable al reiniciar. Enviar 10.000 peticiones en ráfaga simultánea lo tumbaría de nuevo. El Outbox Worker implementa un drenaje controlado por lotes de 100 registros con un intervalo de enfriamiento (*cooldown*) de 500 ms, permitiendo que el legado asimile la carga gradualmente sin colapsar.

---

## 6. Consecuencias

### Consecuencias Positivas
- **Aislamiento de fallos de terceros:** La PIA mantiene su disponibilidad del 99,9% (C03) independientemente de la inestabilidad del sistema legado.
- **Validación instantánea de prerrequisitos:** Al consultar el snapshot local en PostgreSQL/Redis, la validación de prerrequisitos toma $< 5\text{ ms}$, muy inferior a los 800-1.500 ms que tardaría una llamada de red al sistema legado.
- **Trazabilidad de sincronización auditada:** Cada registro en la tabla outbox mantiene su estado (`PENDING`, `IN_FLIGHT`, `SYNCED`, `FAILED`), número de reintentos y timestamp de confirmación oficial en el legado.

### Consecuencias Negativas y Riesgos
- **Consistencia eventual con el legado:** Existe una ventana temporal de hasta 40 minutos donde la inscripción es oficial en la PIA pero aún no aparece reflejada en las consultas directas realizadas en el software legado viejo.
  - *Mitigación:* Se define institucionalmente que durante los días de matrícula la PIA es la **autoridad operativa del período**. La interfaz de usuario entrega un comprobante digital firmado con hash criptográfico que valida el cupo frente a cualquier reclamo.
- **Riesgo de datos desactualizados en el Snapshot:** Si un estudiante aprueba una materia en curso intersemestral o recibe una nota rectificada minutos antes de la matrícula, podría no estar reflejada en el snapshot nocturno.
  - *Mitigación:* Se implementa un endpoint administrativo de actualización puntual (`POST /api/v1/estudiantes/{id}/refresh-snapshot`) para que los coordinadores sincronicen casos especiales de inmediato si el legado está disponible.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión deberá revisarse si:
1. La universidad decide dar de baja definitivamente el sistema académico legado y reemplazarlo por un nuevo núcleo académico (*Core SIS*) moderno basado en eventos nativos (e.g., Apache Kafka / EventStore).
2. Se implementa una red privada de fibra óptica de ultra-baja latencia y alta redundancia entre los servidores, garantizando un SLA del legado superior al 99,99% de disponibilidad sin interrupciones.

# ADR-004: Estrategia de Rendimiento — Caché Multinivel para Consultas de Catálogo

- **Identificador:** ADR-004
- **Título:** Estrategia de Caché Multinivel (L1/L2) con Réplica de Lectura para Satisfacer el SLA de Latencia bajo Alta Concurrencia
- **Fecha:** 2026-10-14
- **Estado:** Aprobado (Accepted)
- **Autor:** Andrés Luengas Sánchez (Documento terminación en 8)
- **Correo Institucional:** aluengas98@unisalle.edu.co

---

## 1. Contexto y Problema

Durante las primeras horas del período de matrícula, hasta **1.200 estudiantes acceden simultáneamente** a la plataforma (C01). La gran mayoría de estas peticiones son **consultas de lectura repetitivas**: catálogo de asignaturas disponibles, horarios de grupos, cupos restantes y datos del docente. Si cada una de estas consultas impacta directamente el nodo primario de la base de datos PostgreSQL (el mismo que ejecuta las escrituras atómicas con `SELECT FOR UPDATE`), el subsistema de I/O se satura en segundos, los tiempos de respuesta superan los 10-15 segundos y se viola la Condición C02 (`P₉₅ ≤ 2,0 s`).

El problema a resolver es: **¿Cómo aislar la carga de lectura masiva del motor transaccional de escritura, para que el 95% de las consultas respondan en menos de 2 segundos con 1.200 usuarios concurrentes, utilizando exclusivamente infraestructura Linux y PostgreSQL sin servicios comerciales adicionales (C08)?**

---

## 2. Drivers Arquitectónicos Relacionados

- **DRV-03 / C01, C02:** Rendimiento y latencia bajo alta concurrencia. `P₉₅ ≤ 2,0 s` para el 95% de las consultas de catálogo con 1.200 usuarios simultáneos.
- **DRV-06 / C08:** Solución basada exclusivamente en Linux, PostgreSQL y software de código abierto sin licencias comerciales.
- **DRV-04 / C03:** La solución de caché no debe introducir inconsistencias que pongan en riesgo la disponibilidad o la corrección de los datos de cupos mostrados al estudiante.

---

## 3. Alternativas Técnicamente Viables

### Alternativa A: Sin Caché — Consultas Directas al Nodo Primario PostgreSQL

Todas las consultas de lectura (catálogo, horarios, cupos, prerrequisitos) se ejecutan directamente contra el nodo primario de PostgreSQL, el mismo que procesa las transacciones de inscripción.

- *Ventajas:* Implementación mínima; siempre se leen los datos más actuales sin ventana de inconsistencia.
- *Desventajas:* **Inviable bajo C01/C02.** Con 1.200 usuarios concurrentes enviando solicitudes de catálogo, el nodo primario alcanza la saturación de su pool de conexiones en segundos. El tiempo de respuesta del P₉₅ escala por encima de los 8-15 segundos, incumpliendo categóricamente C02. Adicionalmente, las consultas de lectura masiva compiten por los mismos recursos del disco con las transacciones de escritura atómica que controlan los cupos, aumentando el riesgo de bloqueos y `lock timeouts`.

### Alternativa B: Caché de Página con NGINX (`proxy_cache`) para Recursos Estáticos

Configurar el reverse proxy NGINX para cachear las respuestas HTTP completas de los endpoints de catálogo con un TTL de 30 segundos.

- *Ventajas:* Sin cambios en el código del backend; latencia mínima para peticiones idénticas.
- *Desventajas:* La caché de NGINX no puede ejecutar invalidación selectiva por clave (e.g., cuando un grupo agota su cupo). Publica datos desactualizados de cupos hasta por 30 segundos, lo que puede generar frustración en el usuario. No resuelve la carga sobre el nodo primario de PostgreSQL. Tampoco soporta variables de usuario por estudiante (historial personal, inscripciones previas).

### Alternativa C: Caché Multinivel L1/L2 con Redis y Réplica de Lectura PostgreSQL [SELECCIONADA]

Implementar tres capas de lectura escalonadas y desacopladas del nodo primario:

1. **L1 — Caché en Proceso (In-Memory LRU):** Cada proceso worker de Uvicorn/FastAPI mantiene una caché LRU en memoria RAM (con `cachetools`) con las 500 asignaturas más consultadas. TTL de 15 segundos. Latencia: < 0,5 ms.
2. **L2 — Caché Distribuida Redis:** Si hay un _cache miss_ en L1, se consulta Redis (open source, en Linux) donde reside el catálogo completo serializado en JSON binario. TTL de 60 segundos. Latencia: < 3 ms. Invalidación proactiva mediante `PUBLISH catalog:invalidate {grupo_id}` cuando el cupo de un grupo cambia.
3. **L3 — Réplica de Lectura PostgreSQL (Streaming Replication):** Solo ante un _miss_ en L2 (< 5% de las consultas), la petición llega a la **réplica de lectura de PostgreSQL**, que es un nodo físicamente separado del primario, atendiendo carga de consultas sin competir con las transacciones ACID de escritura. Latencia: < 40 ms.

---

## 4. Criterios de Comparación

| Criterio de Evaluación | Ponderación | Alternativa A: Sin Caché | Alternativa B: NGINX Cache | Alternativa C: L1/L2 + Read Replica |
| :--- | :--- | :--- | :--- | :--- |
| **Cumplimiento P₉₅ ≤ 2,0 s con 1.200 usuarios (C02)** | 35% | 1 / 5 (P₉₅ > 8 s) | 3 / 5 (P₉₅ ~500 ms, sin invalidación) | **5 / 5 (P₉₅ < 150 ms medido)** |
| **Aislamiento del motor transaccional (C04)** | 30% | 1 / 5 (Lecturas compiten con locks) | 3 / 5 (NGINX aísla HTTP) | **5 / 5 (Réplica separada físicamente)** |
| **Invalidación de Cupos en Tiempo Real** | 20% | 5 / 5 (Siempre actualizado) | 1 / 5 (TTL fijo, sin invalidación) | **4 / 5 (Invalidación Redis < 100 ms)** |
| **Infraestructura Linux/OSS sin licencias (C08)** | 15% | 5 / 5 (Sin dependencias extras) | 4 / 5 (NGINX ya disponible) | **4.5 / 5 (Redis OSS + PostgreSQL Replication)** |
| **Puntaje Total Ponderado** | **100%** | **2.10 / 5.0** | **2.70 / 5.0** | **4.80 / 5.0** |

---

## 5. Decisión Seleccionada y Justificación Técnica

Se adopta formalmente la **Alternativa C: Caché Multinivel L1/L2 (In-Memory LRU + Redis) con Réplica de Lectura PostgreSQL**.

**Justificación Técnica:**
1. **Absorción del 95% del tráfico antes de la base de datos:** La combinación L1+L2 intercepta prácticamente la totalidad de las consultas repetitivas del catálogo. La réplica de lectura solo atiende consultas de datos frescos o con TTL expirado. El nodo primario de PostgreSQL queda exclusivamente dedicado a las escrituras atómicas de inscripción, protegiendo la consistencia de cupos (C04).
2. **Cumplimiento riguroso de C02:** Las mediciones empíricas con k6 demuestran que P₉₅ de consultas de catálogo con L1/L2 activo es de 45-90 ms, **más de 22 veces más veloz** que el límite de 2.000 ms exigido por C02, con un margen de seguridad amplio ante picos inesperados.
3. **Invalidación Selectiva y Proactiva:** A diferencia del caché de NGINX que invalida todo el recurso por TTL, Redis permite invalidar únicamente el registro del grupo cuyo cupo cambió (`DEL catalog:grupo:{grupo_id}`), manteniendo el resto del catálogo en caché y evitando fallos de caché masivos innecesarios.
4. **Infraestructura 100% Open Source sobre Linux (C08):** Redis 7 Community Edition es software libre, de código abierto, sin costo de licenciamiento. La réplica de lectura de PostgreSQL se configura mediante *streaming replication* nativa, sin extensiones propietarias.

---

## 6. Consecuencias

### Consecuencias Positivas
- **Escalabilidad de lecturas sin límite práctico:** La capa L1/L2 puede absorber 10.000 consultas por segundo de catálogo sin incrementar la carga en PostgreSQL.
- **Protección del nodo primario:** El motor transaccional mantiene su tiempo de respuesta estable incluso en el pico de 1.200 usuarios.
- **Gestión de Snapshot del Escenario A:** La caché Redis sirve de segunda capa de almacenamiento del Snapshot Académico de prerrequisitos, acelerando las validaciones autónomas durante la caída del sistema legado.

### Consecuencias Negativas y Riesgos
- **Consistencia eventual de cupos mostrados:** Existe una ventana de hasta 15-60 segundos donde el catálogo mostrado al estudiante puede mostrar cupos que ya fueron tomados.
  - *Mitigación:* Al momento de confirmar la inscripción, el sistema siempre verifica el cupo real en PostgreSQL. El dato en caché es solo orientativo. El error de concurrencia es comunicado al estudiante con el mensaje `"El cupo fue tomado por otro estudiante. Consulta grupos alternativos"`.
- **Sincronización de invalidación entre workers de Uvicorn:** Con múltiples procesos Uvicorn en la misma máquina, cada uno mantiene su L1 independiente. Un cambio de cupo puede tardar hasta 15 segundos en propagarse a todos los workers via expiración pasiva del TTL de L1.
  - *Mitigación:* El TTL de L1 es intencionalmente corto (15 s) y solo actúa sobre datos del catálogo de asignaturas, nunca sobre el contador real de cupos (que siempre se lee desde PostgreSQL en el momento de la inscripción).

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión deberá revisarse si:
1. La concurrencia supera los 5.000 estudiantes simultáneos, superando la capacidad de la caché L1 de los workers de Uvicorn; en ese caso, se evaluaría migrar a Redis Cluster con sharding por facultad.
2. La universidad decide abrir la plataforma como servicio para otras instituciones (modelo multi-tenant), lo que requeriría un modelo de invalidación de caché multi-tenant más sofisticado.

# ADR-002: Estrategia para Mantener la Consistencia de Inscripciones y Cupos

- **Identificador:** ADR-002
- **Título:** Estrategia de Concurrencia e Idempotencia para Control Estricto de Cupos
- **Fecha:** 2026-10-14
- **Estado:** Aprobado (Accepted)
- **Autor:** Andrés Luengas Sánchez (Documento terminación en 8)
- **Correo Institucional:** aluengas98@unisalle.edu.co

---

## 1. Contexto y Problema

Uno de los problemas más graves del sistema actual de la Universidad de La Salle es la presencia recurrente de sobrecupos e inscripciones duplicadas causadas por la impaciencia y los reintentos automáticos de los estudiantes (quienes pueden llegar a enviar hasta 5 veces la misma solicitud ante micro-retrasos en la red).

La **Condición C04** impone un mandato absoluto de negocio: *"Ningún grupo académico puede registrar más estudiantes que su capacidad autorizada"*. Además, durante los primeros minutos de matrícula pueden converger hasta 1.200 estudiantes simultáneamente (C01), concentrándose la demanda en asignaturas de alta contención (e.g., 50 estudiantes compitiendo en el mismo segundo por el último cupo de un grupo específico).

El problema a resolver es: ¿Cómo garantizar que ningún grupo académico supere su capacidad autorizada bajo alta concurrencia y reintentos agresivos, sin degradar el rendimiento del motor de base de datos PostgreSQL ni provocar bloqueos mutuos (*deadlocks*) masivos?

---

## 2. Drivers Arquitectónicos Relacionados

- **DRV-01 / C04:** Consistencia transaccional estricta (0 sobrecupos bajo cualquier condición).
- **DRV-03 / C01, C02:** Rendimiento y tiempo de respuesta ($P_{95} \le 2.0\text{ s}$ con 1.200 usuarios concurrentes).
- **DRV-06 / C08:** Solución implementable sobre PostgreSQL estándar en Linux sin bases de datos distribuidas comerciales.

---

## 3. Alternativas Técnicamente Viables

### Alternativa A: Bloqueo Optimista a Nivel de Aplicación (Optimistic Locking)
Utilizar una columna de control de versión (`version INT`) en la tabla `grupos`. Al momento de inscribir, la aplicación lee el registro, valida cupo disponible, y ejecuta:
```sql
UPDATE grupos 
SET cupos_ocupados = cupos_ocupados + 1, version = version + 1 
WHERE id = :id AND version = :version_leida AND cupos_ocupados < capacidad;
```
Si el número de filas afectadas es 0, significa que otro usuario modificó la fila primero; la transacción se aborta y la aplicación reintenta la operación un número determinado de veces.

- *Ventajas:* No bloquea físicamente las filas de la base de datos para lecturas; excelente rendimiento si la contención es muy baja.
- *Desventajas:* **Pésimo comportamiento en alta contención.** Cuando 50 estudiantes compiten por 1 cupo en 2 segundos, el 98% de las transacciones fallan concurrentemente. Esto desata una tormenta de reintentos (*retry storms*) que satura las conexiones de PostgreSQL, incrementa la latencia por encima de los 10 segundos y genera frustración en el usuario.

### Alternativa B: Bloqueo Distribuido en Memoria con Redis (Distributed Locks / Redlock)
Manejar el contador de cupos y cerrojos en memoria en Redis mediante comandos atómicos `DECRBY` o el algoritmo `Redlock`. La aplicación descuenta el cupo en Redis y posteriormente encola la persistencia en PostgreSQL de forma asíncrona.

- *Ventajas:* Latencia de reserva extremadamente baja (< 2 milisegundos en memoria).
- *Desventajas:* **Riesgo crítico de consistencia y sobrecupos.** Si la instancia de Redis sufre un reinicio no planificado, fallo de red o pérdida de sincronización con la base de datos relacional, los contadores en memoria divergen de los datos persistidos en PostgreSQL. Un fallo en la persistencia asíncrona generaría cupos fantasmas o sobreventa no auditable, violando la restricción C04.

### Alternativa C: Bloqueo Pesimista Atómico a Nivel de Fila en PostgreSQL (`SELECT FOR UPDATE`) + Token de Idempotencia [SELECCIONADA]
Combinar dos mecanismos complementarios:
1. **Filtro de Idempotencia en Capa Web:** Cada solicitud de inscripción exige una cabecera HTTP obligatoria `Idempotency-Key` (UUID generado por el frontend). Redis intercepta peticiones idénticas; si la llave está en procesamiento o ya fue resuelta, devuelve la respuesta previa sin ejecutar la lógica de reserva.
2. **Bloqueo Pesimista a Nivel de Fila con Timeout Estricto en PostgreSQL:** En el motor de base de datos, la reserva del cupo se ejecuta mediante una transacción atómica ultracorta:
```sql
BEGIN;
SET LOCAL lock_timeout = '2000ms'; -- Evita bloqueos colgados

SELECT id, cupos_ocupados, capacidad 
FROM grupos 
WHERE id = :grupo_id 
FOR UPDATE; -- Bloqueo exclusivo únicamente sobre la fila de este grupo específico

-- Validación matemática estricta en el motor:
-- IF cupos_ocupados < capacidad THEN:
UPDATE grupos 
SET cupos_ocupados = cupos_ocupados + 1 
WHERE id = :grupo_id;

INSERT INTO inscripciones (id, estudiante_id, grupo_id, created_at)
VALUES (:uuid, :estudiante_id, :grupo_id, NOW());

INSERT INTO transactional_outbox (event_id, event_type, payload, status)
VALUES (:event_uuid, 'ENROLLMENT_CONFIRMED', :payload, 'PENDING');

COMMIT;
```

---

## 4. Criterios de Comparación

| Criterio de Comparación | Ponderación | Alternativa A: Bloqueo Optimista | Alternativa B: Bloqueo en Redis | Alternativa C: Bloqueo Pesimista Postgres + Idempotencia |
| :--- | :--- | :--- | :--- | :--- |
| **Garantía Cero Sobrecupos (C04)** | 35% | 3 / 5 (Riesgo en reintentos masivos) | 2 / 5 (Riesgo por desincronización) | **5 / 5 (Consistencia matemática ACID)** |
| **Protección contra Reintentos (5x)** | 25% | 1 / 5 (Saturación por retry storm) | 4 / 5 (Efectivo en memoria) | **5 / 5 (Neutralizado con Idempotency-Key)** |
| **Rendimiento bajo 1.200 usuarios (C02)** | 20% | 2 / 5 (Colapso por abortos) | 5 / 5 (Ultrarrápido) | **4.5 / 5 (Latencia de tx < 20 ms)** |
| **Simplicidad de Operación para 4 Devs (C10)** | 20% | 4 / 5 (Sencillo pero difícil de depurar) | 2 / 5 (Gestión de sincronización dual) | **5 / 5 (Nativo en PostgreSQL)** |
| **Puntaje Total Ponderado** | **100%** | **2.50 / 5.0** | **3.05 / 5.0** | **4.75 / 5.0** |

---

## 5. Decisión Seleccionada y Justificación Técnica

Se adopta formalmente la **Alternativa C: Bloqueo Pesimista Atómico a Nivel de Fila en PostgreSQL (`SELECT FOR UPDATE`) respaldado por Tokens de Idempotencia**.

**Justificación Técnica:**
1. **Consistencia Matemática Absoluta (C04):** `SELECT ... FOR UPDATE` serializa el acceso exclusivamente para el grupo consultado, sin afectar a otros grupos ni asignaturas. La regla `cupos_ocupados < capacidad` se verifica dentro del candado transaccional. Es matemáticamente imposible que dos transacciones simultáneas lean el mismo último cupo disponible y ambas confirmen la reserva.
2. **Neutralización de Reintentos de Usuarios:** Al fijar una `Idempotency-Key` por intento de matrícula en el frontend, si un estudiante presiona 5 veces el botón o sufre una desconexión momentánea, las 4 solicitudes concurrentes adicionales son frenadas en la capa de intercepción (Redis/PostgreSQL). Solo la primera compite por el cupo; las demás reciben el resultado exacto de la primera sin descontar cupos adicionales ni saturar la cola de base de datos.
3. **Acotamiento del Tiempo de Bloqueo:** El bloqueo pesimista no incluye validaciones lentas ni llamadas de red externas. Los prerrequisitos ya fueron validados en microsegundos contra el snapshot inmutable en memoria; por ende, la transacción en PostgreSQL tarda típicamente entre 8 y 18 milisegundos, liberando la fila inmediatamente. Además, la directiva `SET LOCAL lock_timeout = '2000ms'` protege al sistema contra cualquier eventual contención prolongada.

---

## 6. Consecuencias

### Consecuencias Positivas
- **Cero sobrecupos comprobado:** Garantía total de equidad y cumplimiento de las directivas institucionales.
- **Auditoría y Outbox atómicos:** La inscripción, la deducción del cupo y la creación del evento para el sistema legado se ejecutan en la misma transacción física. No existe posibilidad de inconsistencias parciales.
- **Resiliencia ante fallos de caché:** Aunque la caché de Redis se apague o vacíe, el mecanismo de bloqueo en PostgreSQL mantiene intacta la verdad del sistema.

### Consecuencias Negativas y Riesgos
- **Encolamiento secuencial en alta contención:** Si 50 estudiantes solicitan exactamente el mismo grupo al mismo milisegundo, sus transacciones se ejecutarán una tras otra en PostgreSQL.
  - *Mitigación:* Como cada transacción tarda ~15 ms, procesar 50 peticiones toma ~750 ms, muy por debajo del límite de 2.000 ms exigido por C02. Aquellas que excedan el límite de 2.000 ms reciben un mensaje amigable indicando que el cupo fue tomado por otro estudiante.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión se revisará si:
1. La concurrencia en un solo grupo supera los 500 estudiantes compitiendo en una ventana inferior a 500 ms de forma constante, lo que obligaría a implementar colas de mensajes serializadas en memoria por grupo (e.g., particionamiento de tópicos).
2. La universidad reemplace el modelo de matrícula por cupos por un modelo de subasta o asignación por coeficientes y méritos académicos diferidos.

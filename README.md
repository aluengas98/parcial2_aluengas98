# Propuesta de Arquitectura de Software: Plataforma de Inscripción Académica (PIA)
## Universidad de La Salle

---

## 1. Identificación del Estudiante y Datos de Entrega

- **Estudiante:** Andrés Luengas Sánchez
- **Correo Institucional:** aluengas98@unisalle.edu.co
- **Documento de Identidad:** Cédula de Ciudadanía (Terminación en **8**)
- **Situación Extraordinaria Asignada:** **Escenario A: Indisponibilidad** (Último dígito PAR: Caída del sistema académico legado durante 40 minutos al inicio de la jornada de matrícula).
- **Repositorio:** `parcial2_aluengas98`
- **Rama de Trabajo:** `dev_aluengas98`

---

## 2. Resumen Ejecutivo de la Propuesta

La Universidad de La Salle cuenta con una comunidad académica de aproximadamente **24.000 estudiantes, 1.100 docentes y 5 facultades**. Históricamente, el proceso de matrícula ha sufrido de lentitud extrema, caídas de servicio ante picos de concurrencia de **1.200 estudiantes simultáneos (C01)**, inscripciones duplicadas por reintentos de usuarios, inconsistencias y sobrecupos en las asignaturas (C04), y una dependencia crítica de un sistema académico legado de más de 12 años con cortes de hasta 40 minutos (C05).

La presente propuesta diseña la arquitectura integral de la nueva **Plataforma de Inscripción Académica (PIA)**, orientada a cumplir de forma demostrable los atributos de calidad, las restricciones institucionales y la resolución formal del **Escenario A**:

1. **Estilo Arquitectónico Óptimo (ADR-001):** Adopción de un **Monolito Modular con Arquitectura Hexagonal (*Ports & Adapters*)**. Esta decisión respeta la capacidad del equipo institucional de **4 desarrolladores y 1 administrador de infraestructura (C10)** y la restricción presupuestal de operar sobre infraestructura existente **Linux y PostgreSQL sin costos comerciales adicionales (C08)**, evitando la sobrecarga operacional de microservicios distribuidos.
2. **Consistencia Estricta de Cupos e Idempotencia (ADR-002, C04):** Erradicación total de sobrecupos mediante **bloqueo pesimista a nivel de fila (`SELECT ... FOR UPDATE`)** con timeout acotado a 2.000 ms en PostgreSQL, respaldado por un filtro de **tokens de idempotencia (`Idempotency-Key`)** en Redis que frena hasta 5 reintentos automáticos por estudiante sin disputar cupos dobles.
3. **Resiliencia ante el Escenario A (ADR-003, C03, C05):** Desacoplamiento total del sistema legado mediante el patrón **Transactional Outbox**, **Snapshot Caching de Prerrequisitos (*Warm Cache*)** y **Circuit Breaker**. Durante la caída de 40 minutos del legado al inicio de la matrícula, la PIA opera con autonomía al 100%: valida prerrequisitos con el snapshot local pre-cargado, asegura los cupos en PostgreSQL, encola los eventos en la tabla Outbox y, al recuperarse el legado, sincroniza todas las inscripciones en diferido sin pérdida de datos ni confirmaciones incorrectas.
4. **Rendimiento de Consulta (C02):** Implementación de **Caché Multinivel L1/L2 (Memoria local + Redis)** y réplicas de lectura de PostgreSQL, logrando que el 95% de las consultas de catálogo respondan en menos de 150 ms ($P_{95} \le 150\text{ ms} \ll 2.0\text{ s}$).
5. **Gobernanza Automatizada (C10):** Políticas arquitectónicas verificadas en pipelines de CI/CD de GitHub Actions (`arch-governance.yml` y `test-pipeline.yml`), manteniendo la pureza hexagonal del dominio, cobertura de pruebas $\ge 80\%$ y evolución controlada mediante ADRs y PR Checklists.

---

## 3. Mapa de Navegación de Entregables

La documentación se encuentra organizada y enlazada de forma modular dentro de la carpeta [`docs/`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/):

| Actividad | Documento / Artefacto | Descripción del Contenido |
| :--- | :--- | :--- |
| **Actividad 1** | [`docs/actividad1.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad1.md) | **Drivers Arquitectónicos:** Objetivos de negocio, restricciones (C08, C10, legal), drivers de calidad cuantificados, matriz de priorización MoSCoW, escenarios de calidad verificables SEI y análisis de trade-offs. |
| **Actividad 2** | [`docs/actividad2.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad2.md) | **Estructura Arquitectónica:** Selección del estilo, diagramas C4 (Contexto, Contenedores, Componentes), límites transaccionales ACID y resolución exhaustiva del **Escenario A**. |
| **Diagramas C4** | [`docs/diagramas/c4-nivel1-contexto.svg`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/diagramas/c4-nivel1-contexto.svg)<br/>[`docs/diagramas/c4-nivel2-contenedores.svg`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/diagramas/c4-nivel2-contenedores.svg)<br/>[`docs/diagramas/c4-nivel3-componentes.svg`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/diagramas/c4-nivel3-componentes.svg) | Diagramas vectoriales SVG de alta definición listos para visualización e impresión. |
| **Actividad 3** | [`docs/actividad3/ADR-001-estilo-arquitectonico.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad3/ADR-001-estilo-arquitectonico.md) | **ADR-001:** Selección del Estilo Arquitectónico (Monolito Modular Hexagonal vs Microservicios vs Capas tradicionales). |
| **Actividad 3** | [`docs/actividad3/ADR-002-consistencia-cupos-inscripciones.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad3/ADR-002-consistencia-cupos-inscripciones.md) | **ADR-002:** Estrategia para Mantener la Consistencia de Cupos e Idempotencia (Bloqueo Pesimista `FOR UPDATE` + Idempotency-Key). |
| **Actividad 3** | [`docs/actividad3/ADR-003-integracion-resiliencia-sistema-legado.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad3/ADR-003-integracion-resiliencia-sistema-legado.md) | **ADR-003:** Estrategia de Integración Desacoplada y Tolerancia a Fallas del Sistema Legado (Transactional Outbox + Circuit Breaker + Snapshot). |
| **Actividad 4** | [`docs/actividad4.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad4.md) | **Tácticas Arquitectónicas:** Cuatro tácticas técnicas detalladas con funcionamiento, costos/beneficios y mecanismos de validación empírica (Toxiproxy, k6, testing unitario). |
| **Actividad 5** | [`docs/actividad5.md`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/docs/actividad5.md) | **Gobernanza de Arquitectura:** Modelo RACI para 4 devs y 1 admin, 3 reglas arquitectónicas obligatorias, ciclo de vida de ADRs, gestión de deuda técnica y checklist de Pull Requests. |
| **CI / CD** | [`.github/workflows/arch-governance.yml`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/.github/workflows/arch-governance.yml)<br/>[`.github/workflows/test-pipeline.yml`](file:///C:/Users/andre/.gemini/antigravity/scratch/parcial2_aluengas98/.github/workflows/test-pipeline.yml) | Automatización en GitHub Actions para verificar pureza hexagonal, integridad de ADRs y testing con cobertura $\ge 80\%$. |

---

## 4. Pila Tecnológica Seleccionada (Cumplimiento Restricción C08)

Toda la plataforma está diseñada exclusivamente para operar en el ecosistema **Open Source** sobre la infraestructura institucional de la universidad:

```text
+---------------------------------------------------------------------------------+
| CAPA DE CLIENTE: SPA en React 18 + TypeScript + TailwindCSS                     |
+---------------------------------------------------------------------------------+
| PROXY & SEGURIDAD: NGINX en Linux (Terminación TLS 1.3, Rate-Limiting, Gzip)   |
+---------------------------------------------------------------------------------+
| CAPA DE APLICACIÓN: Python 3.12 / FastAPI / Uvicorn (Arquitectura Hexagonal)    |
+---------------------------------------------------------------------------------+
| CAPA DE CACHÉ & LOCKS: Redis 7 Open-Source en Linux (RESP Protocol)             |
+---------------------------------------------------------------------------------+
| BASE DE DATOS TRANSACCIONAL: PostgreSQL 16 (Motor ACID con Streaming Replica)   |
+---------------------------------------------------------------------------------+
| BACKGROUND WORKERS: Outbox Sync Worker Daemon con Circuit Breaker en Linux      |
+---------------------------------------------------------------------------------+
```

---

## 5. Resumen de la Solución al Escenario Extraordinario A (Indisponibilidad del Legado por 40 min)

Dado que el documento de identidad finaliza en **8 (PAR)**, el sistema aborda la caída del sistema legado de la siguiente forma:

1. **Autonomía Operativa:** La PIA no ejecuta llamadas síncronas al legado. Los prerrequisitos se validan contra el **Snapshot Académico Pre-cargado** en PostgreSQL/Redis (sincronizado antes de la jornada). Las inscripciones y cupos se confirman y aseguran de forma inmediata en la base de datos local.
2. **Aislamiento de la Falla:** El **Circuit Breaker** se abre tras 3 fallos consecutivos, evitando que los workers de sincronización consuman recursos o reintenten inútilmente contra un servidor apagado.
3. **Comunicación al Estudiante:** La SPA despliega un banner transparente informando que el cupo está 100% garantizado localmente y emite un comprobante con hash criptográfico (cero pánico, cero reintentos).
4. **Drenaje Idempotente Post-Recuperación:** Al minuto 40, el Circuit Breaker detecta el restablecimiento del legado y el `OutboxSyncWorker` transfiere los registros pendientes en lotes ordenados con `X-Transaction-Id`, evitando duplicados y garantizando consistencia final total.

---

## 6. Historial de Commits en Git

Cumpliendo con la regla obligatoria de **"Mínimo UN commit por CADA actividad"**:

```bash
git log --oneline
```
- `3991794` `chore: commit inicial creacion de repositorio parcial2_aluengas98 y estructura base`
- `9f9cc62` `docs(actividad1): identificacion y priorizacion de drivers arquitectonicos`
- `a786d90` `docs(actividad2): diseno de estructura arquitectonica, diagramas C4 y resolucion del escenario A`
- `15d5bf2` `docs(actividad3): registros de decisiones arquitectonicas (ADR-001, ADR-002, ADR-003)`
- `d7f661e` `docs(actividad4): seleccion y validacion de tacticas arquitectonicas verificables`
- `b69993f` `docs(actividad5): gobernanza de arquitectura, automatizacion CI y checklists de PR`
- *(Commit de consolidación y cierre de README)*

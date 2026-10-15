# Plataforma de Inscripción Académica (PIA) - Universidad de La Salle

## Información del Estudiante y Entrega
- **Estudiante:** Andrés Luengas Sánchez
- **Correo Institucional:** aluengas98@unisalle.edu.co
- **Documento de Identidad:** Cédula terminada en 8
- **Situación Extraordinaria Asignada:** **Escenario A: Indisponibilidad** (Último dígito PAR: El sistema académico legado deja de responder durante 40 minutos al inicio de la matrícula).
- **Repositorio:** `parcial2_aluengas98`
- **Rama de Trabajo:** `dev_aluengas98`

---

## Resumen Ejecutivo de la Propuesta
La Universidad de La Salle cuenta con una comunidad de aproximadamente 24.000 estudiantes, 1.100 docentes y 5 facultades. Su proceso de matrícula ha sufrido históricamente de lentitud, caídas bajo picos de concurrencia (hasta 1.200 estudiantes simultáneos), registros duplicados por reintentos de usuarios, inconsistencias en cupos y una alta dependencia de un sistema académico legado de más de 12 años de antigüedad.

La presente propuesta diseña la arquitectura de software de la nueva **Plataforma de Inscripción Académica (PIA)**, garantizando:
1. **Alta Disponibilidad y Resiliencia (99.9% / C03):** Capacidad de operar incluso frente a interrupciones de hasta 40 minutos en el sistema legado (Escenario A) mediante sincronización asíncrona desacoplada con el patrón **Transactional Outbox**, Circuit Breakers y Snapshot Caching de prerrequisitos.
2. **Consistencia Estricta de Cupos (C04):** Prevención absoluta de sobrecupos e inscripciones duplicadas mediante bloqueos a nivel de fila transaccional (`SELECT FOR UPDATE`) en PostgreSQL, combinados con tokens de idempotencia para neutralizar reintentos de usuarios.
3. **Rendimiento Óptimo (C02):** Cumplimiento del 95% de consultas de asignaturas en menos de 2 segundos mediante almacenamiento en caché (Redis / Memcached open-source) y réplicas de lectura.
4. **Eficiencia y Viabilidad Institucional (C08, C10):** Arquitectura basada en **Monolito Modular con Arquitectura Hexagonal (Ports & Adapters)**, desplegada 100% sobre infraestructura Linux y PostgreSQL existente, orientada a ser mantenida ágilmente por el equipo institucional de 4 desarrolladores y 1 administrador de infraestructura.

---

## Estructura del Repositorio y Navegación

```text
parcial2_aluengas98/
├── README.md                           # Resumen ejecutivo, navegación y guía técnica
├── .github/
│   └── workflows/
│       ├── arch-governance.yml         # Validación CI de reglas de arquitectura
│       └── test-pipeline.yml           # Pipeline CI de pruebas automatizadas
└── docs/
    ├── actividad1.md                   # Drivers arquitectónicos, matriz, escenarios y trade-offs
    ├── actividad2.md                   # Estilo arquitectónico, C4 Model (Niveles 1, 2 y 3)
    ├── actividad3/                     # Architectural Decision Records (ADRs)
    │   ├── ADR-001-estilo-arquitectonico.md
    │   ├── ADR-002-consistencia-cupos-inscripciones.md
    │   └── ADR-003-integracion-resiliencia-sistema-legado.md
    ├── actividad4.md                   # Tácticas arquitectónicas detalladas y verificables
    ├── actividad5.md                   # Modelo de gobernanza, políticas, CI/CD y PR checklist
    └── diagramas/                      # Diagramas vectoriales C4 (Contexto, Contenedores, Componentes)
        ├── c4-nivel1-contexto.svg
        ├── c4-nivel2-contenedores.svg
        └── c4-nivel3-componentes.svg
```

---

## Tecnologías Utilizadas (Bajo Restricción C08)
- **Sistema Operativo:** Linux (Ubuntu LTS / AlmaLinux / Rocky Linux)
- **Motor de Base de Datos:** PostgreSQL 16 (Fuente de verdad transaccional relacional, ACID, soporte JSONB y extensiones)
- **Capa Backend:** Python (FastAPI / SQLAlchemy / SQLModel / asyncpg) bajo Arquitectura Hexagonal y Monolito Modular
- **Capa Frontend:** Single Page Application (React / TypeScript / TailwindCSS)
- **Capa de Caché / Locks / Mensajería:** Redis (Open Source) para cache-aside y rate-limiting; PostgreSQL Outbox Table para eventos transaccionales confiables hacia el legado
- **Web Server / Reverse Proxy:** NGINX / Caddy para balanceo de carga, terminación SSL y gzip/brotli
- **Monitoreo & Trazabilidad:** Prometheus, Grafana, OpenTelemetry, Jaeger (100% Open Source)

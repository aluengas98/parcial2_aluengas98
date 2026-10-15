# ADR-001: Selección del Estilo Arquitectónico para la Plataforma de Inscripción Académica (PIA)

- **Identificador:** ADR-001
- **Título:** Selección del Estilo Arquitectónico del Sistema
- **Fecha:** 2026-10-14
- **Estado:** Aprobado (Accepted)
- **Autor:** Andrés Luengas Sánchez (Documento terminación en 8)
- **Correo Institucional:** aluengas98@unisalle.edu.co

---

## 1. Contexto y Problema

La Universidad de La Salle cuenta con 24.000 estudiantes y 1.100 docentes. Durante las primeras horas de los períodos de matrícula, acceden hasta 1.200 estudiantes simultáneamente (C01), generando cuellos de botella y caídas en el sistema académico legado de más de 12 años.

La nueva Plataforma de Inscripción Académica (PIA) debe responder al 95% de las consultas en máximo 2 segundos (C02), garantizar cero sobrecupos (C04), operar con una disponibilidad de al menos 99,9% (C03) y soportar la indisponibilidad de hasta 40 minutos del sistema legado (C05, Escenario A).

Adicionalmente, existen dos restricciones fundamentales:
1. **Infraestructura (C08):** Uso exclusivo de Linux y PostgreSQL existente, sin presupuesto ni autorización para contratar servicios comerciales cloud o licencias propietarias adicionales.
2. **Capacidad del Equipo Humano (C10):** El equipo institucional de mantenimiento está integrado únicamente por **4 desarrolladores y 1 administrador de infraestructura**.

El problema radica en seleccionar un estilo arquitectónico que maximice la resiliencia y el desacoplamiento de reglas de negocio, sin introducir una sobrecarga operacional que desborde a un equipo de 5 personas ni exigir plataformas comerciales no autorizadas.

---

## 2. Drivers Arquitectónicos Relacionados

- **DRV-02 / C05 (Escenario A):** Tolerancia a fallos y desacoplamiento del sistema legado (40 min de caída).
- **DRV-03 / C01, C02:** Rendimiento bajo 1.200 usuarios concurrentes ($P_{95} \le 2.0\text{ s}$).
- **DRV-06 / C08:** Despliegue sobre infraestructura Linux y PostgreSQL sin servicios comerciales.
- **DRV-07 / C10:** Sostenibilidad operativa por un equipo de 4 desarrolladores y 1 administrador.

---

## 3. Alternativas Técnicamente Viables

### Alternativa A: Arquitectura de Microservicios Distribuidos
Descomponer el sistema en servicios independientes desplegados en contenedores (e.g., Servicio de Autenticación, Servicio de Catálogo, Servicio de Inscripciones, Servicio de Cupos, Servicio de Sincronización con Legado, Servicio de Auditoría), comunicados mediante HTTP/gRPC y eventos con Kafka o RabbitMQ.

- *Ventajas:* Escalabilidad independiente por servicio; aislamiento de fallos por proceso; despliegue desacoplado.
- *Desventajas:* Requiere gestionar transacciones distribuidas (Saga / 2PC) para garantizar consistencia de cupos; introduce latencia acumulada de red (*network hops*) que amenaza el SLA de 2 segundos; exige una infraestructura operativa compleja (orquestadores de contenedores como Kubernetes, mallas de servicio, observabilidad distribuida, gestión de múltiples repositorios y pipelines CI/CD).

### Alternativa B: Monolito Tradicional en Tres Capas (Capa Web, Capa de Negocio, Capa de Datos)
Construir una aplicación monolítica estructurada horizontalmente donde los controladores web llaman a clases de servicio genéricas y estas acceden directamente a las tablas de PostgreSQL mediante un ORM acoplado.

- *Ventajas:* Simplicidad de construcción inicial; un solo repositorio y pipeline; facilidad de despliegue en un servidor Linux único con PostgreSQL.
- *Desventajas:* Tiende a degradarse en el antipatrón de "gran bola de lodo" (*Big Ball of Mud*); la lógica de negocio suele mezclarse con dependencias de persistencia y controladores; acopla fuertemente el código al sistema legado, haciendo muy difícil desacoplar la lógica ante una caída de 40 minutos (Escenario A); alto costo de modificación ante nuevas reglas académicas.

### Alternativa C: Monolito Modular con Arquitectura Hexagonal (Puertos y Adaptadores) [SELECCIONADA]
Construir un único artefacto desplegable organizado internamente en módulos funcionales de dominio autónomos (*Bounded Contexts*: Catálogo, Inscripción, Cupos, Reglas Académicas, Auditoría, Integración Legado). Cada módulo encapsula su lógica de negocio pura en un hexágono central, interactuando con el exterior exclusivamente a través de **Puertos** (interfaces) y **Adaptadores** (controladores REST, clientes de base de datos PostgreSQL, clientes Redis, adaptadores del sistema legado).

- *Ventajas:* Aislamiento de dominio absoluto; el núcleo no depende de frameworks ni de bases de datos; permite sustituir o conmutar adaptadores fácilmente (e.g., mock o snapshot ante caídas del legado); despliegue simple y unificado en Linux con PostgreSQL; cero latencia de red inter-módulos (llamadas directas en memoria); mantenible de forma realista por 4 desarrolladores y 1 administrador.
- *Desventajas:* Requiere disciplina estricta de arquitectura para evitar violaciones de dependencias entre módulos; requiere configurar linters y pruebas de arquitectura automáticas en el repositorio.

---

## 4. Criterios de Comparación

| Criterio de Evaluación | Ponderación | Alternativa A: Microservicios | Alternativa B: Monolito en Capas | Alternativa C: Monolito Modular Hexagonal |
| :--- | :--- | :--- | :--- | :--- |
| Viabilidad para 4 Devs + 1 Admin (C10) | 30% | 1 / 5 (Inviable) | 4 / 5 (Favorable) | **5 / 5 (Óptimo)** |
| Costo e Infraestructura Linux/PostgreSQL (C08) | 20% | 2 / 5 (Alto overhead) | 5 / 5 (Muy bajo) | **5 / 5 (Muy bajo y eficiente)** |
| Consistencia Transaccional de Cupos (C04) | 20% | 2 / 5 (Transacciones distribuidas complejas) | 3 / 5 (Riesgo de *race conditions*) | **5 / 5 (ACID nativo local)** |
| Latencia y Rendimiento $P_{95} < 2\text{ s}$ (C02) | 15% | 3 / 5 (Penalización de red) | 4 / 5 (Rápido) | **5 / 5 (Memoria local + Redis)** |
| Desacoplamiento ante Falla de 40 min (Escenario A) | 15% | 4 / 5 (Aislado si hay colas) | 2 / 5 (Acoplado) | **5 / 5 (Puertos / Outbox limpio)** |
| **Puntaje Total Ponderado** | **100%** | **2.25 / 5.0** | **3.70 / 5.0** | **5.00 / 5.0** |

---

## 5. Decisión Seleccionada y Justificación Técnica

Se adopta formalmente el estilo de **Monolito Modular con Arquitectura Hexagonal (Ports & Adapters)**.

**Justificación Técnica:**
1. **Ley de Conway y Eficiencia de Equipo:** Un equipo de 4 desarrolladores y 1 administrador colapsaría intentando operar un ecosistema de microservicios distribuidos (gestión de redes virtuales, configuración de mallas, balanceadores de múltiples servicios, depuración distribuida). El monolito modular proporciona las ventajas de diseño limpio y separación de responsabilidades sin el castigo operacional.
2. **Eliminación de Transacciones Distribuidas:** Al residir los módulos de Inscripciones, Cupos y Outbox en el mismo runtime y compartir la misma base de datos relacional PostgreSQL 16, la consistencia de cupos (C04) se garantiza mediante transacciones ACID locales y cerrojos a nivel de fila (`SELECT FOR UPDATE`), eliminando el riesgo de sobrecupos inherente a arquitecturas con bases de datos descentralizadas.
3. **Resiliencia ante el Escenario A:** La arquitectura hexagonal permite aislar la dependencia del sistema legado detrás de un puerto `LegacyAcademicSystemPort`. Cuando el sistema legado falla por 40 minutos, el núcleo de inscripciones invoca el adaptador de contingencia (que valida prerrequisitos con el snapshot pre-cargado y persiste los eventos en el Outbox), protegiendo la disponibilidad de la plataforma sin tocar el código de negocio.

---

## 6. Consecuencias

### Consecuencias Positivas
- **Despliegue unificado y económico:** Se empaqueta como un servicio único en Linux, simplificando radicalmente el trabajo del administrador de infraestructura.
- **Rendimiento predecible:** Cero sobrecarga de red en validaciones entre módulos; latencias en memoria inferiores a 5 milisegundos para comprobaciones lógicas.
- **Alta testabilidad:** La pureza del dominio permite ejecutar cientos de pruebas unitarias en segundos sin necesidad de levantar bases de datos ni servicios externos.

### Consecuencias Negativas y Riesgos
- **Riesgo de erosión arquitectónica:** Si los desarrolladores no respetan los límites de módulo, el código podría acoplarse con el tiempo.
  - *Mitigación:* Se implementan pruebas automáticas de arquitectura en el pipeline de CI (ver Actividad 5) que fallan el build si un módulo accede directamente al código interno de otro sin pasar por su API pública.
- **Escalado vertical obligatorio inicial:** Todo el monolito escala como una sola unidad en los servidores Linux.
  - *Mitigación:* Con 1.200 usuarios concurrentes (C01), un par de servidores Linux estándar con 8 vCPUs y 16 GB de RAM ejecutando workers de Uvicorn/FastAPI con NGINX soporta sobradamente la carga pico.

---

## 7. Condiciones para Revisar esta Decisión

Esta decisión deberá revisarse si y solo si:
1. La universidad decide migrar formalmente a una nube pública con presupuesto dedicado y el equipo técnico se incrementa a más de 15 ingenieros organizados en células autónomas por facultad.
2. Un módulo específico (e.g., Catálogo de Consultas) requiera escalar horizontalmente de manera independiente con un factor mayor a 10x respecto a los módulos de inscripción, y dicho tráfico no pueda ser absorbido por la capa de caché Redis.

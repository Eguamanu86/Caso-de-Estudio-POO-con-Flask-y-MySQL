# Modelado: Clases OO vs Tablas Físicas MySQL

**Asignatura:** Programación Orientada a Objetos
**Referencia UML:** [01-caso-estudio-poo-enunciado-uml.md](01-caso-estudio-poo-enunciado-uml.md)
**Referencia Implementación:** [03-laboratorio-implementacion-flask-mysql.md](03-laboratorio-implementacion-flask-mysql.md)

---

## Objetivo

Establecer la correspondencia explícita entre el modelo orientado a objetos (clases del dominio) y el modelo relacional físico (tablas MySQL), detallando el DDL completo y ejemplos DML del CRUD.

---

## 1. Mapeo Clases OO → Tablas Relacionales

| Concepto OO                   | Tipo OO              | Tabla / Columna MySQL                              | Observación                                            |
|-------------------------------|----------------------|----------------------------------------------------|--------------------------------------------------------|
| `ReservaLaboratorio`          | Entidad              | `reservas_laboratorio`                             | Una clase de dominio = una tabla principal             |
| `id`                          | Atributo             | `id INT AUTO_INCREMENT PK`                         | Identidad de la entidad                                |
| `laboratorio_id`              | Asociación (FK)      | `laboratorio_id INT NOT NULL`                      | Referencia a tabla `laboratorios`                      |
| `docente_id`                  | Asociación (FK)      | `docente_id INT NOT NULL`                          | Referencia a tabla `docentes`                          |
| `curso_codigo`                | Atributo             | `curso_codigo VARCHAR(30) NOT NULL`                |                                                        |
| `fecha_reserva`               | Atributo             | `fecha_reserva DATE NOT NULL`                      |                                                        |
| `RangoHorario` (composición)  | Objeto de valor      | `hora_inicio TIME`, `hora_fin TIME`                | El objeto de valor se desnormaliza en columnas planas  |
| `EstadoReserva` (enumeración) | Enum                 | `estado VARCHAR(20) NOT NULL DEFAULT 'PENDIENTE'`  | MySQL no requiere ENUM nativo; VARCHAR es más flexible |
| `aprobada_en`                 | Atributo opcional    | `aprobada_en DATETIME NULL`                        | Null hasta que se aprueba                              |
| `deleted_at` (borrado lógico) | Patrón Soft Delete   | `deleted_at DATETIME NULL`                         | No existe en la entidad OO; es infraestructura         |
| `created_at` / `updated_at`   | Infraestructura      | `created_at DATETIME`, `updated_at DATETIME`       | Auditoría; gestionado automáticamente por MySQL        |

### Nota sobre objetos de valor y SQL

> En OO, `RangoHorario` es una clase separada (objeto de valor).
> En el modelo relacional, sus atributos `hora_inicio` y `hora_fin` se almacenan directamente como columnas de `reservas_laboratorio`.
> Esta técnica se llama **desnormalización de objeto de valor** y es la práctica estándar cuando el objeto de valor no tiene identidad propia.

---

## 2. Diagrama Entidad-Relación

```
+------------------+       +-------------------------+       +------------------+
|   laboratorios   |       |  reservas_laboratorio   |       |     docentes     |
+------------------+       +-------------------------+       +------------------+
| PK id            |◄──────| FK laboratorio_id       |       | PK id            |
| nombre           |       | PK id                   |──────►| nombre           |
| capacidad        |       | FK docente_id           |       | email            |
| tipo             |       | curso_codigo            |       | departamento     |
| activo           |       | fecha_reserva           |       +------------------+
+------------------+       | hora_inicio             |
                           | hora_fin                |
                           | estado                  |
                           | aprobada_en             |
                           | deleted_at              |
                           | created_at              |
                           | updated_at              |
                           +-------------------------+
```

---

## 3. DDL — Definición de Estructura de Base de Datos

```sql
-- ============================================================
-- SCHEMA: reservas_db
-- Sistema de Reservas de Laboratorio
-- ============================================================

CREATE DATABASE IF NOT EXISTS reservas_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE reservas_db;

-- ------------------------------------------------------------
-- Tabla: laboratorios
-- Referenciada por reservas_laboratorio.laboratorio_id
-- ------------------------------------------------------------
CREATE TABLE laboratorios (
  id          INT           NOT NULL AUTO_INCREMENT,
  nombre      VARCHAR(100)  NOT NULL,
  capacidad   INT           NOT NULL DEFAULT 30,
  tipo        VARCHAR(50)   NOT NULL DEFAULT 'COMPUTO',
  activo      TINYINT(1)    NOT NULL DEFAULT 1,
  created_at  DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP
                                     ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE INDEX uq_laboratorio_nombre (nombre)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ------------------------------------------------------------
-- Tabla: docentes
-- Referenciada por reservas_laboratorio.docente_id
-- ------------------------------------------------------------
CREATE TABLE docentes (
  id            INT           NOT NULL AUTO_INCREMENT,
  nombre        VARCHAR(150)  NOT NULL,
  email         VARCHAR(150)  NOT NULL,
  departamento  VARCHAR(100)  NULL,
  activo        TINYINT(1)    NOT NULL DEFAULT 1,
  created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP
                                       ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE INDEX uq_docente_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ------------------------------------------------------------
-- Tabla: reservas_laboratorio
-- Entidad principal del sistema
-- Mapea la clase ReservaLaboratorio del modelo OO
-- ------------------------------------------------------------
CREATE TABLE reservas_laboratorio (
  id              INT           NOT NULL AUTO_INCREMENT,

  -- Asociaciones (FK → otras entidades del dominio)
  laboratorio_id  INT           NOT NULL,
  docente_id      INT           NOT NULL,

  -- Atributos de la entidad
  curso_codigo    VARCHAR(30)   NOT NULL,
  fecha_reserva   DATE          NOT NULL,

  -- Objeto de valor RangoHorario desnormalizado
  hora_inicio     TIME          NOT NULL,
  hora_fin        TIME          NOT NULL,

  -- Enumeración EstadoReserva
  estado          VARCHAR(20)   NOT NULL DEFAULT 'PENDIENTE',

  -- Atributo opcional: se llena al aprobar
  aprobada_en     DATETIME      NULL,

  -- Patrón Soft Delete (borrado lógico)
  deleted_at      DATETIME      NULL,

  -- Auditoría — gestionado automáticamente por MySQL
  created_at      DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP
                                         ON UPDATE CURRENT_TIMESTAMP,

  PRIMARY KEY (id),

  -- Integridad referencial
  CONSTRAINT fk_reserva_laboratorio
    FOREIGN KEY (laboratorio_id) REFERENCES laboratorios (id),
  CONSTRAINT fk_reserva_docente
    FOREIGN KEY (docente_id) REFERENCES docentes (id),

  -- Validación del estado en base de datos
  CONSTRAINT chk_estado
    CHECK (estado IN ('PENDIENTE', 'APROBADA', 'RECHAZADA', 'CANCELADA')),

  -- Índices para consultas frecuentes
  INDEX idx_reserva_lookup   (laboratorio_id, fecha_reserva, hora_inicio, hora_fin),
  INDEX idx_estado_fecha      (estado, fecha_reserva),
  INDEX idx_docente_fecha     (docente_id, fecha_reserva)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 4. DML — Manipulación de Datos

### 4.1 INSERT — Datos de prueba

```sql
-- Insertar laboratorios
INSERT INTO laboratorios (nombre, capacidad, tipo) VALUES
  ('Laboratorio A - Planta Baja', 30, 'COMPUTO'),
  ('Laboratorio B - Piso 1',      25, 'COMPUTO'),
  ('Laboratorio C - Piso 2',      20, 'ELECTRONICA');

-- Insertar docentes
INSERT INTO docentes (nombre, email, departamento) VALUES
  ('Ana García',    'ana.garcia@unemi.edu.ec',    'Informática'),
  ('Luis Romero',   'luis.romero@unemi.edu.ec',   'Electrónica'),
  ('María Solano',  'maria.solano@unemi.edu.ec',  'Sistemas');

-- Insertar reservas de prueba (estado inicial: PENDIENTE)
INSERT INTO reservas_laboratorio
  (laboratorio_id, docente_id, curso_codigo, fecha_reserva, hora_inicio, hora_fin)
VALUES
  (1, 1, 'INF-202',  '2026-06-10', '08:00:00', '10:00:00'),
  (1, 2, 'ELE-301',  '2026-06-10', '10:00:00', '12:00:00'),
  (2, 3, 'SIS-401',  '2026-06-11', '14:00:00', '16:00:00'),
  (3, 1, 'INF-101',  '2026-06-12', '09:00:00', '11:00:00');
```

### 4.2 SELECT — Consultas

```sql
-- Todas las reservas activas (sin borrado lógico)
SELECT
  r.id,
  l.nombre              AS laboratorio,
  d.nombre              AS docente,
  r.curso_codigo,
  r.fecha_reserva,
  r.hora_inicio,
  r.hora_fin,
  r.estado,
  r.aprobada_en
FROM reservas_laboratorio r
JOIN laboratorios l ON l.id = r.laboratorio_id
JOIN docentes     d ON d.id = r.docente_id
WHERE r.deleted_at IS NULL
ORDER BY r.fecha_reserva, r.hora_inicio;

-- Reservas filtradas por estado
SELECT * FROM reservas_laboratorio
WHERE estado = 'PENDIENTE'
  AND deleted_at IS NULL;

-- Reservas de un laboratorio en una fecha
SELECT * FROM reservas_laboratorio
WHERE laboratorio_id = 1
  AND fecha_reserva = '2026-06-10'
  AND deleted_at IS NULL
ORDER BY hora_inicio;

-- Verificar conflicto de horario para laboratorio 1 en bloque 09:00–11:00
-- (solo cuenta reservas ACTIVAS: excluye CANCELADA y RECHAZADA)
SELECT COUNT(*) AS total
FROM reservas_laboratorio
WHERE laboratorio_id = 1
  AND fecha_reserva   = '2026-06-10'
  AND deleted_at      IS NULL
  AND estado NOT IN ('CANCELADA', 'RECHAZADA')
  AND ('09:00:00' < hora_fin AND '11:00:00' > hora_inicio);
```

### 4.3 UPDATE — Modificaciones

```sql
-- Aprobar una reserva (cambia estado y registra fecha de aprobación)
UPDATE reservas_laboratorio
   SET estado      = 'APROBADA',
       aprobada_en = NOW(),
       updated_at  = NOW()
 WHERE id = 1
   AND deleted_at IS NULL;

-- Rechazar una reserva
UPDATE reservas_laboratorio
   SET estado     = 'RECHAZADA',
       updated_at = NOW()
 WHERE id = 2
   AND deleted_at IS NULL;

-- Cancelar una reserva
UPDATE reservas_laboratorio
   SET estado     = 'CANCELADA',
       updated_at = NOW()
 WHERE id = 3
   AND deleted_at IS NULL;

-- Actualizar datos de la reserva (bloque horario y curso)
UPDATE reservas_laboratorio
   SET curso_codigo = 'INF-202-LAB',
       hora_inicio  = '09:00:00',
       hora_fin     = '11:00:00',
       updated_at   = NOW()
 WHERE id = 1
   AND deleted_at IS NULL;
```

### 4.4 DELETE — Borrado Lógico

```sql
-- Borrado lógico: se marca deleted_at, el registro NO se elimina físicamente
UPDATE reservas_laboratorio
   SET deleted_at = NOW(),
       updated_at  = NOW()
 WHERE id = 4;

-- Verificar que ya no aparece en consultas normales
SELECT * FROM reservas_laboratorio WHERE deleted_at IS NULL;

-- Consultar todos los registros incluyendo borrados (auditoría)
SELECT * FROM reservas_laboratorio;
```

---

## 5. Comparación Final: Objeto vs Registro

```
Clase OO: ReservaLaboratorio          Registro en MySQL: reservas_laboratorio
─────────────────────────────         ──────────────────────────────────────────
id: int | None = None        ←────►   id INT AUTO_INCREMENT PK
laboratorio_id: int          ←────►   laboratorio_id INT FK → laboratorios
docente_id: int              ←────►   docente_id INT FK → docentes
curso_codigo: str            ←────►   curso_codigo VARCHAR(30)
fecha_reserva: date          ←────►   fecha_reserva DATE
rango: RangoHorario          ←────►   hora_inicio TIME
  .hora_inicio               ←────►   hora_fin TIME
  .hora_fin
estado: EstadoReserva        ←────►   estado VARCHAR(20) CHECK(...)
aprobada_en: datetime | None ←────►   aprobada_en DATETIME NULL
(soft delete — infraestructura) ────►  deleted_at DATETIME NULL
(auditoría de infraestructura) ←───►  created_at DATETIME
                             ←────►   updated_at DATETIME
```

---

## 6. Actividades del Estudiante

### Actividad 1 — Análisis del esquema

Responde las siguientes preguntas basándote en el DDL y el modelo OO:

1. ¿Por qué `hora_inicio` y `hora_fin` son columnas de `reservas_laboratorio` y no una tabla separada?
2. ¿Qué sucede si intentas insertar un registro con `estado = 'VIGENTE'`? ¿Qué restricción lo impide?
3. ¿Por qué `deleted_at` no tiene equivalente en la clase `ReservaLaboratorio`?
4. ¿Qué índice se usa cuando el sistema verifica conflictos de horario? Justifica tu respuesta.

### Actividad 2 — Práctica SQL

Ejecuta las siguientes consultas y anota los resultados:

```sql
-- 1. Lista todas las reservas PENDIENTES del laboratorio 1, ordenadas por hora
SELECT * FROM reservas_laboratorio
WHERE laboratorio_id = 1 AND estado = 'PENDIENTE' AND deleted_at IS NULL
ORDER BY fecha_reserva, hora_inicio;

-- 2. ¿Hay conflicto si se quiere reservar el lab 1 el 2026-06-10 de 09:00 a 11:00?
SELECT COUNT(*) AS conflictos
FROM reservas_laboratorio
WHERE laboratorio_id = 1
  AND fecha_reserva = '2026-06-10'
  AND deleted_at IS NULL
  AND ('09:00:00' < hora_fin AND '11:00:00' > hora_inicio);

-- 3. Aprueba la reserva con id=1 y verifica que aprobada_en se llenó
UPDATE reservas_laboratorio
   SET estado = 'APROBADA', aprobada_en = NOW() WHERE id = 1;
SELECT id, estado, aprobada_en FROM reservas_laboratorio WHERE id = 1;
```

### Actividad 3 — Extensión del modelo

Agrega la siguiente columna al modelo OO y a la tabla física:

- **Nombre:** `motivo_rechazo`
- **Tipo OO:** `str | None`
- **Tipo SQL:** `VARCHAR(255) NULL`
- **Regla:** Solo se llena cuando `estado = 'RECHAZADA'`.

Preguntas:
1. ¿En qué archivo Python agregarías el atributo?
2. ¿Qué sentencia SQL usarías para agregar la columna sin recrear la tabla?
3. ¿Qué método de la entidad necesitas crear o modificar?

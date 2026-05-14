# Caso de Estudio POO — Sistema de Reservas de Laboratorio

**Asignatura:** Programación Orientada a Objetos — UNEMI, Periodo Abril-Junio 2026
**Stack:** Python 3.10+ · Flask 3.0.2 · MySQL 8.x · mysqlclient 2.2.4 · python-dotenv

---

## Documentación

| # | Archivo | Contenido |
|---|---------|-----------|
| 1 | [01-caso-estudio-poo-enunciado-uml.md](01-caso-estudio-poo-enunciado-uml.md) | Enunciado, análisis, UML (clases y secuencia) |
| 2 | [02-modelado-clases-vs-tablas-fisicas.md](02-modelado-clases-vs-tablas-fisicas.md) | Mapeo OO → MySQL, DDL completo, DML de prueba |
| 3 | [03-laboratorio-implementacion-flask-mysql.md](03-laboratorio-implementacion-flask-mysql.md) | Guía paso a paso de implementación en Python |

---

## Requisitos Previos

| Herramienta    | Verificación          |
|----------------|-----------------------|
| Python 3.10+   | `py --version`        |
| pip            | `py -m pip --version` |
| MySQL 8.x      | `mysql --version`     |
| Git Bash       | Terminal recomendada  |

---

## Levantar el Proyecto

### 1. Clonar y entrar al directorio

```bash
git clone <url-del-repositorio>
cd Caso-de-Estudio-POO-con-Flask-y-MySQL
```

### 2. Crear y activar el entorno virtual

```bash
# Crear
py -m venv .venv

# Activar (Git Bash)
source .venv/Scripts/activate

# Activar (PowerShell)
# .\.venv\Scripts\Activate.ps1

# Activar (CMD)
# .venv\Scripts\activate.bat
```

> Si PowerShell bloquea scripts, ejecuta una vez como administrador:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

### 3. Instalar dependencias

```bash
pip install -r requirements.txt
```

### 4. Configurar variables de entorno

Crea el archivo `.env` en la raíz del proyecto (nunca lo subas a git):

```env
DB_HOST=localhost
DB_PORT=3306
DB_NAME=reservas_db
DB_USER=root
DB_PASSWORD=tu_password
```

### 5. Preparar la base de datos

```bash
# Opción A: cargar el schema desde terminal
mysql -u root -p < schema.sql

# Opción B: copiar el DDL desde 02-modelado-clases-vs-tablas-fisicas.md
# y ejecutarlo en Workbench, DBeaver o HeidiSQL
```

Luego inserta datos de prueba (ver sección DML en [02-modelado-clases-vs-tablas-fisicas.md](02-modelado-clases-vs-tablas-fisicas.md)).

### 6. Ejecutar la API

```bash
python run.py
```

La API queda disponible en `http://localhost:5000`.

### 7. Verificar

```bash
# Listar reservas
curl http://localhost:5000/api/v1/reservas

# Crear una reserva
curl -X POST http://localhost:5000/api/v1/reservas \
  -H "Content-Type: application/json" \
  -d '{"laboratorio_id": 1, "docente_id": 1, "curso_codigo": "INF-202", "fecha": "2026-06-01", "hora_inicio": "08:00", "hora_fin": "10:00"}'
```

---

## Estructura del Proyecto

```
.
├── app/
│   ├── domain/          # Entidades y repositorio abstracto
│   ├── application/     # Servicios (lógica de negocio)
│   ├── infrastructure/  # DAO, ConnectionPool, MySQLRepository
│   └── api/             # Controladores Flask (blueprints)
├── run.py               # Punto de entrada
├── schema.sql           # DDL de la base de datos
├── requirements.txt
├── .env                 # Variables de entorno (no incluido en git)
└── .env.example         # Plantilla de variables de entorno
```

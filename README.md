# casino-backend

Backend del **Casino Online** — Experiencia 2 de la asignatura
**Introducción a Herramientas DevOps (ISY1101)**.

API REST en Node.js + Express con PostgreSQL como base de datos.

---

## Stack

- Node.js 20 (sobre `node:20-alpine`)
- Express 4
- PostgreSQL 16 (`postgres:16-alpine` con volumen nombrado `pg_data`)
- JWT para autenticación, bcryptjs para hashes
- `pg` como cliente de Postgres
- Docker (multi-stage build, usuario no root)
- GitHub Actions (CI/CD → Docker Hub)

---

## Estructura

```
backend_intro_devops_casino_vidal/
├── src/
│   ├── server.js                ← bootstrap Express + rutas + /health
│   ├── db/
│   │   ├── pool.js              ← Pool de pg + esperarBD()
│   │   └── seed.js              ← usuarios demo (idempotente)
│   ├── middleware/
│   │   └── auth.js              ← JWT firmar / requiereAuth
│   ├── routes/
│   │   ├── auth.js              ← /api/auth/login | register
│   │   ├── users.js             ← /api/usuarios/me, depositar
│   │   ├── games.js             ← /api/juegos/{slots,roulette,blackjack}
│   │   └── transactions.js      ← /api/transacciones (historial)
│   └── games/
│       ├── slots.js
│       ├── roulette.js
│       └── blackjack.js
├── db/
│   ├── Dockerfile               ← imagen PostgreSQL con init.sql
│   └── init.sql                 ← esquema (tablas usuarios, juegos, sesiones, transacciones)
├── .github/
│   └── workflows/
│       └── build-and-push-dockerhub.yml  ← pipeline CI/CD
├── Dockerfile                   ← multi-stage build (builder + runtime, USER node)
├── docker-compose.yml           ← stack completo: db + backend + frontend
├── .dockerignore
├── .gitignore
├── .env.example
└── package.json
```

---

## Variables de entorno

| Variable         | Default     | Descripción                                  |
|------------------|-------------|----------------------------------------------|
| `PORT`           | `3000`      | Puerto HTTP del servidor                     |
| `JWT_SECRET`     | `cambiame`  | Secreto de firma JWT (cambiar en producción) |
| `JWT_EXPIRES_IN` | `8h`        | Vigencia del token                           |
| `DB_HOST`        | `localhost` | Host de Postgres (`db` en docker-compose)    |
| `DB_PORT`        | `5432`      | Puerto Postgres                              |
| `DB_USER`        | `casino`    | Usuario Postgres                             |
| `DB_PASSWORD`    | `casino`    | Password Postgres                            |
| `DB_NAME`        | `casino_db` | Base de datos                                |
| `CORS_ORIGIN`    | `*`         | Lista CSV de orígenes permitidos             |

>  **Nunca commitear el `.env` real.** Está incluido en `.gitignore`. Usar `.env.example` como referencia.

---

## Endpoints

### Autenticación

| Método | Ruta                 | Descripción                              |
|--------|----------------------|------------------------------------------|
| POST   | `/api/auth/register` | Registro `{ username, email, password }` |
| POST   | `/api/auth/login`    | Login `{ username, password }`           |

### Usuario autenticado (header `Authorization: Bearer <token>`)

| Método | Ruta                           | Descripción                      |
|--------|--------------------------------|----------------------------------|
| GET    | `/api/usuarios/me`             | Datos del usuario y saldo        |
| POST   | `/api/usuarios/me/depositar`   | `{ monto }` — recarga saldo demo |
| GET    | `/api/transacciones?limit=50`  | Historial del usuario            |

### Juegos

| Método | Ruta                             | Descripción                                                   |
|--------|----------------------------------|---------------------------------------------------------------|
| GET    | `/api/juegos`                    | Catálogo (slots, roulette, blackjack)                         |
| POST   | `/api/juegos/slots/jugar`        | `{ apuesta }` → `{ resultado, saldo }`                        |
| POST   | `/api/juegos/roulette/jugar`     | `{ apuestas:[{tipo,valor,monto}] }` → `{ resultado, saldo }` |
| POST   | `/api/juegos/blackjack/iniciar`  | `{ apuesta }` → `{ sesionId, jugador, banca, ... }`           |
| POST   | `/api/juegos/blackjack/accion`   | `{ sesionId, accion: pedir/plantarse/doblar }`                |

### Salud

| Método | Ruta      | Descripción              |
|--------|-----------|--------------------------|
| GET    | `/health` | Estado del servidor + BD |
| GET    | `/`       | Mensaje de bienvenida    |

---

## Usuarios demo (sembrados al arrancar)

| username   | password    | rol     | saldo inicial |
|------------|-------------|---------|---------------|
| `demo`     | `demo1234`  | jugador | $5.000        |
| `jugador1` | `demo1234`  | jugador | $1.000        |
| `admin`    | `admin1234` | admin   | $99.999       |

---

## Como levantar en local

Requisitos: Node 20 y un Postgres accesible.

```bash
cp .env.example .env       # ajustar credenciales
npm install
npm start
# API disponible en http://localhost:3000
```

---

## Docker

### Build local

```bash
docker build -t casino-backend:v1.0.0 .
```

### Levantar el stack completo

```bash
cp .env.example .env       # editar con tus valores
docker compose up -d
docker compose ps
```

### Verificar usuario no root

```bash
docker exec <nombre-contenedor-backend> whoami
# Debe responder: node
```

### Healthcheck

```bash
curl http://localhost:3000/health
# {"status":"ok","db":"up","uptime":...}
```

### Comandos útiles

```bash
# Ver logs en tiempo real
docker compose logs -f backend

# Reiniciar solo el backend
docker compose restart backend

# Entrar al contenedor de la BD
docker exec -it <contenedor-db> psql -U casino -d casino_db

# Ver tablas
\dt

# Bajar el stack y eliminar volúmenes
docker compose down -v

#Esto levanta el stack 
docker compose up -d 

#si deseas pararlo sin la necesidad de borrar
docker compose stop
```

---


### GitHub Secrets requeridos

| Secret                  | Descripción                   |
|-------------------------|-------------------------------|
| `DOCKERHUB_USERNAME`    | Usuario de Docker Hub         |
| `DOCKERHUB_TOKEN`       | Access Token de Docker Hub    |
| `AWS_ACCESS_KEY_ID`     | Credencial AWS Academy        |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS Academy        |
| `AWS_SESSION_TOKEN`     | Token de sesion AWS Academy   |

---

## Despliegue en AWS EC2

|HERRAMIENTAS UTILIZADAS
|----------------|--------------------------------------------------------------
| Instancia      | `ec2-backend` — t3.micro — Amazon Linux 2023                 
| Subred         | Privada (`casino-subnet-private` — 10.0.128.0/20)            
| Security Group | `sg-backend` — puerto 3000 solo desde `sg-frontend`          
| Route table    | 10.0.0.0/16 para local 0.0.0.0/Nat Casino y endpoint para la vpc
| Acceso         | AWS Systems Manager Session Manager (sin SSH público)        
| Imagen         | `kripsv/casino-backend:latest`                               

### Comandos de despliegue en EC2

```bash
# Instalar Docker en EC2 (Amazon Linux 2023)
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Descargar imagen y levantar
docker pull kripsv/casino-backend:latest
docker run -d \
  --name casino-backend \
  -p 3000:3000 \
  --env-file .env \
  kripsv/casino-backend:latest
```

---

## Commits hechos
Commits en dev:9
-5 feats (Agregar el healtcheck y user node del dockerfile)
-Chore config(actualizacion del docekr file y el archvio yml)
-3 Fix orientados al docker compose yml y healthcheck


---

## Repositorio del frontend

[`casino-frontend`](../frontend_intro_devops_casino_vidal)




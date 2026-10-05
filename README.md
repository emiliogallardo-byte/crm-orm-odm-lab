# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.


## Respuestas

**1. Dos motores.**  
Activity es buen candidato para MongoDB porque su `metadata` puede cambiar de estructura según el tipo de actividad. Company y Contact son buenos para PostgreSQL porque tienen una relación clara: una compañía puede tener varios contactos y se puede manejar con una llave foránea.

**2. ORM vs ODM.**  
Un ORM permite trabajar con una base de datos relacional usando objetos y modelos; en este proyecto se usa Sequelize. Un ODM hace algo parecido pero con documentos de una base documental; aquí se usa Mongoose. La diferencia principal es el tipo de base que representa cada uno.

**3. Configuración por variables de entorno.**  
Las variables están definidas en `.devcontainer/docker-compose.yml` y también se muestran en `.env.example`. Es mala práctica ponerlas en los `.js` porque las credenciales quedarían expuestas y sería más difícil cambiarlas. La app usa `postgres` en `DB_HOST` y `mongo` en `MONGODB_URI` porque esos son los nombres de los servicios dentro de Docker, no `localhost`.

**4. Asociaciones.**  
En `models/sequelize/index.js`, `Company` tiene muchos `Contact` usando `companyId` como llave foránea en la tabla `Contact`. El alias `as: 'contacts'` es el nombre con el que Sequelize agrega esa relación al resultado cuando usamos el `include`.

**5. Eager loading.**  
Hacer una consulta para la compañía y otra para sus contactos significa buscar los datos por separado. Con `include` se pueden cargar en la misma operación de Sequelize, dejando la respuesta lista con `contacts`. Para este caso es preferible porque la relación se solicita desde la misma consulta.

**6. Instancia vs consulta.**  
Buscar primero la instancia y después usar `contact.update(...)` permite trabajar con el registro encontrado y devolver esa instancia ya modificada. Con `Model.update(...)` se actualizan registros directamente usando un `where`, pero la operación está más orientada a modificar registros que a trabajar con una instancia concreta y devolverla completa.

**7. Esquema flexible.**  
En `models/mongoose/activity.js`, `metadata` usa `mongoose.Schema.Types.Mixed`. Esto permite guardar objetos diferentes para `CALL`, `EMAIL` y `MEETING`, incluso con arreglos o campos distintos. La desventaja es que se pierde parte de la validación específica que habría si cada campo tuviera un tipo definido.

**8. Sin ref.**  
`contactId` y `userId` son números que apuntan a registros de PostgreSQL y el esquema no tiene `ref` de Mongoose. Por eso Mongoose no puede usar `populate` para traerlos automáticamente. También significa que MongoDB no controla esa relación; por ejemplo, se podría eliminar un User en PostgreSQL y quedar un `userId` en Activity que ya no corresponda a un usuario existente.

**9. Documento actualizado.**  
Antes de corregir el Reto 08, `findByIdAndUpdate()` devolvía por defecto el documento que estaba antes del cambio. En `controllers/activities.js` se agregaron `new: true` para obtener el documento actualizado y `runValidators: true` para validar los datos según el esquema.

**10. Pruebas de comportamiento.**  
Probar el comportamiento permite comprobar que la API realmente responde como debe sin depender de un método interno específico. Así se puede cambiar la implementación mientras se mantenga el mismo resultado para quien usa la API.

**11. Repetibilidad.**  
`tests/setup.js` conecta las dos bases y ejecuta `reset()` antes de cada suite; al final cierra las conexiones. Esto hace que cada conjunto de pruebas empiece con los mismos datos y por eso `npm test` puede dar el mismo resultado cada vez.

**12. Tu experiencia.**  
El reto que más trabajo me dio al revisar el código fue el 08 porque la actualización sí se hacía, pero la respuesta mostraba el documento anterior. Al revisar `controllers/activities.js` y el comportamiento esperado por Jest, quedó claro que hacía falta usar `new: true` y también `runValidators: true`.

## Evidencia

En esta sección debe colocarse la captura real de la terminal del Codespace después de ejecutar:

```bash
npm test
```

La captura debe mostrar completa la línea:

`Test Suites: 9 passed, 9 total`

![npm test con las 9 suites en verde](evidencia-npm-test.png)

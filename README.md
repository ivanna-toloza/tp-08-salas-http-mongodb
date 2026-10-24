# Trabajo práctico 08 - Endpoints persistentes de reservas de salas

Laura Ivanna Toloza

## 1. Punto de partida

Esta carpeta, `tp-08-salas-http-mongodb`, es una copia del TP 07 (`tp-07-salas-mongodb`). El TP 07 quedó intacto. No se copiaron `.git`, `node_modules` ni `.env`; se ejecutó `npm install` con el mismo `package.json` y `package-lock.json`, sin actualizar dependencias. Por eso el manifiesto conserva el nombre `tp-07-salas-mongodb`.

En el TP 07 el sitio trabajaba con un arreglo en memoria y solo los scripts de `scripts/` usaban MongoDB. En este TP **el sitio HTML y la nueva API JSON leen y escriben la misma colección de MongoDB** mediante un único servicio asíncrono.

## 2. Versiones

| Componente | Versión |
| --- | --- |
| Node.js | 22.13 o superior (probado con 24.11.1) |
| Express | 5.2.1 |
| Mongoose | 9.10.3 (exacta) |
| EJS / express-ejs-layouts / Morgan | 6.0.1 / 2.5.1 / 1.12.1 |
| ESLint / Prettier | 10.12.0 / 3.9.10 |

Se verifica con `npm ls express mongoose`. Se mantiene CommonJS.

## 3. Preparación de Atlas

1. Reutilizar el clúster **Atlas Free (M0)** del TP 07.
2. Usar un usuario de base de datos con permiso de lectura y escritura (no es la cuenta de Atlas) y la IP actual en **Network Access**. No habilitar el acceso desde cualquier dirección.
3. La base de este TP es **`reservas_salas_u8`**, colección **`reservas`**. No hace falta crearla desde el panel: aparece al guardar la primera reserva. La base del TP 07 (`reservas_salas_u7`) no se borró ni se migró.

## 4. Entorno y DNS

Copiar `.env.example` como `.env` y reemplazar `USUARIO`, `CLAVE` y `HOST` por los datos reales (codificar caracteres reservados de la contraseña, por ejemplo `@` → `%40`):

```env
PORT=3000
NODE_ENV=development
MONGODB_URI=mongodb+srv://USUARIO:CLAVE@HOST/reservas_salas_u8?retryWrites=true&w=majority
```

`HOST` es un marcador: con esa plantilla tal cual **no** se puede conectar. `.env` está en `.gitignore` y no se entrega.

**DNS:** no se ajustó. En la red usada, la resolución SRV de `mongodb+srv://` funcionó sin cambios (`scripts/comprobar-conexion.js` conectó). Si en otra red aparece `querySrv ECONNREFUSED`, la corrección acordada en clase es agregar en `src/config/base-datos.js`, antes de conectar y una sola vez (no por solicitud), `require("node:dns").setServers(["8.8.8.8", "1.1.1.1"])`, siempre que esos resolutores estén autorizados en la red. No resuelve contraseñas ni IPs no autorizadas.

## 5. Comandos

Desde la raíz del proyecto:

```bash
npm install
node --env-file=.env scripts/comprobar-conexion.js   # conecta y cierra; no crea nada

node --env-file=.env src/index.js                    # inicia el sitio y la API
npm run start:local                                  # equivalente (carga .env)

npm run format                                       # Prettier sobre src y scripts
npm run check                                        # format:check + ESLint
```

`npm start` (sin `--env-file`) solo funciona si el entorno ya tiene `MONGODB_URI`. Para detener el servidor: **Ctrl+C**. El servidor deja de aceptar conexiones, espera las solicitudes activas y recién entonces cierra MongoDB (`Cerrando el servidor...` → `Conexión con MongoDB cerrada.`). Lo mismo ocurre con SIGTERM.

Scripts de terminal (abren la conexión, hacen una tarea y la cierran):

```bash
node --env-file=.env scripts/listar-reservas.js      # solo lee
node --env-file=.env scripts/crear-reserva.js        # crea UNA reserva por ejecución
node scripts/validar-reserva.js                      # valida el modelo sin conectar
node scripts/simular-fallo-servicio.js               # simula una base caída (sin conectar)
```

## 6. Arranque

`src/index.js`:

1. Lee la configuración (`PORT`, `NODE_ENV`).
2. **Espera** `conectarBaseDatos()`.
3. Recién entonces crea el servicio MongoDB, la app y **escucha el puerto**.

Si la URI falta, el DNS falla, Atlas rechaza el usuario o la IP, o el puerto está ocupado, el proceso termina con código 1 y **nunca** muestra `Aplicación disponible`. La conexión se abre una vez y se mantiene entre solicitudes (pool de Mongoose): a diferencia de los scripts, no hay un `finally` que desconecte después de `listen`.

## 7. Estructura por capas

```
tp-08-salas-http-mongodb/
  src/
    index.js                    conecta antes de escuchar; cierre ordenado SIGINT/SIGTERM
    app.js                      middleware, rutas HTML y API, 404 y manejador de errores
    config/
      entorno.js                (sin cambios) puerto, formato de Morgan y MONGODB_URI
      base-datos.js             (sin cambios) conectar y desconectar
    modelos/reserva.js          esquema + campo estado (default "pendiente")
    servicios/
      reservas-mongo.js         NUEVO: listar, obtenerPorId, contar, crear (async)
      reservas.js               referencia en memoria del TP 07, ya no se importa
    controladores/reservas.js   async/await; respuestas HTML y JSON
    rutas/
      reservas.js               router HTML (sin cambios; /nueva antes de /:id)
      reservas-api.js           NUEVO: router JSON /api/reservas
    middleware/
      reservas.js               validación de entrada (tipos estrictos)
      errores.js                NUEVO: middleware de errores de 4 parámetros
      solicitudes.js            (sin cambios) id de solicitud y duración
  scripts/                      diagnóstico y laboratorio de terminal
  postman/                      colección de ejemplo (opcional)
  evidencias/                   registro de las pruebas (registro-pruebas.txt)
  views/  public/               vistas y recursos del sitio
  .env.example                  plantilla sin secretos
```

| Capa | Responsabilidad |
| --- | --- |
| Modelo | Valida y persiste (reglas del esquema, default de `estado`). |
| Servicio | Traduce operaciones a consultas Mongoose y devuelve objetos simples con `id` textual. No conoce Express ni HTTP. |
| Controlador | Espera al servicio y decide la respuesta (render, redirección 302, JSON 200/201/404). |
| Router | Asocia método y dirección. El HTML y la API están separados pero comparten controlador, servicio y modelo. |
| Middleware de errores | Traduce errores a HTTP: entrada inválida 400, fallo inesperado 500. |

### Qué dejó de importarse

- `src/index.js` ya no tiene el arreglo `reservasIniciales` (las 4 reservas semilla) ni importa `crearServicioReservas` de `src/servicios/reservas.js`.
- `src/servicios/reservas.js` se conserva solo como referencia: ningún módulo lo importa, así que hay una sola fuente activa de reservas (MongoDB).
- Nada inserta, borra ni carga semillas al iniciar o al listar.

## 8. Modelo y entrada admitida

| Campo | Regla del esquema |
| --- | --- |
| `estudiante` | String, `required`, `trim` |
| `email` | String, `required`, `trim`, `match: /@/` |
| `sala` | String, `required`, `enum`: Sala Norte, Sala Sur, Sala Multimedia |
| `fecha` | String, `required`, `trim` |
| `turno` | String, `required`, `enum`: Mañana, Tarde, Noche |
| `personas` | Number, `required`, `min: 1`, `max: 6`, entero (`Number.isInteger`) |
| `estado` | String, `enum`: pendiente, confirmada, **`default: "pendiente"`** |

**Dos niveles de validación.** El middleware `validarReserva` filtra la entrada HTTP y el esquema vuelve a validar al guardar (se comprueba con `node scripts/validar-reserva.js`, que rechaza cada caso inválido sin conectarse).

El middleware:

- Usa `req.body ?? {}`: un POST sin cuerpo o con `text/plain` (sin parser) se rechaza con 400 en lugar de romper.
- Acepta los campos de texto **solo si son strings** (antes `String(["Sala Norte"])` pasaba como `"Sala Norte"`).
- Acepta `personas` solo como número o texto numérico no vacío; `true`, `[2]` o `{...}` se rechazan aunque `Number()` pudiera convertirlos.
- Copia **explícitamente** los seis campos a `req.reservaValidada`; el servicio vuelve a copiarlos uno por uno al llamar a `Reserva.create`. `req.body` nunca llega completo a Mongoose, por eso un `estado` o `_id` enviado por el cliente se ignora.
- En la API responde `400` con `{ error, campos }` indicando los campos rechazados; en el formulario responde `400` y vuelve a mostrar los valores enviados.

El esquema y los IDs:

- `estado` no se pide al crear: el default se aplica porque el campo llega `undefined`. Esta API no permite confirmar reservas. El default no es autenticación ni autorización, y no reescribe documentos viejos.
- MongoDB guarda `_id` como `ObjectId`. El servicio usa `.lean()` y construye la representación a mano con `id: documento._id.toString()`: las vistas siguen usando `reserva.id` y la API no expone `_id` ni `__v`.

**Límites que siguen sin controlarse:** email completo (solo se exige `@`), fecha real o futura, solapamiento de sala y turno, cupos compartidos y reservas duplicadas. No hay índice único: dos POST válidos iguales crean dos documentos.

## 9. Endpoints

### API JSON

| Método y dirección | Resultado |
| --- | --- |
| `GET /api/reservas` | 200 y arreglo (vacío también es válido) |
| `GET /api/reservas?sala=Sala%20Norte` | 200 y coincidencias exactas; sin coincidencias, `[]` |
| `GET /api/reservas/:id` | 200 y la reserva |
| `POST /api/reservas` | 201, reserva guardada y cabecera `Location: /api/reservas/<id>` |
| ID mal formado (`/api/reservas/1`) | 400 JSON |
| ID bien formado inexistente | 404 JSON `{ "error": "Reserva no encontrada." }` |
| Creación inválida o JSON mal formado | 400 JSON, sin escritura |
| `sala` repetida o vacía | 400 JSON |
| Ruta API inexistente | 404 JSON `{ "error": "Ruta no encontrada." }` |
| Fallo inesperado (base inaccesible) | 500 JSON `{ "error": "No se pudo completar la operación." }`, sin traza ni secretos |

**Filtro:** el servicio arma el criterio a mano (`{ sala: "<texto>" }`); nunca usa `req.query` completo. Si `sala` llega repetida (Express la entrega como arreglo) o vacía, responde 400. La igualdad es exacta: `sala norte` no coincide con `Sala Norte`. Un parámetro como `sala[$ne]=x` no se convierte en operador de MongoDB.

No hay PUT, PATCH ni DELETE en esta entrega, ni paginación u orden.

### Sitio HTML

| Dirección | Resultado |
| --- | --- |
| `GET /` | Inicio |
| `GET /reservas` | Listado desde MongoDB |
| `GET /reservas/nueva` | Formulario |
| `POST /reservas` | Válido: espera la escritura y redirige con **302** a `/reservas`. Inválido: **400** y conserva los valores |
| `GET /reservas/:id` | Detalle (muestra el id y el estado); ID mal formado 400, inexistente 404 |
| `GET /estado` | JSON `{ servicio, reservas, solicitudId }`; `reservas` es el conteo de MongoDB (`countDocuments`) |

El HTML responde **302** porque un navegador debe pasar a otra página después de enviar un formulario (así, al actualizar, no se reenvía). La API responde **201** porque su cliente necesita saber que se creó un recurso y dónde encontrarlo (`Location`).

### JSON de ejemplo (datos ficticios)

Solicitud `POST /api/reservas`:

```json
{
  "estudiante": "Ana Ejemplo",
  "email": "ana@example.com",
  "sala": "Sala Norte",
  "fecha": "2026-10-21",
  "turno": "Mañana",
  "personas": 2
}
```

Respuesta `201` (el `id` real lo genera MongoDB):

```json
{
  "id": "<24 caracteres hexadecimales>",
  "estudiante": "Ana Ejemplo",
  "email": "ana@example.com",
  "sala": "Sala Norte",
  "fecha": "2026-10-21",
  "turno": "Mañana",
  "personas": 2,
  "estado": "pendiente"
}
```

Respuesta `400` de una creación inválida:

```json
{ "error": "Por favor, completá todos los campos con datos válidos.", "campos": ["personas"] }
```

## 10. Errores

- Los controladores son `async` y **esperan** cada operación del servicio. Con Express 5, un rechazo de esa promesa llega al middleware de errores.
- `src/middleware/errores.js` (cuatro parámetros) se registra **después** de las rutas y del 404:
  - `ValidationError` de Mongoose, ID inválido, filtro inválido y JSON mal formado → **400**.
  - Cualquier otro error (por ejemplo, base inaccesible) → **500** con mensaje genérico. Nunca se devuelve `[]` ni 200 para disimular un fallo.
  - En `/api/...` y `/estado` responde JSON; en el sitio, la página de error con el código correspondiente.
  - En consola registra solo el id de solicitud y el nombre del error, sin URI, contraseñas ni trazas.

**Simulación controlada de un fallo inesperado:** `node scripts/simular-fallo-servicio.js` crea la app con un servicio cuyas operaciones fallan como si la base estuviera caída, sin tocar Atlas. Salida obtenida:

```
GET /api/reservas -> 500 application/json; charset=utf-8 | detalles internos: no
GET /api/reservas/000000000000000000000000 -> 500 application/json; charset=utf-8 | detalles internos: no
POST /api/reservas -> 500 application/json; charset=utf-8 | detalles internos: no
GET /estado -> 500 application/json; charset=utf-8 | detalles internos: no
GET /reservas -> 500 text/html; charset=utf-8 | detalles internos: no
```

## 11. Pruebas con Postman

Iniciar el servidor con `node --env-file=.env src/index.js` y dejarlo corriendo. Opcional: importar `postman/tp-08-reservas.postman_collection.json` (**File → Import**). Trae todas las solicitudes, la variable `baseUrl` (`http://localhost:3000`) y guarda el `id` del POST válido en `idReserva`. Enviar cada solicitud **de a una** con **Send**; no usar el Runner, porque cada POST válido crea otra reserva.

| # | Método | URL | Body | Cabecera de la solicitud | Respuesta esperada |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `http://localhost:3000/api/reservas` | none | — | 200 y arreglo (`[]` si la base es nueva). Anotar la cantidad |
| 2 | POST | `http://localhost:3000/api/reservas` | raw → JSON (el del punto 9) | `Content-Type: application/json` | 201, `id`, los seis campos, `estado: "pendiente"`, y `Location` en **Headers de la respuesta** |
| 3 | GET | `http://localhost:3000` + valor de `Location` | none | — | 200, mismo `id` y mismos campos |
| 4 | GET | `http://localhost:3000/api/reservas/1` | none | — | 400 JSON |
| 5 | GET | `http://localhost:3000/api/reservas/000000000000000000000000` | none | — | 404 JSON |
| 6 | GET | `http://localhost:3000/api/reservas` + Params `sala` = `Sala Norte` | none | — | 200, solo Sala Norte |
| 7 | GET | igual, Params `sala` = `Sala Inexistente` | none | — | 200 y `[]` |
| 8 | GET | dos filas activas en Params: `sala` = `Sala Norte` y `sala` = `Sala Sur` | none | — | 400 JSON |
| 9 | POST | `http://localhost:3000/api/reservas` | raw → JSON, cambiando **un** dato por vez (ver matriz) | `Content-Type: application/json` | 400 JSON con `campos`; volver al GET 1: la cantidad no cambia |
| 10 | POST | igual | none | — | 400 JSON |
| 11 | POST | igual | raw con sintaxis incompleta, p. ej. `{"estudiante": "Ana"` | `Content-Type: application/json` | 400 JSON, sin traza |
| 12 | POST | igual | JSON válido + `"estado": "confirmada"` y `"_id": "000000000000000000000001"` | `Content-Type: application/json` | 201, `estado: "pendiente"` e `id` distinto del enviado (**sí crea**: la cantidad sube 1) |
| 13 | GET | `http://localhost:3000/api/no-existe` | none | — | 404 JSON |
| 14 | GET | `http://localhost:3000/estado` | none | — | 200, `reservas` igual a la cantidad del GET 1 |

Las pruebas del formulario y de las vistas se hacen en el navegador.

## 12. Matriz de resultados

Resultados observados el 23/10/2026 con el servidor local (`node --env-file=.env src/index.js`, puerto 3000) y la base `reservas_salas_u8`, que estaba **vacía** al comenzar. El registro completo está en [`evidencias/registro-pruebas.txt`](evidencias/registro-pruebas.txt). Cada paso numerado muestra método, URL, `Content-Type`, cuerpo enviado, estado, `Location` y respuesta. La columna **Registro** indica el paso o la sección que respalda cada fila. Las mismas solicitudes se reproducen en Postman con la colección de `postman/`.

| Caso | Esperado | Observado | Cantidad antes → después | Registro |
| --- | --- | --- | --- | --- |
| Listado vacío (base nueva) | 200, `[]` | 200, `[]` | 0 | paso 1 |
| Creación válida | 201 y Location | 201, `id` `6adade5f32675795bd1e0660`, `estado: "pendiente"`, `Location: /api/reservas/6adade5f32675795bd1e0660` | 0 → 1 | pasos 2-3 |
| Consulta por ID creado | 200, mismo recurso | 200, mismos campos que la respuesta del POST | — | paso 4 |
| Consulta tras reiniciar | 200, mismo ID | 200, mismo `id` y mismos campos; detalle HTML 200 | — | pasos 55-56 |
| ID "1" | 400 | 400 JSON `{"error":"El identificador debe tener 24 caracteres hexadecimales.","campos":[]}` | — | paso 5 |
| ID válido inexistente (24 ceros) | 404 | 404 JSON `{"error":"Reserva no encontrada."}` | — | paso 6 |
| Estudiante vacío | 400, sin aumento | 400, `campos: ["estudiante"]` | 1 → 1 | pasos 7-8 |
| Email sin @ (`ana.example.com`) | 400, sin aumento | 400, `campos: ["email"]` | 1 → 1 | pasos 9-10 |
| Sala no permitida (`Sala Este`) | 400, sin aumento | 400, `campos: ["sala"]` | 1 → 1 | pasos 11-12 |
| Turno no permitido (`Madrugada`) | 400, sin aumento | 400, `campos: ["turno"]` | 1 → 1 | pasos 13-14 |
| Fecha vacía | 400 | 400, `campos: ["fecha"]` | 1 → 1 | pasos 15-16 |
| Personas 0 | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 17-18 |
| Personas 7 | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 19-20 |
| Personas 2.5 | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 21-22 |
| Personas "abc" | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 23-24 |
| Personas true | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 25-26 |
| Personas arreglo `[2]` | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 27-28 |
| Personas objeto `{"cantidad":2}` | 400 | 400, `campos: ["personas"]` | 1 → 1 | pasos 29-30 |
| POST sin cuerpo (Body none) | 400 JSON | 400 JSON, `campos` con los seis campos | 1 → 1 | pasos 31-32 |
| JSON mal formado | 400 JSON | 400 JSON `{"error":"Revisá los datos enviados.","campos":[]}`, sin traza | 1 → 1 | pasos 33-34 |
| Campos extra `estado: "confirmada"` y `_id` | No los asigna el cliente | 201, `estado: "pendiente"`, `id` `6adade6032675795bd1e0661` (no el `000…001` enviado). Crea una reserva válida | 1 → 2 | pasos 35-36 |
| Filtro exacto `sala=Sala Norte` | Solo coincidencias | 200, 1 resultado, todos de Sala Norte | — | paso 37 |
| Filtro sin coincidencias (`Sala Inexistente`) | 200, `[]` | 200, `[]` | — | paso 38 |
| Sala repetida (`Sala Norte` y `Sala Sur`) | 400 | 400 JSON `{"error":"El parámetro sala debe enviarse una sola vez y con texto.","campos":[]}` | — | paso 39 |
| Listar dos veces | Cantidad estable | 200 y 200 | 2 y 2 | pasos 40-41 |
| Ruta API inexistente | 404 JSON | 404 JSON `{"error":"Ruta no encontrada."}` | — | paso 42 |
| Inicio, CSS y ruta inexistente | 200, 200 y 404 | 200, 200 y 404 (HTML "Página no encontrada") | — | pasos 43-45 |
| Listado y detalle HTML | Leen la base | Listado 200 con la reserva creada; detalle 200 con su `id` y estado pendiente | — | pasos 46-47 |
| Formulario válido | 302 | 302 a `/reservas`; la reserva (`id` `6adade6132675795bd1e0662`) aparece en la API y en el listado HTML | 2 → 3 | pasos 48-51 |
| Formulario inválido (9 personas) | 400, conserva valores | 400, el formulario vuelve con los valores enviados | 3 → 3 | pasos 52-53 |
| `/estado` | Conteo persistente | 200 `{"servicio":"activo","reservas":3,...}`, igual a la cantidad de la API | — | paso 54 |
| Cierre ordenado (Ctrl+C) | Cierra servidor y MongoDB | `Cerrando el servidor...` → `Conexión con MongoDB cerrada.`, código de salida 0 | — | sección I |
| Arranque sin `MONGODB_URI` | No abre el puerto | `No se pudo iniciar la aplicación: ErrorDeConfiguracion` / `Falta MONGODB_URI en el entorno.`, código 1, sin `Aplicación disponible`, puerto 3000 cerrado | — | sección J |
| Base inaccesible durante una operación | 500, sin traza | 500 JSON en `/api/reservas`, `/api/reservas/:id`, POST y `/estado`; 500 HTML en `/reservas`; sin detalles internos | — | sección L |
| `_id` en la colección | Mismo `_id` | `ObjectId("6adade5f32675795bd1e0660")` en `reservas_salas_u8.reservas`, con los mismos campos y `estado: "pendiente"` | 3 documentos | sección K |

## 13. Prueba de persistencia y reinicio

Pasos:

1. GET `/api/reservas` sin filtros para registrar la cantidad inicial.
2. Un solo POST válido. Guardar `id` y `Location`.
3. GET a la URL de `Location`: mismos datos y `estado: "pendiente"`.
4. Detener el servidor con Ctrl+C y volver a iniciarlo con el mismo comando (misma base).
5. Repetir el GET con el **mismo** `id`, sin repetir el POST, y ubicar la reserva en `/reservas` y en su detalle `/reservas/<id>`.
6. Listar dos veces: la cantidad no cambia.
7. Buscar el `_id` en `reservas_salas_u8` → `reservas` (Data Explorer de Atlas).

| Paso | Observado |
| --- | --- |
| Cantidad inicial | 0 (paso 1) |
| `id` creado | `6adade5f32675795bd1e0660` (paso 2) |
| `Location` | `/api/reservas/6adade5f32675795bd1e0660` (paso 2) |
| GET a `Location` antes de reiniciar | 200, mismo recurso (paso 4) |
| Cantidad antes de reiniciar | 3: la reserva del POST válido, la de campos extra y la del formulario (paso 53) |
| Cierre | `Cerrando el servidor...` → `Conexión con MongoDB cerrada.` (sección I) |
| `id` tras reiniciar | `6adade5f32675795bd1e0660`: 200 con los mismos campos (paso 55); detalle en el sitio 200 (paso 56) |
| Cantidad tras reiniciar (dos veces) | 3 y 3: no se perdió ni se duplicó nada (pasos 57-58) |
| `_id` en la colección | `ObjectId("6adade5f32675795bd1e0660")`, con los mismos campos (sección K) |

## 14. Comparación con el TP 07

| Aspecto | TP 07 | TP 08 |
| --- | --- | --- |
| Fuente de datos del sitio | Arreglo en memoria con 4 reservas semilla | MongoDB (`reservas_salas_u8`) |
| Persistencia al reiniciar | Se perdían las reservas del formulario | Se conservan |
| MongoDB | Solo scripts de terminal | Sitio, API y scripts usan el mismo modelo |
| Servicio | Síncrono, devolvía arreglos | Asíncrono (`async`/`await`), devuelve promesas |
| Identificador | Entero calculado con `reduce` (`/reservas/1`) | `ObjectId` expuesto como `id` textual |
| Arranque | `listen` inmediato | Conecta primero, después escucha; cierre ordenado |
| API JSON | Solo `/estado` | `/api/reservas` (listar, filtrar, consultar, crear) |
| Errores | Solo 404 de rutas | Manejador de errores 400/404/500, JSON en la API |
| Validación de tipos | `String(valor)` aceptaba arreglos y objetos | Solo strings y números; tipos raros → 400 |
| Campo `estado` | No existía | Default `pendiente`, no asignable por el cliente |

## 15. Datos sensibles

El repositorio no incluye `.env`, contraseñas, la URI real, `node_modules` ni datos personales reales: `.env` y `node_modules/` están en `.gitignore`, `.env.example` solo tiene marcadores, y las reservas de ejemplo usan datos ficticios. Este servidor es un ejercicio de aprendizaje: no tiene autenticación, autorización ni protección del formulario, por lo que no se publica en Internet.

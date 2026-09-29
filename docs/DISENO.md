# [NOMBRE] — Diseño técnico del MVP (propuesta para aprobación)

Estado: **BORRADOR — pendiente de aprobación**. No hay código de aplicación aún.

---

## 1. Stack propuesto

| Capa | Elección | Por qué |
|---|---|---|
| Lenguaje | TypeScript (estricto) | Pedido explícito; un solo lenguaje front/back. |
| App web | **Next.js (App Router, `output: "standalone"`)** + Server Actions | Un solo contenedor para UI + backend; SSR rápido en celulares de gama baja; imagen Docker pequeña con `standalone`. |
| UI | Tailwind CSS + componentes propios estilo shadcn/ui | Mobile-first sin pelear con CSS; botones grandes para uso con guantes/manos sucias. |
| Base de datos | **PostgreSQL 17** | Pedido explícito; RLS nativo para multi-tenant. |
| ORM / migraciones | **Drizzle ORM** + migraciones SQL versionadas | SQL explícito (necesario para políticas RLS, CHECKs y FKs compuestas); se integra bien con `SET LOCAL` por transacción. Prisma hace esto incómodo. |
| Validación | Zod | Mismo esquema en formulario y servidor; valida patente, RUT, montos y el contenido de los aportes técnicos. |
| Autenticación | Sesiones propias en BD (cookie `httpOnly`, token hasheado) + Argon2id | ~150 líneas, sin dependencia de un proveedor externo, control total de roles y taller activo. |
| Archivos (fotos) | Volumen Docker detrás de una interfaz `Almacenamiento` | Autoalojado sin servicios extra. Se puede cambiar a S3/compatible después sin tocar la lógica. Compresión en el cliente antes de subir (fotos de celular pesan 3–8 MB). |
| PDF | `@react-pdf/renderer` en servidor | Sin Chromium dentro del contenedor (Playwright/Puppeteer suman ~400 MB). |
| Tests | Vitest + PostgreSQL real (servicio en docker-compose de test) | Los tests de aislamiento multi-tenant **tienen** que correr contra Postgres real con RLS; un mock no prueba nada. |
| Despliegue | `docker-compose.yml`: `db`, `migrate` (one-shot), `app` | Compatible directo con Dokploy (Compose). |

---

## 2. Estrategia multi-tenant (tres capas)

1. **Capa de aplicación:** todo acceso a datos de taller pasa por `conTaller(sesion, async (tx) => ...)`, que abre una transacción y ejecuta `SELECT set_config('app.taller_id', $1, true)`. Los servicios nunca reciben `taller_id` desde el cliente; lo toman de la sesión.
2. **Row Level Security en PostgreSQL:** cada tabla del módulo 1 tiene `taller_id NOT NULL`, `ENABLE` + `FORCE ROW LEVEL SECURITY` y política `USING (taller_id = current_setting('app.taller_id')::uuid)` con el mismo `WITH CHECK`. La app se conecta con un rol **sin** `BYPASSRLS` y que **no** es dueño de las tablas; las migraciones usan otro rol. Si falta la variable de sesión, la consulta devuelve 0 filas (falla cerrado).
3. **FKs compuestas `(taller_id, id)`:** una OT del taller A no puede apuntar a un vehículo del taller B aunque alguien adivine el UUID. Lo garantiza la BD, no el código.

Tests obligatorios (ver §6): lectura, escritura, actualización, borrado y referencias cruzadas entre dos talleres, más “sin taller en sesión ⇒ 0 filas”.

---

## 3. Modelo de datos

Convenciones: PK `uuid` (v7), timestamps `timestamptz` en UTC (se muestran en `America/Santiago`, formato `dd-mm-aaaa`), montos en **CLP como `integer`** (sin decimales), cantidades en `numeric(10,2)`.

### 3.1 Identidad y talleres (global)

```
usuarios
  id, email (citext, único), nombre, password_hash,
  es_moderador boolean default false,          -- rol global
  activo, creado_en

talleres
  id, nombre, rut (único, nullable en MVP), razon_social, giro,
  direccion, comuna, region, telefono,
  verificado_en timestamptz null,              -- lo marca un moderador (ver §5 decisión D3)
  creado_en

membresias                                     -- rol dentro de un taller
  usuario_id, taller_id, rol enum('dueno','mecanico'), activo
  PK (usuario_id, taller_id)

sesiones
  id (sha256 del token), usuario_id, taller_activo_id null,
  expira_en, creado_en, user_agent
```

Roles:
- **dueño**: todo en su taller (usuarios, precios, inventario, presupuestos, ajustes).
- **mecánico**: ve y actualiza OTs, sube fotos, ve historial; no ve costos de repuestos ni gestiona usuarios. Puede aportar/confirmar en la base técnica.
- **moderador global**: flag en `usuarios`; modera la base técnica. **No** obtiene acceso a datos de talleres por ser moderador.

### 3.2 Módulo 1 — Gestión de taller (todas con `taller_id` + RLS)

```
clientes
  id, taller_id, nombre, rut null (validado módulo 11), telefono (E.164 +569...),
  email null, notas, creado_en
  UNIQUE (taller_id, rut)

vehiculos
  id, taller_id, cliente_id → clientes(taller_id,id),
  patente (normalizada: mayúsculas, sin guiones/espacios/puntos),
  marca, modelo, anio, motor, vin null, color null,
  variante_id null → tecnica_variantes(id)     -- puente al módulo 2 (opcional)
  km_ultimo int null, km_ultimo_en date null,
  creado_en
  UNIQUE (taller_id, patente)
  CHECK patente ~ '^([A-Z]{2}[0-9]{4}|[B-DF-HJ-LPR-TV-Z]{4}[0-9]{2})$'   -- ver §5 D1 (motos)

ordenes_trabajo
  id, taller_id, numero int (correlativo por taller),
  vehiculo_id → vehiculos(taller_id,id), cliente_id → clientes(taller_id,id),
  estado enum('recibido','diagnostico','esperando_repuesto','en_reparacion','listo','entregado','anulada'),
  mecanico_id null → membresias (debe ser del mismo taller),
  km_ingreso int, motivo_ingreso text, diagnostico text, observaciones text,
  fecha_ingreso, fecha_prometida null, fecha_entrega null,
  creado_por, creado_en
  UNIQUE (taller_id, numero)

ot_eventos                                     -- historial de estados (auditoría)
  id, taller_id, ot_id, estado_anterior, estado_nuevo, usuario_id, nota, creado_en

ot_fotos
  id, taller_id, ot_id, archivo_ruta, mime, bytes, descripcion, subido_por, creado_en

presupuestos
  id, taller_id, ot_id → ordenes_trabajo(taller_id,id), numero int,
  estado enum('borrador','enviado','aprobado','rechazado'),
  validez_dias int default 15, notas,
  neto int, iva int, total int,                -- congelados al pasar a 'enviado'
  aprobado_en null, creado_en
  UNIQUE (taller_id, numero)

presupuesto_items
  id, taller_id, presupuesto_id, orden int,
  tipo enum('mano_obra','repuesto','otro'),
  descripcion, cantidad numeric(10,2), precio_unitario_neto int,
  descuento_pct numeric(5,2) default 0,
  repuesto_id null → repuestos(taller_id,id)

repuestos
  id, taller_id, sku, nombre, marca null, codigo_oem null,
  stock numeric(10,2) default 0, stock_minimo numeric(10,2) default 0,
  costo_neto int null, precio_venta_neto int null, ubicacion null, activo
  UNIQUE (taller_id, sku)

movimientos_inventario                         -- stock = suma de movimientos; `repuestos.stock` es caché
  id, taller_id, repuesto_id, tipo enum('entrada','salida','ajuste'),
  cantidad numeric(10,2), motivo, ot_id null, usuario_id, creado_en

recordatorios
  id, taller_id, vehiculo_id, ot_origen_id null,
  descripcion, fecha_objetivo date null, km_objetivo int null,
  estado enum('pendiente','enviado','completado','descartado'),
  enviado_en null, creado_en
  CHECK (fecha_objetivo IS NOT NULL OR km_objetivo IS NOT NULL)

contadores                                     -- numeración correlativa sin huecos por taller
  taller_id, tipo enum('ot','presupuesto'), ultimo int
  PK (taller_id, tipo)
```

**Historial por vehículo** = OTs + eventos + presupuestos aprobados + fotos + recordatorios, filtrados por `vehiculo_id`. No se duplica en otra tabla.

**Recordatorio por km:** el sistema no conoce el km actual del auto. Se estima con el promedio km/día de las OTs anteriores del vehículo y se muestra como “posiblemente vencido (estimado)”. El envío es un enlace `https://wa.me/569XXXXXXXX?text=...` que el usuario toca; no hay envío automático (eso requeriría la API de WhatsApp Business, fuera del MVP).

### 3.3 Preparación SII (tablas e interfaz, sin implementación)

```
documentos_tributarios
  id, taller_id, ot_id null, presupuesto_id null,
  tipo_dte smallint,                           -- 33 factura, 39 boleta, 61 NC, 56 ND
  folio int null, estado enum('pendiente','emitido','rechazado','anulado'),
  receptor_rut, receptor_razon_social, neto, iva, exento, total,
  proveedor text null, track_id null, respuesta jsonb null, emitido_en null
```

```ts
// src/server/facturacion/proveedor.ts
interface ProveedorDTE {
  emitir(doc: DocumentoTributario): Promise<ResultadoEmision>;
  consultarEstado(trackId: string): Promise<EstadoDTE>;
  anular(doc: DocumentoTributario, motivo: string): Promise<ResultadoEmision>;
}
// Única implementación en el MVP: ProveedorNoConfigurado (lanza error explícito).
```

### 3.4 Módulo 2 — Base técnica colaborativa (global, sin RLS por taller)

Catálogo (lo curan moderadores; los mecánicos pueden *proponer* entradas nuevas):

```
tecnica_marcas     id, nombre (único), slug
tecnica_modelos    id, marca_id, nombre, slug           UNIQUE (marca_id, slug)
tecnica_motores    id, marca_id, codigo, cilindrada_cc, combustible, aspiracion, notas
tecnica_variantes  id, modelo_id, motor_id, anio_desde, anio_hasta, version null, estado enum('propuesta','activa')
                   -- una "ficha" = una variante (marca / modelo / años / motor)
```

Aportes (inmutables: corregir = crear un aporte nuevo que reemplaza al anterior):

```
tecnica_aportes
  id, variante_id,
  tipo enum('torque','fluido','intervalo','falla','procedimiento'),
  contenido jsonb,                               -- validado por Zod según tipo (abajo)
  codigo_dtc text GENERATED (contenido->>'codigo_dtc') STORED   -- índice para búsqueda
  fuente enum('medido_en_terreno','manual_fabricante_propio','etiqueta_vehiculo','otro'),
  fuente_detalle text null,                      -- obligatorio si fuente = 'otro'
  autor_id, taller_id (taller del autor al momento de aportar),
  reemplaza_a_id null → tecnica_aportes(id),
  verificado_por null, verificado_en null,       -- moderador
  oculto boolean, oculto_motivo,                 -- moderación
  creado_en

tecnica_validaciones                             -- UN voto por TALLER por aporte
  id, aporte_id, usuario_id, taller_id,
  tipo enum('confirma','reporta_incorrecto'),
  comentario text null,                          -- obligatorio si reporta
  resuelto_por null, resuelto_en null,           -- moderador resuelve reportes
  creado_en
  UNIQUE (aporte_id, taller_id)
  -- regla: el taller autor no puede votar su propio aporte

tecnica_fotos
  id, aporte_id, paso int null, archivo_ruta, mime, bytes, subido_por, creado_en

moderacion_log
  id, moderador_id, entidad, entidad_id, accion, motivo, creado_en
```

Contenido por tipo (solo estructura, **sin valores**):

| tipo | campos |
|---|---|
| torque | componente, valor, unidad (`Nm`), etapas[] (p. ej. valor + ángulo), secuencia_apriete, notas |
| fluido | sistema (aceite motor, refrigerante, caja, diferencial, frenos, dirección, A/C), capacidad, unidad (`L`), especificacion, con_filtro |
| intervalo | item, cada_km, cada_meses, condicion (`normal`/`severa`) |
| falla | codigo_dtc null, sintoma, causa_encontrada, solucion, km_vehiculo null |
| procedimiento | titulo, herramientas[], pasos[] {orden, texto, foto_id?}, advertencias |

**Nivel de validación** (calculado, se muestra SIEMPRE junto al valor):

1. `verificado_moderador` → si `verificado_en` no es nulo y no hay reportes abiertos.
2. `disputado` → hay ≥1 reporte `reporta_incorrecto` sin resolver (advertencia roja).
3. `confirmado (N talleres)` → N = talleres distintos que confirmaron, excluyendo el taller autor.
4. `sin_verificar` → advertencia amarilla visible: “Dato no verificado. Compruébelo antes de usarlo.”

Un aporte reemplazado queda visible en el historial pero marcado “reemplazado”; sus confirmaciones **no** se heredan.

---

## 4. Estructura del proyecto

```
.
├─ docker-compose.yml            # db + migrate + app (producción / Dokploy)
├─ docker-compose.test.yml       # Postgres efímero para tests de integración
├─ Dockerfile                    # multi-stage, Next standalone
├─ .env.example
├─ drizzle.config.ts
├─ drizzle/                      # migraciones SQL (incluye RLS, roles, CHECKs)
├─ scripts/
│  ├─ seed-demo.ts               # datos DEMO obviamente ficticios
│  └─ crear-moderador.ts
├─ src/
│  ├─ app/
│  │  ├─ (publico)/login, registro-taller
│  │  ├─ (taller)/inicio, ordenes, ordenes/[id], vehiculos, vehiculos/[patente],
│  │  │           clientes, presupuestos/[id], inventario, recordatorios, equipo, ajustes
│  │  ├─ (tecnica)/fichas, fichas/[marca]/[modelo]/[variante], aportar
│  │  ├─ moderacion/
│  │  └─ api/archivos/[id], api/presupuestos/[id]/pdf
│  ├─ server/
│  │  ├─ db/{cliente.ts, conTaller.ts, esquema/{identidad,taller,tecnica,sii}.ts}
│  │  ├─ auth/{sesion.ts, password.ts, permisos.ts}
│  │  ├─ taller/{clientes,vehiculos,ordenes,presupuestos,inventario,recordatorios}.ts
│  │  ├─ tecnica/{catalogo,aportes,validaciones,moderacion}.ts
│  │  ├─ almacenamiento/{interfaz.ts, disco.ts}
│  │  ├─ pdf/presupuesto.tsx
│  │  └─ facturacion/{proveedor.ts, no-configurado.ts}
│  ├─ lib/{patente.ts, rut.ts, formato.ts (CLP, dd-mm-aaaa), whatsapp.ts, iva.ts}
│  └─ components/
└─ tests/
   ├─ unit/        # patente, RUT, montos/IVA, cálculo nivel de validación, permisos por rol
   └─ integracion/ # aislamiento multi-tenant contra Postgres real
```

---

## 5. Decisiones que necesito que confirmes

- **D1 Patentes de motos** (`AA123`, `ABC12`): ¿solo autos en el MVP o aceptar también motos?
- **D2 Descuento de inventario:** propongo descontar stock al pasar la OT a `entregado`, por los repuestos de presupuestos aprobados vinculados a inventario. Alternativa: al aprobar el presupuesto.
- **D3 Anti-manipulación de validaciones:** propongo que solo cuenten para “confirmado por N” los votos de talleres **verificados por un moderador** (RUT revisado). Sin esto, una persona crea 3 talleres y “confirma” su propio dato.
- **D4 Umbral** para mostrar “confirmado”: propongo N ≥ 2 talleres; con N = 1 se muestra “1 taller confirma” pero con la advertencia de no verificado.
- **D5 IVA:** precios se ingresan netos y el presupuesto muestra neto + IVA 19% + total. ¿O tus clientes esperan precios con IVA incluido?
- **D6 Registro de talleres:** ¿autoregistro abierto o solo por invitación tuya en el MVP?

## 6. Tests multi-tenant planificados

Con dos talleres (A y B) y usuarios de cada uno, contra Postgres real con RLS:
- A no lista, no obtiene por id, no actualiza, no borra clientes/vehículos/OTs/presupuestos/repuestos/fotos/recordatorios de B.
- A no puede crear una OT apuntando a un vehículo de B (FK compuesta).
- A no puede asignar como mecánico a un usuario de B.
- Sin taller en la sesión de BD ⇒ 0 filas en todas las tablas del módulo 1.
- El rol de conexión de la app no tiene `BYPASSRLS` ni es dueño de las tablas.
- Misma patente en A y B ⇒ dos vehículos independientes; búsqueda en A nunca devuelve el de B.
- Un mecánico de A no puede acceder a archivos (fotos) de B por URL directa.
- Un moderador sin membresía no ve datos de ningún taller.
- Base técnica: un taller no puede confirmar dos veces el mismo aporte ni confirmar el propio.

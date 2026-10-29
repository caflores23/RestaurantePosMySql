# RestaurantePos

Punto de venta para restaurante. .NET 10 · Blazor (Interactive Server) · MudBlazor 9 · EF Core + **MySQL 8**.

Esta es la versión con MySQL del proyecto (la de SQLite vive en la carpeta `RestaurantePos`). Los espacios de nombres
y las pantallas son los mismos; cambian el proveedor de base de datos, las migraciones y los scripts de `scripts/mysql`.

## Estructura

| Proyecto | Qué contiene |
|---|---|
| `RestaurantePos.Domain` | Entidades y reglas (Usuario, Zona, Mesa, Reserva, Cuenta, Producto…) |
| `RestaurantePos.Contracts` | DTOs compartidos entre API y Web |
| `RestaurantePos.Infrastructure` | `PosDbContext` (MySQL), migraciones de EF Core, datos de prueba, hash de PIN |
| `RestaurantePos.Api` | API REST con JWT: login por PIN, zonas, mesas y cuenta de mesa |
| `RestaurantePos.Web` | Blazor Interactive Server con MudBlazor: acceso con PIN y plano de mesas |

## Base de datos MySQL

Requiere MySQL 8.0 o superior (las columnas calculadas y `utf8mb4_0900_ai_ci` no existen en 5.7). Hay dos caminos;
elige uno:

**A) Con los scripts SQL** (`scripts/mysql`, para un servidor que administras tú):

```bash
mysql -u root -p < scripts/mysql/00-crear-base.sql      # base `restaurante_pos` y usuario `pos_app` (cambia la clave antes)
mysql -u pos_app -p < scripts/mysql/01-esquema.sql      # tablas, llaves, índices y columnas calculadas
mysql -u pos_app -p < scripts/mysql/02-datos-semilla.sql # usuarios, catálogo, salón y datos de demostración
```

**B) Con la aplicación:** crea la base y el usuario con `00-crear-base.sql` y arranca la API. Al iniciar aplica las
migraciones (`MigrateAsync`) y, solo en `Development`, carga los datos de prueba si la base está vacía.

La cadena de conexión es `ConnectionStrings:Pos` de la API. En `appsettings.Development.json` viene un ejemplo con la
clave `CAMBIA_ESTA_CLAVE`; en la práctica define la tuya con user-secrets o con la variable `ConnectionStrings__Pos`:

```bash
dotnet user-secrets set "ConnectionStrings:Pos" "Server=localhost;Port=3306;Database=restaurante_pos;User=pos_app;Password=TU_CLAVE;CharSet=utf8mb4;" --project src/RestaurantePos.Api
```

Los scripts se generan, no se escriben a mano. Tras cambiar el modelo:

```bash
dotnet .tools/dotnet-ef migrations add NombreDelCambio --project src/RestaurantePos.Infrastructure --startup-project src/RestaurantePos.Api
pwsh scripts/mysql/regenerar.ps1   # vuelve a escribir 01-esquema.sql y 02-datos-semilla.sql (no necesita MySQL)
```

(la primera vez, `regenerar.ps1` instala `dotnet-ef` en `.tools/`). Las fechas del script de semilla están escritas
respecto a `NOW()`, así que la demostración (cuentas abiertas, reservas de "en 15 min") sale vigente el día que lo ejecutes.

Diferencias respecto a la versión SQLite:
- MySQL no tiene índices filtrados. Las reglas "una sola cuenta abierta por mesa" y "un solo turno abierto por caja" se
  resuelven con una columna calculada (`MesaAbiertaId`, `CajaAbierta`) que vale NULL cuando la cuenta/turno se cierra,
  más un índice único (los índices únicos de MySQL ignoran los NULL).
- El proveedor es `MySql.EntityFrameworkCore` de Oracle: Pomelo aún no publica una versión para EF Core 10.
- Los textos son `varchar(250)` (con límites menores donde se indexan) porque MySQL no indexa `longtext`.
- El dinero sigue guardándose como centavos (`bigint`).

## Cómo correrlo

Se levantan dos procesos (en dos terminales):

```bash
dotnet run --project src/RestaurantePos.Api --launch-profile http
```

```bash
dotnet run --project src/RestaurantePos.Web --launch-profile http
```

- API: http://localhost:5035 (`/api/health`)
- Web: http://localhost:5254

En `Development`, si la base está vacía, la API carga datos de prueba (`DbSeeder.cs`). Los PINs de prueba están en ese
archivo. Para empezar de cero: `DROP DATABASE restaurante_pos;` y vuelve a crearla con `00-crear-base.sql`.

## Configuración

- `Jwt:Key` (API): mínimo 32 caracteres. En Development viene una clave de desarrollo;
  en producción se define con user-secrets o variable de entorno `Jwt__Key`.
- `Pos:TasaImpuesto` (API): tasa sobre el subtotal, por defecto `0.15`.
- `Api:BaseUrl` y `Restaurante:Nombre` / `Restaurante:Caja` (Web).

## Estado

Hecho:
- Acceso con PIN (con bloqueo tras 5 intentos fallidos).
- Plano de mesas con estados y detalle de cuenta.
- Toma de pedido: abrir mesa con comensales, catálogo por categorías con búsqueda, agregar/quitar productos,
  notas para cocina, "enviar a cocina" (las líneas enviadas quedan bloqueadas y con su hora).

También hecho:
- Cobro y división de cuenta: completa, por persona, por producto o por monto; propina (10/15/20% u otra);
  efectivo (con cambio), tarjeta y transferencia. La cuenta se cierra al saldarse y la mesa queda libre.

- Cocina (KDS) en vivo: al enviar un pedido se crea una comanda por estación (Parrilla, Fríos, Barra);
  la pantalla `/cocina` las muestra en Nuevas → En preparación → Listas, con cronómetro y alertas de
  atención/retraso, filtro por estación y aviso de alergias. Se actualiza por SignalR sin recargar.

- Cierre de caja: turno con fondo inicial; todo pago cuelga del turno abierto (sin turno no se cobra).
  El cierre muestra ventas, cuentas, ticket promedio, propinas y ventas por método, y pide el arqueo de
  billetes y monedas: compara contra el efectivo esperado, exige una nota si hay diferencia y avisa de
  cuentas abiertas o comandas sin entregar antes de cerrar.

- Descuentos, cortesías y cancelaciones (cajero y gerente, siempre con motivo y con registro de quién y cuándo):
  descuento a toda la cuenta (% o monto; el cajero hasta `Pos:DescuentoMaximoCajero`, 20% por defecto, y arriba de
  eso solo un gerente), cortesía de una línea o de algunas unidades, cancelación de un producto ya enviado a
  cocina (cocina lo ve tachado y suena una alerta) y anulación de toda la cuenta. El descuento se aplica antes del
  impuesto y se reparte entre los pagos por producto; una cuenta sin saldo se cierra sola. El cierre de caja
  muestra las incidencias del turno.
- Avisos: en cocina suena una alerta al llegar una comanda, al cancelarse un producto y cuando una comanda cruza
  el umbral de retraso (botón "Sonido" para silenciar; el navegador exige un primer toque en la pantalla). Cuando
  cocina marca una comanda como lista, el mesero dueño de la cuenta recibe un aviso en su tableta (SignalR dirigido
  por usuario, con sonido) y la mesa muestra una campana en el plano.
- Impresión: ticket de cuenta (pre-cuenta con propina sugerida), recibo de la cuenta cobrada, corte parcial (X)
  y corte final (Z) con arqueo y diferencia. Se muestra una vista previa y se imprime con el diálogo del
  navegador, así que sirve con cualquier impresora instalada en el equipo (térmicas de 80 mm incluidas).
- Reservas (uso interno, sin reserva en línea): pantalla `/reservas` (botón "Reservas" de la barra superior) con la
  lista por día, estados (pendiente, atrasada, ya se sentaron, cancelada, no se presentó) y formulario con nombre,
  teléfono, fecha, hora, personas, duración, mesa y notas. El formulario consulta qué mesas están libres para ese
  horario y detecta choques (dos reservas de la misma mesa no pueden encimarse; una que empieza cuando otra termina
  sí). "Sentar" abre la cuenta de la mesa con los comensales que llegaron y deja la reserva cumplida, así la mesa no
  vuelve a salir como reservada al cobrar. En el plano, una reserva solo aparta la mesa cerca de su hora
  (30 min antes y hasta 15 min después); si la mesa sigue ocupada, muestra una etiqueta con la hora de la reserva.

Pendiente: integración con terminal de tarjeta, impresión directa a impresora térmica (ESC/POS, sin diálogo) e
impresión de comandas de cocina, autorización por PIN de gerente para cajeros y meseros, historial de turnos
cerrados, fuentes locales para uso sin internet. De las reservas faltan avisos (por ejemplo al llegar la hora),
juntar mesas para grupos grandes, horario de apertura y recordatorios al cliente.

## API del pedido

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/catalogo` | Categorías con sus productos activos |
| POST | `/api/mesas/{mesaId}/cuenta` | Abre la cuenta de una mesa (`{ comensales }`) |
| POST | `/api/cuentas/{id}/lineas` | Agrega un producto (suma a la línea "por enviar" igual) |
| PUT | `/api/cuentas/{id}/lineas/{lineaId}` | Cambia cantidad y nota (cantidad 0 elimina) |
| DELETE | `/api/cuentas/{id}/lineas/{lineaId}` | Elimina una línea aún no enviada |
| POST | `/api/cuentas/{id}/enviar` | Marca como enviadas a cocina las líneas pendientes |

Las escrituras exigen rol Mesero, Cajero o Gerente. Una línea ya enviada no se puede editar ni borrar.

## API del cobro

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/mesas/{mesaId}/cobro` | Estado de cobro: productos con unidades pagadas, pagos, pagado y pendiente |
| POST | `/api/cuentas/{id}/por-cobrar` | Pide la cuenta (congela el pedido). Exige todo enviado a cocina |
| POST | `/api/cuentas/{id}/reabrir` | Vuelve al pedido; solo si todavía no hay pagos |
| POST | `/api/cuentas/{id}/pagos` | Registra un pago: `monto` (completa/persona/monto libre) **o** `lineas` (por producto) |

Reglas: la propina no puede superar el monto; el efectivo recibido debe cubrir el total; el último pago
absorbe los centavos de redondeo; no se puede mezclar "por producto" con "por monto/persona" salvo para
cobrar el resto completo. El consecutivo de pagos es único por cuenta, así que dos tabletas cobrando a la vez
no pueden duplicar un pago. Los cálculos (`CalculoCobro`) viven en `Contracts` y los usan la API y la pantalla.

## API de ajustes y tickets

| Método | Ruta | Qué hace |
|---|---|---|
| PUT | `/api/cuentas/{id}/descuento` | Aplica un descuento (`tipo`: Porcentaje/Monto/Ninguno, `valor`, `motivo`) |
| POST | `/api/cuentas/{id}/lineas/{lineaId}/cortesia` | Cortesía de la línea o de `cantidad` unidades (`motivo`) |
| DELETE | `/api/cuentas/{id}/lineas/{lineaId}/cortesia` | Quita la cortesía |
| POST | `/api/cuentas/{id}/lineas/{lineaId}/cancelar` | Cancela una línea ya enviada (`motivo`, `cantidad` opcional) |
| POST | `/api/cuentas/{id}/anular` | Anula la cuenta completa sin cobro (`motivo`) |
| GET | `/api/cuentas/{id}/ticket` | Datos del ticket de la cuenta (cuenta, recibo o anulada) |

Los ajustes exigen rol Cajero o Gerente y solo se pueden hacer mientras la cuenta no tenga pagos.
Las cancelaciones y cortesías parciales dividen la línea. El ticket no incluye las líneas canceladas.

## Facturación: clientes, CAI y facturas

Tres tablas: `Clientes`, `Cai` y `Facturas`.

- `Clientes`: `Id`, `Rtn` (único, 14 dígitos), `Nombre`, `RazonSocial`, `Direccion`, `Telefono`, `CorreoElectronico`.
  El cliente es **quien emite las facturas** (el restaurante dueño del CAI).
- `Cai`: `Id`, `ClienteId`, `NumeroCai` (único, con guiones), `RangoInicial` y `RangoFinal` (texto con el formato de la
  factura, `000-001-01-00000001`), `FechaValidaInicial` y `FechaValidaFinal` (columnas `date`).
- `Facturas`: una por cuenta cobrada. Guarda el número (`000-001-01-00000001`), el CAI, el comprador (por ahora
  "Consumidor final") y los importes con los que se emitió.

**Mantenimiento (gerente):** menú Administración → Facturación (`/facturacion`): alta, edición y baja de clientes y de CAI,
con el estado de cada CAI (vigente, aún no vigente, vencido o agotado), cuántas facturas lleva y cuántos números quedan.
Reglas: el RTN no se repite; el número de CAI va con guiones y no se repite; los rangos tienen el formato de la factura, los
dos extremos empiezan igual (`000-001-01-`) y el final no es menor que el inicial; dos CAI no pueden autorizar el mismo
número; un CAI con facturas no se edita ni se borra, y un cliente con CAI no se borra.

**Cómo se factura:**
- La facturación se activa sola cuando el cliente emisor tiene al menos un CAI. Mientras no, las cuentas se cobran con
  recibo, como antes.
- Ya activa, la cuenta que se salda al registrar su último pago toma el siguiente número del CAI **vigente** (dentro de sus
  fechas y con números disponibles; si hay varios, el que vence primero) y queda ligada a él. Si un CAI se agota, pasa al
  siguiente. El número y el CAI se guardan junto con el pago, en una sola transacción.
- Sin CAI vigente la cuenta **no se puede cerrar**: al pedir la cuenta y al registrar el pago la API responde con el motivo,
  y la pantalla de facturación muestra el aviso. Un gerente debe registrar o renovar el CAI.
- Dos cuentas que se cierran a la vez no pueden tomar el mismo número (candado en la API e índice único en la base).
- El impuesto de la factura es el de la cuenta (`Pos:TasaImpuesto`, 15%).
- El recibo se imprime como **FACTURA**: razón social, RTN, dirección y teléfono del emisor, número de factura, comprador,
  y al pie el CAI, el rango autorizado y la fecha límite de emisión.
- Con más de un cliente registrado hay que indicar cuál emite en `Pos:ClienteEmisorId` (API); con uno solo se usa ese.

Pendiente: capturar el RTN y el nombre del comprador al cobrar (hoy sale "Consumidor final"), anular facturas y reportes
fiscales. Si tu base ya existía, la API aplica las migraciones sola al arrancar; también están los scripts de
`scripts/mysql/cambios/` (aplícalos en orden). En una base nueva, `01-esquema.sql` ya incluye todo.

## Mantenimiento del menú

Solo el Gerente (botón "Menú" de la barra superior, pantalla `/menu`): agrega, edita, desactiva y borra categorías y
platillos (nombre, precio, categoría, estación de cocina que lo prepara, etiqueta como "Popular" o "Picante", y si
está disponible). Un platillo desactivado deja de aparecer al tomar pedidos; los cambios se ven la próxima vez que
un mesero abre un pedido. Un cambio de precio no toca las cuentas ya armadas (cada línea guarda el precio con el que
se vendió). Un platillo con ventas no se puede borrar, solo desactivar; una categoría solo se borra si está vacía.
No se repiten nombres (sin distinguir mayúsculas ni acentos) y el precio se redondea a centavos.

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/admin/menu` | Categorías con todos sus platillos (también los desactivados) y si tienen ventas |
| POST / PUT / DELETE | `/api/admin/menu/categorias[/{id}]` | Crea, renombra o borra una categoría vacía |
| POST / PUT / DELETE | `/api/admin/menu/productos[/{id}]` | Crea, edita (incluye activar/desactivar) o borra un platillo sin ventas |

Cada operación responde con el menú completo ya actualizado. Pendiente: reordenar categorías y platillos.

## API de reservas

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/reservas?fecha=yyyy-MM-dd` | Reservas del día (hoy por defecto), todos los estados, por hora |
| GET | `/api/reservas/mesas?inicio=…&duracion=…&personas=…&excluir=` | Todas las mesas con su disponibilidad para ese horario (`excluir` = reserva que se edita) |
| POST | `/api/reservas` | Crea una reserva (`mesaId`, `fechaHora`, `nombre`, `telefono`, `personas`, `notas`, `duracionMinutos`) |
| PUT | `/api/reservas/{id}` | Edita una reserva pendiente; con `version` (la que vio quien edita) devuelve 409 si otra tableta la cambió antes |
| POST | `/api/reservas/{id}/sentar` | Abre la cuenta de la mesa (`comensales`, por defecto las personas de la reserva) y la deja cumplida; con `mesaEsperadaId` devuelve 409 si la reserva cambió de mesa |
| POST | `/api/reservas/{id}/no-se-presento` | El cliente no llegó; solo cuando ya es la hora |
| POST | `/api/reservas/{id}/cancelar` | Cancela con `motivo` obligatorio; solo Cajero y Gerente |

Crear, editar, sentar y "no se presentó": Mesero, Cajero y Gerente. Reglas: 1 a 30 personas y no más que la
capacidad de la mesa; duración de 30 a 240 minutos (90 por defecto); una hora pasada se acepta hasta 15 minutos
atrás y hasta 120 días hacia adelante; se sientan reservas de hoy o que caen en la ventana del plano (aunque
crucen la medianoche) y solo con la mesa libre. Altas, ediciones, sentar, cancelar y "no se presentó" se serializan
en la API y compiten por el estado de la reserva: si dos tabletas actúan a la vez, la segunda recibe un 409.
Cocina no accede a `/api/reservas` y el plano que recibe no trae datos de reservas.
Las horas son hora local del servidor: la API y la Web deben correr en la zona horaria del restaurante.
En `Pos:` (API) se ajustan `ReservaDuracionMinutos` (90; es la duración para quien llame a la API sin indicarla, el
formulario propone 90 y deja elegir de 1 a 3 horas), `ReservaAnticipacionMinutos` (30, cuándo el plano empieza a
marcar la mesa como reservada) y `ReservaToleranciaMinutos` (15, cuánto se guarda la mesa después de la hora).
Las migraciones de EF Core mantienen la tabla de reservas al día al arrancar la API.

## API de cocina y tiempo real

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/cocina/comandas` | Comandas nuevas, en preparación y listas (las más viejas primero) |
| POST | `/api/cocina/comandas/{id}/iniciar` | Nueva → En preparación |
| POST | `/api/cocina/comandas/{id}/lista` | Nueva o En preparación → Lista |
| POST | `/api/cocina/comandas/{id}/entregar` | Lista → Entregada (sale de la pantalla) |
| Hub | `/hubs/cocina` | SignalR: evento `ComandasCambiaron` (la pantalla recarga por REST) |
| Hub | `/hubs/salon` | SignalR dirigido al mesero dueño de la cuenta: evento `ComandaLista` |

Solo los roles Cocina y Gerente las ven y las mueven. El usuario "Cocina" entra directo a `/cocina`; el Gerente
llega con el botón "Cocina" de la barra superior. El token JWT viaja en la consulta (`access_token`) solo para
el hub, porque los WebSockets no permiten encabezados; en producción conviene servir todo por HTTPS.
Los umbrales de atención y retraso se configuran en `Cocina:AtencionMinutos` / `Cocina:RetrasoMinutos` (Web).

## API de caja (turno)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/api/turno` | Resumen del turno abierto (404 si no hay): ventas, métodos, efectivo esperado, pendientes |
| POST | `/api/turno/abrir` | Abre el turno con un fondo inicial |
| POST | `/api/turno/cerrar` | Cierra con el arqueo (`conteo`), `nota` y `confirmarPendientes` |

Solo Cajero y Gerente (el botón "Cierre de caja" aparece en la barra superior). Reglas: un solo turno abierto
por caja; el servidor recalcula todo al cerrar y valida las denominaciones; si el efectivo contado difiere del
esperado, la nota es obligatoria; si quedan cuentas abiertas o comandas sin entregar, hay que confirmarlo.
El efectivo esperado es fondo + ventas en efectivo + propinas en efectivo (las propinas de tarjeta y
transferencia no están en el cajón). Las denominaciones (`Pos:Denominaciones`) y el nombre de la caja
(`Pos:Caja`) tienen valores por defecto en la API; no definas `Denominaciones` en appsettings porque un
arreglo configurado se suma al de por defecto.

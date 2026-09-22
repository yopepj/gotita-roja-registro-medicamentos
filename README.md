# Sistema de registro de pacientes y entrega de medicamentos donados

Aplicación web ASP.NET Core MVC + Entity Framework Core + SQL Server.

Reemplaza el formulario de Google y la planilla de Excel con los que la
institución viene registrando las donaciones de medicamentos desde que la
gobernación dejó de permitir que se archive la receta en papel.

---

## Qué resuelve

La receta médica debe devolverse al paciente, así que la institución necesita
conservar un respaldo digital para responder ante una auditoría. El
procedimiento provisional —escanear receta y carnet en cada visita y volcar todo
a Excel— duplica los datos de los pacientes que vuelven y consume el
almacenamiento en la nube más rápido de lo previsto.

El sistema registra a cada paciente **una sola vez**. Cada visita posterior
agrega una entrega a su historial, con su receta escaneada, sin volver a pedir
el carnet.

La instalación inicial está pensada para una ONG con dos usuarias: una PC
principal ejecuta la aplicación, SQL Server Express y el almacenamiento de
documentos; la segunda computadora trabaja desde el navegador a través de la
red privada de la institución. SQL Server y las carpetas nunca se comparten
directamente.

---

## Requisitos

| Componente | Versión mínima | Notas |
|---|---|---|
| SQL Server | 2019 | Express alcanza: el límite de 10 GB es holgado |
| .NET SDK | 8.0 | Es LTS; con soporte hasta noviembre de 2026 |
| Visual Studio | 2022 (17.8+) | Opcional: sirve `dotnet` por línea de comandos |
| SSMS | Cualquiera | Para ejecutar los scripts de la carpeta `BaseDeDatos` |

> Si tenés instalado sólo el SDK 9, funciona igual: el SDK 9 compila proyectos
> `net8.0` siempre que esté presente el *targeting pack* de .NET 8, que Visual
> Studio 2022 instala por defecto. Si tu Visual Studio se queja, instalá el
> [SDK de .NET 8](https://dotnet.microsoft.com/download/dotnet/8.0) o cambiá
> `<TargetFramework>` a `net9.0` en `SistemaDonaciones.Web.csproj`.

---

## Puesta en marcha

### 1. Crear la base de datos

Abrir SSMS y ejecutar, **en este orden**. Los tres primeros van siempre, también
en la instalación real de la institución; del cuarto en adelante son datos de
demostración y **no deben ejecutarse en producción**.

1. `BaseDeDatos/01-esquema.sql`
   Las primeras líneas crean la base `DonacionMedicamentos` si no existe y la
   seleccionan con `USE`. Crea tablas, restricciones, índices, triggers,
   vistas, procedimientos almacenados y los catálogos base (sexo, países,
   condiciones de atención, tipos de ingreso, unidades de medida y roles).

   **Este script es el esquema completo.** En una máquina nueva no hace falta
   ejecutar nada más para tener la estructura definitiva.

2. `BaseDeDatos/02-catalogo-territorial.sql`
   Carga los 9 departamentos, 112 provincias y 340 municipios con sus códigos
   INE. Sin esto los desplegables de procedencia del paciente quedan vacíos y
   las estadísticas por departamento y municipio no muestran nada.

3. `BaseDeDatos/03-catalogos-operativos.sql`
   Diagnósticos con código CIE-10, establecimientos de salud y unidades de
   medida adicionales. **No es opcional:** `Atencion.IdDiagnostico` es
   `NOT NULL`, así que sin diagnósticos cargados no se puede registrar ni una
   sola entrega. La lista es editable desde Administración; los códigos CIE-10
   son un punto de partida y conviene que alguien del área de salud los valide
   antes de usarlos en informes oficiales.

En este punto la base ya está lista para operar. **Iniciar la aplicación una
vez** para que cree el usuario administrador antes de seguir.

4. *(Sólo demostración)* `BaseDeDatos/04-datos-demo-farmacia.sql`
   Medicamentos y 31 lotes: el caso de costeo FIFO con dos precios que planteó
   la cliente, un medicamento con tres lotes a tres precios, lotes agotados,
   vencidos, por vencer y un ingreso anulado con su motivo.

5. *(Sólo demostración)* `BaseDeDatos/05-datos-demo-pacientes.sql`
   81 pacientes y 131 atenciones repartidas en los últimos 14 meses. Está
   calibrado para que se vean funcionando el origen contra la residencia, los
   pacientes del exterior, la dirección desglosada y los cinco colores de la
   escala del mapa. El encabezado del script trae la tabla de lo que debería
   pintar el mapa, para contrastarla contra la pantalla.

6. *(Sólo demostración)* `BaseDeDatos/06-limpiar-datos-demo.sql`
   Selecciona los datos mediante las marcas reservadas `DEMO-*`, rutas `demo/`
   y documentos de siete dígitos en la serie `99xxxxx`. No toca catálogos ni
   usuarios. Antes de ejecutarlo en una base compartida hay que revisar los
   conteos y confirmar que ninguna ficha real utilice esas marcas, además de
   disponer de un respaldo.

### 2. Configurar la conexión

En `SistemaDonaciones.Web/appsettings.json`:

```json
"ConnectionStrings": {
  "BaseDatos": "Server=localhost\\SQLEXPRESS;Database=DonacionMedicamentos;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=True"
}
```

Con usuario y contraseña de SQL Server en lugar de autenticación de Windows:

```
Server=MI-SERVIDOR;Database=DonacionMedicamentos;User Id=sa;Password=...;TrustServerCertificate=True;MultipleActiveResultSets=True
```

> **No dejar contraseñas en `appsettings.json` cuando el proyecto esté en
> producción o en un repositorio.** Usar *User Secrets* en desarrollo
> (primero `dotnet user-secrets init --project SistemaDonaciones.Web` y luego
> `dotnet user-secrets set "ConnectionStrings:BaseDatos" "..." --project SistemaDonaciones.Web`)
> y variables de entorno en el servidor.

### 3. Ejecutar

Desde Visual Studio: abrir `SistemaDonaciones.sln` y presionar F5. Desde la
línea de comandos:

```powershell
Set-Location SistemaDonaciones.Web
dotnet restore
dotnet run
```

Con la configuración incluida, la aplicación atiende en **`http://localhost:5000`**
desde la propia PC y en **`http://IP-LOCAL:5000`** desde otra computadora de la
red. Es así con cualquier perfil: la clave `Urls` de `appsettings.json` tiene
prioridad sobre la dirección que declara cada perfil en `launchSettings.json`,
de modo que `http`, `https` y `red-local` escuchan en el mismo lugar y el perfil
`https` no sirve HTTPS. Lo único que distingue a `red-local` es que muestra la
página de error que vería la usuaria en vez de la traza. Para volver a las
direcciones de cada perfil, ver *Arquitectura local y trabajo desde dos
computadoras* en las notas de despliegue.

Si Windows solicita permiso de red, habilitar únicamente redes privadas.

En el primer arranque, si la base no tiene ningún usuario, la aplicación crea el
administrador con los valores de la sección `UsuarioInicial`. Si esa sección no
está configurada, usa los predeterminados:

| Usuario | Contraseña |
|---|---|
| `admin` | `Cambiar.2026` |

Esa contraseña es pública —figura en el código fuente—, así que **hay que
cambiarla en el primer ingreso**, desde el menú del usuario → *Cambiar
contraseña*. Para que la instalación no arranque nunca con ella, definir
`UsuarioInicial:Contrasena` con *User Secrets* o una variable de entorno antes
del primer inicio.

Para ver en desarrollo la página de error que verá la usuaria en producción
(sin trazas ni detalles internos), agregar `"Errores": { "MostrarDetalle": false }`
a la configuración. Sin esa clave, en Development se sigue mostrando la traza.

---

## Estructura del proyecto

```
SistemaDonaciones/
├── BaseDeDatos/
│   ├── 01-esquema.sql              Estructura completa y al día (fuente de verdad)
│   ├── 02-catalogo-territorial.sql Departamentos, provincias y municipios
│   ├── 03-catalogos-operativos.sql Diagnósticos CIE-10 y establecimientos
│   ├── 04-datos-demo-farmacia.sql  Demostración: medicamentos y lotes
│   ├── 05-datos-demo-pacientes.sql Demostración: pacientes y atenciones
│   └── 06-limpiar-datos-demo.sql   Borra sólo lo de la demostración
│
└── SistemaDonaciones.Web/
    ├── Data/
    │   └── AppDbContext.cs         Mapeo Database First al esquema
    ├── Infraestructura/            Búsqueda, decimales, escala del mapa, red local
    ├── Models/
    │   ├── Entidades/              Mapeo de las columnas usadas de tablas y vistas
    │   └── ViewModels/             Modelos de las pantallas
    ├── Servicios/
    │   ├── ServicioContrasenas.cs  PBKDF2, sin dependencias externas
    │   ├── ServicioArchivos.cs     Recetas y carnets, con deduplicación
    │   ├── ServicioCatalogos.cs    Normaliza el texto libre
    │   ├── ServicioPacientes.cs    Alta, búsqueda y ficha
    │   ├── ServicioAtenciones.cs   Alta de entregas (transaccional)
    │   ├── ServicioFarmacia.cs     Inventario y costeo FIFO
    │   ├── ServicioReportes.cs     Estadísticas y exportación
    │   ├── ServicioBitacora.cs     Registro de auditoría de los cambios
    │   ├── ServicioBackup.cs       Copia de seguridad de la base (.bak)
    │   ├── ServicioBackupSegundoPlano.cs  La ejecuta cada N minutos
    │   └── InicializadorDatos.cs   Verificación y usuario inicial
    ├── Controllers/
    ├── Views/
    └── wwwroot/
```

---

## Decisiones de diseño que conviene poder explicar

### Database First, no migraciones

El esquema se escribió a mano y es la fuente de verdad. Las clases de C#
describen lo que ya existe. Se decidió así porque el esquema usa recursos que
las migraciones automáticas de Entity Framework no saben generar: triggers,
columnas calculadas, índices filtrados y procedimientos almacenados. Dejar que
EF genere la base habría significado renunciar a todo eso.

Consecuencia práctica: **no ejecutar `Add-Migration` ni `Update-Database`**. Los
cambios de estructura se hacen en `01-esquema.sql` y se reflejan a mano en
`AppDbContext`.

### La edad no se guarda

`Paciente.Edad` es una columna calculada no persistida. Guardar la edad
violaría la tercera forma normal —es un dato derivado de la fecha de
nacimiento— y además quedaría desactualizada en cada cumpleaños. Se recalcula
en cada lectura.

### Un archivo idéntico no se guarda dos veces

`ArchivoDigital.HashSha256` es único. Antes de escribir nada al disco, el
sistema calcula el SHA-256 del archivo subido y busca ese hash. Si ya está,
reutiliza el registro existente. Es la defensa concreta contra el problema que
originó el proyecto: el mismo carnet escaneado en cada visita.

### Sin receta escaneada no hay entrega

`Atencion.IdArchivoReceta` es `NOT NULL`. La regla más dura del sistema vive
como restricción del motor, no como una validación que se pueda olvidar en el
código.

### Nada se borra

`TR_Atencion_ImpedirBorrado` es un trigger `INSTEAD OF DELETE` que convierte
cualquier intento de borrado en un cambio de estado a `ANULADA`. Un archivo que
respalda una auditoría no puede tener huecos.

### Costeo FIFO, no promedio

La cliente lo pidió textualmente: un medicamento que en 2013 costó 50 Bs y hoy
cuesta 150 Bs no puede valuarse por promedio, porque da un monto irreal para sus
estadísticas. Cada ingreso es un lote con su propio costo, y las entregas
consumen del más antiguo al más nuevo.

El reparto lo hace `sp_ConsumirLotesFifo` **sin cursor**: una suma corrida
(`SUM() OVER`) dice cuánto stock hay acumulado hasta cada lote, y con eso se
resuelve en un solo `INSERT` qué lotes participan y cuánto sale de cada uno.

### Autenticación propia, no ASP.NET Core Identity

El esquema ya define `Usuario` y `Rol`. Identity habría agregado siete tablas
propias que duplican esas dos, ensuciando el modelo entidad-relación, que es
justamente lo que se evalúa. Se usan cookies de autenticación con derivación
PBKDF2-HMAC-SHA256 de la biblioteca base de .NET: sin paquetes de terceros, una
dependencia menos que mantener.

### Los archivos no se sirven como estáticos

Recetas y carnets viven fuera de `wwwroot` y se entregan a través de
`ArchivosController`, que exige sesión iniciada. Si estuvieran en `wwwroot`,
cualquiera con la URL podría ver documentos médicos.

---

## Reglas comunes de validación y búsqueda

- **Importes y cantidades** se aceptan con coma o con punto decimal
  (`12,34` y `12.34` valen lo mismo; `1.234,56` y `1,234.56` también). Lo
  resuelve `Infraestructura/EnlazadorDecimal.cs`, y el navegador aplica la
  misma regla en `wwwroot/js/validacion-es.js`. Máximo dos decimales.
- **Búsquedas** (pacientes, entregas, farmacia, autocompletado): sin distinguir
  mayúsculas ni tildes; cada palabra escrita tiene que coincidir con el
  comienzo del carnet o de un nombre o apellido (en medicamentos y números de
  formulario, con cualquier parte). Un criterio sin letras ni números (por
  ejemplo, sólo un emoji) no devuelve nada. Reglas en
  `Infraestructura/CriterioBusqueda.cs` y en `sp_BuscarPaciente`.
- **Formulario de paciente**: el teclado sólo deja escribir lo que el servidor
  acepta (dígitos en el carnet; letras, espacios, apóstrofo y guion en nombres
  y apellidos). Filtro en `site.js`, atributo `data-filtro`; la expresión
  regular del modelo es la misma. En el alta, el país de origen viene
  preseleccionado en Bolivia: una ficha sin país queda fuera del filtro
  «Bolivia» de las estadísticas.
- **Fechas**: no se aceptan fechas futuras de nacimiento, atención ni ingreso
  de lote, ni años anteriores a 1900 (nacimiento) o 2000 (atención e ingreso).
- **Número de formulario**: el campo muestra una vista previa; el número
  definitivo se asigna al guardar, dentro de la transacción.

---

## Notas de despliegue

### Dónde guardar los archivos escaneados

La sección `Almacenamiento:RutaRaiz` de `appsettings.json` define la carpeta
raíz. En desarrollo es una carpeta relativa al proyecto; en el servidor conviene
una ruta absoluta en un volumen con respaldo:

```json
"Almacenamiento": {
  "RutaRaiz": "C:\\SistemaDonaciones\\Datos\\Archivos",
  "TamanoMaximoMb": 10
}
```

La carpeta debe estar **fuera** del directorio publicado y **debe incluirse en
la copia de seguridad**: la base guarda la ruta y el hash, no el binario.

### Arquitectura local y trabajo desde dos computadoras

La decisión vigente para la primera etapa es una instalación local en Windows:

- **PC principal:** aplicación publicada, SQL Server Express y carpeta de
  documentos.
- **PC operadora:** sólo navegador y una cuenta propia del sistema.
- **Router:** mantiene ambos equipos en la misma red privada; no publica ningún
  puerto hacia Internet.

La configuración incluida viene preparada para ese esquema: la aplicación
atiende a toda la red privada por HTTP en el puerto 5000, y la PC operadora
entra con `http://IP-DE-LA-PC-PRINCIPAL:5000`. Intervienen dos partes de
`appsettings.json`:

```json
"Urls": "http://0.0.0.0:5000",

"RedLocal": {
  "Habilitada": true,
  "UrlPublica": "",
  "Puerto": 5000,
  "PermitirHttpSinTls": true
}
```

- **`Urls`** es lo que abre la aplicación a la red: `0.0.0.0` significa
  escuchar en todas las interfaces del equipo, no sólo en `localhost`.
- **`RedLocal:Habilitada`** muestra en el tablero la dirección que hay que
  pasarle a la operadora. Con `UrlPublica` vacía la arma sola a partir de la
  IPv4 privada del equipo y de `Puerto`.
- **`RedLocal:PermitirHttpSinTls`** permite iniciar sesión por HTTP. **Va
  siempre junto con `Habilitada`:** fuera de modo Desarrollo, si falta alguna
  de las dos la cookie de sesión se emite como segura, el navegador la descarta
  sobre HTTP y el ingreso falla sin ningún mensaje de error.

Si la PC principal cambia de IP, la dirección cambia con ella. Conviene
reservarle una IP fija en el router (reserva DHCP) o poner en `UrlPublica` el
nombre del equipo, por ejemplo `http://PC-GOTITA:5000`.

HTTP sirve en una red controlada, pero **no cifra contraseñas, cookies ni datos
médicos**: cualquiera conectado a la misma red —incluido el wifi de invitados si
comparte segmento— puede leerlos. Por eso:

- No abrir el puerto 5000 en el router ni habilitar conexiones remotas hacia
  SQL Server.
- Cuando Windows pida permiso para la aplicación, permitirla **sólo en redes
  privadas**, y dejar la regla del Firewall limitada a la PC operadora.
- El paso siguiente es HTTPS: una URL estable como `https://PC-GOTITA:5443` en
  `Urls` y `UrlPublica`, un certificado confiable configurado en Kestrel, y
  `PermitirHttpSinTls` en `false`.

Para que la aplicación vuelva a atender sólo en la PC principal, quitar la
línea `Urls` y poner `Habilitada` y `PermitirHttpSinTls` en `false`.

Una eventual migración a nube queda para una fase posterior. Si la base se muda
a Azure SQL, hay que activar `EnableRetryOnFailure` en `Program.cs` **y**
envolver las transacciones de `ServicioAtenciones` y `ServicioPacientes` en
`CreateExecutionStrategy().ExecuteAsync(...)`. Además hay que desactivar
`Backup:Habilitado`: Azure SQL no admite `BACKUP DATABASE ... TO DISK` y trae
sus propias copias automáticas.

### Copias de seguridad

Dos cosas que respaldar, y ninguna sirve sin la otra:

1. La base de datos.
2. La carpeta de archivos escaneados.

**La base se respalda sola.** Un servicio en segundo plano ejecuta
`BACKUP DATABASE` cada 30 minutos y deja el `.bak` en
`<RutaRaiz>/bd_backups`, conservando los 10 más recientes. También se puede
generar uno a mano desde **Administración → Copias de seguridad**, que lista
los existentes. Se configura en `appsettings.json`:

```json
"Backup": {
  "Habilitado": true,
  "IntervaloMinutos": 30,
  "MaximoArchivos": 10
}
```

Tres condiciones para que funcione:

- **SQL Server tiene que estar en la misma PC que la aplicación.** El `.bak`
  lo escribe el propio SQL Server en *su* disco; la pantalla lista el disco de
  la aplicación. Es el caso de la instalación prevista.
- **La cuenta del servicio de SQL Server** (por lo general
  `NT Service\MSSQL$SQLEXPRESS`) necesita permiso de escritura sobre esa
  carpeta. Sin él, cada intento falla con *acceso denegado* y queda sólo en el
  registro de la aplicación.
- Se hace una copia al arrancar, antes de esperar el primer intervalo.

Esos `.bak` están en la misma máquina que la base: protegen contra un error de
carga, no contra la rotura del disco ni un robo. **Copiar periódicamente
`bd_backups` y la carpeta de escaneados a un disco externo o a otro equipo.**

La pantalla **Reportes → Respaldos exportados** deja constancia de cada copia
que salga de la institución: qué período abarcaba, cuándo se hizo y quién la
hizo.

> **El archivo exportado cambió de columnas.** El 12/09/2026, donde antes había
> `Departamento; Provincia; Municipio`, pasaron a ser seis: las tres del lugar
> de origen y las tres de la residencia, con el nombre puesto. El 13/09 se
> agregó `Pais de origen` delante de ellas, que en las filas de pacientes
> extranjeros es lo único que trae la procedencia. En total, las columnas
> posteriores se corrieron cuatro lugares respecto del formato del 27/08. Si
> alguien armó una planilla de Excel sobre el formato anterior, hay que rehacer
> las referencias.

---

## Puntos que quedaron pendientes de confirmar con la cliente

Surgieron durante el relevamiento de requerimientos. El sistema hoy resuelve
cada uno con una decisión razonable, pero conviene validarlas:

| Punto | Cómo está resuelto hoy |
|---|---|
| Login con usuario y contraseña | Implementado, con roles Administrador y Operador |
| Permisos por usuario | Operador registra y consulta; Administrador además gestiona usuarios, catálogos, bitácora y copias de seguridad |
| Formatos permitidos | JPG, PNG, WEBP y PDF, hasta 10 MB |
| Numeración del formulario | Vista previa correlativa anual (`2026-0001`), de sólo lectura; el valor definitivo se reserva al guardar |
| Reportes estadísticos | Por sexo, condición y grupo etario; por medicamento; tablero de porcentajes y mapa de calor por departamento |
| Exportación a Excel | CSV con BOM UTF-8, que Excel abre directamente |
| Farmacia en la primera versión | Incluida, con costeo FIFO |
| Trabajo desde dos computadoras | Habilitado en la configuración incluida, por HTTP en el puerto 5000 dentro de la red privada; HTTPS pendiente |
| Proveedor de nube | No aplica en la primera etapa: SQL Server Express y archivos locales; se evaluará más adelante |

---

## Equipo de desarrollo

Sistema donado a la Asociación Benéfica Gotita Roja. Para consultas sobre el
código, la instalación o el mantenimiento:

| # | Nombre | Correo |
|:-:|---|---|
| 1 | Julián A. Cruz Justiniano | hooasoiio@gmail.com |
| 2 | Leonardo Zeballos Valda | leonardo587933@gmail.com |
| 3 | Yimy Serrano Palacios | yimysp@gmail.com |
| 4 | Josue Mujica Cachicatari | josuelegion1@gmail.com |

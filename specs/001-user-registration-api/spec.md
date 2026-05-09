# Feature Specification: User Registration API

**Feature Branch**: `001-user-registration-api`  
**Created**: 2026-05-05  
**Status**: Draft  
**Input**: User description: "Como desarrollador requiero crear una API de registro de usuarios para almacenar la informacion de los mismos. Como criterios de aceptacion: El API debe ser con el context path /users, el API debe ser un tipo POST que recibe un JSON con los siguientes datos del usuario: nombre, apellido, direccion, telefono, correo. El servicio debe almacenar los datos del usuario en memoria, y el API debe responder los mismos datos del usuario con un ID de tipo UUID autogenerado. Validar que el correo no sea repetido (debe ser unico). Validar que los campos nombre, apellido, direccion, telefono y correo sean obligatorios; si algun campo requerido no se envia debera retornar un error 400 indicando que campo falta. Si el servicio funciona, debe retornar un status 201; si el usuario ya se encuentra creado debe retornar un 400 con un mensaje de usuario existente."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registrar un nuevo usuario (Priority: P1)

Un consumidor del servicio (por ejemplo, una aplicación cliente o un sistema externo) envía la información de un nuevo usuario — nombre, apellido, dirección, teléfono y correo — al endpoint de registro. El sistema valida los datos, almacena el usuario y devuelve la información registrada junto con un identificador único generado por el sistema.

**Why this priority**: Es el caso de uso completo y único de esta entrega. Sin él no hay funcionalidad; con él la entrega es viable como MVP.

**Independent Test**: Enviar una petición POST a `/users` con un cuerpo JSON que contenga los cinco campos requeridos y verificar que la respuesta tenga estado 201, que el cuerpo de respuesta contenga los mismos cinco campos y un identificador UUID válido recién generado, y que una consulta posterior con el mismo correo sea rechazada como duplicada.

**Acceptance Scenarios**:

1. **Given** que no existe ningún usuario registrado con el correo `ana@example.com`, **When** un cliente envía POST `/users` con `{nombre, apellido, direccion, telefono, correo: "ana@example.com"}` válidos, **Then** el sistema responde con estado 201 y un cuerpo JSON que contiene los cinco campos enviados más un campo `id` con un UUID generado.
2. **Given** que ya existe un usuario con correo `ana@example.com`, **When** un cliente envía POST `/users` con un cuerpo cuyo correo es `ana@example.com` (en cualquier combinación de mayúsculas/minúsculas), **Then** el sistema responde con estado 400 y un mensaje que indica que el usuario ya existe.
3. **Given** un cuerpo de petición al que le falta el campo `telefono`, **When** un cliente envía POST `/users`, **Then** el sistema responde con estado 400 e indica explícitamente que el campo faltante es `telefono`.
4. **Given** un cuerpo de petición al que le faltan varios campos requeridos, **When** un cliente envía POST `/users`, **Then** el sistema responde con estado 400 y enumera todos los campos requeridos que faltan.
5. **Given** un registro exitoso de usuario, **When** se realiza una segunda petición con el mismo correo pero con nombre, apellido, dirección o teléfono diferentes, **Then** el sistema responde con estado 400 y mensaje de usuario existente (la unicidad se determina por correo, no por la combinación de campos).

---

### Edge Cases

- ¿Qué ocurre si un campo requerido se envía como cadena vacía o solo espacios en blanco? Debe tratarse como ausente y disparar la respuesta 400 con el nombre del campo.
- ¿Qué ocurre si el cuerpo de la petición no es JSON válido o falta por completo? Debe responder 400 con un mensaje claro de error de formato.
- ¿Qué ocurre si dos peticiones intentan registrar el mismo correo de forma concurrente? Solo una debe quedar registrada; la segunda debe responder 400 (usuario existente).
- ¿Qué ocurre si el correo se envía con diferencias de mayúsculas/minúsculas o espacios al inicio/fin (`"  Ana@Example.com  "` vs `"ana@example.com"`)? Debe normalizarse y tratarse como duplicado.
- ¿Qué ocurre si el cuerpo incluye campos adicionales no esperados? Deben ignorarse; la respuesta solo refleja los cinco campos definidos más el `id`.
- ¿Qué ocurre si el correo no tiene un formato válido (por ejemplo, sin `@`)? Debe responder 400 indicando que el correo no es válido.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST exponer un endpoint para el registro de un nuevo usuario sobre el recurso `/users` que acepte una petición POST con cuerpo JSON.
- **FR-002**: El cuerpo de la petición MUST aceptar exactamente los siguientes cinco campos del usuario: `nombre`, `apellido`, `direccion`, `telefono`, `correo`.
- **FR-003**: El sistema MUST tratar los cinco campos del FR-002 como obligatorios; un valor ausente, nulo o compuesto únicamente por espacios en blanco se considera ausente.
- **FR-004**: Si uno o más campos obligatorios están ausentes, el sistema MUST responder con estado HTTP 400 e incluir en la respuesta el nombre exacto de cada campo faltante.
- **FR-005**: El sistema MUST validar que el campo `correo` cumpla un formato de correo electrónico válido; si no lo cumple, MUST responder con estado HTTP 400 indicando que el correo no es válido.
- **FR-006**: El sistema MUST generar un identificador único de tipo UUID para cada usuario registrado exitosamente; el cliente NUNCA proporciona el identificador.
- **FR-007**: El sistema MUST persistir los datos del usuario registrado en almacenamiento en memoria durante el tiempo de vida del proceso del servicio.
- **FR-008**: El sistema MUST garantizar que el campo `correo` sea único entre todos los usuarios registrados, comparando de forma insensible a mayúsculas/minúsculas y sin espacios envolventes.
- **FR-009**: Si el correo enviado ya existe (según la regla del FR-008), el sistema MUST responder con estado HTTP 400 y un mensaje que indique que el usuario ya existe.
- **FR-010**: Cuando el registro es exitoso, el sistema MUST responder con estado HTTP 201 y un cuerpo JSON que contenga los cinco campos enviados por el cliente y el identificador `id` generado.
- **FR-011**: El sistema MUST manejar peticiones concurrentes con el mismo correo de modo que solo una quede registrada; las demás MUST recibir la respuesta de usuario existente del FR-009.
- **FR-012**: El sistema MUST ignorar los campos adicionales no contemplados en FR-002 sin fallar la petición; la respuesta del FR-010 MUST contener únicamente los seis campos definidos (`nombre`, `apellido`, `direccion`, `telefono`, `correo`, `id`).
- **FR-013**: El sistema NO debe registrar (en logs centralizados ni en la respuesta) el contenido de los campos personales del usuario más allá de lo necesario para depuración mínima; los logs cruzados de la operación deben permitir trazabilidad por `id` o por correlación, sin exponer todos los datos personales en texto claro.

### Key Entities

- **User**: Representa a una persona registrada en el sistema.
  - `id` — identificador único del usuario, generado por el sistema (UUID). Inmutable tras la creación.
  - `nombre` — nombre del usuario. Obligatorio.
  - `apellido` — apellido del usuario. Obligatorio.
  - `direccion` — dirección física del usuario. Obligatoria.
  - `telefono` — número de contacto telefónico. Obligatorio.
  - `correo` — dirección de correo electrónico. Obligatoria, única entre todos los usuarios, almacenada de forma normalizada para la comparación de unicidad.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las peticiones POST a `/users` con los cinco campos requeridos válidos y un correo no registrado previamente reciben una respuesta con estado 201 y un cuerpo que incluye los cinco campos enviados más un `id` de formato UUID.
- **SC-002**: El 100% de las peticiones POST a `/users` a las que les falta uno o más campos requeridos reciben una respuesta con estado 400 que enumera explícitamente cada campo faltante por su nombre.
- **SC-003**: El 100% de las peticiones POST a `/users` con un correo ya registrado reciben una respuesta con estado 400 y un mensaje que comunica claramente que el usuario ya existe.
- **SC-004**: Después de N registros exitosos consecutivos, una consulta sobre el almacenamiento en memoria devuelve exactamente N usuarios distintos, cada uno con un `id` UUID único y un correo único (insensible a mayúsculas/minúsculas).
- **SC-005**: Bajo una ráfaga de peticiones concurrentes con el mismo correo, queda registrado exactamente un usuario y todas las demás peticiones reciben la respuesta de usuario existente con estado 400.
- **SC-006**: Un nuevo desarrollador puede ejecutar el endpoint manualmente desde una herramienta como Postman o `curl` y verificar los tres flujos principales (éxito, campo faltante, correo duplicado) en menos de 5 minutos siguiendo la documentación del feature.

## Assumptions

- El almacenamiento en memoria es aceptable para esta entrega; los datos NO necesitan sobrevivir reinicios del proceso (criterio de aceptación explícito del usuario).
- La unicidad del correo se aplica de forma insensible a mayúsculas/minúsculas y con `trim` de espacios envolventes (estándar de la industria para correos electrónicos).
- "Campo obligatorio" abarca: ausente del JSON, presente con valor `null` y presente con cadena vacía o únicamente espacios en blanco.
- No se requiere autenticación ni autorización para este endpoint (registro abierto); cualquier control de acceso será un requisito separado.
- El formato del campo `telefono` no está restringido más allá de "presente y no vacío" — no se valida un patrón específico (E.164 u otro) en esta entrega.
- La respuesta de error 400 se devuelve con un cuerpo JSON estructurado y consistente; el formato exacto del cuerpo se decidirá durante la fase de diseño técnico, pero MUST permitir al cliente identificar (a) el código de error y (b) los nombres de los campos involucrados.
- El endpoint debe seguir las prácticas de versionado mandatadas por la constitución del proyecto (Principio III); el "context path `/users`" provisto por el usuario describe el recurso lógico — la ruta efectiva en producción incluirá el segmento de versión (por ejemplo, `/api/v1/users`).
- La observabilidad (logs de entrada/salida, duración, excepciones) se implementará vía AOP a nivel de plataforma (Principio IV de la constitución) y NO se incluye como requisito funcional adicional aquí.

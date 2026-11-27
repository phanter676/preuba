MANUAL DE OPERACIÓN Y ARQUITECTURA DEL CÓDIGO
Sistema de Monitoreo de Cámaras IP – EstadoCámaras
Proyecto: EstadoCámaras
Organización: LosMatus
Tipo de sistema: Aplicación web de monitoreo y administración de cámaras IP
Versión del documento: 1.0
Estado: Implementación funcional en desarrollo
Audiencia: Desarrolladores backend, frontend, administradores de sistemas y personal técnico.
________________________________________
1. VISIÓN GENERAL Y ARQUITECTURA DEL SISTEMA
1.1. Propósito principal
EstadoCámaras es un sistema web diseñado para administrar y monitorear cámaras IP dentro de una infraestructura de videovigilancia.
Su objetivo principal es permitir que el personal responsable pueda conocer el estado de sus cámaras, identificar posibles interrupciones de conectividad, consultar el historial de eventos y recibir notificaciones cuando se detecten situaciones que requieran atención.
El sistema realiza verificaciones periódicas de los dispositivos registrados mediante diferentes mecanismos:
•	Ping: comprueba si la dirección IP responde a solicitudes ICMP.
•	TCP: comprueba si el puerto configurado de la cámara acepta conexiones.
•	MAC: obtiene la dirección física del dispositivo mediante la tabla ARP del sistema operativo y la compara con la dirección MAC registrada.
A partir de estos resultados, el sistema determina el estado de cada cámara, actualiza la información almacenada y registra los cambios relevantes.
Adicionalmente, permite administrar usuarios, roles y técnicos responsables de recibir las notificaciones.
1.2. Funcionalidades principales
Módulo	Responsabilidad
Autenticación	Registro, inicio de sesión y generación de tokens JWT.
Gestión de cámaras	Registrar, consultar, actualizar, retirar y restaurar cámaras.
Monitoreo	Ejecutar verificaciones de conectividad y disponibilidad.
Historial	Registrar los cambios de estado y eventos relacionados con MAC.
Técnicos	Administrar los destinatarios de las notificaciones.
Notificaciones	Enviar alertas mediante WhatsApp, Telegram o consola.
Dashboard	Mostrar cámaras, estadísticas, estados e historial.
1.3. Stack tecnológico
Backend
Tecnología	Uso
Java 17	Lenguaje de programación.
Spring Boot 4.1.0	Framework principal.
Spring Web MVC	Exposición de API REST.
Spring Data JPA	Persistencia y consultas a la base de datos.
Hibernate	Implementación ORM y administración del esquema.
Spring Security	Autenticación y autorización.
JWT 0.12.6	Administración de tokens de acceso.
MySQL	Base de datos relacional.
Maven	Gestión de dependencias y construcción.
Lombok	Reducción de código repetitivo.
MapStruct	Dependencia de mapeo; el proyecto también contiene mappers manuales.
Frontend
Tecnología	Uso
React 19	Construcción de interfaces.
JavaScript	Lógica del cliente.
Vite 8	Servidor de desarrollo y compilación.
CSS	Diseño y presentación visual.
oxlint	Análisis estático del código frontend.
Servicios externos
•	WhatsApp Cloud API de Meta.
•	Telegram Bot API.
•	MySQL como almacenamiento persistente.
•	Servicios nativos del sistema operativo para ejecutar ping y consultar ARP.
1.4. Arquitectura general
El proyecto utiliza una arquitectura monolítica en el backend, organizada mediante capas y paquetes funcionales.
El frontend está separado en un directorio independiente, pero consume los servicios REST proporcionados por Spring Boot.
No se trata de una arquitectura de microservicios: el monitoreo, la autenticación, la administración y las notificaciones se ejecutan dentro de la misma aplicación backend.
Diagrama de arquitectura
                 USUARIO
                    |
                    v
          +--------------------+
          |   FRONTEND REACT   |
          |                    |
          | Dashboard          |
          | Cámaras            |
          | Técnicos           |
          | Historial          |
          | Autenticación      |
          +--------------------+
                    |
                    | HTTP / JSON
                    | JWT
                    v
          +--------------------+
          |   SPRING BOOT      |
          |                    |
          | Controllers        |
          | Services           |
          | Security           |
          | DTO / Mappers      |
          +--------------------+
                    |
          +---------+----------+
          |                    |
          v                    v
 +------------------+  +------------------+
 |   REPOSITORIES   |  | MONITOREO         |
 |                  |  |                  |
 | CamaraRepository |  | PingVerificador  |
 | TecnicoRepository|  | TcpVerificador   |
 | HistorialRepo    |  | MacVerificador   |
 | UsuarioRepository|  | EvaluarEstado    |
 +------------------+  +------------------+
          |                    |
          v                    v
 +------------------+  +------------------+
 |      MYSQL       |  | RED DE CÁMARAS   |
 |                  |  |                  |
 | cámaras          |  | IP               |
 | usuarios         |  | Puerto TCP       |
 | técnicos         |  | MAC / ARP        |
 | historial        |  +------------------+
 +------------------+
                              |
                              v
                    +--------------------+
                    |   NOTIFICACIONES   |
                    |                    |
                    | WhatsApp           |
                    | Telegram           |
                    | Consola            |
                    +--------------------+
Responsabilidad de cada capa
Controller: recibe las peticiones HTTP, obtiene los parámetros y delega las operaciones a los servicios.
Service: contiene la lógica de negocio y coordina las operaciones entre repositorios y otros componentes.
Repository: proporciona acceso a la base de datos mediante Spring Data JPA.
Entity: representa las entidades persistentes de MySQL.
DTO: define los objetos utilizados para recibir solicitudes y devolver respuestas.
Mapper: transforma entidades en DTO y viceversa.
Security: controla la autenticación y autorización mediante JWT y roles.
Monitoreo: concentra los mecanismos de verificación de conectividad y evaluación de estados.
Notificador: abstrae los distintos canales de comunicación para enviar alertas.
________________________________________
2. ESTRUCTURA DEL REPOSITORIO Y DIRECTORIOS
2.1. Estructura general
El repositorio está organizado de la siguiente manera:
camaraProyecto-main/
│
├── README.md
│
├── estadoCamaras/
│   ├── pom.xml
│   ├── .env.example
│   ├── mvnw
│   ├── mvnw.cmd
│   │
│   ├── db/
│   │   └── migrate_empresa_to_single_company.sql
│   │
│   └── src/
│       ├── main/
│       │   ├── java/
│       │   │   └── com/losMatus/estadoCamaras/
│       │   │       │
│       │   │       ├── config/
│       │   │       ├── controller/
│       │   │       ├── dto/
│       │   │       ├── entity/
│       │   │       ├── enums/
│       │   │       ├── mapper/
│       │   │       ├── monitoreo/
│       │   │       ├── notificador/
│       │   │       ├── repository/
│       │   │       ├── security/
│       │   │       ├── service/
│       │   │       │
│       │   │       └── EstadoCamarasApplication.java
│       │   │
│       │   └── resources/
│       │       ├── application.properties
│       │       └── static/
│       │
│       └── test/
│           └── java/
│
└── frontend/
    ├── package.json
    ├── package-lock.json
    ├── vite.config.js
    ├── index.html
    │
    ├── public/
    │
    └── src/
        ├── api/
        ├── components/
        ├── context/
        ├── styles/
        ├── utils/
        ├── App.jsx
        └── main.jsx
2.2. Backend: organización de paquetes
config/
Contiene las configuraciones generales de Spring.
Archivos principales:
•	CorsConfig.java: configuración de acceso entre orígenes.
•	SecurityConfig.java: definición de las reglas de seguridad, autenticación y autorización.
Cuando se requiera modificar las políticas de acceso, se debe revisar principalmente SecurityConfig.
controller/
Contiene los controladores REST que exponen los endpoints.
Archivo	Responsabilidad
AuthController.java	Registro e inicio de sesión.
CamaraController.java	Operaciones relacionadas con cámaras.
TecnicoController.java	Administración de técnicos.
ApiExceptionHandler.java	Tratamiento centralizado de determinados errores REST.
Los controladores no deberían contener operaciones complejas de negocio. Su responsabilidad es recibir solicitudes y delegar el procesamiento.
dto/
Contiene los objetos de transferencia de datos.
Ejemplos:
•	CamaraRequestDTO: datos necesarios para registrar o actualizar una cámara.
•	CamaraResponseDTO: información devuelta al cliente.
•	HistorialEstadoResponseDTO: representación del historial.
•	LoginRequestDTO: credenciales de inicio de sesión.
•	RegisterRequestDTO: datos de registro.
•	AuthResponseDTO: respuesta de autenticación.
•	TecnicoRequest: datos para registrar o modificar técnicos.
•	TecnicoResponse: información devuelta sobre técnicos.
•	ApiErrorResponse: estructura de errores de la API.
Se utilizan DTO para evitar exponer directamente las entidades de persistencia.
entity/
Contiene las entidades JPA que representan las tablas de la base de datos.
Camara.java
Representa una cámara registrada.
Campos importantes:
•	id
•	nombre
•	ip
•	puerto
•	ubicacion
•	macEsperada
•	ultimaMacDetectada
•	fallosConsecutivos
•	estado
•	ultimaVerificacion
•	eliminada
HistorialEstado.java
Representa un evento registrado durante el monitoreo.
Contiene:
•	Cámara relacionada.
•	Estado anterior.
•	Estado nuevo.
•	Tipo de evento.
•	Motivo.
•	Fecha del cambio.
Tecnico.java
Representa a los técnicos que pueden recibir notificaciones.
Incluye nombre, teléfono y estado de actividad.
Usuario.java
Representa las cuentas que acceden al sistema.
Contiene correo electrónico, contraseña codificada y rol.
enums/
Contiene enumeraciones utilizadas para mantener valores controlados.
EstadoCamara.java:
public enum EstadoCamara {
    ACTIVA,
    INACTIVA,
    DEGRADADA,
    ALERTA,
    DESCONOCIDO
}
Rol.java:
public enum Rol {
    ADMIN,
    USER
}
TipoEvento.java:
public enum TipoEvento {
    CAMBIO_ESTADO,
    ALERTA_MAC
}
mapper/
Contiene las clases encargadas de convertir entidades y DTO.
•	CamaraMapper.java
•	TecnicoMapper.java
•	UsuarioMapper.java
Si se modifica la estructura de un DTO o una entidad, se debe comprobar si es necesario actualizar su mapper correspondiente.
monitoreo/
Es uno de los paquetes más importantes del sistema.
Contiene la lógica especializada para comprobar el estado de las cámaras.
Clase	Función
MonitoreoScheduler.java	Programa la ejecución periódica.
PingVerificador.java	Comprueba conectividad mediante ping.
TcpVerificador.java	Comprueba disponibilidad del puerto TCP.
MacResolver.java	Obtiene la MAC mediante ARP.
MacVerificador.java	Compara la MAC detectada con la esperada.
EvaluarEstado.java	Determina el estado de la cámara.
ResultadoCheck.java	Representa el resultado de una comprobación.
Verificador.java	Interfaz común para verificadores.
Importante: para modificar la lógica de detección de estados, revisar EvaluarEstado.java y MonitoreoService.java.
notificador/
Contiene la implementación de los diferentes canales de notificación.
•	Notificador.java: contrato común.
•	ConsolaNotificador.java: registra las notificaciones en consola.
•	TelegramNotificador.java: integra Telegram.
•	WhatsappNotificador.java: integra WhatsApp Cloud API.
La selección del canal se realiza mediante configuración, no modificando directamente el servicio de monitoreo.
repository/
Contiene las interfaces de acceso a datos.
•	CamaraRepository.java
•	HistorialEstadoRepository.java
•	TecnicoRepository.java
•	UsuarioRepository.java
Extienden JpaRepository, por lo que aprovechan los métodos CRUD proporcionados por Spring Data JPA.
security/
Contiene los componentes de seguridad basados en JWT.
Clase	Función
CustomUserDetailsService.java	Busca usuarios durante la autenticación.
UserDetailsImpl.java	Adapta la entidad Usuario a UserDetails.
JwtService.java	Genera y valida tokens JWT.
JwtAuthenticationFilter.java	Intercepta solicitudes y procesa el token.
service/
Contiene la lógica principal de negocio.
•	CamaraService.java
•	MonitoreoService.java
•	TecnicoService.java
•	UsuarioService.java
Especial atención a MonitoreoService.java: coordina los verificadores, calcula el estado, actualiza la cámara, registra el historial y solicita el envío de notificaciones.
2.3. Frontend: organización
El frontend está construido con React y Vite.
api/
client.js centraliza las llamadas HTTP hacia el backend.
Se encarga de gestionar el token JWT, realizar solicitudes y procesar errores de la API.
components/
Contiene los componentes visuales.
Componente	Responsabilidad
AuthScreen.jsx	Inicio de sesión y registro.
Dashboard.jsx	Panel principal.
Sidebar.jsx	Navegación.
StatsGrid.jsx	Estadísticas generales.
CameraTable.jsx	Tabla de cámaras.
CameraFormModal.jsx	Registro y edición.
HistoryModal.jsx	Consulta del historial.
RetiredCameras.jsx	Cámaras retiradas.
TechniciansSection.jsx	Administración de técnicos.
Modal.jsx	Componente reutilizable de diálogo.
Brand.jsx	Identidad visual.
context/
AuthContext.jsx administra el estado de autenticación del frontend.
Incluye el manejo de sesión y las operaciones de inicio y cierre de sesión.
La sesión se almacena en sessionStorage.
styles/
Contiene los estilos CSS del panel.
utils/
format.js contiene funciones auxiliares para formatos, estados, fechas y búsqueda.
________________________________________
3. FLUJOS DE DATOS Y PROCESOS CRÍTICOS
3.1. Flujo de una petición REST
Una solicitud normal sigue el siguiente recorrido:
Cliente React
     |
     v
Solicitud HTTP
     |
     v
JwtAuthenticationFilter
     |
     v
Spring Security
     |
     v
Controller
     |
     v
Service
     |
     v
Repository
     |
     v
MySQL
     |
     v
Service
     |
     v
DTO / Mapper
     |
     v
Respuesta JSON
     |
     v
Cliente React
Paso a paso
1.	El usuario ejecuta una operación desde el panel.
2.	El frontend construye la solicitud HTTP.
3.	Si el endpoint requiere autenticación, envía el token JWT.
4.	Spring Security comprueba la autenticación y los permisos.
5.	El controlador recibe la solicitud.
6.	El servicio ejecuta las reglas de negocio.
7.	El repositorio consulta o modifica MySQL.
8.	El resultado se convierte a DTO cuando corresponde.
9.	Spring devuelve la respuesta HTTP.
3.2. Flujo de autenticación
El sistema utiliza autenticación sin estado mediante JWT.
Registro
Endpoint:
POST /auth/register
Proceso:
1.	El usuario envía correo y contraseña.
2.	AuthController recibe los datos.
3.	UsuarioService utiliza UsuarioMapper.
4.	La contraseña se codifica mediante BCrypt.
5.	Se asigna automáticamente el rol USER.
6.	Se almacena el usuario en MySQL.
7.	Se devuelve la respuesta correspondiente.
El registro público no permite asignar directamente el rol ADMIN.
Inicio de sesión
Endpoint:
POST /auth/login
Proceso:
1.	El usuario envía sus credenciales.
2.	AuthenticationManager realiza la autenticación.
3.	CustomUserDetailsService obtiene el usuario.
4.	Spring Security comprueba la contraseña.
5.	JwtService genera el token.
6.	Se devuelve el token junto con el rol del usuario.
Las siguientes solicitudes protegidas deben incluir:
Authorization: Bearer <token>
El token tiene una duración configurada de 24 horas.
3.3. Flujo de monitoreo automático
Este es el proceso central del sistema.
La clase MonitoreoScheduler inicia el monitoreo automáticamente.
Configuración actual:
@Scheduled(
    fixedDelay = 60000,
    initialDelay = 5000
)
public void ejecutar() {
    monitoreoService.monitorearTodas();
}
Esto significa:
•	La primera ejecución comienza aproximadamente cinco segundos después del inicio de la aplicación.
•	Las siguientes ejecuciones se realizan con un retraso de 60 segundos después de finalizar la ejecución anterior.
Secuencia de monitoreo
MonitoreoScheduler
        |
        v
MonitoreoService
        |
        v
Buscar cámaras no retiradas
        |
        v
Verificar PING
        |
        v
Verificar TCP (si tiene puerto)
        |
        v
Obtener MAC (si responde)
        |
        v
EvaluarEstado
        |
        v
Determinar estado
        |
        v
Actualizar cámara
        |
        v
¿Cambió el estado?
      /       \
    SÍ         NO
    |           |
    v           v
Guardar       Mantener
historial     historial
    |
    v
¿Corresponde notificar?
    |
    v
Enviar alerta
Verificación de ping
La clase responsable es:
PingVerificador.java
Ejecuta el comando nativo ping dependiendo del sistema operativo.
En Windows utiliza:
ping -n 1 -w 2000 DIRECCION_IP
En Linux utiliza:
ping -c 1 -W 2 DIRECCION_IP
El tiempo de espera configurado es de aproximadamente dos segundos.
En Windows se comprueba la presencia de ttl= en la salida para evitar interpretar determinados mensajes de host inaccesible como respuestas satisfactorias.
Verificación TCP
La clase responsable es:
TcpVerificador.java
Utiliza Socket para intentar establecer una conexión con la IP y el puerto registrados.
Ejemplo conceptual:
socket.connect(
    new InetSocketAddress(ip, puerto),
    timeoutMs
);
Si la conexión se establece, el puerto se considera disponible.
Si existe un timeout, rechazo de conexión o problema de ruta, se registra un resultado negativo.
El timeout predeterminado es de 2000 milisegundos.
Verificación de MAC
Participan dos clases:
•	MacResolver.java
•	MacVerificador.java
MacResolver obtiene la dirección MAC consultando la tabla ARP del sistema operativo.
En Linux utiliza:
/proc/net/arp
En Windows utiliza:
arp -a DIRECCION_IP
La dirección encontrada se normaliza al formato:
AA:BB:CC:DD:EE:FF
Posteriormente, MacVerificador compara la dirección detectada con la MAC esperada registrada para la cámara.
Los posibles resultados son:
•	MAC coincidente.
•	MAC diferente.
•	MAC no disponible.
Limitación de red: ARP permite conocer directamente la MAC del dispositivo cuando este se encuentra en el mismo segmento de red de capa 2 que el servidor. Si existe un router entre ambos, puede aparecer la MAC del gateway en lugar de la MAC real de la cámara.
Por ello, no debe interpretarse automáticamente una MAC no encontrada como una cámara desconectada.
3.4. Evaluación del estado
La clase EvaluarEstado.java determina el estado final a partir de los resultados de las comprobaciones.
Condición	Estado
Ping y TCP no responden	INACTIVA
MAC detectada y diferente de la esperada	ALERTA
Ping responde, pero TCP falla	DEGRADADA
Existe respuesta y no se detectan las condiciones anteriores	ACTIVA
Cuando una cámara no tiene puerto configurado, TCP no se ejecuta.
Cuando no existe una respuesta de ping ni TCP, no se realiza la consulta de MAC.
El estado DESCONOCIDO se utiliza como estado inicial de una cámara que todavía no tiene una clasificación determinada.
3.5. Reintentos ante una caída
El comportamiento está implementado en MonitoreoService.java.
Configuración:
private static final int REINTENTOS = 3;
private static final long ESPERA_REINTENTO_MS = 3000;
Cuando una cámara pasa a parecer INACTIVA desde otro estado, el sistema realiza verificaciones adicionales antes de confirmar el cambio.
Proceso:
1.	Ejecuta una primera comprobación.
2.	Si detecta una posible caída nueva, espera tres segundos.
3.	Ejecuta una segunda comprobación.
4.	Si continúa INACTIVA, vuelve a esperar tres segundos.
5.	Repite hasta completar los reintentos configurados.
Los reintentos no se aplican de la misma manera a una cámara que ya estaba registrada como INACTIVA.
Este proceso evita que una interrupción momentánea genere inmediatamente una alerta definitiva.
3.6. Actualización del historial
La clase MonitoreoService determina si el estado nuevo es diferente al anterior.
Cuando existe un cambio, se construye un registro de HistorialEstado.
Se almacena:
•	Identificador de la cámara.
•	Estado anterior.
•	Estado nuevo.
•	Tipo de evento.
•	Motivo.
•	Fecha y hora.
Los tipos de evento utilizados son:
CAMBIO_ESTADO
ALERTA_MAC
El historial se consulta mediante:
GET /api/camaras/{id}/historial
Las verificaciones periódicas no generan un registro de historial por cada ejecución. Se registra cuando se produce un cambio de estado.
3.7. Flujo de notificaciones
El sistema utiliza la interfaz:
public interface Notificador {
    void notificar(
        Camara camara,
        EstadoCamara anterior,
        EstadoCamara nuevo,
        String motivo
    );
}
Esto permite implementar diferentes mecanismos de comunicación sin acoplar el monitoreo a un proveedor específico.
La selección se configura mediante:
NOTIFICACIONES_CANAL=whatsapp
Canales disponibles:
•	whatsapp
•	telegram
•	consola
¿Cuándo se envían las notificaciones?
El sistema notifica cuando una cámara entra o sale de los estados:
•	INACTIVA.
•	ALERTA.
También contempla avisar durante la primera verificación después del arranque si una cámara ya se encuentra en uno de esos estados.
No se envía una notificación por cada comprobación periódica.
Destinatarios
Los destinatarios son los técnicos activos registrados en la base de datos.
La consulta utilizada es:
tecnicoRepository.findByActivoTrue();
En el canal WhatsApp, los envíos a los diferentes técnicos se realizan utilizando CompletableFuture y un ejecutor con cinco hilos.
Un error de envío a un técnico se registra y no impide intentar enviar a los demás.
Si no existen técnicos activos, el sistema registra un aviso en el log.
3.8. Administración de cámaras
El flujo de administración utiliza:
CamaraController
       |
       v
CamaraService
       |
       v
CamaraMapper
       |
       v
CamaraRepository
       |
       v
MySQL
Las operaciones principales son:
•	Crear una cámara.
•	Listar cámaras activas.
•	Consultar una cámara por identificador.
•	Actualizar información.
•	Retirar una cámara.
•	Consultar cámaras retiradas.
•	Restaurar una cámara.
•	Consultar su historial.
Eliminación lógica
El sistema no elimina físicamente la cámara cuando se ejecuta la operación DELETE.
En su lugar, cambia el campo:
eliminada = true;
Esto permite conservar la información y el historial anterior.
La restauración vuelve a establecer:
eliminada = false;
3.9. Manejo de errores
La clase:
ApiExceptionHandler.java
Utiliza @RestControllerAdvice para centralizar el tratamiento de determinadas excepciones.
Actualmente contempla:
Excepción	Respuesta
MethodArgumentNotValidException	HTTP 400
HttpMessageNotReadableException	HTTP 400
ResponseStatusException	Código HTTP correspondiente
Los errores de validación devuelven información sobre los campos que presentan problemas.
Los servicios utilizan ResponseStatusException para situaciones como:
•	Cámara no encontrada.
•	Técnico no encontrado.
•	Cámara ya activa.
•	Teléfono de técnico duplicado.
No todas las excepciones posibles tienen un manejador específico en esta implementación.
3.10. Logging
El proyecto utiliza SLF4J mediante Lombok y la anotación:
@Slf4j
Ejemplo:
log.error(
    "Error monitoreando la cámara {}: {}",
    camara.getId(),
    e.getMessage()
);
Los registros permiten identificar:
•	Fallos durante el monitoreo.
•	Problemas de conexión.
•	Errores de notificación.
•	Ausencia de técnicos activos.
•	Problemas relacionados con la integración externa.
No existe una configuración específica de almacenamiento de logs en archivos dentro del repositorio. Por defecto, deben revisarse los registros de salida del proceso Java o del servicio que lo ejecute.
3.11. Concurrencia y procesamiento en segundo plano
El monitoreo se ejecuta mediante Spring Scheduler.
La iteración de cámaras y sus reintentos se realizan de forma secuencial dentro del proceso programado.
Por lo tanto, el tiempo total de una ejecución depende de:
•	Cantidad de cámaras.
•	Tiempo de respuesta de ping.
•	Tiempo de conexión TCP.
•	Tiempo de resolución de MAC.
•	Cantidad de reintentos.
Las notificaciones WhatsApp utilizan procesamiento concurrente mediante CompletableFuture.
El conjunto de cámaras verificadas desde el arranque utiliza ConcurrentHashMap.newKeySet() para mantener el seguimiento de forma segura frente a accesos concurrentes.
________________________________________
4. CONFIGURACIÓN DEL ENTORNO DE DESARROLLO LOCAL
4.1. Prerrequisitos
Para trabajar con el proyecto se requieren las siguientes herramientas:
Herramienta	Requisito
JDK	Java 17
Maven	Se puede utilizar Maven Wrapper incluido.
Node.js	20.19 o superior, o 22.12 o superior.
npm	Incluido con Node.js.
MySQL	Servidor de base de datos.
IDE	IntelliJ IDEA, Eclipse o VS Code.
Git	Para control de versiones.
No existe una configuración Docker incluida en el repositorio.
4.2. Clonar el repositorio
git clone URL_DEL_REPOSITORIO
Ingresar a la carpeta:
cd camaraProyecto-main
El repositorio contiene dos partes principales:
•	estadoCamaras: backend.
•	frontend: interfaz web.
4.3. Configuración de MySQL
Crear la base de datos:
CREATE DATABASE estado_camaras
CHARACTER SET utf8mb4;
Crear un usuario:
CREATE USER 'monitoreo'@'localhost'
IDENTIFIED BY 'una_clave_segura';
Asignar permisos:
GRANT ALL PRIVILEGES ON estado_camaras.*
TO 'monitoreo'@'localhost';
La aplicación utiliza Hibernate con:
spring.jpa.hibernate.ddl-auto=update
Esto permite actualizar el esquema de la base de datos al iniciar la aplicación.
No reemplaza un sistema de migraciones versionadas.
4.4. Variables de entorno del backend
Ingresar a:
cd estadoCamaras
Copiar el archivo de ejemplo:
Copy-Item .env.example .env
Completar las variables:
# Base de datos
DB_URL=jdbc:mysql://localhost:3306/estado_camaras
DB_USERNAME=monitoreo
DB_PASSWORD=tu_contraseña

# Seguridad
JWT_SECRET=secreto_aleatorio_de_32_bytes_o_mas

# Canal de notificaciones
NOTIFICACIONES_CANAL=whatsapp

# WhatsApp
WHATSAPP_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=

# Telegram
TELEGRAM_TOKEN=
TELEGRAM_CHAT_ID=
Consideraciones importantes
•	El archivo .env contiene información sensible.
•	No debe subirse al repositorio.
•	JWT_SECRET debe ser aleatorio, tener al menos 32 caracteres y ser diferente en cada instalación.
•	El archivo .env se carga desde el directorio de trabajo de la aplicación.
•	Si se modifica una variable, es necesario reiniciar el backend.
La configuración se encuentra en:
src/main/resources/application.properties
4.5. Ejecutar el backend
Desde la carpeta estadoCamaras:
En Windows:
.\mvnw.cmd spring-boot:run
En Linux:
./mvnw spring-boot:run
El servidor utiliza el puerto 8080 por defecto.
Dirección:
http://localhost:8080
4.6. Ejecutar el frontend
Abrir otra terminal:
cd frontend
Instalar dependencias:
npm install
Ejecutar el servidor de desarrollo:
npm run dev
Abrir:
http://localhost:5173
El archivo vite.config.js contiene la configuración del proxy.
Las solicitudes hacia:
/api
/auth
se redirigen al backend.
Por defecto:
http://localhost:8080
Si el backend utiliza otra dirección, se puede crear:
frontend/.env.local
Con:
BACKEND_URL=http://localhost:9090
4.7. Compilar el backend
Desde estadoCamaras:
.\mvnw.cmd clean package
En Linux:
./mvnw clean package
El resultado se genera en:
estadoCamaras/target/
El archivo JAR puede ejecutarse mediante:
java -jar target/estadoCamaras-0.0.1-SNAPSHOT.jar
Al ejecutar el JAR, debe respetarse el directorio de trabajo donde se encuentra el .env.
4.8. Pruebas del backend
Ejecutar:
.\mvnw.cmd test
El proyecto contiene pruebas para:
•	Contexto de Spring Boot.
•	Manejo de excepciones de la API.
•	Validación de CamaraRequestDTO.
La prueba de contexto requiere que la base de datos configurada esté disponible.
No se observa una suite completa de pruebas de integración para el monitoreo de cámaras, ni pruebas específicas para todos los servicios de seguridad y notificación.
4.9. Pruebas del frontend
Ejecutar el análisis estático:
npm run lint
Generar la compilación:
npm run build
El resultado se almacena en:
frontend/dist/
Para probar esa compilación:
npm run preview
________________________________________
5. OPERACIÓN, DESPLIEGUE Y MONITOREO (OPS)
5.1. Pipeline CI/CD
En la versión del repositorio revisada no se encuentra una configuración de CI/CD.
No se observan archivos de workflows de GitHub Actions, configuración de Jenkins ni otros pipelines automatizados.
Actualmente, el proceso de construcción se realiza mediante los comandos de Maven y npm.
Flujo de construcción actual
Código fuente
     |
     v
Compilación Backend
     |
     v
Pruebas Maven
     |
     v
Generación del JAR
     |
     v
Build Frontend
     |
     v
Generación de archivos estáticos
     |
     v
Despliegue manual
Automatización recomendada
Para futuras versiones se recomienda implementar un pipeline que incluya:
1.	Verificación de compilación Java.
2.	Ejecución de pruebas unitarias.
3.	Análisis estático del frontend.
4.	Construcción de React.
5.	Generación del JAR.
6.	Publicación de artefactos.
7.	Despliegue controlado.
Esto permitiría detectar errores antes de incorporar cambios a producción.
5.2. Estrategia de despliegue
La documentación del repositorio contempla la ejecución del backend como un JAR en Windows Server.
No se incluye una configuración Docker.
Despliegue del backend
Requisitos:
•	Java 17 compatible con el sistema operativo.
•	MySQL.
•	Archivo .env.
•	Acceso a la red de cámaras.
•	Acceso a Internet si se utilizan notificaciones externas.
Ejemplo de ubicación:
C:\monitoreo\
│
├── estadoCamaras-0.0.1-SNAPSHOT.jar
└── .env
Ejecución:
cd C:\monitoreo

java -jar estadoCamaras-0.0.1-SNAPSHOT.jar
Para mantener la aplicación ejecutándose después de cerrar una sesión, se recomienda configurarla como servicio mediante NSSM o mediante el Programador de tareas de Windows.
Es importante establecer el directorio de trabajo correcto para que Spring pueda localizar el .env.
También se recomienda:
•	Utilizar una cuenta de servicio con permisos mínimos.
•	Configurar reinicio automático ante fallos.
•	Restringir el acceso al archivo de secretos.
•	Configurar respaldos de MySQL.
Despliegue del frontend
La compilación se genera mediante:
npm run build
Los archivos resultantes se encuentran en:
frontend/dist/
Existen dos alternativas documentadas:
Alternativa A: mismo servidor que Spring Boot
Copiar los archivos generados al directorio:
estadoCamaras/src/main/resources/static/
Después, volver a compilar el JAR.
Esta alternativa requiere revisar la configuración de Spring Security para permitir correctamente los recursos estáticos de React, especialmente /assets/**.
Alternativa B: servidor independiente
Publicar el contenido de dist/ en un servidor web separado y configurar la URL de la API y las reglas CORS correspondientes.
Antes de utilizar esta alternativa se debe revisar la configuración actual de CorsConfig y las reglas de SecurityConfig, ya que no todos los orígenes y rutas están habilitados actualmente.
5.3. Configuración de notificaciones
WhatsApp
El sistema utiliza WhatsApp Cloud API de Meta.
Variables necesarias:
NOTIFICACIONES_CANAL=whatsapp
WHATSAPP_TOKEN=TOKEN
WHATSAPP_PHONE_NUMBER_ID=ID_DEL_NUMERO
La plantilla configurada es:
alerta_camara
Idioma:
es
La plantilla debe estar aprobada por Meta y utilizar cinco parámetros:
1.	Nombre de la cámara.
2.	Ubicación.
3.	Dirección IP.
4.	Estado actual.
5.	Motivo.
Los teléfonos registrados para los técnicos deben incluir el código de país, sin el signo + ni espacios.
Ejemplo:
51987654321
Telegram
Variables:
NOTIFICACIONES_CANAL=telegram
TELEGRAM_TOKEN=TOKEN_DEL_BOT
TELEGRAM_CHAT_ID=ID_DEL_CHAT
Consola
Permite utilizar un canal de notificación local para pruebas o instalaciones que no requieran comunicación externa.
5.4. Monitoreo operativo del propio sistema
El sistema realiza monitoreo de cámaras, pero no cuenta actualmente con una plataforma independiente de observabilidad.
Para diagnosticar problemas se recomienda revisar:
Problema	Dónde revisar
Aplicación no inicia	Consola del proceso Java.
Error de conexión MySQL	application.properties, .env y logs.
Cámara inactiva	Resultado de ping y TCP.
MAC no encontrada	Tabla ARP y configuración de red.
Notificaciones no llegan	Logs del notificador y credenciales externas.
Error de autenticación	JWT y configuración de Spring Security.
Problemas del frontend	Consola del navegador y Network.
Consideraciones de red
Para que el monitoreo funcione correctamente, el servidor debe tener conectividad hacia las direcciones IP de las cámaras.
Se deben considerar:
•	Segmentación de redes.
•	Reglas de firewall.
•	Rutas entre subredes.
•	Permisos ICMP.
•	Disponibilidad de los puertos TCP.
•	Visibilidad de la tabla ARP.
La obtención de MAC no está garantizada cuando las cámaras se encuentran detrás de routers.
5.5. Base de datos y mantenimiento
La base de datos contiene información operativa importante:
•	Usuarios.
•	Cámaras.
•	Técnicos.
•	Historial de estados.
Se recomienda realizar respaldos periódicos.
La configuración actual:
spring.jpa.hibernate.ddl-auto=update
permite que Hibernate actualice el esquema, pero no proporciona un control formal de versiones de las migraciones.
El repositorio contiene:
estadoCamaras/db/migrate_empresa_to_single_company.sql
Este script está destinado a instalaciones antiguas que todavía utilizan la relación entre cámaras y empresas.
No debe ejecutarse indiscriminadamente en una base nueva.
Antes de cualquier migración se debe realizar un respaldo y comprobar que la estructura de la base corresponda con la esperada por el script.
________________________________________
6. GUÍA DE SUPERVIVENCIA PARA EL PRIMER DÍA
6.1. ¿Por dónde empezar a entender el código?
Un nuevo desarrollador no necesita leer todos los archivos del repositorio inmediatamente.
Se recomienda seguir este orden:
Primero: comprender la aplicación
Leer:
README.md
Después revisar:
EstadoCamarasApplication.java
application.properties
pom.xml
Segundo: entender la API
Revisar:
controller/
dto/
service/
repository/
Comenzar con el flujo de cámaras.
Tercero: entender el monitoreo
Leer en este orden:
MonitoreoScheduler.java
MonitoreoService.java
PingVerificador.java
TcpVerificador.java
MacResolver.java
MacVerificador.java
EvaluarEstado.java
Este orden permite entender cómo se inicia una verificación, cómo se obtienen los resultados y cómo se determina el estado final.
Cuarto: entender las notificaciones
Revisar:
notificador/Notificador.java
notificador/WhatsappNotificador.java
notificador/TelegramNotificador.java
Quinto: comprender la seguridad
Revisar:
SecurityConfig.java
JwtAuthenticationFilter.java
JwtService.java
CustomUserDetailsService.java
UserDetailsImpl.java
Sexto: entender el frontend
Comenzar por:
frontend/src/main.jsx
frontend/src/App.jsx
frontend/src/context/AuthContext.jsx
frontend/src/api/client.js
frontend/src/components/Dashboard.jsx
6.2. Antipatrones y errores que deben evitarse
1. Colocar lógica de negocio en los controladores
Evitar implementar operaciones complejas directamente en los controladores.
La lógica debe mantenerse en los servicios correspondientes.
2. Modificar directamente el estado de una cámara sin revisar el monitoreo
El estado se determina mediante EvaluarEstado.
Si se modifican las reglas sin revisar el servicio de monitoreo, pueden generarse inconsistencias entre el resultado de las verificaciones, el historial y las notificaciones.
3. Confundir ping con disponibilidad del puerto
Una cámara puede no responder a ICMP y, aun así, aceptar conexiones TCP.
Por eso, no se debe asumir que un ping fallido significa necesariamente que el dispositivo está desconectado.
4. Suponer que siempre se puede obtener la MAC
La resolución ARP depende de la topología de red.
Una cámara ubicada en otra subred puede no proporcionar su MAC real al servidor de monitoreo.
5. Eliminar físicamente cámaras de la base de datos
El proyecto utiliza eliminación lógica para conservar el historial.
Se debe utilizar el mecanismo implementado de retiro y restauración.
6. Modificar los DTO sin actualizar los mappers
Los mappers son responsables de transformar los datos entre entidades y DTO.
Un cambio en los campos puede afectar las respuestas de la API.
7. Exponer información sensible
Nunca se deben subir al repositorio:
•	Contraseñas de MySQL.
•	Tokens de WhatsApp.
•	Tokens de Telegram.
•	Claves JWT.
•	Archivos .env reales.
8. Cambiar las reglas de Spring Security sin revisar los endpoints
Una modificación incorrecta puede permitir operaciones administrativas a usuarios sin privilegios o bloquear funcionalidades legítimas del frontend.
9. Suponer que las notificaciones se envían en cada comprobación
El sistema notifica ante determinadas transiciones de estado y durante ciertas condiciones del primer monitoreo tras el arranque.
No se debe modificar este comportamiento sin considerar el riesgo de generar notificaciones repetitivas.
10. Cambiar el intervalo de monitoreo sin evaluar el rendimiento
Las verificaciones se ejecutan secuencialmente.
Aumentar la cantidad de cámaras o los tiempos de espera puede provocar que una ejecución tarde más de lo previsto.
Antes de modificar los intervalos se debe analizar el tiempo total de ejecución.
6.3. Endpoints principales de la API
Autenticación
Método	Endpoint	Descripción
POST	/auth/register	Registrar usuario.
POST	/auth/login	Iniciar sesión.
Cámaras
Método	Endpoint	Descripción
GET	/api/camaras	Listar cámaras activas.
GET	/api/camaras/retiradas	Listar cámaras retiradas.
GET	/api/camaras/{id}	Consultar cámara.
GET	/api/camaras/{id}/historial	Consultar historial.
POST	/api/camaras	Registrar cámara.
PUT	/api/camaras/{id}	Actualizar cámara.
DELETE	/api/camaras/{id}	Retirar cámara.
POST	/api/camaras/{id}/restaurar	Restaurar cámara.
Técnicos
Método	Endpoint	Descripción
GET	/api/tecnicos	Listar técnicos.
GET	/api/tecnicos/{id}	Consultar técnico.
POST	/api/tecnicos	Registrar técnico.
PUT	/api/tecnicos/{id}	Actualizar técnico.
DELETE	/api/tecnicos/{id}	Desactivar técnico.
POST	/api/tecnicos/{id}/reactivar	Reactivar técnico.
Permisos
USER:
•	Consultar cámaras.
•	Consultar cámaras retiradas.
•	Consultar historial.
ADMIN:
•	Todas las operaciones de USER.
•	Registrar cámaras.
•	Actualizar cámaras.
•	Retirar y restaurar cámaras.
•	Administrar técnicos.
6.4. Preguntas frecuentes para el nuevo desarrollador
¿Dónde se modifica el tiempo entre verificaciones?
En MonitoreoScheduler.java, mediante fixedDelay.
¿Dónde se modifica el tiempo de espera del ping?
En PingVerificador.java.
¿Dónde se modifica el timeout TCP?
En TcpVerificador.java, mediante la propiedad monitoreo.timeout-ms, cuyo valor predeterminado es 2000 ms.
¿Dónde se determina si una cámara está activa o inactiva?
En EvaluarEstado.java.
¿Dónde se implementan los reintentos?
En MonitoreoService.java.
¿Dónde se guarda el historial?
En HistorialEstadoRepository.java, coordinado desde MonitoreoService.java.
¿Dónde se modifica el canal de notificaciones?
En el archivo .env, mediante NOTIFICACIONES_CANAL.
¿Dónde se configuran los permisos?
En SecurityConfig.java.
¿Dónde se modifican los endpoints de cámaras?
En CamaraController.java.
¿Dónde se modifican las consultas a MySQL?
En los repositorios correspondientes.
________________________________________
7. ESTADO ACTUAL Y CONSIDERACIONES PARA FUTURAS VERSIONES
Para facilitar la incorporación de nuevos desarrolladores, es importante diferenciar las capacidades actuales de las mejoras todavía pendientes.
7.1. Funcionalidades implementadas
•	Administración de cámaras IP.
•	Registro y actualización de cámaras.
•	Retiro lógico y restauración.
•	Verificación mediante ping.
•	Verificación de puertos TCP.
•	Obtención y comparación de MAC.
•	Evaluación automática de estados.
•	Reintentos ante determinadas caídas.
•	Historial de cambios.
•	Autenticación JWT.
•	Autorización mediante roles.
•	Administración de técnicos.
•	Notificaciones mediante WhatsApp.
•	Implementación de canales Telegram y consola.
•	Dashboard web con React.
•	Consulta de cámaras retiradas e historial.
7.2. Mejoras previstas
De acuerdo con la documentación y estructura revisadas, se identifican las siguientes oportunidades:
1.	Implementar verificación RTSP para comprobar la disponibilidad del servicio de video.
2.	Incorporar migraciones versionadas de base de datos.
3.	Mejorar la estrategia de despliegue y automatizar CI/CD.
4.	Incorporar respaldos y recuperación de datos.
5.	Implementar una comprobación de salud del servicio.
6.	Ampliar las pruebas unitarias y de integración.
7.	Mejorar la observabilidad mediante métricas y alertas del propio backend.
8.	Evaluar mejoras de concurrencia si aumenta considerablemente la cantidad de cámaras.
Estas mejoras deben desarrollarse sin comprometer la estabilidad de las funcionalidades actuales.
________________________________________
8. CONCLUSIÓN
EstadoCámaras es una aplicación de monitoreo y administración de cámaras IP construida sobre Java y Spring Boot, con una interfaz web desarrollada en React.
Su arquitectura separa las responsabilidades de presentación, lógica de negocio, persistencia, seguridad, monitoreo y notificación.
El componente central es MonitoreoService, que coordina las verificaciones de conectividad, la evaluación de estados, la persistencia de cambios y el envío de alertas.
Para incorporarse al equipo, el desarrollador debe comprender especialmente los siguientes componentes:
•	CamaraService: administración de cámaras.
•	MonitoreoService: coordinación del monitoreo.
•	EvaluarEstado: reglas de clasificación.
•	MacResolver: resolución de direcciones MAC.
•	MonitoreoScheduler: ejecución periódica.
•	SecurityConfig: protección de endpoints.
•	WhatsappNotificador: comunicación con los técnicos.
La recomendación principal es mantener la separación de responsabilidades existente, respetar las reglas de seguridad y comprender las condiciones reales de red antes de modificar los mecanismos de monitoreo.
Fin del documento.


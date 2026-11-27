# Manual técnico de arquitectura, operación y soporte

## Propósito y alcance

Este documento describe cómo funciona la aplicación de monitoreo de cámaras IP, qué comunicaciones necesita y cómo interpretar sus resultados. Está dirigido a personal técnico que instala, opera o recibe el sistema.

El comportamiento descrito corresponde al código actual. La aplicación comprueba conectividad de red y disponibilidad de un puerto TCP opcional; no muestra, graba ni valida el video. Las notificaciones por WhatsApp están implementadas mediante Meta WhatsApp Cloud API.

## 1. Resumen de la arquitectura

La aplicación backend está desarrollada con Java y Spring Boot. Consulta las cámaras almacenadas en MySQL, prueba su conectividad desde el servidor, actualiza su estado y, ante ciertos cambios, solicita a Meta el envío de notificaciones a los técnicos activos.

```text
Operador -> navegador -> aplicación Spring Boot <-> MySQL
                               |
                               | ICMP, TCP y consulta local de ARP
                               v
Cámaras IP -> switch -> router o equipo de capa 3

Aplicación Spring Boot -> HTTPS saliente -> Meta WhatsApp Cloud API
                                                -> WhatsApp del técnico
```

El servidor inicia las pruebas hacia las cámaras. Las cámaras no envían su estado al servidor ni se comunican con Meta. Los teléfonos reciben mensajes a través de la infraestructura de WhatsApp; no establecen una conexión directa con la red de cámaras.

El operador utiliza el panel web y la API de la aplicación para gestionar cámaras y técnicos. La API utiliza JWT para autenticar solicitudes; las funciones administrativas requieren el rol correspondiente. La persistencia se implementa con Spring Data JPA/Hibernate y MySQL.

La aplicación se configura mediante variables de entorno o un archivo `.env` protegido. Entre las variables principales están las credenciales de MySQL (`DB_URL`, `DB_USERNAME`, `DB_PASSWORD`), el secreto de JWT y las credenciales de WhatsApp. El canal de notificación es WhatsApp por defecto y también puede configurarse para Telegram o consola. No se deben incluir secretos reales en el repositorio. La configuración de instalación y ejecución se documenta también en `README.md`.

## 2. Conceptos de red

### Dirección IP, máscara y subred

La IP identifica lógicamente un equipo, por ejemplo `172.16.8.5`. La máscara determina qué parte corresponde a la subred. Los equipos de una misma subred pueden comunicarse directamente en la LAN; para llegar a otra subred se necesita una ruta a través de un router o equipo de capa 3. La aplicación se dirige a la IP registrada para cada cámara.

### Dirección MAC y ARP

La MAC identifica una interfaz en el segmento local y se utiliza para entregar tramas. ARP permite conocer la MAC del siguiente equipo local a partir de una IP. Si el destino está en otra subred, el servidor resuelve la MAC de su gateway, no la MAC remota de la cámara. ARP puede mostrar información en caché y no es un mecanismo de autenticación.

### Switch, router y gateway

Un switch de capa 2 reenvía tramas dentro de la red local usando las direcciones MAC. Un router o switch de capa 3 enruta paquetes IP entre subredes. El gateway es el siguiente salto al que un equipo envía tráfico destinado a otra red. Las IP y topologías de laboratorio son ejemplos y deben reemplazarse por el plan de direccionamiento real de la empresa.

### NAT y salida a Internet

En muchas redes, NAT traduce las direcciones privadas al salir a Internet y permite asociar las respuestas con conexiones iniciadas desde dentro. Las políticas corporativas pueden usar firewalls o proxies además de NAT. Para WhatsApp, el requisito de la aplicación es que el servidor pueda iniciar HTTPS hacia Meta; no necesita una conexión entrante desde Internet.

### ICMP, ping y TCP

Ping usa ICMP Echo Request/Echo Reply. Una respuesta confirma que hubo respuesta ICMP desde la IP, no que el video funcione. Cámaras y firewalls pueden bloquear ICMP.

La prueba TCP intenta abrir una conexión a la IP y puerto configurados. Algunos ejemplos habituales en equipos son `80`/`443` para web, `554` para RTSP o `37777` en ciertos equipos Dahua. Se debe confirmar el puerto habilitado en cada cámara. La aplicación no inicia sesión ni intercambia datos de video. Un rechazo o timeout puede deberse al servicio, la ruta o un firewall; no demuestra por sí solo que el equipo esté apagado.

### DNS

DNS traduce nombres a direcciones IP. Se utiliza, por ejemplo, para resolver `graph.facebook.com` al enviar avisos. Las cámaras se registran por IP, por lo que DNS no interviene en las pruebas hacia ellas.

## 3. Topología de red y segmentación

En una instalación típica, las cámaras se conectan a switches de acceso. El switch reenvía las tramas dentro de la LAN. Si el servidor y la cámara están en subredes distintas, el router o switch de capa 3 enruta el tráfico entre ellas. La red real puede usar VLAN, firewalls, varios routers u otras rutas; no debe asumirse una topología específica sin validarla con el equipo de redes.

Para monitorear cámaras en otra subred se necesita:

- Una ruta desde el servidor hacia la subred de cámaras y una ruta de retorno.
- Reglas de firewall/ACL que permitan desde la IP del servidor las pruebas ICMP y el puerto TCP configurado.
- Una regla que permita el tráfico de respuesta.

Una o más tarjetas de red en el servidor no sustituyen el enrutamiento. Un servidor con varias interfaces debe configurarse de acuerdo con la arquitectura de la empresa; rutas, métricas o gateways incorrectos pueden provocar tráfico asimétrico o pérdida de conectividad. No se recomienda agregar una segunda interfaz solo para intentar obtener la MAC. La comunicación IP/TCP puede funcionar a través de routers aunque la MAC real de la cámara no sea visible por ARP.

Si se usa un laboratorio de Packet Tracer u otro diagrama, sus subredes e IP son ejemplos de ese laboratorio, no requisitos fijos del sistema.

## 4. Proceso de monitoreo

El planificador `MonitoreoScheduler` inicia el primer ciclo cinco segundos después del arranque. Al terminar cada ciclo, espera 60 segundos antes del siguiente. Las cámaras se procesan una por una, por lo que la duración del ciclo se suma a esa espera y puede aumentar cuando hay muchas cámaras o timeouts.

Para cada cámara no retirada, `MonitoreoService`:

1. Ejecuta ping a la IP registrada.
2. Si se configuró un puerto, intenta una conexión TCP a `IP:puerto`.
3. Si ping o TCP respondió, consulta la tabla ARP local para intentar obtener la MAC.
4. Evalúa el estado con los resultados.
5. Si el resultado es una caída nueva, reintenta hasta tres veces con pausas de tres segundos antes de confirmar `INACTIVA`.
6. Guarda la última verificación y el estado en MySQL. Si hubo cambio, también guarda un evento en el historial.
7. Si el cambio cumple las reglas de notificación, llama al canal configurado.

Los timeouts de ping y TCP son de aproximadamente dos segundos. El puerto TCP es opcional; ping siempre se intenta.

### Estados

| Estado | Condición aplicada por el código |
|---|---|
| `DESCONOCIDO` | Estado inicial de una cámara antes de obtener su primer resultado. |
| `ACTIVA` | Ping o TCP respondió, y no se detectó una MAC distinta ni una condición de degradación. |
| `INACTIVA` | Ni ping ni el puerto TCP configurado respondieron. No demuestra que la cámara esté apagada. |
| `DEGRADADA` | Ping respondió, pero el puerto TCP configurado no respondió. |
| `ALERTA` | Hubo respuesta y se detectó una MAC distinta de la MAC esperada registrada. |

Si ping está bloqueado pero el puerto TCP responde, la cámara puede quedar `ACTIVA`. Si ping y TCP están bloqueados, puede aparecer `INACTIVA` aunque el equipo esté encendido. Si no se configura un puerto, no se puede detectar el estado de ese servicio TCP.

## 5. Recorrido de los paquetes

En la misma subred, el sistema operativo del servidor necesita una dirección MAC de destino para entregar las tramas. Si no tiene una entrada ARP vigente, resuelve la IP local mediante ARP. A continuación, el servidor ejecuta ping y, si está configurado, abre la conexión TCP. El switch reenvía las tramas dentro de la LAN; no determina el estado de la cámara.

Si la IP de destino está en otra subred, el servidor envía los paquetes al siguiente salto definido por su tabla de rutas, normalmente un gateway. El router reenvía los paquetes hacia la red destino y las respuestas deben poder regresar al servidor. En ese caso, la consulta ARP del servidor normalmente refleja la MAC del siguiente salto, no la MAC remota de la cámara.

Por lo tanto, el sistema operativo puede seleccionar una interfaz según sus rutas y configuración; la aplicación no fuerza el uso de una segunda tarjeta de red.

## 6. Comprobación de MAC y limitaciones

El sistema consulta la tabla ARP del sistema operativo después de intentar ping/TCP. En Windows ejecuta `arp -a` para la IP; en Linux lee `/proc/net/arp`. Normaliza separadores y mayúsculas antes de comparar con la MAC esperada.

ARP solo permite identificar la MAC del vecino de capa 2. Si hay un router entre el servidor y la cámara, normalmente se observa la MAC del gateway. Además, la tabla puede conservar entradas en caché. Por ello, la MAC no se usa para decidir si la cámara responde y puede generar una alerta incorrecta si se registra como esperada en una topología enrutada.

Una MAC tampoco es un mecanismo de autenticación: puede cambiar o suplantarse. Para evitar conflictos de direcciones, la administración de red puede usar asignaciones IP controladas o reservas DHCP, según la política de la empresa.

## 7. Notificaciones por WhatsApp

### Cuándo se genera un aviso

El código notifica cuando hay un cambio cuyo estado anterior o nuevo es `INACTIVA` o `ALERTA`. En la primera comprobación del servidor también puede avisar si la cámara ya aparece en uno de esos estados.

| Cambio | ¿Notifica? |
|---|---|
| `DESCONOCIDO -> ACTIVA` | No |
| `DESCONOCIDO -> INACTIVA` | Sí, si la falta de respuesta se confirma |
| `ACTIVA -> INACTIVA` | Sí |
| `INACTIVA -> ACTIVA` | Sí, notificación de recuperación |
| Cambio entre `ACTIVA` y `DEGRADADA` | No, por sí solo |
| Entrada o salida de `ALERTA` | Sí |

Los mensajes no se agrupan. Cada evento se envía por separado a cada técnico activo. Por ejemplo, si 100 cámaras generan un evento notificable y hay dos técnicos activos, se pueden producir hasta 200 envíos de mensaje.

### Recorrido del mensaje

1. `MonitoreoService` detecta el cambio y llama al notificador configurado.
2. `WhatsappNotificador` obtiene los técnicos activos y construye una petición HTTPS con la plantilla `alerta_camara`, el token de acceso, el ID del número emisor y el teléfono del destinatario.
3. La petición sale del servidor a `graph.facebook.com` por HTTPS, normalmente TCP 443. La red puede usar NAT o un proxy de salida; no se necesita una conexión entrante desde Internet al servidor para enviar avisos.
4. Meta valida y procesa la solicitud. Una respuesta aceptada por la API no equivale necesariamente a confirmación de entrega al teléfono.
5. WhatsApp entrega el mensaje al destinatario mediante su propia infraestructura. El servidor de monitoreo y el teléfono no se conectan directamente.

El cuerpo de la plantilla incluye nombre, ubicación, IP, estado y motivo. Se requiere que la plantilla exista y esté aprobada en Meta, y que los valores de configuración y destinatarios sean válidos.

### Límites del comportamiento actual

- Si falla el envío a un destinatario, el error queda en los logs. El sistema no tiene una cola persistente ni reintentos automáticos de mensajes pendientes.
- El estado de la cámara se guarda independientemente del resultado de WhatsApp. Durante el mismo proceso, si la cámara sigue en el mismo estado, el mensaje fallido no se reenvía automáticamente. Al reiniciar la aplicación, una cámara que siga `INACTIVA` o en `ALERTA` puede generar un nuevo aviso en su primera verificación.
- El código no configura webhooks para confirmar estados de entrega o recibir mensajes.
- Sin técnicos activos, no se envía WhatsApp y se registra una advertencia en el log.

## 8. Límites y consideraciones operativas

- `INACTIVA` significa que no respondieron las pruebas habilitadas; no confirma que la cámara esté apagada.
- Una respuesta de ping o TCP no confirma que exista video, que se esté grabando o que la imagen sea correcta.
- La validación de MAC por ARP no identifica la MAC remota a través de un router y no es una medida de autenticación.
- El ciclo actual procesa las cámaras secuencialmente. Un conjunto grande de cámaras sin respuesta puede hacer que el ciclo tarde bastante más de un minuto.
- Las notificaciones fallidas no se almacenan en una cola para reenvío. Conviene supervisar los logs del servicio y la configuración de Meta.
- Las cámaras no necesitan acceso a Internet para que el servidor compruebe su IP local; el servidor sí requiere acceso de salida a Meta para WhatsApp.

## 9. Validación y diagnóstico

Ejecuta las pruebas desde el propio servidor, para comprobar el mismo camino de red que usa la aplicación. Sustituye la IP y el puerto de ejemplo por valores autorizados y reales:

```powershell
ping 172.17.1.83
Test-NetConnection 172.17.1.83 -Port 37777
arp -a 172.17.1.83
ipconfig
Test-NetConnection graph.facebook.com -Port 443
```

Interpretación:

- `TcpTestSucceeded: True` confirma que en ese momento se pudo establecer una conexión TCP desde el servidor a ese puerto.
- Un ping fallido no descarta que TCP funcione; ICMP puede estar bloqueado.
- Una prueba TCP fallida puede deberse a puerto incorrecto, servicio no disponible, ruta o firewall; no demuestra por sí sola que la cámara esté apagada.
- `arp -a` muestra la información local disponible. En una red enrutada podría mostrar el gateway.
- La conectividad a `graph.facebook.com:443` no valida el token, la plantilla ni que el teléfono haya recibido el mensaje; esos aspectos se revisan en los logs y en la configuración de Meta.

| Síntoma | Revisiones iniciales |
|---|---|
| Cámara `INACTIVA`, aunque parece encendida | IP, ruta de ida y vuelta, ICMP, puerto TCP configurado y reglas de firewall. |
| Cámara `DEGRADADA` | El ping respondió, pero el puerto TCP configurado no. Confirma el puerto y las ACL/firewalls. |
| Cámara `ALERTA` en otra VLAN | Revisa la MAC esperada; ARP puede estar mostrando la MAC del gateway. |
| No se recibe WhatsApp | Técnicos activos, canal configurado, token, ID del número emisor, plantilla aprobada, permisos, salida HTTPS y logs. |
| Muchas alertas al mismo tiempo | Actualmente se envía un mensaje por evento de cámara y por cada técnico activo; no hay agrupación. |

## 10. Glosario

- **ACL:** regla de control de acceso que permite o bloquea tráfico.
- **ARP:** mecanismo para conocer la MAC del siguiente equipo local a partir de una IP.
- **Gateway:** puerta de enlace usada para enviar tráfico a otra red.
- **ICMP:** protocolo utilizado por ping.
- **JPA/Hibernate:** tecnología ORM que mapea objetos Java a registros de base de datos.
- **MAC:** dirección de interfaz usada para entregar tramas en el segmento local.
- **NAT:** traducción de direcciones que suele realizarse al conectar una red privada con otras redes.
- **Plantilla de WhatsApp:** mensaje aprobado por Meta con texto fijo y parámetros.
- **Puerto TCP:** número que identifica un servicio de red dentro de un equipo.
- **RTSP:** protocolo utilizado por algunos equipos para transmitir video.
- **Subred:** conjunto de direcciones IP definido por una máscara.
- **VLAN:** separación lógica de redes en una infraestructura física compartida.
- **Webhook:** endpoint al que un servicio externo puede enviar eventos; este proyecto no configura uno para WhatsApp.

## 11. Mejoras posibles

Estas son opciones de evolución, no capacidades que deban darse por implementadas:

- Agrupar alertas simultáneas y añadir una cola persistente con política de reintentos.
- Procesar cámaras en paralelo con límites de concurrencia y tiempos de espera controlados.
- Consultar confirmaciones de entrega de Meta mediante webhooks, si se requiere seguimiento.
- Comprobar el flujo RTSP u otro mecanismo del fabricante si se necesita validar disponibilidad de video.
- Añadir métricas de disponibilidad, informes e importación masiva de cámaras.

## 12. Referencias de implementación

- Planificador: `src/main/java/com/losMatus/estadoCamaras/monitoreo/MonitoreoScheduler.java`
- Lógica del ciclo y notificaciones: `src/main/java/com/losMatus/estadoCamaras/service/MonitoreoService.java`
- Ping y TCP: `src/main/java/com/losMatus/estadoCamaras/monitoreo/PingVerificador.java` y `TcpVerificador.java`
- Estados: `src/main/java/com/losMatus/estadoCamaras/monitoreo/EvaluarEstado.java`
- ARP/MAC: `src/main/java/com/losMatus/estadoCamaras/monitoreo/MacResolver.java`
- WhatsApp: `src/main/java/com/losMatus/estadoCamaras/notificador/WhatsappNotificador.java`

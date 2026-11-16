# Mailflare para Dano Studios

Estado al 16 de noviembre de 2026: preparación; no desplegado, sin buzones activos ni DNS modificado.

## Dominio y repositorio

- Repositorio de instalación: https://github.com/danysepulveda20-oss/mailflare
- Código upstream revisado localmente: 5330fa4fff72cb0f38564822ff873e8ab854dee8, versión package.json 0.5.5.
- NIC Chile respondió «Nombre de dominio no existe» para danostudios.cl; DNS NS respondió NXDOMAIN. Esto es una consulta puntual, no una reserva ni una prueba de propiedad.
- Registrar danostudios.cl en NIC Chile antes de configurar correo. Repetir la consulta al comprar.
- Conservar LICENSE y avisos del proyecto (AGPL-3.0). Esta preparación no es una auditoría de seguridad completa.

## Diseño

Internet → Cloudflare Email Routing → Worker mailflare → cola mailflare-inbound → D1 + R2 → webmail.
Webmail → cola mailflare-outbound → Cloudflare Email Sending → destinatario.
D1 contiene cuentas, mensajes e índices; R2 guarda MIME original, adjuntos y respaldos.
Durable Object RealtimeHub entrega actualizaciones en vivo; cron ejecuta respaldos.
La cola mailflare-agent y el binding AI existen en upstream; mantener la IA sin activar hasta decidir su uso y costo.
Usar inicialmente el webmail. JMAP está implementado; importar desde IMAP no equivale a ofrecer un servidor IMAP para Outlook.

## Costos y límites

Cloudflare Workers Paid parte de US$5/mes. Email Sending incluye 3.000 envíos/mes y cobra US$0,35 por 1.000 adicionales según su tarifa consultada. Dominio, impuestos y excesos de Workers, D1, R2, Queues, Images o AI no están incluidos en esa estimación.
Email Routing no cobra por recepción, pero ejecutar y almacenar el correo consume recursos.
La cuota diaria de envío depende de la cuenta; comprobarla antes de producción.
Alternativa: Cloudflare para recepción y Resend para envío, sujeto a sus cuotas (plan Free: 100 diarios); no se configura en esta preparación.

Importante: la documentación actual de Cloudflare indica 25 MiB por mensaje, pero ESTE código usa un presupuesto de 5 MiB para envíos Cloudflare y convierte adjuntos grandes en enlaces. No prometer 25 MiB de adjuntos directos. Esos enlaces pueden descargarse por quien los posea y caducan según la aplicación (actualmente 30 días).
Mailflare no incorpora un motor antivirus completo. Tampoco ofrece por sí mismo una garantía de entrega o recuperación. Probar spam, adjuntos y restauración antes de usarlo como único correo profesional.

Fuentes:
- https://developers.cloudflare.com/workers/platform/pricing/
- https://developers.cloudflare.com/email-service/platform/pricing/
- https://developers.cloudflare.com/email-service/platform/limits/
- https://resend.com/pricing
- [Proveedores](providers.md), [seguridad](security.md), src/lib/email/cloud-attachment-utils.ts.

## 1. Registrar y delegar el dominio

1. Registrar danostudios.cl desde la cuenta del propietario en https://www.nic.cl/.
2. Añadir la zona a la cuenta Cloudflare que alojará Mailflare.
3. Configurar en NIC Chile exactamente los nameservers asignados por Cloudflare.
4. Esperar a que Cloudflare muestre la zona Active. Verificar NS y propiedad antes de continuar.
5. Si aparecen registros existentes, exportarlos antes de cualquier cambio. No sustituir MX de otro servicio sin plan de migración.

No se incluyen valores MX, DKIM ni identificadores ficticios: deben obtenerse de la cuenta real.

## 2. Credenciales

Usar dos credenciales independientes, sin pegarlas en chat, GitHub, archivos versionados, URLs o argumentos de comandos.

- Despliegue: el acceso de Cloudflare Builds o un token limitado a la cuenta con Workers Scripts Edit, D1 Edit, Workers R2 Storage Edit, Queues Edit y Account Settings Read. Workers Routes Edit solo si se configura una ruta propia.
- Ejecución: secreto de Worker CF_TOKEN, limitado a la zona danostudios.cl: Zone Read, DNS Edit, Zone Settings Edit y Email Routing Rules Edit. Añadir Email Sending Edit en la cuenta para enviar.

Guardar CF_TOKEN en Cloudflare como Secret, no como variable de texto visible. No usar Global API Key. Consultar los nombres vigentes de permisos en [deployment.md](deployment.md).
No configurar GITHUB_UPDATE_TOKEN ni MAILFLARE_UPDATE_TOKEN: el actualizador upstream reemplaza el árbol completo y elimina personalizaciones.

## 3. Desplegar desde esta copia

Después de revisar e integrar la preparación:
https://deploy.workers.cloudflare.com/?url=https://github.com/danysepulveda20-oss/mailflare

Seleccionar la cuenta propietaria de la zona y mantener el nombre EXACTO mailflare, porque la referencia interna WORKER_SELF_REFERENCE usa ese nombre.
Comprobar que el flujo use este repositorio; si propone crear otra copia, conectar el repositorio existente desde Workers & Pages.
Raíz del proyecto: /. Instalar con npm ci. Comando de despliegue: npm run deploy (ya incluye build; no duplicarlo en otro paso).
No incluir migraciones remotas en el comando de despliegue.

Recursos esperados:

| Recurso | Nombre / binding |
| --- | --- |
| Worker | mailflare |
| D1 | mailflare / DB |
| R2 privado | mailflare-raw / BUCKET |
| Cola de entrada | mailflare-inbound / INBOUND_QUEUE |
| Cola de salida | mailflare-outbound / OUTBOUND_QUEUE |
| Cola del asistente | mailflare-agent / AGENT_QUEUE |
| Durable Object | RealtimeHub / REALTIME |

Verificar que el aprovisionamiento haya creado y enlazado todos los recursos. No crear duplicados si ya existen.
No publicar database_id ni configuración particular de la cuenta en la plantilla reutilizable.
Configurar CF_EMAIL_WORKER_NAME=mailflare. APP_URL debe ser la URL HTTPS real del despliegue; cambiarla al adoptar mail.danostudios.cl.
Activar R2 y el plan de Workers adecuado desde la cuenta del propietario; revisar cualquier cargo antes de aceptarlo.

## 4. Proteger y completar el primer inicio

El registro inicial crea al primer administrador y se cierra después. No dejar una instancia vacía expuesta al público.
Antes de habilitar el acceso web público, restringir el acceso HTTP al instalador con Cloudflare Access y comprobar que tanto el dominio personalizado como workers.dev y URLs de vista previa no permitan eludir la restricción. Si no se puede garantizar esto, detener el despliegue público hasta preparar el control de acceso.
Turnstile reduce automatización, pero no acredita que quien reclama el primer administrador sea el propietario. Si se usa, configurar NEXT_PUBLIC_TURNSTILE_SITE_KEY en la compilación y TURNSTILE_SECRET_KEY como secreto.
Abrir /setup bajo esa protección; el asistente aplica las migraciones y permite crear al administrador. El propietario introduce su contraseña o passkey directamente.
Verificar después que /api/auth/register rechace nuevos registros. No abrir el servicio hasta confirmar que el administrador pertenece al propietario.

## 5. Activar correo y DNS

1. Conectar danostudios.cl desde el administrador con recepción Cloudflare y, si está habilitado y contratado, envío Cloudflare.
2. Completar onboarding de Email Sending en Cloudflare.
3. Crear el buzón que el propietario elija; contacto@danostudios.cl es una propuesta, no un buzón creado.
4. Revisar los MX, SPF y DKIM generados por los proveedores. Mantener un solo registro SPF por hostname; combinar fuentes autorizadas si corresponde.
5. Empezar DMARC en modo de observación (p=none); no publicar una dirección de informes que todavía no exista. Pasar a quarantine/reject después de comprobar alineación y todos los remitentes legítimos.
6. Añadir mail.danostudios.cl como dominio personalizado del Worker solo cuando la zona esté activa y verificar TLS.
7. No confundir el hostname del webmail con los MX de recepción. No usar un registro web A/CNAME como sustituto del MX.
8. Mantener privados R2 y respaldos; no activar r2.dev público.

## 6. Criterios de aceptación

- Compilación, tipos, lint y pruebas automáticas aprobados.
- Usuario sin sesión no puede leer mensajes ni adjuntos privados; registro inicial cerrado.
- Recibir desde dos proveedores externos, responder y enviar a ambos.
- Revisar cabeceras: SPF, DKIM y DMARC alineados. Revisar spam y rebotes.
- Probar adjuntos pequeños y uno grande: verificar cuándo llega enlace y su caducidad.
- Verificar permisos entre buzones si se añaden más usuarios.
- Probar copia y restauración aisladas, incluyendo datos D1 Y objetos R2. La copia de registros de base de datos no sustituye una copia independiente de objetos.
- Vigilar fallos de colas, límites y consumo; activar alertas de facturación. No habilitar IA ni autoenvíos sin necesidad.
- Documentar la URL real y la fecha de prueba antes de anunciar el correo.

## Actualizaciones y recuperación

La comprobación Checks se habilita para este repositorio en esta preparación. Si GitHub exige activar Actions en el fork, hacerlo desde Actions y repetir las comprobaciones.
Actualizar mediante una rama y revisión, conservando personalizaciones. No usar Dashboard Update de upstream.
Antes de actualizar: copia de datos y objetos, revisión de cambios, comprobaciones y compilación.
Después de desplegar: aplicar migraciones pendientes desde Admin si corresponde.
Para fallos de código, volver a una versión conocida del Worker. Un rollback de código NO revierte D1: revisar compatibilidad de esquema y restaurar únicamente mediante un procedimiento probado.

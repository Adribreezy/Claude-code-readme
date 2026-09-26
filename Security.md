# SECURITY.md

# Reglas de seguridad

## 1. Modelo de amenaza

Asume que el agente puede ejecutar cualquier comando que su usuario pueda ejecutar,
que puede ser inducido a hacerlo por contenido no confiable (una pagina web, un
fichero, un issue, la salida de una herramienta) y que cualquier secreto visible
dentro del contenedor puede acabar en un prompt, un log o una peticion de red.
Diseña el despliegue para que el peor caso sea acotado.

## 2. Aislamiento del contenedor

- Ejecuta como usuario sin privilegios. Nunca como root.
- `cap_drop: [ALL]` y `no-new-privileges:true`.
- Mantén el perfil seccomp por defecto. No uses `seccomp:unconfined` ni `privileged`.
- Sistema de ficheros de solo lectura (`read_only: true`) con tmpfs para lo temporal.
- Limites de memoria, CPU y procesos (`mem_limit`, `cpus`, `pids_limit`).
- No uses `network_mode: host`, `pid: host` ni `ipc: host`.

## 3. Montajes

- Monta unicamente el directorio de trabajo y el de configuracion.
- Nunca montes `/var/run/docker.sock`: equivale a dar root sobre el host.
- Nunca montes `/`, el directorio personal del host, ni el agente SSH del host.
- Monta en solo lectura todo lo que no necesite escritura.
- Un directorio de trabajo por proyecto o por persona.

## 4. Secretos

- No metas secretos en la imagen, en el Dockerfile ni en el repositorio.
- Las variables de entorno son visibles con `docker inspect` y para todos los procesos
  del contenedor. Para secretos sensibles prefiere ficheros montados de solo lectura
  con permisos estrictos.
- Usa tokens de minimo privilegio, de solo lectura, con caducidad y uno por servicio.
- Claves SSH dedicadas y de despliegue, limitadas a un repositorio. Nunca tu clave personal.
- Nunca compartas tu suscripcion, tu sesion ni tu clave de API con otra persona.
- El directorio de configuracion (`data/claude/`) contiene sesion e historial: trátalo como un secreto.
- Si un secreto se ha mostrado en un log, un chat o un commit, considerarlo comprometido y rotarlo.

## 5. Red y exposicion

- Red propia para el contenedor. Sin acceso a redes internas que no necesite.
- Filtra la salida (egress) con una lista de destinos permitidos: la API del proveedor,
  tu servidor git y los registros de paquetes que uses. Observa el trafico real antes
  de cerrar la lista.
- Bloquea desde el contenedor los rangos privados que no necesite y el endpoint de
  metadatos de la nube (`169.254.169.254`).
- Publica puertos solo en `127.0.0.1`. El acceso remoto va por VPN, tunel SSH o proxy
  inverso con autenticacion.

## 6. Autenticacion y acceso

- No asumas que una interfaz web de terceros trae autenticacion. Verificalo en su codigo.
- Pon delante autenticacion fuerte (SSO o doble factor) y TLS.
- Acceso individual y revocable. Nada de cuentas compartidas.
- Recuerda que quien accede a la interfaz tiene una shell con tus credenciales.

## 7. Cadena de suministro

- Fija versiones de la imagen base y de los paquetes npm (nada de `latest`).
- Considera fijar la imagen base por digest.
- Revisa el codigo y los permisos de cualquier paquete, plugin o servidor MCP de
  terceros antes de instalarlo.
- Analiza la imagen con un escaner de vulnerabilidades en cada construccion.
- Reconstruye con regularidad para recibir parches.

## 8. Herramientas y servidores MCP

- Lista explicita de los servidores MCP permitidos. Ninguno por defecto.
- Cada servidor con un token propio, de solo lectura salvo necesidad demostrada.
- Nada de credenciales de administrador de infraestructura dentro del contenedor.
- Revisa periodicamente que servidores y plugins hay instalados.

## 9. Permisos de Claude Code

- No uses el modo que omite permisos (`--dangerously-skip-permissions`) en una
  instancia compartida ni con acceso a algo que importe.
- Define reglas `deny` para ficheros de secretos y comandos peligrosos, y `ask` para
  acciones irreversibles (push, borrados, despliegues).
- Si tu version lo admite, usa la configuracion gestionada para imponer estas reglas
  y que el usuario no pueda relajarlas.
- Estas reglas son una capa mas, no sustituyen al aislamiento.

## 8b. Datos y privacidad

- Lo que ve el agente se envia al proveedor del modelo. No proceses datos personales
  ni regulados sin base legal y sin acuerdo con el proveedor.
- Mantén los secretos fuera del alcance del agente con reglas `deny` y con montajes
  que no los incluyan.
- Informa a cada usuario de que sus prompts y ficheros salen del contenedor.

## 10. Multiusuario

- Un contenedor y un volumen por persona.
- Sin secretos compartidos entre personas.
- Limites de gasto por clave de API para acotar el impacto de un abuso.

## 11. Registro y monitorizacion

- Guarda los logs del contenedor y rotalos.
- Comprueba que los logs no contienen secretos.
- Alertas de reinicios anormales, picos de CPU o memoria y trafico de salida inesperado.
- Revisa periodicamente quien tiene acceso.

## 12. Respuesta a incidentes

1. Para el contenedor: `docker compose stop`.
2. Revoca y rota todas las credenciales que tuviera (API, tokens, claves SSH).
3. Conserva los logs y el volumen para analizarlos antes de borrarlos.
4. Revisa el historial de los repositorios a los que tuvo acceso.
5. Reconstruye desde una imagen limpia y con secretos nuevos.

## 13. Lista de comprobacion antes de compartir

- [ ] Corre como usuario sin privilegios, con `cap_drop: ALL` y seccomp por defecto
- [ ] Sin `privileged`, sin socket de Docker, sin montajes del host
- [ ] Ningun secreto en el repositorio, en su historial ni en la imagen
- [ ] `.env.example` sin valores reales y `.gitignore` correcto
- [ ] Versiones fijadas (imagen, Claude Code y paquetes)
- [ ] Puertos solo en loopback, acceso por VPN o proxy autenticado
- [ ] Filtro de salida de red configurado
- [ ] Reglas de permisos `deny` y `ask` configuradas
- [ ] Servidores MCP en lista explicita y con tokens de solo lectura
- [ ] Cada persona con su propio contenedor, volumen y credenciales
- [ ] Limites de recursos y de gasto establecidos

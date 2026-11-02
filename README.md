# Práctica Docker — Servicios Telemáticos

**Estudiante:** Jose Alejandro Castillo Arce (2230181) — Universidad Autónoma de Occidente
**Docker Hub:** https://hub.docker.com/r/jjalejandrox/ubuntuweb (etiqueta `v1`)
**Entorno:** Windows + VirtualBox 7.2 + Vagrant, box `ubuntu/jammy64` (20241002.0.0)

| VM | IP | RAM |
|---|---|---|
| `servidorUbuntu` | 192.168.100.3 | 2 GB |
| `clienteUbuntu` | 192.168.100.2 | 1 GB |

## 0. Problema encontrado: las VM no arrancaban en frío

`vagrant up` terminaba en `Timed out while waiting for the machine to boot` (20 min). Pasó también con la caja `bento/ubuntu-22.04`.

- El log de VirtualBox mostró `HMR3Init: Attempting fall back to NEM: VT-x is not available`: VirtualBox trabajaba sobre el hipervisor de Windows (Docker Desktop/WSL2).
- La consola de la VM mostró `watchdog: BUG: soft lockup - CPU#1 stuck for 370s` y `rcu_sched self-detected stall`.
- `bcdedit /set hypervisorlaunchtype off` quedó guardado, pero `systeminfo` siguió detectando un hipervisor.
- **Solución:** conjunto de ajustes en el `Vagrantfile`: 1 CPU, `--paravirtprovider none` y `--uartmode1 disconnected` (no se aisló cuál fue el decisivo). El arranque en frío pasó de más de 20 min a **1 min 17 s** (1:22:23 → 1:23:40) y se repitió 3 veces con éxito.

Evidencia: [00_vm_servidorUbuntu.png](evidencias/00_vm_servidorUbuntu.png)

## 1. Instalación de Docker (parte 2)

Docker CE 29.9.2, Compose v5.6.1 y Buildx 0.38.1. Se corrigió la guía: `signedby=` → `signed-by=`. `hello-world` funcionó y `docker.service` quedó `enabled` y `active (running)`. Storage driver `overlayfs` (Docker 29 usa el almacén de imágenes de containerd) y cgroup `systemd`.

## 2. Servidor httpd en el puerto 8800 (parte 3)

`docker pull httpd` (177 MB en disco) y `docker run -d --name web1 -p 8800:80 httpd`. El navegador mostró "It works!". El log mostró dos orígenes: `172.17.0.1` (curl desde la VM) y `192.168.100.1` (navegador de Windows). Luego se detuvo y borró el contenedor; la imagen se conservó.

Evidencia: [02_httpd_navegador.png](evidencias/02_httpd_navegador.png)

## 3. Imagen propia con Apache sobre Ubuntu (parte 4) y Ejercicio 1

Carpeta `test_docker/` con `Dockerfile` (ubuntu + apache2 + `COPY html1/ /var/www/html/`) y una página con mis datos. Imagen `jjalejandrox/ubuntuweb` (346 MB frente a 163 MB de `ubuntu`).

| Construcción | Tiempo |
|---|---|
| Primera | 71,5 s |
| Segunda, sin cambios | 1,9 s (pasos 2 a 6 en `CACHED`) |
| `--no-cache` | 72,1 s |

Apache respondió con `Server: Apache/2.4.66 (Ubuntu)`, distinto del `Apache/2.4.69 (Unix)` de la imagen `httpd`: es la imagen que construí.

Evidencia: [03_sitio_personalizado.png](evidencias/03_sitio_personalizado.png)

## 4. Docker Hub y prueba desde otra VM (parte 5) — Ejercicio 1

Se usó un token de acceso personal (Read & Write) en lugar de la contraseña; no se incluye en el repositorio. `docker tag` creó `v1`; `latest` y `v1` comparten el ID `647693e16157` (una imagen, dos nombres). `docker push` terminó con `v1: digest: sha256:647693e161572ce76d926343a621f72c5736cbb56102f73486f952ef958fed79`. Las capas de `ubuntu` y una de `httpd` salieron `Mounted from library/...`, y solo se subieron las nuevas.

En `clienteUbuntu` se instaló Docker, **sin `docker login`** (repositorio público), y `docker run --name webcliente -d -p 9900:80 jjalejandrox/ubuntuweb:v1` descargó 9 capas con el **mismo digest** del push. El sitio se ve en `192.168.100.2:9900`, y `curl http://192.168.100.3:9000` desde el cliente devolvió el `<h1>` del servidor (comunicación VM a VM). Tras apagar la VM, `webcliente` quedó `Exited (137)`: `docker run` sin `--restart` no revive el contenedor.

Evidencia: [04_dockerhub_repo.png](evidencias/04_dockerhub_repo.png), [04_cliente_navegador.png](evidencias/04_cliente_navegador.png)

## 5. Copiar archivos al contenedor (parte 6)

`test_docker2/voldocker/` con `index.html` y `pagina1.html`. Se corrigió la guía: el `COPY` usa `voldocker/` y no `html1/`. La construcción tardó 4,3 s porque los pasos 2 a 5 estaban en caché. Contenedor `webcontainer` en el puerto 9910; `docker exec -it webcontainer /bin/bash` mostró los dos archivos en `/var/www/html`. Al editar el archivo en el host, el contenedor seguía sirviendo `version 1`: el `COPY` fija el contenido al construir.

Evidencia: [05_testdir_navegador.png](evidencias/05_testdir_navegador.png)

## 6. Ejercicio 2 — Volúmenes

| Mecanismo | Comando | Resultado |
|---|---|---|
| Bind mount | `-v /vagrant/test_docker2/voldocker:/var/www/html` (puerto 9920) | El contenedor leyó el archivo del host (`version 2`, luego `version 3`) sin reconstruir; el 9910 siguió en `version 1`. Un archivo creado dentro apareció en el host (doble sentido). Editando desde el Bloc de notas de Windows el cambio se vio en el navegador. `inspect`: `bind ... RW=true`. |
| Volumen con nombre | `--mount source=datos_web,destination=/var/www/html` | Vacío, Docker copió el contenido de la imagen (`version 1`). Se modificó a `version 4 (volumen)`, se borró el contenedor, y uno nuevo mostró `version 4 (volumen)`. Ubicación: `/var/lib/docker/volumes/datos_web/_data` (el artículo de 2018 decía `vfs`). `inspect`: tipo `volume`. |
| Solo lectura | `...,readonly` | Leer funciona; escribir da `Read-only file system`; `RW=false`. Borrar el volumen en uso falló con `volume is in use`. |

Bind mount: carpeta propia, ideal para desarrollo. Volumen con nombre: administrado por Docker, para datos que deben persistir. Solo lectura: acceso sin riesgo de modificar los datos.

Evidencia: [06_bind_mount_9920.png](evidencias/06_bind_mount_9920.png)

## 7. Docker Compose (nginx + redis)

Se omitió la línea `version` (obsoleta). `docker compose config` añadió `name: dockercomposetest` y una red `default`. Con `up -d` se crearon 2 contenedores y la red `dockercomposetest_default`. Sin `ports:`, `curl` al 9090 dio `000`. Tras añadir `9090:80/tcp`, `up -d --no-deps web` recreó solo `web` y `database` conservó su antigüedad. `--scale database=4` creó 4 réplicas de redis (no publica puertos); con `web` no se podría, por el puerto ocupado. `down` borró contenedores y red.

Evidencia: [07_compose_nginx_navegador.png](evidencias/07_compose_nginx_navegador.png)

## 8. Ejercicio 3 — Compose + Flask + MySQL

Repositorio de la guía clonado en `docker-tutorial/` (`app/`, `db/`, `docker-compose.yml`). Las imágenes antiguas (`python:3.6`, `mysql:5.7`) funcionaron; no hizo falta la nota ARM. La app respondió `{"favorite_colors": [{"Lancelot": "blue"}, {"Galahad": "yellow"}]}` en `192.168.100.3:5000`.

**Cómo funciona:**
- **Host `db`, no `localhost`.** Dentro de un contenedor `localhost` es el propio contenedor. Compose crea una red y un DNS interno que resuelve cada servicio por su nombre.
- **Puerto 3306 interno y 32000 externo.** La app usa el 3306 por la red interna; `32000:3306` publica MySQL hacia la VM.
- **`init.sql`.** Se monta con `./db:/docker-entrypoint-initdb.d/:ro` (bind mount de solo lectura). MySQL ejecuta esos scripts solo al inicializar una base vacía.
- **`--host=127.0.0.1`.** Con `localhost` el cliente usa un socket local e ignora el puerto.

**Experimentos:**

| Experimento | Resultado |
|---|---|
| A: `INSERT` por el 32000 | La app mostró a `Arturo` |
| B: `inspect` y `restart db` | Dos montajes: `bind` (init.sql, `RW=false`) y un **volumen anónimo** en `/var/lib/mysql`. Tras el reinicio, 8 intentos con `200` y Arturo siguió. |
| C: `down` y `up` | Arturo **desapareció**: nuevo contenedor, volumen nuevo y `init.sql` ejecutado otra vez |
| Orden de arranque | Con volumen nuevo: `000` (Flask aún no escucha), `500` (Flask sin MySQL) y luego `200`. Compose no espera a que MySQL esté listo. |

Evidencia: [08_flask_navegador.png](evidencias/08_flask_navegador.png)

## 9. Observaciones

- `python:3.6` y `mysql:5.7` están fuera de soporte: sirven para aprender, no para producción.
- `version` y `links` están obsoletos en Compose v5 y solo generan avisos.
- `docker login` guarda credenciales sin cifrar en `/root/.docker/config.json`. Se usó un token y ese archivo no se incluye en el repositorio. Las credenciales `root/root` son de laboratorio.
- Los `RUN` del Dockerfile están en capas separadas, y un `apt clean` en capa aparte no reduce el tamaño.

## 10. Estructura del repositorio

`Vagrantfile`, `test_docker/` (ejercicio 1), `test_docker2/` (parte 6 y ejercicio 2), `dockerComposeTest/` (Compose), `docker-tutorial/` (ejercicio 3, código de la guía) y `evidencias/`.

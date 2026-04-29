# Instalación de PostgreSQL y Docker en el RAID

Esta guía asume que el SO ya está instalado y las particiones `/produccion` y `/desarrollo` ya están montadas dentro del arreglo RAID. Verificar antes de comenzar:

```bash
lsblk
df -vh
```

---

## 📋 Tabla de Contenidos

1. Configuración Inicial del Servidor
2. PostgreSQL en el RAID
3. Docker en el RAID
4. Verificación General
5. Contenedor de PostgreSQL
6. Contenedor de PgAdmin

---

## 1. Configuración Inicial del Servidor

Conectarse por SSH desde MobaXterm y convertirse en superusuario:

```bash
sudo su
```

### Actualizar zona horaria

```bash
date
timedatectl
timedatectl set-timezone America/Costa_Rica
timedatectl
date
```

### Instalar herramientas de red

```bash
apt update
apt install net-tools
ifconfig
```

> ⚠️ Omitir `apt upgrade` si así lo indica el enunciado.

---

## 2. PostgreSQL en el RAID

Los datos se instalarán en `/produccion`.

### Instalar PostgreSQL

```bash
apt install postgresql postgresql-contrib
```

### Detener el servicio

```bash
/etc/init.d/postgresql stop
```

### Dar permisos al directorio del RAID

```bash
chown -R postgres:postgres /produccion
```

### Inicializar los datos en el RAID

```bash
sudo -u postgres /usr/lib/postgresql/16/bin/initdb -D /produccion/postgresql/16/main
```

Si todo salió bien debe aparecer `Success` al final.

### Redirigir PostgreSQL al RAID

```bash
vim /etc/postgresql/16/main/postgresql.conf
```

Buscar con `/data_directory` y cambiar por:

```
data_directory = '/produccion/postgresql/16/main'
```

Guardar con `Escape` → `:wq` → `Enter`.

### Iniciar PostgreSQL

```bash
/etc/init.d/postgresql start
```

### Verificar que apunta al RAID ✅

```bash
sudo -u postgres psql -c "SHOW data_directory;"
```

Debe mostrar:

```
/produccion/postgresql/16/main
```

### Cambiar clave del usuario postgres

```bash
sudo -i -u postgres
psql
alter user postgres with password 'linux';
```

### Habilitar acceso remoto

```bash
vim /etc/postgresql/16/main/postgresql.conf
```

Modificar:

```
listen_addresses = '*'
```

```bash
vim /etc/postgresql/16/main/pg_hba.conf
```

Agregar al final:

```
host    all    all    192.168.56.0/24    md5
host    all    all    172.17.0.0/24      md5
```

### Reiniciar el servicio

```bash
/etc/init.d/postgresql stop
/etc/init.d/postgresql start
/etc/init.d/postgresql status
```

---

## 3. Docker en el RAID

Los datos se instalarán en `/desarrollo`.

### Instalar dependencias

```bash
apt install apt-transport-https
apt install ca-certificates
apt install curl
apt install gnupg-agent
apt install software-properties-common
```

### Agregar la clave GPG oficial de Docker

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### Configurar el repositorio estable

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Instalar Docker Engine

```bash
apt update
apt-get install docker-ce docker-ce-cli containerd.io docker-compose
```

### Verificar que Docker está corriendo

```bash
service docker status
```

### Redirigir Docker al RAID

```bash
systemctl stop docker
systemctl stop containerd
```

```bash
vim /etc/docker/daemon.json
```

Escribir dentro:

```json
{
  "data-root": "/desarrollo/docker"
}
```

Guardar con `Escape` → `:wq` → `Enter`.

### ⚠️ Redirigir containerd al RAID (IMPORTANTE)

> Sin este paso, containerd almacena las capas de imágenes en `/var/lib/containerd` dentro del disco del SO (sda3 de ~4GB), lo que causa el error **"no space left on device"** al hacer `docker pull` ya que el disco del SO tiene espacio muy limitado.

Crear el directorio en el RAID:

```bash
mkdir -p /desarrollo/containerd
```

Editar la configuración de containerd:

```bash
vim /etc/containerd/config.toml
```

Localizar la línea que dice `#root = "/var/lib/containerd"` (línea 17 aproximadamente), **descomentarla y cambiar la ruta:**

```toml
root = "/desarrollo/containerd"
```

Guardar con `Escape` → `:wq` → `Enter`.

Limpiar caché anterior para liberar espacio en el SO:

```bash
rm -rf /var/lib/containerd/*
```

### Iniciar los servicios

```bash
systemctl start containerd
systemctl start docker
```

---

## 4. Verificación General ✅

Ejecutar todos estos comandos para confirmar que todo está correctamente configurado antes de continuar:

```bash
# Estado del arreglo RAID
cat /proc/mdstat
sudo mdadm --detail /dev/md127
```

Resultado esperado:
```
State : clean
Active Devices : 2
Failed Devices : 0
```

```bash
# Ver particiones y montajes
lsblk
df -vh
```

```bash
# PostgreSQL apunta al RAID
sudo -u postgres psql -c "SHOW data_directory;"
```

Debe mostrar: `/produccion/postgresql/16/main`

```bash
# Docker apunta al RAID
docker info | grep "Docker Root Dir"
```

Debe mostrar: `Docker Root Dir: /desarrollo/docker`

```bash
# Containerd apunta al RAID
cat /etc/containerd/config.toml | grep root
```

Debe mostrar: `root = "/desarrollo/containerd"`

```bash
# Espacio usado en cada partición del RAID
du -sh /produccion/
du -sh /desarrollo/

# Limpiar residual de docker en el SO (si existe)
du -sh /var/lib/docker/
rm -rf /var/lib/docker/*
```

---

## 5. Contenedor de PostgreSQL

```bash
docker pull postgres

docker image ls

docker run --name contenedor-psql -e POSTGRES_PASSWORD=linux -d -p 2022:5432 postgres

docker ps
```

> ⚠️ `docker run` solo se ejecuta una vez (crea el contenedor). Para iniciarlo luego usar:

```bash
docker start contenedor-psql
```

---

## 6. Contenedor de PgAdmin

```bash
docker run -d -p 8080:80 --name pgadmin-contenedor \
  -e 'PGADMIN_DEFAULT_EMAIL=user@domain.com' \
  -e 'PGADMIN_DEFAULT_PASSWORD=linux' \
  dpage/pgadmin4
```

Acceder desde el navegador de Windows:

```
http://IP-SERVIDOR:8080
```

| Campo | Valor |
|-------|-------|
| Usuario | correo definido en `PGADMIN_DEFAULT_EMAIL` |
| Contraseña | definida en `PGADMIN_DEFAULT_PASSWORD` |

---

## 📝 Notas finales

**Sobre los discos de diferente tamaño:**
El uso de discos de 20 GB y 30 GB fue intencional para evidenciar que en RAID 1, el arreglo siempre se limita al disco de menor capacidad. Los 10 GB sobrantes del disco de 30 GB se desperdician. Si al momento de configurar el RAID fue necesario "recortar" el disco mayor para que funcionara, eso es equivalente al mismo comportamiento: en la práctica, el resultado es idéntico.

**Sobre las particiones `/produccion` y `/desarrollo`:**
Ambas particiones están dentro del arreglo RAID 1, por lo que todo lo almacenado en ellas — bases de datos de PostgreSQL, imágenes y contenedores de Docker — tiene redundancia automática. Si uno de los discos falla, el otro mantiene una copia exacta de toda la información.

---

> 📝 **Nota final sobre los discos de diferente tamaño:**  
> El uso de discos de 20 GB y 30 GB fue intencional para evidenciar que en RAID 1, el arreglo siempre se limita al disco de menor capacidad. Los 10 GB sobrantes del disco de 30 GB se desperdician. Si al momento de configurar el RAID fue necesario "recortar" el disco mayor para que funcionara, eso es equivalente al mismo comportamiento: en la práctica, el resultado es idéntico.

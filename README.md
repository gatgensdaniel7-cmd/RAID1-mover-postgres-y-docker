# 🗄️ Instalación de PostgreSQL y Docker en el RAID

> Esta guía asume que el SO ya está instalado y las particiones `/produccion` y `/desarrollo` ya están montadas dentro del arreglo RAID.
> Verificar antes de comenzar:
> ```bash
> lsblk
> df -vh
> ```

---

## 📋 Tabla de Contenidos

1. [Configuración Inicial del Servidor](#1-configuración-inicial-del-servidor)
2. [PostgreSQL en el RAID](#2-postgresql-en-el-raid)
3. [Docker en el RAID](#3-docker-en-el-raid)
4. [Contenedor de PostgreSQL](#4-contenedor-de-postgresql)
5. [Contenedor de PgAdmin](#5-contenedor-de-pgadmin)

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

> ⚠️ **Omitir `apt upgrade`** si así lo indica el enunciado.

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

Si todo salió bien debe aparecer **Success** al final.

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
```

```sql
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

vim /etc/docker/daemon.json
```

Escribir dentro:
```json
{
  "data-root": "/desarrollo/docker"
}
```

Guardar con `Escape` → `:wq` → `Enter`.

```bash
systemctl start containerd
systemctl start docker
```

### Verificar que apunta al RAID ✅

```bash
docker info | grep "Docker Root Dir"
```

Debe mostrar:
```
Docker Root Dir: /desarrollo/docker
```

---

## 4. Contenedor de PostgreSQL

```bash
docker pull postgres

docker image ls

docker run --name contenedor-psql -e POSTGRES_PASSWORD=linux -d -p 2022:5432 postgres

docker ps
```

> ⚠️ `docker run` solo se ejecuta **una vez** (crea el contenedor). Para iniciarlo luego usar:
> ```bash
> docker start contenedor-psql
> ```

---

## 5. Contenedor de PgAdmin

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

> 📝 **Nota final sobre los discos de diferente tamaño:**  
> El uso de discos de 20 GB y 30 GB fue intencional para evidenciar que en RAID 1, el arreglo siempre se limita al disco de menor capacidad. Los 10 GB sobrantes del disco de 30 GB se desperdician. Si al momento de configurar el RAID fue necesario "recortar" el disco mayor para que funcionara, eso es equivalente al mismo comportamiento: en la práctica, el resultado es idéntico.

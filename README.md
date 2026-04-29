# Despliegue de servicio

**Cloud Computing: Servicios y Aplicaciones**

**Pablo García Alvarado** – <pablog.alvarado@proton.me>  
Escuela Técnica Superior de Ingeniería Informática y de Telecomunicación  
Universidad de Granada

<p align="center">
  <img src="https://github.com/user-attachments/assets/dde05572-856d-429a-b17a-f6b6e36ed2bd"
       alt="image"
       width="500" />
</p>

**29/04/2026, Granada**

---

## Índice

1. [Introducción](#introducción)
2. [Entorno de desarrollo y de producción utilizado](#entorno-de-desarrollo-y-de-producción-utilizado) 
3. [Tarea 1](#tarea-1)  
4. [Tarea 2](#tarea-2)  
5. [Tarea 3](#tarea-3)  
6. [Conclusiones](#conclusiones)  
7. [Bibliografía](#bibliografía)

---

## Introducción

El objetivo de esta práctica es adquirir experiencia en el despliegue y gestión de servicios interconectados mediante contenedores, utilizando herramientas como Podman, podman-compose y Kubernetes.

A lo largo de la práctica se pretende comprender cómo diseñar e implementar distintas arquitecturas de servicios en función de los requisitos del sistema, prestando especial atención a aspectos como la persistencia de datos, la autenticación de usuarios, la escalabilidad, la gestión de réplicas, el balanceo de carga y la monitorización.

Además, se busca trabajar con configuraciones orientadas a la alta disponibilidad y al uso concurrente por parte de múltiples usuarios, acercando el entorno de trabajo a escenarios reales de administración y despliegue de servicios en infraestructuras modernas.

---

## Entorno de desarrollo y de producción utilizado

La práctica se ha desarrollado utilizando un entorno basado en contenedores, trabajando principalmente desde terminal mediante `podman` y `podman-compose`.

El entorno utilizado ha sido el siguiente:

- Sistema operativo local: Windows, utilizando terminal Git Bash para la conexión y gestión de ficheros.
- Entorno de despliegue: servidor @galeon.ugr.es proporcionado para la práctica.
- Gestor de contenedores: Podman.
- Herramienta de composición de servicios: podman-compose.
- A lo largo de la práctica se han trabajado distintos despliegues de servicios mediante contenedores y Kubernetes. En la primera parte se desplegó una arquitectura basada en LDAP, ownCloud, una base de datos y HAProxy. Posteriormente, se amplió el trabajo hacia escenarios de balanceo, escalabilidad y despliegue en Kubernetes mediante Minikube, utilizando recursos como pods, deployments, services y réplicas.

---



## Tarea 1

El objetivo de esta primera tarea ha sido desplegar una arquitectura de servicios interconectados mediante contenedores usando `podman` y `podman-compose`. En concreto, se ha configurado un entorno formado por ownCloud, una base de datos MariaDB, un servidor LDAP y un servicio Redis.

La finalidad principal era disponer de una instancia funcional de ownCloud accesible desde navegador, con persistencia de datos y autenticación de usuarios mediante LDAP. De esta forma, se trabaja una arquitectura similar a la que podría encontrarse en un entorno real, donde la aplicación principal no gestiona todos los elementos por sí sola, sino que depende de servicios externos especializados.

Los servicios desplegados en esta tarea han sido:

- `pablo_owncloud`: aplicación web ownCloud.
- `pablo_mariadb`: base de datos MariaDB para ownCloud.
- `pablo_redis`: servicio Redis usado como apoyo por ownCloud.
- `pablo_ldap`: servidor LDAP para gestionar usuarios externos.

<img width="1491" height="160" alt="p1_cc_2" src="https://github.com/user-attachments/assets/1d3a85c5-aabe-4088-b2b1-03aa8ef08726" />

Antes de comenzar el despliegue, se comprobó el entorno de trabajo y las versiones de las herramientas disponibles. Para ello se utilizaron comandos básicos de identificación del sistema y versiones:

```bash
hostname
whoami
pwd
podman --version
podman compose version
```

<img width="485" height="146" alt="image" src="https://github.com/user-attachments/assets/57fa6eb8-c69e-44b3-a03e-e43d753313fd" />

A continuación, se creó una estructura de directorios para organizar los ficheros de configuración, scripts, capturas y datos persistentes de los servicios:

```bash
mkdir -p practica-owncloud/{ldap,scripts,capturas,data/owncloud,data/mariadb,data/ldap/database,data/ldap/config}
cd practica-owncloud
```

Se decidió separar los datos de cada servicio en subdirectorios dentro de data/, de forma que la información persistente no quedara mezclada con los ficheros de configuración.

El despliegue se definió mediante un archivo podman-compose.yml. Para editarlo se hizo uso de nano. 

```yaml
version: "3.8"

services:
  mariadb:
    image: docker.io/library/mariadb:10.11
    container_name: pablo_mariadb
    environment:
      MARIADB_ROOT_PASSWORD: rootpass
      MARIADB_DATABASE: owncloud
      MARIADB_USER: owncloud
      MARIADB_PASSWORD: owncloudpass
    volumes:
      - ./data/mariadb:/var/lib/mysql:Z
    networks:
      - owncloud-net

  redis:
    image: docker.io/library/redis:7
    container_name: pablo_redis
    command: ["redis-server", "--appendonly", "yes"]
    networks:
      - owncloud-net

  ldap:
    image: docker.io/osixia/openldap:1.5.0
    container_name: pablo_ldap
    environment:
      LDAP_ORGANISATION: "Example Inc."
      LDAP_DOMAIN: "example.org"
      LDAP_ADMIN_PASSWORD: "admin"
    ports:
      - "20111:389"
      - "20112:636"
    volumes:
      - ./data/ldap/database:/var/lib/ldap:Z
      - ./data/ldap/config:/etc/ldap/slapd.d:Z
    networks:
      - owncloud-net

  owncloud:
    image: docker.io/owncloud/server:10.15
    container_name: pablo_owncloud
    depends_on:
      - mariadb
      - redis
      - ldap
    ports:
      - "20110:8080"
    environment:
      OWNCLOUD_DOMAIN: "galeon:20110"
      OWNCLOUD_TRUSTED_DOMAINS: "galeon,localhost,127.0.0.1"
      OWNCLOUD_DB_TYPE: "mysql"
      OWNCLOUD_DB_NAME: "owncloud"
      OWNCLOUD_DB_USERNAME: "owncloud"
      OWNCLOUD_DB_PASSWORD: "owncloudpass"
      OWNCLOUD_DB_HOST: "mariadb"
      OWNCLOUD_ADMIN_USERNAME: "admin"
      OWNCLOUD_ADMIN_PASSWORD: "admin"
      OWNCLOUD_REDIS_ENABLED: "true"
      OWNCLOUD_REDIS_HOST: "redis"
    volumes:
      - ./data/owncloud:/mnt/data:Z
    networks:
      - owncloud-net

networks:
  owncloud-net:
```

Durante los primeros intentos de despliegue, algunos contenedores no se iniciaron correctamente. Para diagnosticar el problema se consultaron los logs de MariaDB, LDAP y ownCloud mediante los siguientes comandos:

```bash
podman logs pablo_mariadb --tail=80
podman logs pablo_ldap --tail=80
podman logs pablo_owncloud --tail=80
```

La revisión de los logs permitió detectar que el problema estaba relacionado con los permisos de los directorios utilizados como volúmenes persistentes. Al trabajar con Podman, los procesos que se ejecutan dentro de los contenedores utilizan usuarios internos concretos. Por tanto, si los directorios del sistema anfitrión no tienen los permisos adecuados, servicios como MariaDB, OpenLDAP u ownCloud no pueden escribir correctamente sus datos.

Para solucionarlo se detuvieron los contenedores y se ajustó la propiedad de los directorios persistentes usando podman unshare chown:

```bash
podman-compose down

podman unshare chown -R 999:999 ./data/mariadb
podman unshare chown -R 911:911 ./data/ldap/database ./data/ldap/config
podman unshare chown -R 33:33 ./data/owncloud

podman-compose up -d
```

---

## Tarea 2

---

## Tarea 3

---

## Conclusiones

---

## Bibliografía

1. Guion de la práctica 1 de Cloud Computing: servicios y aplicaciones, última vez consultado el 29 de abril de 2026 en https://github.com/j-m-benitez/cc2526/blob/main/practice1/README.md#entrega-de-la-pr%C3%A1ctica-a-traves-de-prado-documentaci%C3%B3n-y-evaluaci%C3%B3n-de-la-pr%C3%A1ctica
2.  

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
- Entorno de despliegue: servidor `galeon.ugr.es` proporcionado para la práctica.
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

Se decidió separar los datos de cada servicio en subdirectorios dentro de `data/`, de forma que la información persistente no quedara mezclada con los ficheros de configuración.

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

En esta configuración se definió una red interna llamada `owncloud-net`, compartida por todos los servicios. Esta decisión permite que los contenedores se comuniquen entre sí usando sus nombres de servicio, como `mariadb`, `redis` o `ldap`, sin necesidad de exponer todos los puertos al exterior.

Solo se publicaron los puertos necesarios para la prueba desde el navegador o desde el exterior del entorno: el puerto `20110` para acceder a ownCloud, el `20111` para LDAP y el `20112` para LDAPS. MariaDB y Redis no se publicaron hacia fuera porque únicamente deben ser utilizados por los servicios internos de la arquitectura.

Durante los primeros intentos de despliegue, algunos contenedores no se iniciaron correctamente. Para diagnosticar el problema se consultaron los logs de MariaDB, LDAP y ownCloud mediante los siguientes comandos:

```bash
podman logs pablo_mariadb --tail=80
podman logs pablo_ldap --tail=80
podman logs pablo_owncloud --tail=80
```

La revisión de los logs permitió detectar que el problema estaba relacionado con los permisos de los directorios utilizados como volúmenes persistentes. Al trabajar con Podman, los procesos que se ejecutan dentro de los contenedores utilizan usuarios internos concretos. Por tanto, si los directorios del sistema anfitrión no tienen los permisos adecuados, servicios como MariaDB, OpenLDAP u ownCloud no pueden escribir correctamente sus datos.

Para solucionarlo se detuvieron los contenedores y se ajustó la propiedad de los directorios persistentes usando `podman unshare chown`:

```bash
podman-compose down

podman unshare chown -R 999:999 ./data/mariadb
podman unshare chown -R 911:911 ./data/ldap/database ./data/ldap/config
podman unshare chown -R 33:33 ./data/owncloud

podman-compose up -d
```

Se volvió a comprobar que los servicios se hubieran terminado de levantar correctamente con `podman ps` y `podman ps -a`.

<img width="1491" height="160" alt="p1_cc_2" src="https://github.com/user-attachments/assets/1d3a85c5-aabe-4088-b2b1-03aa8ef08726" />

Una vez desplegado el contenedor LDAP, se comprobó que el servicio respondía correctamente mediante una búsqueda LDAP ejecutada dentro del propio contenedor:

```bash
podman exec pablo_ldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin
```

Este comando permite consultar el árbol LDAP usando autenticación simple. La opción -H indica la dirección del servidor LDAP, -b establece la base de búsqueda, -D indica el usuario con el que se realiza la consulta y -w proporciona la contraseña.

A continuación, se creó una unidad organizativa llamada People, destinada a agrupar los usuarios de la organización. Para ello se creó un fichero LDIF:

```bash
nano ldap/add_ou_people.ldif
```

El contenido del fichero fue el siguiente:

```ldif
dn: ou=People,dc=example,dc=org
objectClass: top
objectClass: organizationalUnit
ou: People
description: Unidad organizativa para usuarios de la empresa
```

Posteriormente, el fichero se copió al contenedor LDAP:

```bash
podman cp ldap/add_ou_people.ldif pablo_ldap:/tmp/add_ou_people.ldif
```

Y se añadió la unidad organizativa al directorio mediante ldapadd:

```bash
podman exec pablo_ldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  -f /tmp/add_ou_people.ldif
```

La decisión de crear ou=People se tomó para mantener una estructura LDAP ordenada. En lugar de añadir los usuarios directamente bajo dc=example,dc=org, se agrupan dentro de una unidad organizativa específica, lo cual facilita su búsqueda, filtrado y posterior integración con ownCloud.

Una vez creada la unidad organizativa, se añadieron dos usuarios al directorio LDAP: pablo y lorena. Para ello se prepararon dos ficheros LDIF:

```bash
nano ldap/pablo.ldif
nano ldap/lorena.ldif
```

Estos ficheros definían los atributos necesarios para que los usuarios pudieran ser reconocidos posteriormente por ownCloud, incluyendo atributos como uid, cn, sn, uidNumber, gidNumber, homeDirectory, loginShell y la clase de objeto inetOrgPerson.

Después, los ficheros se copiaron al contenedor LDAP:

```bash
podman cp ldap/pablo.ldif pablo_ldap:/tmp/pablo.ldif
podman cp ldap/lorena.ldif pablo_ldap:/tmp/lorena.ldif
```

Y se importaron en el directorio con ldapadd:

```bash
podman exec pablo_ldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  -f /tmp/pablo.ldif

podman exec pablo_ldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  -f /tmp/lorena.ldif
```

Una vez añadidos los usuarios, se les asignó contraseña mediante ldappasswd:

```bash
podman exec pablo_ldap ldappasswd -s pablo123 \
  -w admin \
  -D "cn=admin,dc=example,dc=org" \
  -x "uid=pablo,ou=People,dc=example,dc=org"

podman exec pablo_ldap ldappasswd -s lorena123 \
  -w admin \
  -D "cn=admin,dc=example,dc=org" \
  -x "uid=lorena,ou=People,dc=example,dc=org"
```

El uso de ficheros LDIF permite definir las entradas LDAP de forma clara y reproducible. Además, separar cada usuario en un fichero distinto facilita realizar cambios o añadir nuevos usuarios en el futuro.

Para comprobar que los usuarios se habían añadido correctamente al directorio, se realizó una búsqueda LDAP sobre la unidad organizativa `People`:

```bash
podman exec pablo_ldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b ou=People,dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin
```

Además, se realizó una búsqueda filtrando por la clase de objeto inetOrgPerson, ya que esta clase sería utilizada posteriormente por ownCloud para identificar qué entradas del directorio debían considerarse usuarios válidos:

```bash
podman exec pablo_ldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  "(objectClass=inetOrgPerson)" uid cn dn
```

<img width="740" height="953" alt="p1_cc_1" src="https://github.com/user-attachments/assets/2a04d056-2451-4954-9431-18447e080d25" />

El resultado mostró correctamente los usuarios pablo y lorena, por lo que se confirmó que el servidor LDAP estaba funcionando y que las entradas habían sido creadas correctamente.

Una vez comprobado que el servidor LDAP contenía los usuarios correctamente, se configuró ownCloud accediendo al servicio a través de http://galeon:20110, para utilizar LDAP como sistema externo de autenticación. Para ello se accedió con el usuario administrador de ownCloud y se habilitó/configuró la autenticación LDAP desde el apartado de administración.

Como todos los contenedores forman parte de la red interna `owncloud-net`, ownCloud puede comunicarse con el servidor LDAP usando el nombre del servicio definido en `podman-compose.yml`:

```text
ldap:389
```

Esta decisión evita depender de direcciones IP internas de contenedores, que podrían cambiar al recrear el entorno. Usar el nombre del servicio hace que la configuración sea más estable y reproducible.

En la configuración de ownCloud se comprobó que la conexión con LDAP era correcta, apareciendo el estado Configuration OK. Además, en la pestaña de usuarios se configuró el filtro para que ownCloud reconociera únicamente entradas LDAP de tipo inetOrgPerson.

objectClass=inetOrgPerson

Con este filtro, ownCloud detectó los dos usuarios creados previamente en LDAP.

También se configuró el atributo utilizado por ownCloud para permitir el inicio de sesión de los usuarios LDAP. En este caso, se utilizó el atributo `uid`, de forma que los usuarios pudieran iniciar sesión con nombres como `pablo` o `lorena`.

El filtro de login utilizado fue:

```text
(&(|(objectclass=inetOrgPerson))(|(uid=%uid)))
```

Este filtro indica que ownCloud debe buscar entradas que pertenezcan a la clase inetOrgPerson y cuyo atributo uid coincida con el nombre introducido en el inicio de sesión.

Para comprobar la configuración, se utilizó la herramienta de verificación de ownCloud introduciendo el usuario pablo. El sistema devolvió el mensaje User found and settings verified, confirmando que ownCloud era capaz de localizar correctamente al usuario en LDAP.

Tras finalizar la configuración de LDAP en ownCloud, se probó el inicio de sesión con los usuarios creados en el directorio. Se comprobó que los usuarios `pablo` y `lorena` podían acceder correctamente y que ownCloud mostraba sus perfiles con el nombre completo asociado a sus entradas LDAP.

<img width="1918" height="1033" alt="p1_cc_5" src="https://github.com/user-attachments/assets/68df90fb-2b7f-4ab0-b643-0a8469b9f84c" />

<img width="1918" height="1030" alt="p1_cc_6" src="https://github.com/user-attachments/assets/b9407477-1596-4899-b3be-512cd267301c" />

La persistencia de datos se configuró mediante volúmenes enlazados a directorios locales dentro del proyecto. En concreto, se utilizaron los siguientes montajes:

```yaml
volumes:
  - ./data/mariadb:/var/lib/mysql:Z
  - ./data/ldap/database:/var/lib/ldap:Z
  - ./data/ldap/config:/etc/ldap/slapd.d:Z
  - ./data/owncloud:/mnt/data:Z
```

Esta configuración permite que los datos principales de la arquitectura no dependan únicamente del ciclo de vida de los contenedores. Si un contenedor se detiene o se recrea, los datos almacenados en los directorios locales se mantienen.

En concreto:

MariaDB conserva la base de datos utilizada por ownCloud.
LDAP conserva tanto la base de datos del directorio como su configuración.
ownCloud conserva sus datos internos en el directorio asociado.

Durante el desarrollo de la tarea se detuvieron y levantaron los servicios en varias ocasiones mediante:

```bash
podman-compose down
podman-compose up -d
```

Tras reiniciar los contenedores, se verificó que la arquitectura seguía funcionando y que los usuarios LDAP continuaban disponibles.

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

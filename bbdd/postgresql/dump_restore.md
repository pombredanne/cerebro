http://www.postgresql.org/docs/8.1/static/backup.html
https://www.opsdash.com/blog/postgresql-backup-restore.html

https://www.pgbarman.org/index.html
Solución completa de backup y restore simplificada
NOTA: Usar esto para hacer backups!!

La idea es que el backup es un servicio que debe estar corriendo todo el rato, llevándose los WAL y de vez en cuando haciendo basebackup.


Los roles y tablespaces no están dentro de ninguna database, están a nivel global.


Dos tipos de backups:
 - lógicos (lo que hace pg_dump): saca el contenido de una base de datos
   Cons:
     - hace queries y carga la bbdd y si metemos mas parallel jobs, más carga
     - no permite arrancar un standy server
     - ocupa mucho espacio
     - mala performance
     - puede joder el filesystem cache
     - escribir el dump genera I/O
     - restore muy lento para ddbb grandes!
     - locks que bloquean DDL
   Pros:
     - flexibilidad
     - solo hace falta un user read only
     - se pueden restaurar solo ciertos objetos (usando el formato custom)
     - se puede modificar el SQL a mano antes de restaurarlo (usando el formato SQL)
     - comprime
     - ocupa poco espacio

 - físicos (base backup): copia de los ficheros de pg_data
   Cons:
     - ocupa más??
     - se hace un backup de todo (no podemos elegir ciertas databases)
     - tiene que restaurarse en un postgres igual (arquitectura, version, compile flags and paths)
     - genera mucho I/O (lectura de todos los ficheros)
   Pros:
     - más rápidos

Parece que lo mejor es tener un hot standby server donde realizar los backups (pero tenemos el coste de tener otro server).
Y realizar full backups periodicamente mientras almacenamos continuamente los ficheros WAL, esto nos permite restaurar en un punto determinado del tiempo (PITR, point-in-time recovery)
  mirar como se restaura un PITR en https://www.opsdash.com/blog/postgresql-backup-restore.html#point-in-time-recovery-pitr

También podemos hacer un backup lógico en un hot standby. Tener en cuenta: https://dba.stackexchange.com/questions/30626/running-pg-dump-on-a-hot-standby-server

Un full backup cada n días y un incremental backup (WAL files) cada hora.

Monitorizar que estamos realizando los backups, el tiempo que tardan, probar a restaurar los últimos backups y el tiempo de restauración:
  you should also have another cron job that picks up a recent backup and tries to restore it into an empty database, and then deletes the database. This ensures that your backups are accessible and usable


# Backup lógico
Se un dump de los datos, no da la database tal cual.
Permite mover datos entre distintas releases.
Nos permite sacar solo algunas tablas, o solo obtener los schemas de las tablas.

pg_dumpall -g
  dump solo los elementos globales (roles y tablespaces)

pg_dumpall
  usar ~/.pgpass porque nos pedirá la password después de cada dump de cada db

pg_dump siempre es compatible con versiones antiguas. Siempre mejor usar el último psql aunque ataquemos a bbdd antiguas.

Podemos solo hacer dump de los datos o solo del schema.

## Formato custom
Lo mejor es siempre usar el archive (custom) format.
Nos permite pasar a sql file con pg_restore.

El formato dir lo bueno es que nos permite ejecutar en paralelo.

Custom, más potente. Nos permite a la hora de restaurar elegir el orden o seleccionar que restaurar:
Lleva compresión (mirar parámetro -Z).
  Más compresión, limitado por la CPU
  Menos compresión, limitado por I/O

pg_dump -Fc -d prueba -f prueba.custom
  -x si no queremos guardar informacion de roles (permisos de las tablas)
  -j=N para meter varios hilos
  -Z=n para seleccionar la compresion

Si queremos seleccionar la tabla de un schema en partiular:
schemaName.tableName

Ver contenido del dump (los schemas, no los datos):
pg_restore -l prueba.custom > fichero.list

Podemos usar la salida de este comando para quitar (o comentar ";") lo que no queremos restaurar y luego usar "pg_restore --use-list=fichero.list fichero.dump" para recuperar solo lo que queramos.

Ver todo el contenido:
pg_restore fichero.custom | less

Restaurar (CUIDADO, muy lento para ddbb grandes):
createdb prueba
pg_restore -v -e -Fc -d prueba -1 /backup/prueba.custom
  -1 todo en una única transaccion, si falla deja vacía la bbdd. Si no ponemos el parametro y falla por una constraint, tendremos igualmente los datos importados
  -v verbose
  -e exit on error
  -Fc format custom
  --no-privileges --no-owner "role XXX does not exist"

Podemos quitar -1 y usar -jN para paralelizar (no compatible con -1)
Mejorar performance con fsync=off en el file system? (https://www.hagander.net/talks/Backup%20strategies.pdf)

Meter -j va a consumir mucha memoria. Va a usar muchas veces la maintenance_working_mem para crear los índices.


## Formato SQL plano
Sin compresión
  pg_dump -d dbname -n public -f outfile.sql

  Backup de todas las bases de datos:
  pg_dumpall > outfile

  Ejemplo con postgres en docker:
  docker exec -it -u postgres postgres pg_dump -d basedatos -t tabla > tabla.sql

Restaurar simple
  createdb dbname
  psql dbname < infile
   o
  psql -f infile -d dbname

La forma correcta (-1 indica que se haga todo en una única transacción, "o todo o nada"):
PGOPTIONS='--client-min-messages=warning' psql -X -q -1 -v ON_ERROR_STOP=1 --pset pager=off -d mydb_new -f mydb.sql -L restore.log

-1, hacer en una única transaction
-X, no cargar el ~/.psqlrc

Tras el restaure, ejecutar ANALYZE:
vacuumdb --analyze-in-stages
  genera las estadísticas en tres fases, para ir conociendo un poco sobre las tablas.
  Si intentamos generarlas de golpe tenemos un gap tan grande entre no tener nada que puede ser muy costoso


Con compresión
  # su postgres
  $ pg_dump dbname | gzip > filename.gz

  Restauración
    # su postgres
    $ createdb dbname
    $ gunzip -c filename.gz | psql dbname



-c, pone unos drops para borrar todo antes de hacer un recover, cómodo para desarrollo para destruir&crear rápido
-C, incluir los crearate database
--insert, para ser más compatible, usar INSERT en vez de COPY, pero será más lento
  -D, para añadir también los nombres de las columnas a los INSERT


## Backup periodico en cron
https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux

Los scripts y el fichero de conf están en backup-scripts
pg_backup.sh - hace simplemete el backup
pg_backup_rotated.sh - hace el backup y rota los ficheros

IMPORTANTE:
Por defecto el comando es 'psql -h "$HOSTNAME"', esto hace que el usuario intente acceder via localhost.
La autenticación por defecto permite al usuario postgres acceder pero via socket.
Una solución sin cambiar la autentificación es quitar '-h "$HOSTNAME"' de los comandos psql.

Tenemos que crear /home/backups/database/postgresql/ y darle permisos al usuario postgres
mkdir -p /home/backups/database/postgresql/ && chown postgres:postgres -R /home/backups/database/
chgrp postgres /home/backups; chmod g+rwx /home/backups

La entrada de cron debe configurarse para ejecutarse como el usuario postgres.
Los scripts pg_backup.sh y pg_backup_rotated.sh van a buscar el fichero pg_backup.config en su mismo directorio.
También le podemos pasar la ubicación del fichero con el parámetro -c

cp pg_backup.config /etc/ && chown root:postgres /etc/pg_backup.config && chmod 640 /etc/pg_backup.config
cp pg_backup.sh /usr/local/sbin/ && chmod 755 /usr/local/sbin/pg_backup.sh
cp pg_backup_rotated.sh /usr/local/sbin/ && chmod 755 /usr/local/sbin/pg_backup_rotated.sh

/etc/cron.d/postgresql
# Generate backup all days at 02:30
30 02 * * * postgres /usr/local/sbin/pg_backup_rotated.sh -c /etc/pg_backup.config





# Base backup / físico
https://blog.2ndquadrant.com/what-does-pg_start_backup-do/
Esto hace un backup de un postgres entero, de los ficheros binarios.

Se hace conectando un cliente con el protocolo de replicación y obteniendo una copia consistente de PGDATA tras el final de alguna transacción.

Necesitamos explicitar un usuario que pueda conectarse de este modo (pg_hba.conf):
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   replication     postgres                                trust

kill -HUP PID_POSTGRES


Necesitamos configurar el max_wal_senders a, al menos, 4 (dos conex para el pg_basebackup y otras dos extra por si se desconectase pudiese inmediatamente reconectar)
max_wal_senders = 4
wal_level = replica  # puede tener cierto impacto en performance para algunos comandos (crear tablas, indices, etc) https://www.postgresql.org/docs/9.6/static/populate.html#POPULATE-PITR
systemctl restart postgres (requiere reinicio)

Para evitar perder datos entre el comienzo del backup y el fin, es necesario que se obtengan tambien los ficheros transaction log (WAL), tendremos que poner el -D

Si queremos hacer un base backup parece que hay que usar
pg_basebackup --xlog-method=stream -D /path/to/dir -P
  con "stream" no podemos usar tar. Lo mejor es usar stream (el de por defecto)
  el espacio consumido será lo que ocupe PGDATA
  tiene bastante impacto en el I/O
  --xlog-method=stream se lleva los WAL mientras dure el backup (valor por defecto)
pg_basebackup --xlog-method=fetch --format=tar -z -D /path/to/dir -P
  --xlog-method=fetch it is necessary for the wal_keep_segments parameter to be set high enough that the log is not removed before the end of the backup. If the log has been rotated when it's time to transfer it, the backup will fail and be unusable.

  -P show progress (hace el backup algo más lento)
  --format=tar -Z: generamos ficheros .tar.gz por cada tablespace
  -Z [0-9] nos permite especificar mayor/menor tasa de compresión


Si por otro lado ya nos estamos llevando los ficheros WAL, solo tenemos que hacer el basebackup
Podemos usar el "archive_command = %p /archiveDir/%f", que, cuando se llene un WAL, se copiará a otro directorio.
También podemos usar pg_receivewal con el que nos vamos llevando los WAL files.


Restaurar, parar postgres, mover los ficheros al PGDATA y arrancar.

NO HACER!
psql -U usuario -d postgres -p ${POSTGRESQL_PORT} -c "select pg_start_backup('Daily Backup'), current_timestamp"
tar -cvzf ${BACKUP_DIR}/data-$(date +%Y%m%d).tar.gz ${DATA_BASE_DIR}
psql -U usuario -d postgres -p ${POSTGRESQL_PORT} -c "select pg_stop_backup(), current_timestamp"

Estos comandos fuerzan a hacer un checkpoint y escribir todo a disco para poder hacer una copia de los ficheros a nivel de sistema de ficheros.
Ejecutar pg_start_backup no limita que se siga escribiendo al data directory.

The key point is that the base backup is NOT a consistent copy of the database. You might have copied every file, but all the data is taken at different times. So its wrong. Until you recover the database with the WAL changes that occurred between the start backup and the stop backup.
FIN NO HACER!




# Point in time recovery (PITR)
Nos permite, a partir de un base backup, ir aplicando los WAL files hasta la hora que queramos, de esta manera dejaremos la bbdd en un punto en el tiempo determinado.



# Restaurar
Muchas veces será más facil usar PITR en otro server para ir al momento antes de un problema y recuperar esos datos en la bd de producción.
Si solo hacemos recuperación del backup, perderemos los nuevos datos escritos desde el problema hasta el momento actual.


Si tenemos que restaurar un backup:
  1. Recuperar de un base backup (copiar el DATADIR)
  2. Definir el recovery target (edit recovery.conf)
     En la linea "reocver_command" debemos especificar donde están los WAL: "cp /path/con/los/wal/%f %p"
     Aqui podemos especificar PITR. recovery_target_time (también podemos poner el lsn, xid o target_name, que tendremos que haber especificado antes. Especificar si la queremos incluir o no)
     recovery_target='immediate', para lo antes posible (base backup + los mínimos wal para que sea consistente)
     con recovery_target_action definimos que pasa cuando se recupere: promote (moverla a master), apagarse, pausarse (este nos permite consultar la db sin que la db se mueva)
       shutdown aplica los wal al base dir, conseguimos un datadir "compacto", que funciona por si solo, sin wal files.
       si lo ponemos como promote, al terminar y arrancar renombrara el fichero recovery.conf y los ficheros wal ahora serán 00000002....
  3. start database server



# Limpiar
Generalmente tendremos varios base_backups y luego un archiveDir con todos los wal.
Si queremos borrar backups antiguos, tendremos que chequear el LSN de start del backup que queremos borrar y podremos borrar los wal previos a esos.

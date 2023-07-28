# SeaweedFS

Deployment of SeaweedFS, a distributed object store and file system

## Tareas comunes

### Cambio de replicación y rebalanceo de volúmenes

Ejecutar desde un nodo `volume`, por ejemplo:

```sh
weed shell -master swfs-master-1:9333,swfs-master-2:9333,swfs-master-3:9333

lock

# cambia ajuste de replicación por defecto
volume.configure.replication -replication=100

# aplica cambio de replicación
volume.fix.replication

# reparte equitativamente los volúmenes entre los nodos
volume.balance -force

unlock

exit
```

### Migración de store usado por filer

Ejecutar desde un nodo `filer`.

Primero, se guarda un backup del store actual:

```sh
weed shell -master swfs-master-1:9333,swfs-master-2:9333,swfs-master-3:9333 -filer 127.0.0.1:8888 -filerGroup filesystem

# fija directorio dentro de swfs desde el que guardar
fs.cd /

# genera fichero .meta con el backup del store
fs.meta.save

exit
```

Tras relanzar el servicio con el nuevo store, se carga el backup:

```sh
weed shell -master swfs-master-1:9333,swfs-master-2:9333,swfs-master-3:9333 -filer 127.0.0.1:8888 -filerGroup filesystem

# carga fichero de backup en el nuevo store
fs.meta.load 127.0.0.1:8888-*.meta

exit

# elimina fichero backup
rm 127.0.0.1:8888-*.meta
```

### Mantenimiento de store etcd (en desuso)

Ejecutar desde un nodo `etcd`:

```sh
# listar miembros del cluster con sus campos
etcdctl member list -w fields

# listar alarmas lanzadas
etcdctl alarm list

# comprobar estado del servicio local en detalle
etcdctl --write-out=table endpoint status

# capturar el número de la última revisión
rev=$(etcdctl --endpoints=:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')

# compactar las revisiones anteriores a la capturada
etcdctl compact $rev

# desfragmentar el espacio tras la compactación
etcdctl defrag

# eliminar las alarmas para salir del modo mantenimiento
etcdctl alarm disarm
```

## Plugin de volúmenes Docker

### Compilación e instalación del plugin

Se ha hecho uso del plugin [katharostech/docker-plugin_seaweedfs](https://github.com/katharostech/docker-plugin_seaweedfs/tree/fb624481819998223bb2043171caaea1e29a4a7f), concretamente en el commit `fb62448` donde se utiliza seaweedfs en su versión `3.43`. Esto nos aporta mejoras en cuanto a compatibilidad y rendimiento, pero con la pega de que no se ha actualizado su versión compilada ([katharostech/seaweedfs-volume-plugin](https://registry.hub.docker.com/r/katharostech/seaweedfs-volume-plugin)) para su uso directo. Ya está disponible en [redmic/seaweedfs-volume-plugin](https://registry.hub.docker.com/r/redmic/seaweedfs-volume-plugin).

Por ello, se hace necesaria la descarga del repositorio (también está disponible una copia estática del commit `fb62448` en `plugins/seaweedfs-volume-plugin`), la compilación del plugin y la subida del resultado para su uso en las diferentes máquinas:

```sh
git clone https://github.com/katharostech/docker-plugin_seaweedfs.git
cd docker-plugin_seaweedfs
make clean rootfs

# si no se ha hecho antes, identificarse en DockerHub
docker login

docker plugin create redmic/seaweedfs-volume-plugin:v3.43 ./plugin
docker plugin push redmic/seaweedfs-volume-plugin:v3.43
docker plugin rm redmic/seaweedfs-volume-plugin:v3.43

docker plugin create redmic/seaweedfs-volume-plugin:latest ./plugin
docker plugin push redmic/seaweedfs-volume-plugin:latest
docker plugin rm redmic/seaweedfs-volume-plugin:latest

make clean
```

Luego, en cada nodo se puede instalar mediante:

```sh
docker plugin install --alias seaweedfs redmic/seaweedfs-volume-plugin:latest \
  HOST=localhost:8888 \
  REMOTE_PATH=/docker-volumes \
  MOUNT_OPTIONS="-volumeServerAccess=publicUrl -dataCenter=dc-$(docker node inspect -f '{{.ID}}' self) -cacheDir=/tmp -cacheCapacityMB=2048"
```

O si ya estaba instalado, pero se desea actualizar o cambiar a esta compilación:

```sh
docker plugin disable -f seaweedfs:latest

docker plugin upgrade --grant-all-permissions seaweedfs:latest redmic/seaweedfs-volume-plugin:latest

docker plugin enable seaweedfs:latest
```

### Configuración del plugin

Para ajustar la configuración del plugin cuando ya estaba funcionando:

```sh
docker plugin disable -f seaweedfs:latest

docker plugin set seaweedfs:latest \
  HOST=127.0.0.1:8888 \
  MOUNT_OPTIONS="..."

docker plugin enable seaweedfs:latest
```

Las variables disponibles son:

* `HOST`: Referencia al servicio `filer`, en este caso siempre ha de ser local.
* `MOUNT_OPTIONS`: Acepta cualquier parámetro de `weed mount`, ver su salida de ayuda para más información.
* `REMOTE_PATH`: Ruta raíz donde almacenar los volúmenes dentro de seaweedfs.
* `ROOT_VOLUME_NAME`: Nombre de volumen global con todo el contenido. Si se deja vacío no se genera.
* `LOG_LEVEL`: Nivel de salida de logs generados por el plugin. Consultar desde la máquina con `journalctl -f -n 100 -u docker`.

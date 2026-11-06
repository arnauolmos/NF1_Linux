## Fase 1. Preparació de l'entorn

Per al client, utilitzarem Zorin OS 18 amb configuració de "xarxa NAT". El server tindrem la interfície per treballar amb "xarxa només-amb-amfitrió" i l'altra amb "xarxa NAT". Això ens permetrà que el client tingui accés a Internet i que el servidor pugui comunicar-se amb el client.

Ens assegurarem que ambdues màquines tinguin accés a Internet i que es puguin comunicar entre elles a través de la xarxa només-amb-amfitrió i actualitzarem els sistemes amb les últimes actualitzacions disponibles.

---

## Fase 2: Preparació del servidor

Abans de compartir res, hem de preparar els usuaris i els directoris al Servidor.

1. **Creació de Grups:** Crear dos grups per al client: `devs` i `admins`.
2. **Creació d'Usuaris:** 
   - Crear un usuari `dev01` (membre del grup `devs`).
   - Crear un usuari `admin01` (membre del grup `admins`).
3. **Creació de Directoris (al Servidor):**
   - Crear el directori per als projectes de desenvolupament: `/srv/nfs/dev_projects`
   - Crear el directori per a les eines d'administració: `/srv/nfs/admin_tools`
4. **Permisos del Servidor (El punt clau):**
   - Es vol que els *developers* tinguin control total sobre els seus projectes.
   - Es vol que els administradors tinguin control sobre les seves eines.
   - En tots dos casos, l'usuari propietari serà `root`.
5. **Instal·lació i configuració:** Com a pas final, s'instal·laran els paquets necessaris per al servei NFS al servidor i es configurarà l'exportació dels directoris amb les opcions adequades.

*Nota:* Perquè aquesta pràctica funcioni correctament, heu de replicar aquests usuaris i grups al client, o, idealment, assegurar-vos que els UID i GID (els números d'identificació) coincideixin a les dues màquines.

---

## Fase 3: L'Exportació d'Administració (El Dilema del root_squash)

El client necessita que el directori `/srv/nfs/admin_tools` sigui accessible per l'equip d'administradors. 

A vegades, l'usuari `root` del client (que sou vosaltres, els consultors) necessitarà escriure en aquest directori per instal·lar eines.

 Aquí mostrarem un error típic i la seva solució.

### Prova 1 (L'error comú)
1. Exportar el directori `/srv/nfs/admin_tools` amb les opcions `rw,sync`.
````
   arnau@sox-aodl:~$ sudo exportfs -ra

   arnau@sox-aodl:~$ sudo exportfs -v
````
2. Des del client, muntar aquest recurs compartit a `/mnt/admin_tools`. Com a `root` del client, intentar crear un fitxer dins d'aquest directori muntat.

```
arnau@aarnau-zorin:~$ sudo mount -t nfs 10.0.2.6:/srv/nfs/admin_tools /mnt/admin_tools


arnau@aarnau-zorin:~$ sudo touch /mnt/admin_tools/prova_root.txt
touch: cannot touch '/mnt/admin_tools/prova_root.txt': Permission denied


arnau@aarnau-zorin:~$ ls -l /mnt/admin_tools/
total 0
```
3. Verificar quin és el propietari del fitxer creat. Per què? Justificar la resposta amb l'explicació tècnica de `root_squash`.
````
Encara que creïs el fitxer com a root al client, al servidor no apareixerà com a propietari root.

Això passa perquè NFS utilitza per defecte l'opció root_squash, que transforma les peticions de l'usuari root del client en un usuari anònim (normalment nobody o nfsnobody).

A més, com que aquest usuari anònim no té permisos sobre el directori, és possible que la creació del fitxer doni un error de permisos
````
### Prova 2 (La Solució)
1. Modificar l'exportació del directori `/srv/nfs/admin_tools` per incloure l'opció `no_root_squash`.

````
/srv/nfs/admin_tools 10.0.2.9/8(rw,sync,no_root_squash)
````
2. Al client, desmuntar i tornar a muntar el recurs compartit.
````
arnau@aarnau-zorin:~$ sudo umount /mnt/admin_tools
arnau@aarnau-zorin:~$ sudo mount -t nfs 10.0.2.6:/srv/nfs/admin_tools /mnt/admin_tools 
````
3. Com a `root` del client, intentar crear un fitxer dins d'aquest directori muntat. Observeu quin és el propietari del fitxer creat aquesta vegada. Ha canviat alguna cosa? Justificar la resposta amb l'explicació tècnica de `no_root_squash`.
````
arnau@aarnau-zorin:~$ sudo touch /mnt/admin_tools/prova_no_root_squash.txt
arnau@aarnau-zorin:~$ ls -l /mnt/admin_tools/
total 0
-rw-r--r-- 1 root root 0 nov  6 22:56 prova_no_root_squash.txt

Amb no_root_squash, el servidor no transforma l'usuari root del client en un usuari anònim. Per tant, les operacions realitzades com a root al client es mantenen com a operacions de root al servidor.

````
---

## Fase 4: L'Exportació de Desenvolupament (Permisos rw vs ro)

1. Editar `/etc/exports` per afegir dues exportacions per al mateix directori. El client vol que la xarxa d'administració (p.ex., `192.168.56.0/24`) hi pugui escriure, però que la xarxa de consultors (simularem que és una altra IP, p.ex., `192.168.56.150`) només pugui llegir.
2. Si les dues màquines es veuen per xarxa NAT, useu una IP consistent amb el rang de la xarxa NAT. Per exemple, `10.0.2.100`.
3. Des del client, muntar el recurs compartit a `/mnt/dev_projects` i provar d'escriure-hi com a usuari `dev01`. Hauria de funcionar.
4. Desmuntar el recurs i canviar manualment la IP del client a `192.168.56.100`. Tornar a provar d'escriure al directori muntat com a usuari `dev01` i comprovar que només funciona la lectura.
5. Canvieu d'usuari al client a `admin01` i torneu a provar d'escriure al directori muntat. Comproveu que no es pot escriure, ja que `admin01` no és membre del grup `devs` (permisos locals del sistema de fitxers).

---

## Fase 5: Muntatge Automàtic amb /etc/fstab

És evident que els usuaris no poden estar muntant manualment els recursos compartits cada vegada que reinicien el sistema. Per això, es configurarà el muntatge automàtic mitjançant el fitxer `/etc/fstab` al client.

1. Editar el fitxer `/etc/fstab` al client per afegir les entrades necessàries per muntar automàticament els recursos compartits NFS al directori `/mnt/admin_tools` i `/mnt/dev_projects` durant l'inici del sistema.
2. Executar la comanda `mount -a` per provar les entrades sense reiniciar.
3. Reiniciar el client i verificar que els recursos compartits s'han muntat correctament.
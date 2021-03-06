======================
HA (High Availability)
======================

|product| supporta una configurazione in HA limitata ad alcuni scenari specifici.

Il cluster è composto da due nodi in modalità active-passive:
il nodo master (o nodo primario) fornisce il servizio, mentre il nodo slave (o nodo secondario) entra in gioco in caso
di fallimento del nodo master.
Entrambi i nodi condividono uno storage DRBD in modalità active-passive.

Questa configurazione supporta:

* IP virtuali collegati sulla rete green
* Servizi clusterizzati che salvano i dati sullo storage condiviso


*Esempio*

Il demone MySQL risponde ad un IP virtuale e salva i propri dati all'interno del DRBD.
In caso di guasto del nodo master, il servizio viene fatto ripartire sul nodo secondario.

I client si collegano al server MySQL usando l'IP virtuale.


Limitazioni
===========

* Il servizio LDAP, e tutti i servizi che dipendono da esso, non possono essere clusterizzati.
  Si consiglia l'utilizzo di un LDAP esterno
* Sono supportati solo i dispositivi di fencing di tipo STONITH


Requisiti hardware
==================

La procedura prevede l'installazione su due nodi gemelli. Ogni nodo dovrà avere

* un disco o una partizione dedicata allo storage condiviso DRBD (Distributed Replicated Block Device)
* due interfacce di rete per creare un bond con ruolo *green*, entrambe le interfacce
  devono essere collegate agli switch della rete locale

E' necessario possedere due switch all'interno della LAN, per esempio SW1 e SW2.
Creare un bond su ciascun nodo usando due interfacce di rete. 
Ogni nodo deve essere collegato ad entrambi gli switch SW1 e SW2.


Fence device
------------

Ogni nodo dovrà essere collegato ad almeno un fence device già correttamente configurato.

Con il termine *fencing* si intende la disconnessione di un nodo dallo storage condiviso.
Il *fence device* è il dispositivo hardware che consente tale disconnessione usando
il metodo STONITH (Shoot The Other Node In The Head), ovvero togliendo l'alimentazione al nodo guasto.

Si consiglia l'utilizzo di PDU (Power Distribution Unit), 
ma anche dispositivi IPMI (Intelligent Platform Management Interface) possono essere usati con alcune limitazioni.

Collegamenti esterni:

* lista dei dispositivi supportati: https://access.redhat.com/articles/28603
* maggiori informazioni sul fencing: http://clusterlabs.org/doc/crm_fencing.html
 

Installazione
=============

Prima di iniziare:

* collegare i due nodi come previsto e assicurarsi che il nodo secondario sia spento, procedere poi installando |product| sul nodo primario
* assicurarsi che il nome host del nodo primario sia *ns1*. Esempio: ns1.mydomain.com. 
  Scegliere ora anche il dominio, che *non* potrà essere cambiato in seguito

Nodo primario
-------------

Il nodo primario sarà quello che fornirà i servizi in caso di normale funzionamento.
Per prima cosa è necessario configurare una partizione o un disco riservato allo storage
condiviso DRBD.

Configurazione storage per DRBD
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Aggiungere un disco (per esempio vdb)
* Partizionare il disco 

::

 parted /dev/vdb mklabel gpt
 parted /dev/vdb --script -- mkpart primary 0% 100%

* Creare sul disco una volume fisico 

::

 pvcreate /dev/vdb1

* Estendere il volume group 

::

 vgextend VolGroup /dev/vdb1

* Creare un volume logico per il DRBD 

::

 lvcreate -n lv_drbd -l 100%FREE VolGroup


Software
^^^^^^^^

Le opzioni del cluster sono salvate all'interno della chiave di configurazione *ha* che deve essere
configurata specularmente su entrambi i nodi.

Eseguire i passi di configurazione come riportati di seguito.

* Configurare il bond sulle interfacce green.

* Installare i servizi del cluster:

::

 yum install nethserver-ha

* Installare il software, esempio MySQL:

::

  yum install nethserver-mysql

* Configurare l'IP virtuale, ed informare il cluster sugli IP green di entrambi i nodi:

::

 config setprop ha VirtualIP <GREEN_IP_HA>
 config setprop ha NS1 <NS1_GREEN_IP>
 config setprop ha NS2 <NS2_GREEN_IP>


* Applicare le modifiche e avviare i servizi sul nodo primario: 

::

 signal-event nethserver-ha-save


Al termine della configurazione, il nodo primario è pronto ad erogare i servizi.
Per verificare lo stato del cluster, eseguire: ::

 pcs status

Configurazione servizi
^^^^^^^^^^^^^^^^^^^^^^

I servizi clusterizzati devono essere controllati dal demone che gestisce le risorse del cluster (pacemaker),
per tanto è necessario disattivare il riavvio automatico all'interno di |product|: ::

 service mysqld stop
 chkconfig mysqld off
 /sbin/e-smith/config settype mysqld clustered

I seguenti comandi configurano un'istanza di MySQL che salva i dati sul DRBD ed è legata all'IP virtuale: ::

 /usr/sbin/pcs cluster cib /tmp/mycluster
 /usr/sbin/pcs -f /tmp/mycluster resource create DRBDData ocf:linbit:drbd drbd_resource=drbd00 op monitor interval=60s
 /usr/sbin/pcs -f /tmp/mycluster resource master DRBDDataPrimary DRBDData master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 is-managed="true" notify=true
 /usr/sbin/pcs -f /tmp/mycluster resource create VirtualIP IPaddr2 ip=`config getprop ha VirtualIP` cidr_netmask=`config getprop ha VirtualMask` op monitor interval=30s
 /usr/sbin/pcs -f /tmp/mycluster resource create drbdFS Filesystem device="/dev/drbd/by-res/drbd00" directory="/mnt/drbd" fstype="ext4" 
 /usr/sbin/pcs -f /tmp/mycluster resource create mysqld lsb:mysqld
 /usr/sbin/pcs -f /tmp/mycluster resource create sym_var_lib_asterisk ocf:heartbeat:symlink params target="/mnt/drbd/var/lib/mysql" link="/var/lib/mysql" backup_suffix=.active
 /usr/sbin/pcs -f /tmp/mycluster resource create sym_etc_my.pwd ocf:heartbeat:symlink params target="/mnt/drbd/etc/my.pwd" link="/etc/my.pwd" backup_suffix=.active
 /usr/sbin/pcs -f /tmp/mycluster resource create sym_root_.my.cnf ocf:heartbeat:symlink params target="/mnt/drbd/root/.my.cnf" link="/root/.my.cnf" backup_suffix=.active

 /usr/sbin/pcs -f /tmp/mycluster constraint order promote DRBDDataPrimary then start drbdFS
 /usr/sbin/pcs -f /tmp/mycluster constraint colocation add drbdFS with DRBDDataPrimary INFINITY with-rsc-role=Master
 /usr/sbin/pcs -f /tmp/mycluster resource group add mysqlha drbdFS VirtualIP sym_var_lib_mysql sym_etc_my.pwd sym_root_.my.cnf var_lib_nethserver_secrets mysqld

 /usr/sbin/pcs cluster cib-push /tmp/mycluster

Per verificare lo stato del cluster e dei servizi: ::

 pcs status

Si veda la documentazione ufficiale di pacemaker per maggiori informazioni

Nodo secondario
---------------

* Installare |product| sul nodo secondario
* Assicurarsi che l'hostname del nodo secondario sia *ns2* e che il dominio sia lo stesso del nodo primario
* Procedere alla configurazione storage DRBD come sul nodo primario
* Installare e configurare il software come per il nodo primario
* Configurare l'IP virtuale, le opzioni NS1 e NS2, quindi applicare la configurazione:

  ::
 
   signal-event nethserver-ha-save


Passi finali
------------

* Abilitare lo STONITH, digitando su uno qualsiasi dei nodi il seguente comando: 

::

 pcs property set stonith-enabled=true

* Configurare i fence device, i comandi possono essere eseguiti su uno qualsiasi dei nodi.
  Esempio per fence con libvirt, dove i nodi sono macchine virtuali ospitate da un virtualizzatore con IP 192.168.1.1: 

::

 pcs  stonith create Fencing fence_virsh ipaddr=192.168.1.1 login=root passwd=myrootpass pcmk_host_map="ns1.nethserver.org:ns1;ns2.nethserver.org:ns2" pcmk_host_list="ns1.nethserver.org,ns2.nethserver.org"

* Configurare un indirizzo mail a cui inviare le notifiche in caso di guasto:

::

  pcs resource create MailNotify ocf:heartbeat:MailTo params email="admin@nethserver.org" subject="Cluster notification"

* E' fortemente consigliato cambiare la password di root da interfaccia web su entrambi i nodi. 
  La password di root è infatti utilizzata per impartire ordini ai nodi del cluster.

Fencing con IPMI
----------------

Molti server possiedono un'interfaccia di gestione preinstallata conosciuta con vari nomi commerciali come
ILO (HP), DRAC (Dell) o BMC (IBM). Tutte queste interfacce rispettano lo standard IPMI.
Dal momento che l'interfaccia di gestione controlla solo il nodo su cui è installata,
è necessario configurare almeno due dispositivi di fence, uno per ciascun nodo.

Se il dominio del cluster è ``nethserver.org``, usare i seguenti comandi: ::

 pcs stonith create ns2Stonith fence_ipmilan pcmk_host_list="ns2.nethserver.org" ipaddr="ns2-ipmi.nethserver.org" login=ADMIN passwd=ADMIN timeout=4 power_timeout=4 power_wait=4 stonith-timeout=4 lanplus=1 op monitor interval=60s
 pcs stonith create ns1Stonith fence_ipmilan pcmk_host_list="ns1.nethserver.org" ipaddr="ns1-ipmi.nethserver.org" login=ADMIN passwd=ADMIN timeout=4 power_timeout=4 power_wait=4 stonith-timeout=4 lanplus=1 op monitor interval=60s

Dove ns1-ipmi.nethserver.org e ns2-ipmi.nethserver.org sono i nomi host associati agli IP delle interfacce di gestione.

Inoltre, assicurarsi che la risorsa stonith risieda sul nodo corretto: ::

 pcs constraint location ns2Stonith prefers ns1.nethserver.org=INFINITY
 pcs constraint location ns1Stonith prefers ns2.nethserver.org=INFINITY


Guasti e ripristino
===================

Un cluster a due nodi può tollerare solo un guasto alla volta.

.. note::
   Qualora si utilizzino i dispositivi di fence di tipo IPMI, il cluster non è in grado di gestire 
   la perdita di alimentazione di un nodo, in quanto il dispositivo di fence è alimentato dal nodo stesso.

   In questo caso è necessario confermare manualmente lo spegnimento del nodo eseguendo questo comando: ::

     pcs stonith confirm <failed_node_name>

Nodi guasti
-----------

Quando un nodo non risponde all'heartbeat, il nodo viene escluso dal cluster.
Tutti i servizi clusterizzati sono disabilitati al boot per evitare problemi in caso di fencing:
un nodo che è stato spento da un evento di fencing, necessita probabilmente di manutenzione prima di rientrare 
nel cluster.

Per inserire nuovamente il nodo nel cluster, eseguire: ::

 pcs cluster start


Fence device irraggiungibili
----------------------------

Il cluster controlla periodicamente lo stato dei dispositivi di fence configurati.
Se un dispositivo non è raggiungibile, verrà considerato in stato fermo (stopped).

Dopo aver ripristinato il dispositivo di fence, informare il cluster sullo stato
di ciascun dispositivo con il seguente comando: ::

  crm_resource --resource <stonith_name> --cleanup --node <node_name>

Disaster recovery
-----------------

In caso di guasto hardware, è possibile reinstallare il nodo è raggiungerlo al cluster.
I servizi clusterizzati saranno automaticamente configurati e i dati verranno sincronizzati fra i nodi.

Seguire questi passi.

1. Installare |product| sulla macchina.
2. Ripristinare il backup della configurazione del nodo. Se non si possiede il backup della configurazione,
   riconfigurare il server e assicurarsi di installare il pacchetto ``nethserver-ha``.
3. Eseguire l'evento per unire il nodo al cluster: ::

     signal-event nethserver-ha-save


Backup
======

Il backup deve essere configurato su entrambi i nodi ed eseguito su una condivisione di rete.
Solo il nodo primario effettuerà realmente il backup, il backup del nodo secondario
verrà automaticamente abilitato qualora il nodo primario sia guasto.

In caso di guasto di entrambi i nodi, reinstallare il nodo primario, 
ripristinare il backup della configurazione e avviare il cluster: ::

 signal-event nethserver-ha-save

Infine ripristinare il backup dei date e, al termine, riavviare il sistema.

Se si desidera eseguire il backup dei dati contenuti nel DRBD,
assicurarsi di aggiungere le directory all'interno del file :file:`custom.include`.

Esempio: ::

  echo "/mnt/drbd/var/lib/mysql" >> /etc/backup-data.d/custom.include


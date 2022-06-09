# NUT

How to Install NUT in SmartGrid System

## Instalar NUT ##################

apt install nutapt install nut

################### Configurar NUT ###################
´´´
$ nano /etc/nut/nut.conf
´´´´
--------------- Archivo nut.conf -------------------

MODE=standalone

################## Configurar ups.conf ##############

nano /etc/nut/ups.conf

--------------- Archivo ups.conf -------------------

maxretry = 3

[ups1]

  driver = snmp-ups
  
  port = x.x.x.31
  
  snmp_version = v1
  
  desc = "UPS High Priority"
  
  community = public

[ups2]
  
  driver = snmp-ups
  
  port = x.x.x.62
  
  snmp_version = v1
  
  desc = "UPS Aux"
  
  community = lsst_network

############## Configurar upsd.conf #################

nano /etc/nut/upsd.conf

-------------- Archivo upsd.conf --------------------
                                                   
/etc/netplan# cat /etc/nut/upsd.conf
MAXAGE 15
STATEPATH /var/run/nut
LISTEN 127.0.0.1 3493
LISTEN 172.19.221.221 3493
MAXCONN 1024

############## Inicio de servicio NUT ###############

service nut-server start
upsc ups1
upsc ups2

############## Configuracion upsd.users #############

nano /etc/nut/upsd.users

-------------- Archivo upsd.users -------------------

[admin]
        password = "password"
        actions = SET FSD
        instcmds = ALL
        upsmon master

############### Configuracion upsmon.conf ###########

nano /etc/nut/upsmon.conf

--------------- Archivo upsmon.conf -----------------

MONITOR ups1 1 "admin" "password" master

RUN_AS_USER nut

MINSUPPLIES 1

SHUTDOWNCMD "/opt/ups/halt_server.sh"

POLLFREQ 5

POLLFREQALERT 5

HOSTSYNC 15

DEADTIME 15

POWERDOWNFLAG /etc/killpower

NOTIFYCMD "/sbin/upssched"

NOTIFYMSG ONLINE "UPS: Normal state"

NOTIFYMSG ONBATT "UPS: On battery"

NOTIFYMSG LOWBATT "UPS: Battery low"

NOTIFYMSG FSD "UPS: Starting shutdown"

NOTIFYMSG COMMOK "UPS: Communication restored"

NOTIFYMSG COMMBAD "UPS: Communication lose"

NOTIFYMSG SHUTDOWN "UPS: Shutting down"

NOTIFYMSG REPLBATT "UPS: Replace battery"

NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC

NOTIFYFLAG ONBATT SYSLOG+WALL+EXEC

NOTIFYFLAG LOWBATT SYSLOG+WALL+EXEC

NOTIFYFLAG FSD SYSLOG+WALL+EXEC

NOTIFYFLAG COMMOK SYSLOG+WALL+EXEC

NOTIFYFLAG COMMBAD SYSLOG+WALL+EXEC

NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC

NOTIFYFLAG REPLBATT SYSLOG+WALL+EXEC

RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 0

################### Configurar upssched.conf ##############

nano /etc/nut/upssched.conf

------------------- Archivo upssched.conf -----------------

############# Network UPS Tools - upssched.conf ###########
CMDSCRIPT /opt/ups/ups_event.sh

##Hay que crear la ruta /var/run/nut/upssched/ si no existe con propietario nut:nut
#PIPEFN /var/run/nut/upssched/upssched.pipe
#LOCKFN /var/run/nut/upssched/upssched.lock
#/var/run/nut/upssched/ es borrada periódicamente por el sistema con lo cual
#upssched deja de funcionar. La solución mas cómoda es usar /tmp/ para almacenar 
#ambos ficheros:
PIPEFN /tmp/upssched.pipe
LOCKFN /tmp/upssched.lock

############### Si hay corte de corriente, espera 300 seg y apaga ##############
AT ONBATT * START-TIMER  ups-on-battery-shutdown  30000000

############### Si vuelve la corriente, se cancela el timer ####################
AT ONLINE * CANCEL-TIMER  ups-on-battery-shutdown

######### Si hay corte de corriente, espera 15 seg antes de notificarlo ########
AT ONBATT * START-TIMER ups-on-battery 15
AT ONLINE * CANCEL-TIMER ups-on-battery

#En los siguientes eventos, llama al script de notificacion para que lo procese.
AT ONLINE * EXECUTE ups-back-on-line
AT REPLBATT * EXECUTE ups-change_battery
AT LOWBATT * EXECUTE ups-low-battery
AT COMMOK * EXECUTE ups-comunication-ok
AT COMMBAD * EXECUTE ups-comunication-bad

#################### Configuracion Usuario Nut ##################

visudo

-------------------- Archivo sudoers.tmp ----------------------

#################### al Final del Archivo #######################

nut ALL = (ALL:ALL) NOPASSWD: /bin/systemctl, /sbin/shutdown

#################### Configuacion config ########################

nano /opt/ups/config

-------------------- Archivo config -----------------------------

################ IDENTIFICACION GATEWAY #########################

GW_ID="lsstalarm@netdev.cl"

########### NOMBRE DE LA PERSONA QUE RECIBIRA EL MAIL ###########

MAIL_NAME="LSST Alarm"

########### MAIL AL QUE SE ENVIARA LA INFO ######################

MAIL_RCPT="lsstalarm@netdev.cl"

################### Configuracion mail.sh #######################

nano /opt/ups/mail.sh

------------------- Archivo mail.sh------------------------------


#!/bin/bash

source config

function mailSend() {
    
    TMP_FILE="/tmp/ups.email"
    
    /bin/echo "To: $MAIL_NAME <$MAIL_RCPT>" > $TMP_FILE
    
    /bin/echo "Subject: $1" >> $TMP_FILE
    
    /bin/echo "" >> $TMP_FILE

    /bin/echo "$2" >> $TMP_FILE

    /usr/sbin/sendmail -v $MAIL_RCPT < $TMP_FILE 

    /bin/rm $TMP_FILE
}

################### Configurar ups_event.sh ######################

#!/bin/bash
DEBUG=0

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
export PATH
TZ='America/Santiago'
export TZ

function log() {
  if [ $DEBUG -eq 1 ]; 
  then 
    logger $1; 
  fi
}

log "ups_event.sh: running"

cd /opt/ups
source config
log "ups_event.sh: loaded config"
source mail.sh
log "ups_event.sh: loadad mail.sh"

MESSAGE_MAIL=""
EVENT_TYPE=$1
APAGADO="0"
FECHA=$(/bin/date '+%Y-%m-%d %H:%M:%S%z')

case "$EVENT_TYPE" in
    "ups-on-battery-shutdown")
        MESSAGE_MAIL="UPS on battery: shutdown now"
        APAGADO="1"
        ;;
    "ups-on-battery")
        MESSAGE_MAIL="WARN: UPS on battery"
        ;;
    "ups-comunication-bad")
        MESSAGE_MAIL="Communications with UPS lost"
        ;;
    "ups-change_battery")
        MESSAGE_MAIL="UPS battery needs to be replaced"
        ;;
    "ups-back-on-line")
        MESSAGE_MAIL="INFO: UPS on line power"
        ;;
    "ups-low-battery")
        MESSAGE_MAIL="UPS battery is low"
        ;;
    "ups-comunication-ok")
        MESSAGE_MAIL="Communications with UPS established"
        ;;
esac

log "ups_event.sh: case finished, sending email..."
mailSend  "[$FECHA] $MESSAGE_MAIL"  "[UPS] Event $EVENT_TYPE de SAI IoT GW $GW_ID"
/bin/echo "[$FECHA] SAI: $MESSAGE_MAIL" >> /var/log/ups.log
log "ups_event.sh: mail sent, waiting.."

if [ $APAGADO = "1" ]
then
    /bin/sleep 30
    /opt/ups/halt_server.sh
fi
log "ups_event.sh: everything done, exit."

exit 0


#################### Depuracion y Resolucion de Servicios ###########

service nut-server restart
service nut-monitor restart

systemctl status nut-server.service
systemctl status nut-monitor.service

journalctl -xe


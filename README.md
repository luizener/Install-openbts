
Instalación de openbts 5.0 en Ubuntu 14.04 y la tarjeta USRP B210 de ETTUS

Las nuevas versiones de openbts liberadas por Range Networks incluye mejoras en el código, pero también cuenta con un “error” respecto a las tarjetas B2x0 de ettus Research, por ende se realizan modificaciones en la instalación de openbts, tal como se mencionan el el siguiente paso a paso.

Necesario instalar Ubuntu Desktop 14.04 64 bits en instalación limpia (para este caso se usa Ubuntu GNOME 14.04.4 LTS). 

Luego se procede a actualizar paquetes y repositorios, con el fin de tener los paquetes necesarios actualizados, no se recomienda actualizar a Ubuntu 16.04 LTS por problemas de compatibilidad debido al cambio del sistema de ficheros upstart por systemd (la inicializacion de aplicaciones de openbts requiere upstart).

Al iniciar el sistema operativo recién instalado se ejecútan las instrucciones

$ sudo apt-get update
$ sudo apt-get upgrade

Instalar los siguientes paquetes adicionales requeridos previamente a la instalación del driver UHD y de openbts

$ sudo apt-get install g++ erlang libreadline6-dev bind9 ntp autoconf
$ sudo apt-get install libboost-all-dev

el segundo dirá que no se cumplen dependencias y no instala, acto seguido se ejecuta la instrucción

$ sudo apt-get -f install

ahí instala todas las dependencias que requiere libboost-all-dev

Luego es necesario descargar una versión de UHD que funcione con la tarjeta B210, desde https://sourceforge.net/p/openbts/mailman/message/35276776/ se recomienda la versión 003.007.003, la cual se descarga e instala

$ wget http://files.ettus.com/binaries/uhd_stable/uhd_003.007.003-release/uhd_003.007.003-release_Ubuntu-14.04-x86_64.deb 
$ sudo dpkg -i uhd_003.007.003-release_Ubuntu-14.04-x86_64.deb 

Así ya se tiene instalada la tarjeta USRP con la versión de UHD correcta, se recomienda conectar a un puerto USB 3.0 y verificar usando los comandos

$ uhd_find_devices
$ uhd_usrp_probe

a partir de aquí se pasa al instructivo indicado en el handbook Getting_Started_with_OpenBTS_Range_Networks.pdf de la pagina 25, 

instalar git

$ sudo apt-get install git

obtener la carpeta fuente y el código, preferiblemente en una carpeta de fácil acceso

$ git clone https://github.com/RangeNetworks/dev.git
$ cd dev
$./clone.sh

Verificar que se vaya a compilar la versión 5.0 
$ ./switchto.sh 5.0

el siguiente paso no esta en el handbook, pero se recomienda realizar para garantizar enlaces simbólicos entre los diferentes elementos a instalar, (paso hallado en https://antonraharja.com/2016/03/16/instalasi-dan-konfigurasi-openbts-5-0/)

$ cd /dev/liba53
$ sudo make install
$ sudo ldconfig
$ cd ..

luego se incluye el repositorio encargado de que funcione correctamente smqueue

$ sudo apt-get install software-properties-common python-software-properties 
$ sudo add-apt-repository ppa:chris-lea/zeromq 
$ sudo apt-get update

acto seguido, con un editor de texto se procede a editar el archivo build.sh, ya que hay una dependencia que no se cumple (Ubuntu 14.04 incluye el paquete libzmq3 y no permite instalar libzmq5) y tampoco se debe instalar la versión por defecto en Ubuntu de UHD ya que previamente se ha instalado la version correcta para la tarjeta usada, las lineas a modificar son las siguientes

installIfMissing libzmq5 se reemplaza por installIfMissing libzmq3

se dejan como comentario o se eliminan las siguientes lineas

if [ "$MANUFACTURER" == "Ettus" ]; then 
        installIfMissing libuhd-dev 
        installIfMissing libuhd003 
        installIfMissing uhd-host 
fi

se guarda el archivo sin cambiar el nombre y se procede a ejecutar

$ ./build.sh B210 

B210 es la tarjeta que se esta configurando, esta referencia se cambia si es otro el modelo de tarjeta USRP que se requiere configurar.

A continuación se instalan los paquetes *.deb generados en el paso anterior y que se encuentran almacenados en la carpeta

$ cd BUILDS/fecha-de-compilación

Terminando la instalación de los paquetes pedirá 2 veces confirmación de reemplazo de 2 paquetes del sistema y ser reemplazados por paquetes de openbts, a lo que se responde afirmativamente, (esto con el fin de que asterisk funcione correctamente).

Hasta aquí ya esta instalado openbts, se requiere reiniciar el sistema operativo para que todos los cambios realizados queden asignados correctamente.


Cambios adicionales realizados a la instalación recomendada por el libro de Range Networks:

El registro desde nodemanager funciona y agrega los nuevos celulares a la red para los servicios sipauthserve y smqueue, para asterisk no funciona y por ende se deben agregar los numeros teléfonicos (MSISDN) manualmente como se muestra a continuación:

1. se agregan los IMSI junto al MSISDN asignado usando

sudo asterisk -rx "database put IMSI IMSI####### MSISDN"
sudo asterisk -rx "database put PHONENUMBER MSISDN IMSI#######" 

así para cada numero que sea agregado, luego hay que editar el archivo /etc/asterisk/extensions.conf, incluyendo la siguiente linea

\# include extensions-custom.conf

acto seguido se crea el archivo /etc/asterisk/extensions-custom.conf
y se pega la siguiente información

[from-openBTS]

exten => _XXXX.,1,Verbose(Dialplan started)

same = n,Set(CALLER_IMSI=${CALLERID(num)})

same = n,Verbose(Get CID from CALLER_IMSI: ${CALLER_IMSI})

same = n,Set(CID=${DB(IMSI/${CALLER_IMSI})})

same = n,Set(CALLERID(num)=${CID})

same = n,Verbose(Get IMSI from EXTEN: ${EXTEN})

same = n,Set(IMSI=${DB(PHONENUMBER/${EXTEN})})

same = n,Dial(SIP/00101100010/${IMSI})


tener presente las X en la segunda linea, las cuales se deben reemplazar por los primeros números de la red que usted haya elegido, y así asterisk sabrá que debe permitir aquellos números que empiecen por este numero, ejemplo si usted elige 456-1234567 se recomienda que coloque en la parte resaltada los números 456.

el siguiente paso es verificar que los números se hayan incluido correctamente, usando la instrucción

$sudo asterisk -rx "database show"

luego se guardan los cambios de configuración realizados en asterisk usando 

sudo asterisk -rx "dialplan reload"

Recuerde que cada nuevo numero a aprovisionar debe estar en el listado NodeManager y en asterisk usando los procedimientos adecuados. (no es necesario volver a editar los archivos de asterisk, este procedimiento solamente debe realizarse una vez).





Referencias

https://sourceforge.net/p/openbts/mailman/openbts-discuss/

https://antonraharja.com/2016/03/16/instalasi-dan-konfigurasi-openbts-5-0/

http://openbts.org/site/wp-content/uploads/ebook/Getting_Started_with_OpenBTS_Range_Networks.pdf




http://crasseux.com/books/ctutorial/Writing-a-makefile.html
La idea es dar una serie de reglas a "make" para que sepa que debe recompilar.
Según el timestamp de los distintos ficheros y las relaciones entre los ficheros conseguirá solo recompilar las partes necesarias.
El típico ejemplo es un programa en c con headers, source files, objects files y el binario.
Al modificar un fichero make, gracias a las reglas del Makefile, sabrá que cosas debe recompilar, evitando tener que recompilar todo el proyecto.

Si hacemos make, se ejecutará el primer "goal" disponible (que no empieze por punto)

Una opción típica al compilar es que los targets y los requisites sean ficheros.
Y vamos poniendo en cada target los ficheros que hacen falta para compilar este.
De esta manera vamos creando una jerarquía, que en el caso de modificar algún fichero, make sabrá que tiene que recompilar para cumplir con todos los target.
Ejemplo: http://crasseux.com/books/ctutorial/A-simple-makefile.html#A%20simple%20makefile



Esquema básico:
target ... : prerequisites ...
  command


Variables
nombre = valor de la variable
La usamos como: $(nombre)


Si queremos crear targets custom que no sean compilaciones de ficheros, usaremos .PHONY para decirle a make que no intente buscar un fichero con el nombre del target:
.PHONY : clean
clean :
  rm edit $(objects)

Tambien podemos definir todos los PHONY al final:
.PHONY: build docker-run ...



Definir un goal/target por defecto:
.DEFAULT_GOAL := help




El ejemplo más sencillo es compilar un programa en c cuando hagamos 'make'.

Para ello el Makefile será:
program:
	gcc file.c -o ejecutable

Tener en cuenta que es un tabulador lo que hay antes de gcc

Si hacemos make sin tener Makefile ejecutará:
make test -> gcc test.c -o test
Tambien vale, al menos, para c++


Si queremos meter un script en bash:
$$(echo "hola")
`echo "otra opcion"`

Pillar version de un fichero:
VERSION=$$(cat VERSION)

Loops y usar la variable ($$ para que no la resuelva antes de llamar al echo):
for num in 1 2 3 4 ; do \
  echo $$num ; \
done


# Autoconf
The file 'configure.ac' (or 'configure.in') is used to create 'configure' by a program called 'autoconf'.

Parece que lo tipico es ejecutar:
aclocal -I m4
autoconf
autoheader
automake -a
automake



Ejemplo de Makefile:

# This is how we want to name the binary output
BINARY=dcip_eventhandler

# These are the values we want to pass for Version and BuildTime
VERSION=0.2.0

# Setup the -ldflags option for go build here, interpolate the variable values
LDFLAGS=-ldflags "-X main.app_version=${VERSION}"

all: build rpm

build:
	go build ${LDFLAGS} -o ${BINARY} *.go

rpm:
	fpm -s dir -t rpm -n dcip_eventhandler -a x86_64 --prefix /usr/bin -v ${VERSION} ${BINARY}



# Leer de user:
test:
  read -p "Dime tu user de LDAP: " user; \
  read -p "Dime tu pass de LDAP: " pass; \
  echo "Tus cred son $$user $$pass"


# Definir comandos de un target
@comando
  esto hará que make no muestre el comando antes de ejecutarlo

-comando
  esto hace que el comando tenga RC=0, asi que seguirá ejecutando el resto de cmds del target
  podemos combinarlo con @ de cualquier manera: "-@" o "@-"
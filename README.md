El proyecto de bet4talent está dividido en 4 grandes bloques. Cada uno de ellos se encarga de una y solo una cosa, y se comunica con los demás a través de colas y eventos, por lo que todos los bloques trabajan de forma isolada del resto de ellos.

* Crawlers - Bloque encargado de trabajar con crawlers
* Pipe - Todos los usuarios recogidos por los crawlers son procesados por este proyecto
* Nod - Encargado de la parte de matching
* Search - Vertical único de usuarios

## Filosofía global

Tras la arquitectura de bet4talent se esconda la necesidad inmediata de trabajar
con bloques independientes, a la par que autosuficientes. Para tener una idea 
más clara de la urgencia de trabajar de esta forma, es necesario pensar que cada 
uno de los bloques debería poderse tratar como un proyecto separado, y con un 
caracter plug-&-play, de tal forma que en un futuro, cuando haya una realidad de 
refactoring, los cambios puedan hacerse sin demasiado impacto hacia el proyecto 
general.

La arquitectura general se basa en PHP y Symfony, y aún trabajando con Symfony a 
nivel de Framework, es importante tener clara cual es la diferencia entre 
Symfony Framework y Symfony Componentes, ya que a nivel de librerías se utilizan
los componentes de Symfony, mientras que a nivel de aplicación se utiliza 
Symfony Framework.

Se podría decir que cada una de las microaplicaciones que tiene el proyecto 
bet4talent se podría dividir en tres pequeños subproyectos. Libería, Bundle, 
Aplicación.

Librería - Simple librería PHP. Solo acoplada a otras librerías PHP, como pueden 
ser las de Symfony o otras de Github. Importante saber que una librería PHP 
nunca debería requerir, ni en producción (require) ni a nivel de testing 
(require-dev) ningun Bundle, pues este debería requerir, si estuviera bien 
hecho, FrameworkBundle o otros Bundles.

Bundle - Exposición de una librería a una aplicación Symfony. Esto implica la 
definición explícita de como la capa de dominio es expuesta mediante el 
componente Dependency Injection, de como las entidades son mapeadas en Doctrine 
y de cuales son las dependencias con otros bundles. Para esto se utiliza la 
librería [BaseBundle](http://github.com/mmoreram/BaseBundle) y 
[SimpleDoctrineMapping](http://github.com/mmoreram/SimpleDoctrineMapping), ambas 
librerías para incrementar la calidad de los bundles.

App - Pequeñas agrupaciones de Bundles (normalmente uno) y su pertinente 
exposición a otras capas de comunicación, como puede ser HTTP (Controladores y 
introducción a capa de servicio mediante Request/Response), Consola (Comandos 
utilizando Input y Output) y la comunicación entre ellas a partir de colas. En 
este caso se utiliza para dicha finalidad Redis y la aplicación 
(http://github.com/mmoreram/RSQueueBundle) --Anexo 1--

Es importante destacar que en esta arquitectura es muy importante la forma que 
tienen estos paquetes de utilizarse, siempre a partir de requirements en 
composer, y de forma unidireccional. Es lógico pensar pues que un bundle debería 
requerir la librería, y la app, el bundle.

## Definiciones comunes

Hay definiciones que son comunes a nivel de proyecto. Por ejemplo, las ips de 
los microservicios, o el nombre de las colas. Dichas definitiones siempre deben 
hacerse dentro de una librería PHP llamada 
[PHP Foundations](http://github.com/bet4talent/php-foundations). Por supuesto, 
dichas fundaciones son específicas para cada uno de los lenguajes y por ahora, 
la información está añadida de forma implícita en la librería de PHP. En caso 
que se quisiera que todas las librerías de lenguaje se nutrieran del mismo 
repositorio, sin importar el lenguaje de programación, se podría crear un 
repositorio llamado *bet4talent/foundations* con documentos yml y los valores 
pertinentes. Cada uno de los lenguajes (php-foundations, perl-foundations, 
go-foundations, podría leer estos documentos y exponerlos a sus librerías 
relativas).

Cada repositorio que necesite acceder a estos valores deberá incluir 
php-foundations en su bloque de require en composer.

## Satis

Dado que los repositorios de bet4talent son privados (al menos por ahora) y dado 
que existe una dependencia entre muchos de ellos, ha sido necesario crear una 
instancia de Satis (un proyecto creado por los autores de Composer y base de la 
instalación de Packagist) para poder resolver dichas dependencias. Satis está 
instalado en una máquina de AWS y para poder acceder a él es necesario una clave 
privada adjunta en el correo conjunto a la documentación. Para poder acceder a 
la máquina es necesario entrar con el usuario *ubuntu*.

Dentro de la máquina está el proyecto satis instalado escuchando el puerto 8010, 
por lo que la configuración de los composer.json de todos los proyectos que 
deséen tener dependencias internas debe ser de la siguiente forma:

``` yml
{
   "repositories": [
        {
            "type": "composer",
            "url": "https://satis.bet4talent.com"
        }
    ],
}
```

## Workflow

El workflow de la aplicación bet4talent es el siguiente:

* Los parsers generan datos completamente isolados, cada uno de ellos generando 
nuevos perfiles específicos de cada plataforma. Por ejemplo, los usuarios de 
Linkedin no serán los mismos que los usuarios de Github, ni tendrán la misma 
estructura. Cada uno de ellos, en el momento que genera un nuevo perfil, emite 
un evento en cola llamado *pipeline.queue.crud_user_updated*. El evento se puede 
encontrar en la librería php-foundation con el namespace 
*Foundation\Pipeline::CRUD_USER_UPDATED_QUEUE*. Los datos de este evento (aunque 
vaya por cola se considera un evento con un solo listener tipo Worker ya que la 
cola es del tipo Producer/Consumer) son crudos. Esto significa que aunque cada 
una de las plataformas haya guardado dichos datos de una forma única, el sistema 
debe entender un usuario de la misma forma, sea el origen que sea.
* El proyecto pipe es el que se encarga de todo el pipeline, o sea, orquestrar 
todo el sistema. la clase ClientCommand es la que se encarga de escuchar el 
evento anterior, recoger los datos en plano que lleva la cola y llamar el método 
addUserFromCrude de la clase Client.
* Los Feeders se encargan de volver a mandar datos a los parsers utilizando otra 
cola llamada Foundation\Pipeline::LINK_FEED_QUEUE. Esta cola es del tipo 
Publisher/Subscriber, por lo que todos los parsers deberían escucharlo para
detectar si ellos deberían recoger los datos. Mayormente se utiliza para 
realimentar el sistema con nuevos links encontrados en perfiles, añadiendo 
cierto peso de conexión entre el original y el encontrado.
* Los Guessers son lanzados cuando se ha creado un perfil con referencia a 
otro perfil (por ejemplo, hemos parseado Linkedin y hemos encontrado el usuario 
A con un enlaze a Github. Hemos parserado Github y hemos encontrado el perfil B, 
por lo que tendremos a B con el usuario relacionado A). Se dedican a comprobar 
ciertos factores que puedan argumentar similitudes entre ambos perfiles y mandar 
resultados a NOD.
* Un Matcher es el que se encarga de notificar resultados de similitudes a un 
microservicio. Está implementado con ports and adapters, y hay implementados dos 
adaptadores, el que utiliza NOD como servicio y el que utiliza NOD como 
microservicio REST.
* Un Pusher es el que se encarga de notificar al sistema de indexación 
(buscador) que un usuario ha sido actualizado (él o la base de relaciones de 
NOD).
 
## Nod

Nod es la base relacional y de conexiones entre usuarios de diferentes fuentes. 
Se ha implementado como un proyecto completamente aparte y desacoplado del 
sistema para que pueda trabajar y evolucionar sin necesidad de otras 
dependencias. El proyecto NOD sigue las dinámicas siguientes.

* NOD ofrece un servicio con dos métodos.
* El primero es para la introducción de datos. Es simple. Se trata de decir que 
A de la fuente Linkedin y B de la fuente Github son iguales en un 40 (sobre 
100). A partir de múltiples llamadas de este tipo, con distintos usuarios y 
distintas fuentes, se elabora un mapa relacional con pesos que simbolizan la 
relación y conexión entre usuarios de distintas páginas.
* Cada una de las llamadas lleva un identificador, asociado a quién ha creado 
esta relación. Por ejemplo, el Guesser encargado de definir si dos perfiles 
tienen la misma foto, tiene el identificador guesser_same_image. Imaginemos que 
la tiene, y manda [guesser_same_image, 1000, Linkedin, 278, Github, 30], o lo 
que equivale a decir que guesser_same_image dice que el usuario 1000 de linkedin 
y el usuario 278 de Github son el mismo con un 30% de probablidad. Si el mismo 
guesser decide a posterior que el valor debe ser mayor puede mandar otra linea 
con otro valor y el mismo identificador y el valor será substituido. Todos los 
valores con distintos identificadores pero mismos nodos [Linkedin, Github] o 
viceversa, serán sumados a la hora de computar el peso total de la arista.
* Para consultar se necesita una fuente, un usuario y el peso mínimo que 
consideremos que dos usuario son iguales. El resultado será equivalente a los 
usuarios de distintas fuentes que sean la misma que el que hemos pedido 
(siempre, por defecto, linkedin).

Nod está disponible en formato servicio PHP (librería o servicio de Symfony) o 
en su equivalente en capa REST (controlador + ruta).

## Anexos

### Anexo1

Se ha creado posteriormente una nueva organización llamada RSQueue con la 
separación entre la librería PHP de RSQueue y el bundle que expone la librería a 
las aplicaciones Symfony, RSQueueBundle. 
[Enlace Organización](http://github.com/rsqueue)

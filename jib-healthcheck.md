# Spring Cloud con Jib y Docker Compose

Cuando desplegamos arquitecturas de microservicios frecuentemente nos encontramos con dependencias entre contenedores, que obliga a arrancarlos en un orden específico.

Nos debemos asegurar de que cada contenedor ha arrancado correctamente y que los servicios que implementa están disponibles antes de arrancar los dependientes. Con Kubernetes u OpenShift este control lo proporcionan distintos mecanismos de los orquestadores, pero mientras desarrollamos en nuestro local, nos puede interesar una infraestructura más liviana orquestada con Docker Compose.

En este articulo expongo una aproximación todavía parcial en el contexto de una aplicación completa de comercio electrónico con una arquitectura CQRS-ES usando Axon Framework

Por una parte iremos un paso más allá del uso común de Jib, implementado una imagen base que proporcione funcionalidades adicionales a las imágenes propias del proyecto

Por otra, implementaremos un sistema de HealthCheck de contenedores parametrizable, que permita a cada contenedor realizar las comprobaciones que le interesen.

La arquitectura del Sistema de Comercio Electronico usado se basa, de hecho ES, la expuesta en el libro [Practical Microservices Architectural Patterns: Event-Based Java Microservices with Spring Boot and Spring Cloud](https://www.amazon.es/Practical-Microservices-Architectural-Patterns-Event-Based/dp/1484245008), pero dockerizada y con las adaptaciones que se verán.

Los fuentes de este articulo están subidos a github:

* [ecom-system](https://github.com/juancpaz/ecom)
* [ecom-config-repo](https://github.com/juancpaz/ecom-config-repo)
* [docker-compose]()

No es el objeto de este articulo exponer la arquitectura CQRS-ES, ni otros detalles que aquí se mencionen muy de pasada, y es un trabajo en progreso, con muchos flecos todavía sin resolver que, con el tiempo, espero tener la oportunidad de ir comentando en futuros artículos.

## La idea general

Jib es un plugin de maven/gradle que automaticamente gestiona las capas de las imagenes Docker generadas por el build, discrimina elementos de S.O, dependencias java y el programa propiamente dicho, para reutilizarlas entre contenedores, y para que durante el build/push/pull no sea necesario trasmitir imagenes completas. 

Era una evolución lógica en tecnología de contenedores para JVM, considerando que la mayor parte de un programa Java son dependencias, no tiene sentido subir imagenes de varios GB's sólo por haber cambiado un literal.

![alt text](ecom-05.png)

Otra caracteristica de Jib es que no usa Dockerfile's, todo lo necesario para definir la imagen se debe declarar en el pom.xml de maven, p.e. 

<details><summary>Declaración Jbi en pom.xml (Click para expandir)</summary>


```xml
<plugin>
	<groupId>com.google.cloud.tools</groupId>
	<artifactId>jib-maven-plugin</artifactId>
	<version>2.1.0</version>
	<configuration>
		<from>
			<image>registry.gitlab.com/juancpaz/ecom/ecom-base-image:0.0.1-SNAPSHOT</image>
		</from>
		<to>
			<image>ecom/${project.artifactId}:${project.version}</image>
		</to>
		<container>
			<entrypoint>
				<shell>bash</shell>
				<option>-c</option>
				<arg>chmod +x /entrypoint.sh &amp;&amp; sync &amp;&amp; /entrypoint.sh --mainclass com.juancpaz.ecom.registry.DiscoveryServer --healthcheck</arg>
			</entrypoint>					
			<ports>
				<port>8761</port>
			</ports>
			<creationTime>USE_CURRENT_TIMESTAMP</creationTime>
		</container>
	</configuration>
	<executions>
		<execution>
			<phase>install</phase>
			<goals>
				<goal>dockerBuild</goal>
			</goals>
		</execution>
	</executions>				
</plugin>
```

</p></details>

Con Jib declaramos la imagen base, la imagen que queremos generar y el entrypoint del container entre otros elementos, para los detalles se puede consultar la documentación de [Jib](https://github.com/GoogleContainerTools/jib)

Por defecto, Jib genera una imagen con un entrypoint

```java
java ${JAVA_OPTS} -noverify -XX:+AlwaysPreTouch -cp /app/resources/:/app/classes/:/app/libs/*
```

La idea me la dio el hecho de que por una parte se pueda incluir código ejecutable arbitrario como entrypoint, y por otra, que una imagen herede todo lo incluido en su imagen base.

Podemos personalizar nuestra imagen base tanto como sea necesario incluyendo, por ejemplo, paquetes linux adicionales...

```bash
# En el entrypoint.sh
function install_packages() {
	echo -n "Installing pakages... "
	apt-get update > /dev/null 2>&1
	for package in $(cat /packages.txt); do
		apt-get install -y ${package} > /dev/null 2>&1
	done
	echo "Done"
}

install_packages
```

o realizar operaciones más complejas como comprobar el estado de servicios externos.

La imagen base se comportara, permitidme la licencia, como una imagen abstracta, y las imagenes que la heredan como imagenes concretas. La operación de HealthCheck se declara en la base, pero la declaración de HealthCheks y la ejecución propiamente dicha de la comprobación se realiza en la imagen concreta.

![alt text](ecom-01.png)

El sistema que he implementado, declara comprobaciones en ficheros yaml, auqnue también entiende json y xml, por ejemplo:

```yaml
name: "healthchecks-1"
description: "Gateway Server Healthchecks"
healthChecks:
- type: "REST"
  name: "check-discovery-server-8761"
  description: "Check Configuration Server"
  delay: 5
  timeout: 30
  url: "http://ecom-discovery-server:8761/actuator/health"
  expected: 
    fields:
    - name: "status"
      value: "UP"
```

Declara un healthcheck contra el actuator del discovery de Eureka con un timeout de 30 segundos, si transcurrido el timeout el discovery no ha respondido con status UP, la ejecución del Gateway finaliza:

```bash
if [ "${ac_skip_healthcheck}" = "true" ]; then
	echo "Health Check skipped"
else
	if [ "${ac_command}" = "healthcheck" ]; then		
		echo "Executing Healthchecks"
		exec_healthchecks
	fi
fi
if [ "${ac_skip_service}" = "false" ]; then
	if [ ! "${ac_healthcheck_result}" = "ERROR" ]; then
		entry_point
	fi
else 
	echo "Skipping Service Execution"
fi
```

Antes de entrar en detalles de como se implementó, y cómo se usa, el mecanismo de heatlhcheck, voy a repasar brevemente la arquitectura del sistema al que da soporte. 

Se trata de una aplicación compleja de comercio electrónico, la típica tienda online, con su gestión de inventario, carrito, usuarios, compras, etc. Uso esta aplicación porque mi intención es estudiar el patrón CQRS-ES, y el [Framework AXON](https://axoniq.io/)

## Contenedores

El sistema se estructura en varios componentes, más o menos independientes:

* Infraestructura:
  - configuration-server
  - discovery-server
  - gateway-server
  - admin-server
  - security-service
  - user-service
* Servicios:
  - product-service
  - history-service
  - cart-service
  - shipping-service
* Aplicación:
  - web-server
* Externos:
  - MongoDB
  - RabbitMQ
  - MySql


Infraestructura, servicios, web y externos

![alt text](ecom-03.png)

## Dependencias

![alt text](ecom-04.png)

## Sistema de HealthCheck

![alt text](ecom-02.png)

### El entrypoint en baseimage

### Declaración de health checks

### Ejecución de HealthChecks

### Creación de nuevos HEalthChecks

## Referencias

https://enmilocalfunciona.io/construccion-de-imagenes-docker-con-jib/

## Temas pendientes y próximos pasos

* Uso de distintos orquestadores, distintas configuraciones en los mismos fuentes.
* Generalizar y crear un starter como los de spring boot, para reutilizar esta arquitectura en otros proyectos
* Implementar otros tipos de HealthCheck
* Implementar timeout global para el conjunto de healthchecks

## Notas

Habria que detener completamente el arranque de un contenedor hasta q no se de una condicion, pero docker compose arranca los contenedores uno tras otro, sin esperar, segun el orden de declaración en el yaml

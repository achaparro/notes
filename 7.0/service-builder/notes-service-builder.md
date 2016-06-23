#Service Builder
Construido bajo las tecnologías:
- Hibernate
- Spring
- OSGI

A partir de XML con plantillas de fremarker genera;
- local service
. Service
- Persistance
- SOAP-AXIS: deprecated
- JSON-RPC
- HTTP Tunnel

También ficheros de base de datos, mapeos de hibernate, caché, tests de persistencia, etc.

Para moodulos y core.

Es un jar + Gradle plugin

## Definición del XML
El finder siempre genera un índice por defecto. Se puede deshabilitar.
Los índices ayudan en las búsquedas pero perjudican las escrituras.

### Dependencias
Las relaciones con otros objetos generan referencias en las clases que obtienen los bean de esos otros objetos:

Cuando declaramos en el XML `<reference>`

Genera, por ejemplo, UserLocalServiceBaseImpl:
    @BeanReference(type = com.liferay.portal.kernel.service.LayoutLocalService.class)
    protected com.liferay.portal.kernel.service.LayoutLocalService layoutLocalService;

Para localizar el bean busca en el contexto de Spring.

Esto con OSGI cambia dado que necesario el bean de otro contexto diferente al módulo actual.
Genera ahora un @ServiceReference:
    @ServiceReference(type = com.liferay.ratings.kernel.service.RatingsStatsLocalService.class)
    protected com.liferay.ratings.kernel.service.RatingsStatsLocalService ratingsStatsLocalService;

Si no detecta que la entidad pertenece al módulo, la referencia la crea como servicio.
(esto no añade la dependencia de gradle, deberemos añadir dicha dependencia incluyendo la versión)

En tiempo de ejecución, si alguna dependencias del @ServiceReference no está disponible, el conexto de Spring no arranca y por tanto el módulo no estará disponible.

En cada módulo, al generar un jar, se genera un fichero context-dependencies que contiene todos los servicios de los que dependen y si alguno no está disponible el contexto de la aplicación no arrancará.

### Opciones adicionales
Tienes muchas opciones definidas por defecto pero se pueden cambiar en el XML:
- MvccVersion enabled
- Cache enabled
- Trash enabled
- Etc.

## Proceso de generación Java
Toda la lógica está en la clase `ServiceBuilder.java`

## Plugin de Gradle
Por estar dentro del build de Liferay se aplica siempre este plugin a este módulo. Con esto dispones de una sección de configuración y tareas gradle del service builder.
Por ejemplo:
/bookmarks-service/build.gradle

Ya podemos aplicar propiedades del service builder:
buildService {
    apiDir = "../bookmarks-api/src/main/java"
    testDir = "../bookmarks-test/src/testIntegration/java"
}

En blade está hecho:
    - gradle
    - liferayGradle:
        + Service builder sample (te añade la dependencia con el módulo de service builder)
        https://github.com/liferay/liferay-blade-samples/blob/master/liferay-gradle/blade.servicebuilder.svc/build.gradle 

## Caché
Los finder que se generan automáticamente llevan su caché (incluidos en el XML)

Los manuales generados por ti, por ejemplo con custom SQL, no tienen cachñe generada.

## Interceptores
En los servicios generados con Service Builder tienes:
- Transacciones
- Indexación
- System stadistics
- Retry
- Etc.

La llamada a un *LocalService se pasa por los interceptores de cada uno de los anteriores en un orden en concreto, tanto al inicio como al final de la invocación.
Ver:
/portal-master/portal-impl/src/META-INF/base-spring.xml

Por ejemplo:
/portal-master/portal-impl/src/com/liferay/portal/cache/thread/local/ThreadLocalCacheAdvice.java
/portal-master/portal-impl/src/com/liferay/portal/search/IndexableAdvice.java

Tienes la posibilidad de declarar de métodos after y before, ejecutando en medio el método del servicio (invoke)

La lógica se suele basar en las anotaciones sobre los métodos. Ver /portal-master/portal-web/docroot/dtd/liferay-service-builder_7_0_0.dtd

No se puede crear un módulo/plugin que añada un interceptor. Quizás se puede conseguir con las propiedades de spring del portal.

### Base de datos y transacciones
Se basa en la de Spring y Hibernate. Ver propiedad `transaction.manager.impl`

Cuando llamo a un método de un servicio se crea una transcción (en el proxy que tenemos), si llamas a otro servicio en su lógica se reutiliza la transacción porque ya no pasa por el proxy (no crea una instancia de ese servicio sino que se usa el bean)

**¿Transaccionalidad entre módulos?**
Es igual en los módulos, una transacción por cada llamada a un método de un servicio aunque se llamen a servicios de otro módulo dentro.

**Crear tus entidades en otro datasource**
No es recomendable porque las operaciones no estarían en la misma transaccin. Sólo podría ser viable si las operaciones con esas entidades no se mezclan con operaciones con entidades de Liferay.
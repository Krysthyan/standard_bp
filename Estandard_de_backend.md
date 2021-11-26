# Estándar de Backend

### Configuración de IDEs

La configuración a realizar es el estilo de google, se puede acceder a los archivos en el siguiente repositorio **[Google style](https://github.com/google/styleguide)**

Se seleccionará el estilo de java para el editor o IDE que se utilize.

### Nombrado del proyecto

Cada repositorio o proyecto tiene un patrón de como deben ser nombrados. La estructa que se va a emplear es la siguiente:

**\<name_1>-\<name_2>-\<name_3>**

* name_1 -> iniciales de tu celula (Crédito)
* name_2 -> microservice architecture
* name_3 -> nombre del proyecto

Por ejemplo: **crd-msa-customer**

### Estructura de paquetes

Para el manejo de la estructura de de los paquetes se tomará como referencia [Domain driven design](https://es.wikipedia.org/wiki/Dise%C3%B1o_guiado_por_el_dominio), en donde tendremos la siguiente estructura:

#### Proyectos REST

~~~
* helm
  * development.yaml 
  * production.yaml
  * staging.yaml
* src
  * main
    * java
      * com.pichincha.<celula>.<nombre-proyecto>
        * config
        * respository
        * service
        * controller
        * domain
          * enums
        * util
    * resources
      - application.yml
      - application-dev.yml
      - application-staging.yml
      - application-prod.yml
      * db.migration
  * test
    * java
      * com.pichincha.<celula>.<nombre-proyecto>
        * configuration
        * respository
        * service
        * controller
        * util
- azure-pipelines.yml
- .gitignore
- README.md
- pom.xml
- settings.xml
- Dockerfile


~~~



#### Proyectos WebFlux

~~~
* helm
  * development.yaml
  * production.yaml
  * staging.yaml
* src
  * main
    * java
      * com.pichincha.<celula>.<nombre-proyecto>
        * config
        * respository
        * handler
        * service
        * domain
          * enums
        * util
    * resources
      - application-dev.yml
      - application-staging.yml
      - application-prod.yml
      * db.migration
  * test
    * java
      * com.pichincha.<celula>.<nombre-proyecto>
        * configuration
        * respository
        * service
        * util
        * handler
- azure-pipelines.yml
- .gitignore
- README.md
- pom.xml
- settings.xml
- Dockerfile
~~~

##### Heml

Se trata de los archivos de configuración de variables en los diferentes ambientes, **dev**, **staging**, **prod**

##### Config

En este paquete se incluyen todas las clases que hagan referencia a configuraciones del proyecto, como por ejm:

* _ApplicationProperties.java_ (Mapear las variables establecidas en los resources @ConfigurationProperties)
* _Constants.java_ (Establecer las diferentes constantes que va a tener el proyecto)
* _DatabaseConfiguration.java_ (Configuración de la base de datos que se este utilizando)
* _CacheDBConfiguration.java_ (Configurar la base de datos que se esté utilizando en el proyecto)
* _JacksonConfiguration.java_ (Configuración de la serialización de los objetos del proyecto)
* _LogginConfiguration.java_ (Configuración de los logs del proyecto)
* _SecurityConfiguration.java_ (Configuración de la seguridad manejada en el proyecto, como las rutas permitidas y los roles permidos en caso de existir)
* _WebfluxConfiguration.java_ (En caso de ser el proyecto de tipo webflux, en esta clase se coloca las diferentes configuraciones establecidas para el proyecto)

##### Repository

Este paquete sirve para agregar las diferentes interfaces e implementaciones para la obtención de datos de una DB u realizar peticiones a otros microservicios, como por ejm:

* Repository

* ~~~java
  public interface CustomerRepository {
    Mono<ContractCustomerResponseDto> find(String identification, String identificationType);
  }
  ~~~

  * impl

    * ~~~java
      @Component
      @RequiredArgsConstructor
      public class CustomerRepositoryImpl implements CustomerRepository {
      
        private final WebClientHelper webClientHelper;
        private final TransactionTrackingHelper transactionTrackingHelper;
        
        @Value("${url.get.customer.contract-data}")
        private String urlGetCustomerContractData;
      
        public Mono<ContractCustomerResponseDto> find(String identification, String identificationType){
          return webClientHelper.builder()
              .header(transactionTrackingHelper.getMdcHeaders()).build().get()
              .uri(urlGetCustomerContractData, identification, identificationType)
              .retrieve().bodyToMono(ContractCustomerResponseDto.class);
        }
      }
      ~~~

##### Service

En este paquete se realiza todas las interfaces e implementaciones de los servicios (logica) de nuestro proyecto, por ejm:

* service

*  ~~~java
   public interface UrlService {
     Mono<HashResponseDto> create(OtpRequest request);
   }
   ~~~

  * impl

  * ~~~ java
    @Slf4j
    @Service
    @RequiredArgsConstructor
    public class UrlServiceImpl implements UrlService {
      private final CreateOfferStoreService createOfferStoreService;
      @Override
      public Mono<HashResponseDto> create(OtpRequest request) {
        return Mono.zip(Mono.just(request).subscribeOn(Schedulers.parallel()),
            createOfferStoreService.find(request.getIdentification(), 		     request.getIdentificationType()).subscribeOn(Schedulers.parallel()))
            .flatMap(this::validateStatusOffer);
      }
    
      private Mono<HashResponseDto> validateStatusOffer(Tuple2<OtpRequest, CreateOfferModel> tuple) {
        updateAndSaveCreateOffer(tuple);
        return offerStoreService.delete(request.getIdentification(), request.getIdentificationType())
            .flatMap(deleteStatus -> {
              return Mono.just(HashResponseDto.builder()
                  .code(deleteStatus.getCode())
                  .message(deleteStatus.getMessage())
                  .build());
            });
      }
    ~~~

##### Controller

Paquete donde se guardan las diferentes clases que contienen los endpoints de los servicios que va a tener el proyecto, por ejm:

* controller

* ~~~ java
  @Slf4j
  @RequiredArgsConstructor
  @RestController
  @RequestMapping("${endpoint.url}")
  public class CheckOfferController {
      
    private String commonHeaderCrdDeviceIdKey;
  
    @PostMapping(path = "/check/offer", produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<CommonResponseDto> checkOffer(@RequestBody PersonDto personDto,
            @RequestHeader Map<String, String> headers) {
      
      headers.entrySet().forEach(mapEntry -> MDC.put(mapEntry.getKey(), mapEntry.getValue()));
      MDC.put(commonHeaderCrdSesionIdKey, commonHeaderCrdSesionIdKey);
  
      if (BooleanUtils.isFalse(validateCaptcha(personDto.getIdentification(), headers.get(RECAPTCHA_TOKEN)))) {
        return ResponseEntity.ok(CommonResponseDto.builder()
                .code(StatusCode.INVALID_RECAPTCHA.getCode())
                .message(StatusCode.INVALID_RECAPTCHA.getMessage()).build());
      }
  ~~~

##### Haldler

--------------- no se como explicar esta parte ---------------------

##### Domains

En este paquete se guardará los modelos utilizados en el proyectos. Dentro de este paquete existe uno adicional que es el de enum, paquete que sirve para guardar las diferentes clases tipo enum que tenga el proyecto.

~~~java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder(toBuilder = true)
public class CreateOfferRequestDto {
  
  private String identification;
  private String identificationType;
  private String timeWorking;
  private String cif;
  private String type;
  private ProductDto product;
  private List<JobStatusDto> jobStatus;
}
~~~

Para la creación de los modelos se recomienda utilizar la dependencia Lombok, la misma que facilita la implementación de código repetitivo, como los gets y sets de las variables, constructores, la anotación Builder, etc.

##### Util

Paquete para las clases que tienen funciones utilizadas en algunas partes del proyecto.

En este paquete también se puede agregar las clases TransactionTrackingUtil y WebClientUtil

* TransactionTrackingUtil (Componente para inyectar las cabeceras de las peticiones a los demas microservicios cuando sean invocados)
* WebFluxClientUtil | RestClientUtil (Componente que tiene la funcionalidad de invocar otros microservicios con la configuración establecida en esta clase).

##### Resources

Aquí se encuentran los archivos de las variables configuradas en el proyecto, existen 4 archivos de variables

* application.yml (configuración y variables para ambiente local)
* application-dev.yml (configuración y variables para ambiente de desarrollo)
* application-staging.yml (configuración y variables para ambiente de pruebas)
* application-prod.yml (configuración y variables para ambiente de producción)

Adicional se encuentra un paquete denominado db.migration que tiene como objetivo almacenar los diferentes scripts SQL que vaya modificando la base de datos, para el nombramiento de estos scripts, van a seguir el siguiente patron:

* V\<anio>\<mes>\<dia>\<hora>\<minuto>__<create|update|delete>\_\<funcionalidad>.sql
  * V202101011201\_\_create_table_parameter.sql

### Nomenclatura

- Todo el código debe ser escrito en inglés, esto incluye a todo el proyecto (clases, paquetes, código)
- No se debe dejar código comentado
- Los comentarios agregados deben ser para documentar el código.
- En el caso del uso del lenguaje Java, se recomienda el uso de las librerías **lombok** y **mapstruct**
- Utilizar los lineamientos de [Clean code of  Robert C. Martin](https://www.amazon.com/-/es/Robert-C-Martin/dp/0132350882)

#### clases

- El nombre de las clases deben ser a referencia de lo que va hacer la clase.
- Para su nombramiento se debe utilizar el estilo _CamelCase_ en el caso de **Java** y _snake_case_ en el caso de **Python**.
- En el nombre no se debe incluir prefijos o sufijos como DAO, URL, HTML.
- Clases que sean implementación de una _interfaz_ se agregará a su nombre la terminación **Impl**

Mencionar sobre los Dtos

#### Metodos

---------- estandard para los metodos -------------

### Documentación

Para la documentación existen dos secciones, que se trata de:

- Documentación de código
- Documentación para APIs

#### Documentación de código

Para documentar el código se utiliza el estandard de de JAVADOC.

No se permite realizar documentación dentro de los métodos o dejar código comentado en el proyecto.

En el caso de Interfaces e implementaciones, se debe realizar la documentación en la interfaz y no en la implementación.

#### Documentación para APIs

Para la documentación en Java utilizar la dependencia _spring-openapi_ para automatizar la generación de documentación API en formato JSON / YAML / y APIs HTML. 

Esta documentación se puede completar con comentarios utilizando anotaciones de swagger-api, como por ejemplo:

~~~ java
@Operation(summary = "Create customer")
@ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Successfully created a customer"),
        @ApiResponse(responseCode = "400", description = "Bad Request"),
        @ApiResponse(responseCode = "401", description = "Authorization denied"),
        @ApiResponse(responseCode = "500", description = "Unexpected system exception"),
        @ApiResponse(responseCode = "502", description = "An error has occurred with an upstream service")
})
@PostMapping(consumes = JSON)
public ResponseEntity createCustomer(@Valid @RequestBody CustomerInfo customerInfo, UriComponentsBuilder uriBuilder)
    throws Exception {
    ...
}
~~~

### Pruebas

Utilizar JUnit 5

Utilziar JaCoCo para la cobertura de pruebas unitarias -> 75%

#### Unitarias

------- pruebas unitarias ----------

#### Integración

--------- pruebas de integración ----------

### Buenas prácticas

mencionar solid single Respo

#### Standard dependecias en pom

- Colocar los números de versión de las dependencias dentro de la sección de _\<properties\>_
- Incluir las versiones de todas las dependencias
- No incluir el número de versiones de las dependecias que forman parte de la lista de spring boot.
- No mezclar varias versiones de una dependecia en un proyecto, utilizar _mvn dependency:tree_ para identificar versiones conflictivas de las dependecias.

#### Reglas de Logging *** borrar -> Estandard escritura de logs EventLog()  

- Nunca utilizar System.Out o System.Err
- Utilizar siempre SLF4J, Utilize la anotación _@Slf4j_ a nivel de clase
- Utilizar siempre _Logback_ que viene incluido en spring boot.
- No utilizar Java Util Logging (JUL)

#### Favorezca la inyección de constructor en lugar de @Autowired

- Mantiene el código limpio, no nos permite crear dependecias circulares entre beans
- Facilita la creación de instancias.
- La mejor práctica para la inyección de dependecias en usar _Lombok_. Donde se debe declar una propiedad final del tipo interfaz y anotar la clase usando _@RequiredArgsConstructor_ 

#### Manejo de excepciones globales

- Spring boot proporciona dos formas principales de manejar excepciones a nivel global

  - HandlerExceptionResolver para definir la estrategia global de manejo de excepciones.

  - Anotar los controladores con la anotación _@ExceptionHandler_. Es útil para determinados casos, por ejemplo:

    ~~~java
    @ExceptionHandler(NotFoundException.class)
    @ResponseBody
    public ResponseEntity<ErrorResponse> handleNotFoundException(NotFoundException exception) {
        ErrorResponse errorResponse = ErrorResponse.builder()
                .errorCode(HttpStatus.NOT_FOUND.toString())
                .errorKey(UNKNOWN_DATA_ITEM.name())
                .errorMessage(exception.getMessage()).build();
        return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
    }
    ~~~

#### Validación de beans

#### Paginación y clasificación con Spring Data JPA

#### Exponer puntos finales de comprobación de estado

#### Aplicar estilo de verificación

#### Liquibase para la gestión de cambioos en el esquema de base de datos -> revisar otras opciones

#### utilizar maven wrapper

#### Patrones de arquitectura -> si se pone hay que hace revisiones con arch

#### Patrones de diseno



### Plantillas base

#### Spring rest

#### Spring webflux






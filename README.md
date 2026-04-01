# Curso de Testing en Java: JUnit 5, Mockito y Spring Boot

Curso completo de pruebas unitarias e integración en Java, cubriendo desde los fundamentos con JUnit 5 hasta testing avanzado con Spring Boot.

---

## Tabla de Contenidos

1. [JUnit 5 - Fundamentos](#1-junit-5---fundamentos)
2. [Mockito - Mocks y Stubs](#2-mockito---mocks-y-stubs)
3. [Mockito Avanzado - Spies, Captors y Order](#3-mockito-avanzado---spies-captors-y-order)
4. [Spring Boot - Testing de Servicios](#4-spring-boot---testing-de-servicios)
5. [Spring Boot - Testing de Repositorios JPA](#5-spring-boot---testing-de-repositorios-jpa)
6. [Spring Boot - Testing de Controllers con MockMvc](#6-spring-boot---testing-de-controllers-con-mockmvc)
7. [Spring Boot - Integration Testing con TestRestTemplate](#7-spring-boot---integration-testing-con-testresttemplate)
8. [Spring Boot - Integration Testing con WebTestClient](#8-spring-boot---integration-testing-con-webtestclient)
9. [Patrones y Buenas Practicas](#9-patrones-y-buenas-practicas)
10. [Comparativa de Frameworks](#10-comparativa-de-frameworks)

---

## 1. JUnit 5 - Fundamentos

**Modulo:** `junit5_app`

JUnit 5 (Jupiter) es el framework de pruebas unitarias estandar para Java. Reemplaza a JUnit 4 con una arquitectura modular y nuevas funcionalidades.

### Ciclo de Vida de un Test

JUnit 5 ofrece anotaciones para controlar la ejecucion antes y despues de cada test o de toda la clase:

```java
@BeforeAll
static void beforeAll() {
    // Se ejecuta UNA vez antes de todos los tests de la clase
}

@BeforeEach
void setUp() {
    // Se ejecuta ANTES de cada test individual
}

@AfterEach
void tearDown() {
    // Se ejecuta DESPUES de cada test individual
}

@AfterAll
static void afterAll() {
    // Se ejecuta UNA vez despues de todos los tests de la clase
}
```

- `@BeforeAll` y `@AfterAll` deben ser `static` (a menos que se use `@TestInstance(Lifecycle.PER_CLASS)`).
- `@BeforeEach` es ideal para inicializar objetos frescos en cada test y evitar estado compartido.

### Inyeccion de Metadata

JUnit 5 permite inyectar `TestInfo` y `TestReporter` como parametros del metodo:

```java
@BeforeEach
void setUp(TestInfo testInfo, TestReporter testReporter) {
    testReporter.publishEntry("Ejecutando: " + testInfo.getDisplayName());
}
```

### Assertions (Aserciones)

Las aserciones son la base de toda prueba. Verifican que el resultado obtenido coincida con el esperado.

| Asercion | Descripcion |
|----------|-------------|
| `assertEquals(expected, actual)` | Verifica igualdad por valor |
| `assertNotEquals(a, b)` | Verifica que no sean iguales |
| `assertTrue(condition)` | Verifica que la condicion sea `true` |
| `assertFalse(condition)` | Verifica que la condicion sea `false` |
| `assertNotNull(object)` | Verifica que el objeto no sea `null` |
| `assertNull(object)` | Verifica que el objeto sea `null` |
| `assertSame(a, b)` | Verifica que sean la misma instancia (referencia) |
| `assertThrows(Exception.class, () -> ...)` | Verifica que se lance una excepcion |
| `assertTimeout(duration, () -> ...)` | Verifica que la ejecucion no exceda un tiempo |

#### assertAll - Agrupacion de Aserciones

Permite ejecutar multiples aserciones y reportar **todas** las que fallen, en lugar de detenerse en la primera:

```java
assertAll(
    () -> assertEquals("Oscar", cuenta.getPersona()),
    () -> assertEquals(1000, cuenta.getSaldo().intValue()),
    () -> assertEquals("Banco Nacional", cuenta.getBanco().getNombre())
);
```

#### Mensajes Lazy con Lambdas

Los mensajes de error se pueden construir con lambdas para evitar la creacion innecesaria de strings cuando el test pasa:

```java
assertEquals(expected, actual, () -> "El saldo esperado era " + expected + " pero fue " + actual);
```

### Excepciones

```java
@Test
void testDineroInsuficiente() {
    Exception exception = assertThrows(DineroInsuficienteException.class, () -> {
        cuenta.debito(new BigDecimal("2000"));
    });
    assertEquals("Dinero Insuficiente", exception.getMessage());
}
```

### Organizacion de Tests

#### @DisplayName

Proporciona nombres descriptivos legibles para los tests:

```java
@Test
@DisplayName("Test que verifica el nombre de la cuenta")
void testNombreCuenta() { ... }
```

#### @Nested - Clases Anidadas

Permite agrupar tests relacionados en clases internas:

```java
@Nested
@DisplayName("Tests de operaciones con cuenta")
class CuentaOperacionesTest {

    @Test
    void testDebito() { ... }

    @Test
    void testCredito() { ... }
}
```

#### @Tag - Etiquetas

Permite categorizar y filtrar tests para ejecucion selectiva:

```java
@Tag("cuenta")
@Test
void testSaldo() { ... }

@Tag("banco")
@Test
void testTransferencia() { ... }
```

Se filtran desde Maven con:
```xml
<groups>cuenta</groups>
<excludedGroups>banco</excludedGroups>
```

### Ejecucion Condicional

JUnit 5 permite habilitar o deshabilitar tests segun el entorno de ejecucion:

```java
// Por sistema operativo
@EnabledOnOs(OS.WINDOWS)
@DisabledOnOs(OS.LINUX)

// Por version de Java
@EnabledOnJre(JRE.JAVA_11)
@DisabledOnJre(JRE.JAVA_8)

// Por propiedad del sistema
@EnabledIfSystemProperty(named = "ENV", matches = "dev")

// Por variable de entorno
@EnabledIfEnvironmentVariable(named = "JAVA_HOME", matches = ".*jdk.*")
```

#### Assumptions (Suposiciones)

Las assumptions cancelan el test (no lo fallan) si la condicion no se cumple:

```java
@Test
void testSaldoDev() {
    boolean esDev = "dev".equals(System.getProperty("ENV"));
    assumeTrue(esDev); // Si no es dev, el test se omite
    assertEquals(1000, cuenta.getSaldo().intValue());
}

@Test
void testSaldoCondicional() {
    assumingThat("dev".equals(System.getProperty("ENV")), () -> {
        // Este bloque solo se ejecuta si la condicion es verdadera
        assertEquals(1000, cuenta.getSaldo().intValue());
    });
}
```

### Tests Repetidos

```java
@RepeatedTest(value = 5, name = "Repeticion {currentRepetition} de {totalRepetitions}")
void testDebitoRepetido(RepetitionInfo info) {
    if (info.getCurrentRepetition() == 3) {
        System.out.println("Estamos en la repeticion 3");
    }
    cuenta.debito(new BigDecimal("100"));
    assertNotNull(cuenta.getSaldo());
}
```

### Tests Parametrizados

Permiten ejecutar el mismo test con diferentes datos de entrada, eliminando duplicacion:

```java
// Con @ValueSource
@ParameterizedTest
@ValueSource(strings = {"100", "200", "500", "1000"})
void testDebitoValueSource(String monto) {
    cuenta.debito(new BigDecimal(monto));
    assertTrue(cuenta.getSaldo().compareTo(BigDecimal.ZERO) >= 0);
}

// Con @CsvSource (inline)
@ParameterizedTest
@CsvSource({"1, 100", "2, 200", "3, 500"})
void testDebitoCsv(String index, String monto) {
    cuenta.debito(new BigDecimal(monto));
    assertTrue(cuenta.getSaldo().compareTo(BigDecimal.ZERO) > 0);
}

// Con @CsvFileSource (archivo externo)
@ParameterizedTest
@CsvFileSource(resources = "/data.csv")
void testDebitoArchivoCsv(String monto) {
    cuenta.debito(new BigDecimal(monto));
    assertNotNull(cuenta.getSaldo());
}

// Con @MethodSource (metodo estatico)
@ParameterizedTest
@MethodSource("montoList")
void testDebitoMethodSource(String monto) {
    cuenta.debito(new BigDecimal(monto));
    assertTrue(cuenta.getSaldo().compareTo(BigDecimal.ZERO) > 0);
}

static List<String> montoList() {
    return Arrays.asList("100", "200", "500", "1000");
}
```

### Timeout

```java
@Test
@Timeout(value = 5, unit = TimeUnit.SECONDS)
void testTimeout() throws InterruptedException {
    TimeUnit.SECONDS.sleep(4);
}

@Test
void testTimeoutAssertions() {
    assertTimeout(Duration.ofSeconds(5), () -> {
        TimeUnit.SECONDS.sleep(4);
    });
}
```

---

## 2. Mockito - Mocks y Stubs

**Modulo:** `app-mockito`

Mockito permite crear objetos simulados (mocks) de las dependencias para probar clases de manera aislada, sin depender de implementaciones reales.

### Conceptos Clave

- **Mock:** Objeto simulado que reemplaza una dependencia real. No ejecuta logica real.
- **Stub:** Comportamiento programado en un mock (`when(...).thenReturn(...)`).
- **Verify:** Verificar que un metodo del mock fue invocado.

### Configuracion

```java
@ExtendWith(MockitoExtension.class)
class ExamenServiceImplTest {

    @Mock
    ExamenRepository examenRepository;

    @Mock
    PreguntaRepository preguntaRepository;

    @InjectMocks
    ExamenServiceImpl service; // Inyecta los mocks automaticamente
}
```

- `@ExtendWith(MockitoExtension.class)`: Inicializa Mockito automaticamente.
- `@Mock`: Crea un mock de la interfaz o clase.
- `@InjectMocks`: Crea la instancia real e inyecta los mocks en el constructor o setters.

### Stubbing - Definir Comportamiento

```java
// Retornar un valor
when(examenRepository.findAll()).thenReturn(Datos.EXAMENES);

// Retornar segun argumento
when(preguntaRepository.findPreguntasPorExamenId(anyLong()))
    .thenReturn(Datos.PREGUNTAS);

// Lanzar excepcion
when(examenRepository.findAll())
    .thenThrow(new IllegalArgumentException("Error"));
```

### Argument Matchers

Los matchers permiten flexibilidad al definir stubs y verificaciones:

```java
when(preguntaRepository.findPreguntasPorExamenId(anyLong()))
    .thenReturn(Datos.PREGUNTAS);

when(examenRepository.guardar(any(Examen.class)))
    .thenReturn(Datos.EXAMEN);
```

| Matcher | Descripcion |
|---------|-------------|
| `any()` | Cualquier objeto |
| `any(Clase.class)` | Cualquier objeto del tipo |
| `anyLong()` | Cualquier `long` |
| `anyString()` | Cualquier `String` |
| `anyList()` | Cualquier `List` |
| `eq(valor)` | Exactamente ese valor |
| `isNull()` | Valor `null` |

### Verificacion de Invocaciones

```java
verify(examenRepository).findAll();
verify(preguntaRepository).findPreguntasPorExamenId(5L);
verify(examenRepository).guardar(any(Examen.class));
verify(preguntaRepository, never()).findPreguntasPorExamenId(99L);
```

### Answers - Respuestas Dinamicas

Para logica mas compleja en los stubs:

```java
when(examenRepository.guardar(any(Examen.class))).then(new Answer<Examen>() {
    Long secuencia = 8L;

    @Override
    public Examen answer(InvocationOnMock invocation) throws Throwable {
        Examen examen = invocation.getArgument(0);
        examen.setId(secuencia++);
        return examen;
    }
});
```

---

## 3. Mockito Avanzado - Spies, Captors y Order

**Modulo:** `app-mockito 2`

### Spy vs Mock

| Caracteristica | Mock | Spy |
|----------------|------|-----|
| Comportamiento por defecto | No hace nada (retorna null/0/false) | Ejecuta el metodo real |
| Uso principal | Simular dependencias completamente | Sobreescribir metodos puntuales |
| Requiere | Interfaz o clase | Clase concreta (con implementacion) |

```java
@Spy
ExamenRepositoryImpl examenRepository; // Spy sobre clase concreta

@Mock
PreguntaRepository preguntaRepository; // Mock de interfaz

@InjectMocks
ExamenServiceImpl service;
```

#### Stubbing en Spy

Con spies se usa `doReturn().when()` en lugar de `when().thenReturn()` para evitar que se ejecute el metodo real durante el stubbing:

```java
// Correcto para spy:
doReturn(Datos.EXAMENES).when(examenRepository).findAll();

// Incorrecto para spy (ejecutaria el metodo real):
// when(examenRepository.findAll()).thenReturn(Datos.EXAMENES);
```

### Argument Matchers Personalizados

```java
// Con lambda
verify(examenRepository).guardar(argThat(arg ->
    arg.getNombre() != null && arg.getNombre().contains("Matematicas")
));

// Con clase custom
verify(examenRepository).guardar(argThat(new MiArgumentMatcher()));

class MiArgumentMatcher implements ArgumentMatcher<Examen> {
    @Override
    public boolean matches(Examen examen) {
        return examen != null && examen.getId() != null;
    }
}
```

### Verificacion de Orden de Invocaciones

```java
InOrder inOrder = inOrder(examenRepository, preguntaRepository);
inOrder.verify(examenRepository).findAll();
inOrder.verify(preguntaRepository).findPreguntasPorExamenId(anyLong());
inOrder.verify(examenRepository).guardar(any(Examen.class));
```

### Verificacion de Cantidad de Invocaciones

```java
verify(examenRepository, times(1)).findAll();       // Exactamente 1 vez
verify(examenRepository, atLeast(1)).findAll();      // Al menos 1 vez
verify(examenRepository, atLeastOnce()).findAll();   // Al menos 1 vez
verify(examenRepository, atMost(3)).findAll();       // Maximo 3 veces
verify(examenRepository, atMostOnce()).findAll();    // Maximo 1 vez
verify(examenRepository, never()).guardar(any());    // Nunca invocado

// Verificar que no hubo NINGUNA interaccion con el mock
verifyNoInteractions(preguntaRepository);
```

### doCallRealMethod - Llamar Metodo Real en Mock

```java
doCallRealMethod().when(examenRepository).findAll();
```

Util cuando necesitas comportamiento real en un mock para un metodo especifico.

---

## 4. Spring Boot - Testing de Servicios

**Modulo:** `springboot_test_services_test`

### @SpringBootTest

Carga el contexto completo de Spring, permitiendo tests de integracion con beans reales:

```java
@SpringBootTest
class SpringbootTestApplicationTests {

    @MockBean  // Reemplaza el bean en el contexto de Spring
    CuentaRepository cuentaRepository;

    @MockBean
    BancoRepository bancoRepository;

    @Autowired  // Inyecta el servicio real (con los mocks inyectados)
    CuentaService service;
}
```

### @MockBean vs @Mock

| Anotacion | Framework | Alcance |
|-----------|-----------|---------|
| `@Mock` | Mockito | Solo dentro de la clase de test |
| `@MockBean` | Spring Boot Test | Reemplaza el bean en todo el contexto de Spring |

`@MockBean` es necesario cuando el servicio bajo prueba es un bean de Spring que recibe sus dependencias via inyeccion de Spring.

### Ejemplo de Test de Servicio

```java
@Test
void testTransferir() {
    // Arrange
    when(cuentaRepository.findById(1L)).thenReturn(Datos.crearCuenta001());
    when(cuentaRepository.findById(2L)).thenReturn(Datos.crearCuenta002());
    when(bancoRepository.findById(1L)).thenReturn(Datos.crearBanco());

    // Act
    service.transferir(1L, 2L, new BigDecimal("100"), 1L);

    // Assert
    BigDecimal saldoOrigen = service.revisarSaldo(1L);
    BigDecimal saldoDestino = service.revisarSaldo(2L);
    assertEquals("900", saldoOrigen.toPlainString());
    assertEquals("2100", saldoDestino.toPlainString());

    // Verify
    verify(cuentaRepository, times(3)).findById(1L);
    verify(cuentaRepository, times(3)).findById(2L);
    verify(cuentaRepository, times(2)).save(any(Cuenta.class));
    verify(bancoRepository, times(2)).findById(1L);
    verify(bancoRepository).save(any(Banco.class));
    verify(cuentaRepository, never()).findAll();
}
```

### Test de Excepciones en Servicio

```java
@Test
void testTransferirDineroInsuficiente() {
    when(cuentaRepository.findById(1L)).thenReturn(Datos.crearCuenta001());
    when(cuentaRepository.findById(2L)).thenReturn(Datos.crearCuenta002());
    when(bancoRepository.findById(1L)).thenReturn(Datos.crearBanco());

    assertThrows(DineroInsuficienteException.class, () -> {
        service.transferir(1L, 2L, new BigDecimal("1200"), 1L);
    });

    verify(cuentaRepository, never()).save(any(Cuenta.class));
}
```

---

## 5. Spring Boot - Testing de Repositorios JPA

**Modulo:** `springboot_test_data_jpa`

### @DataJpaTest

Configura automaticamente una base de datos en memoria (H2) y solo carga los componentes JPA:

```java
@DataJpaTest
class IntegracionJpaTest {

    @Autowired
    CuentaRepository cuentaRepository;
}
```

**Caracteristicas de @DataJpaTest:**
- Auto-configura un `DataSource` con H2 en memoria
- Aplica los scripts `import.sql` automaticamente
- Cada test se ejecuta en una transaccion que hace rollback al finalizar
- Solo carga repositorios, entidades y configuracion JPA (no controllers ni servicios)

### Operaciones CRUD Testeadas

```java
// Buscar por ID
@Test
void testFindById() {
    Optional<Cuenta> cuenta = cuentaRepository.findById(1L);
    assertTrue(cuenta.isPresent());
    assertEquals("Oscar", cuenta.orElseThrow().getPersona());
}

// Buscar por atributo custom
@Test
void testFindByPersona() {
    Optional<Cuenta> cuenta = cuentaRepository.findByPersona("Oscar");
    assertTrue(cuenta.isPresent());
    assertEquals("1000.00", cuenta.orElseThrow().getSaldo().toPlainString());
}

// Guardar nueva entidad
@Test
void testSave() {
    Cuenta cuenta = new Cuenta(null, "Pepe", new BigDecimal("3000"));
    Cuenta resultado = cuentaRepository.save(cuenta);
    assertEquals("Pepe", resultado.getPersona());
    assertEquals("3000", resultado.getSaldo().toPlainString());
}

// Actualizar entidad
@Test
void testUpdate() {
    Cuenta cuenta = cuentaRepository.findById(1L).orElseThrow();
    cuenta.setSaldo(new BigDecimal("3800"));
    Cuenta resultado = cuentaRepository.save(cuenta);
    assertEquals("3800", resultado.getSaldo().toPlainString());
}

// Eliminar entidad
@Test
void testDelete() {
    Cuenta cuenta = cuentaRepository.findById(2L).orElseThrow();
    cuentaRepository.delete(cuenta);
    assertThrows(NoSuchElementException.class, () -> {
        cuentaRepository.findById(2L).orElseThrow();
    });
    assertEquals(1, cuentaRepository.findAll().size());
}
```

---

## 6. Spring Boot - Testing de Controllers con MockMvc

**Modulo:** `springboot_test_web_controllers`

### @WebMvcTest

Prueba la capa web de manera aislada, sin levantar un servidor HTTP real:

```java
@WebMvcTest(CuentaController.class)
class CuentaControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private CuentaService cuentaService;

    @Autowired
    private ObjectMapper objectMapper;
}
```

**Caracteristicas de @WebMvcTest:**
- Solo carga el controller especificado y sus dependencias web
- No carga servicios, repositorios ni base de datos
- Requiere `@MockBean` para las dependencias del controller
- Usa `MockMvc` para simular requests HTTP

### Test de Endpoint GET

```java
@Test
void testDetalle() throws Exception {
    // Arrange
    when(cuentaService.findById(1L)).thenReturn(Datos.crearCuenta001().orElseThrow());

    // Act & Assert
    mvc.perform(get("/api/cuentas/1").contentType(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andExpect(content().contentType(MediaType.APPLICATION_JSON))
        .andExpect(jsonPath("$.persona").value("Oscar"))
        .andExpect(jsonPath("$.saldo").value("1000"));

    verify(cuentaService).findById(1L);
}
```

### Test de Endpoint POST

```java
@Test
void testTransferir() throws Exception {
    // Arrange
    TransaccionDto dto = new TransaccionDto();
    dto.setCuentaOrigenId(1L);
    dto.setCuentaDestinoId(2L);
    dto.setMonto(new BigDecimal("100"));
    dto.setBancoId(1L);

    // Act & Assert
    mvc.perform(post("/api/cuentas/transferir")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(dto)))
        .andExpect(status().isOk())
        .andExpect(content().contentType(MediaType.APPLICATION_JSON))
        .andExpect(jsonPath("$.mensaje").value("Transferencia realizada con exito"));

    verify(cuentaService).transferir(1L, 2L, new BigDecimal("100"), 1L);
}
```

### Test de Endpoint GET - Lista

```java
@Test
void testListar() throws Exception {
    List<Cuenta> cuentas = Arrays.asList(
        Datos.crearCuenta001().orElseThrow(),
        Datos.crearCuenta002().orElseThrow()
    );
    when(cuentaService.findAll()).thenReturn(cuentas);

    mvc.perform(get("/api/cuentas").contentType(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andExpect(content().contentType(MediaType.APPLICATION_JSON))
        .andExpect(jsonPath("$[0].persona").value("Oscar"))
        .andExpect(jsonPath("$[1].persona").value("Jhon"))
        .andExpect(jsonPath("$", hasSize(2)));
}
```

### JSONPath - Expresiones Comunes

| Expresion | Descripcion |
|-----------|-------------|
| `$.campo` | Campo del objeto raiz |
| `$[0]` | Primer elemento del array |
| `$[0].campo` | Campo del primer elemento |
| `$.length()` | Tamanio del array |
| `$` | Objeto/array raiz |

---

## 7. Spring Boot - Integration Testing con TestRestTemplate

**Modulo:** `springboot_test_TestRestTemplate`

### Configuracion

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CuentaControllerTestRestTemplateTest {

    @Autowired
    private TestRestTemplate client;

    @Autowired
    private ObjectMapper objectMapper;

    @LocalServerPort
    private int puerto;
}
```

**Diferencia clave con MockMvc:**
- `TestRestTemplate` levanta un servidor HTTP **real** en un puerto aleatorio
- Las requests viajan por la red (localhost), no se simulan
- Se prueban todas las capas: controller, service, repository, base de datos
- Es un test de **integracion completa**

### Test de GET con TestRestTemplate

```java
@Test
@Order(1)
void testDetalle() {
    ResponseEntity<Cuenta> respuesta = client.getForEntity("/api/cuentas/1", Cuenta.class);

    assertEquals(HttpStatus.OK, respuesta.getStatusCode());
    assertEquals(MediaType.APPLICATION_JSON, respuesta.getHeaders().getContentType());

    Cuenta cuenta = respuesta.getBody();
    assertNotNull(cuenta);
    assertEquals("Oscar", cuenta.getPersona());
    assertEquals("1000.00", cuenta.getSaldo().toPlainString());
}
```

### Test de POST con TestRestTemplate

```java
@Test
@Order(2)
void testTransferir() throws JsonProcessingException {
    TransaccionDto dto = new TransaccionDto();
    dto.setCuentaOrigenId(1L);
    dto.setCuentaDestinoId(2L);
    dto.setMonto(new BigDecimal("100"));
    dto.setBancoId(1L);

    ResponseEntity<String> respuesta = client.postForEntity("/api/cuentas/transferir", dto, String.class);

    assertEquals(HttpStatus.OK, respuesta.getStatusCode());
    assertEquals(MediaType.APPLICATION_JSON, respuesta.getHeaders().getContentType());

    // Usar JsonNode para parseo flexible
    JsonNode json = objectMapper.readTree(respuesta.getBody());
    assertEquals("Transferencia realizada con exito", json.path("mensaje").asText());
}
```

### Test de DELETE con TestRestTemplate

```java
@Test
@Order(5)
void testEliminar() {
    ResponseEntity<Void> respuesta = client.exchange(
        "/api/cuentas/3",
        HttpMethod.DELETE,
        null,
        Void.class
    );
    assertEquals(HttpStatus.NO_CONTENT, respuesta.getStatusCode());
    assertFalse(respuesta.hasBody());
}
```

### Tests Ordenados y con Estado

Con `@TestMethodOrder(MethodOrderer.OrderAnnotation.class)` y `@Order(n)`, los tests se ejecutan en secuencia y comparten estado de la base de datos. Esto permite probar flujos completos:

1. `@Order(1)` - Consultar saldo inicial
2. `@Order(2)` - Realizar transferencia
3. `@Order(3)` - Verificar saldo actualizado
4. `@Order(4)` - Crear nueva cuenta
5. `@Order(5)` - Eliminar cuenta

---

## 8. Spring Boot - Integration Testing con WebTestClient

**Modulo:** `springboot_test_WebTestClient`

### Configuracion

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CuentaControllerWebTestClientTests {

    @Autowired
    private WebTestClient client;

    @Autowired
    private ObjectMapper objectMapper;
}
```

**WebTestClient** es el cliente HTTP reactivo de Spring WebFlux. Ofrece una API fluida y encadenable para testing.

**Dependencia requerida:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <scope>test</scope>
</dependency>
```

### Test de GET con WebTestClient

```java
@Test
@Order(1)
void testDetalle() {
    client.get().uri("/api/cuentas/1")
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.persona").isEqualTo("Oscar")
        .jsonPath("$.saldo").isEqualTo(1000);
}
```

### Test de POST con WebTestClient

```java
@Test
@Order(2)
void testTransferir() {
    TransaccionDto dto = new TransaccionDto();
    dto.setCuentaOrigenId(1L);
    dto.setCuentaDestinoId(2L);
    dto.setMonto(new BigDecimal("100"));
    dto.setBancoId(1L);

    client.post().uri("/api/cuentas/transferir")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(dto)
        .exchange()
        .expectStatus().isOk()
        .expectBody()
        .jsonPath("$.mensaje").isNotEmpty()
        .jsonPath("$.mensaje").value(is("Transferencia realizada con exito"))
        .jsonPath("$.transaccion.cuentaOrigenId").isEqualTo(1);
}
```

### Test de GET Lista con WebTestClient

```java
@Test
@Order(4)
void testListar() {
    client.get().uri("/api/cuentas")
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentType(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$[0].persona").isEqualTo("Oscar")
        .jsonPath("$[0].saldo").isEqualTo(900)
        .jsonPath("$").value(hasSize(2));
}

// Alternativa: parsear a lista de objetos
@Test
@Order(4)
void testListar2() {
    client.get().uri("/api/cuentas")
        .exchange()
        .expectStatus().isOk()
        .expectBodyList(Cuenta.class)
        .hasSize(2);
}
```

### consumeWith - Procesamiento Custom

```java
@Test
void testDetalle2() {
    client.get().uri("/api/cuentas/1")
        .exchange()
        .expectStatus().isOk()
        .expectBody(Cuenta.class)
        .consumeWith(response -> {
            Cuenta cuenta = response.getResponseBody();
            assertNotNull(cuenta);
            assertEquals("Oscar", cuenta.getPersona());
        });
}
```

### Comparativa TestRestTemplate vs WebTestClient

| Aspecto | TestRestTemplate | WebTestClient |
|---------|------------------|---------------|
| Paradigma | Sincrono/bloqueante | Reactivo/no-bloqueante |
| API | Metodos imperativos | Fluent/encadenable |
| Respuesta | `ResponseEntity<T>` | Assertions encadenadas |
| JSON | Manual con `JsonNode` | `jsonPath()` integrado |
| Listas | `getForEntity` + cast | `expectBodyList(T.class)` |
| Ideal para | APIs REST tradicionales | APIs modernas y reactivas |

---

## 9. Patrones y Buenas Practicas

### Patron AAA (Arrange-Act-Assert)

Todos los tests del curso siguen esta estructura:

```java
@Test
void testTransferir() {
    // Arrange (Given) - Preparar datos y mocks
    when(repository.findById(1L)).thenReturn(cuenta);

    // Act (When) - Ejecutar la accion
    service.transferir(1L, 2L, monto, bancoId);

    // Assert (Then) - Verificar resultados
    assertEquals(expected, actual);
    verify(repository).save(any());
}
```

### Clase de Datos para Tests

Centralizar datos de prueba en una clase `Datos`:

```java
public class Datos {
    public static Optional<Cuenta> crearCuenta001() {
        return Optional.of(new Cuenta(1L, "Oscar", new BigDecimal("1000")));
    }

    public static Optional<Cuenta> crearCuenta002() {
        return Optional.of(new Cuenta(2L, "Jhon", new BigDecimal("2000")));
    }

    public static Optional<Banco> crearBanco() {
        return Optional.of(new Banco(1L, "Banco Nacional", 0));
    }
}
```

### Piramide de Testing Aplicada

```
        /  E2E  \         <- TestRestTemplate / WebTestClient (pocos)
       / Integra \        <- @SpringBootTest + @MockBean (moderados)
      /  Unitarios \      <- JUnit 5 + Mockito (muchos)
     /______________\
```

| Nivel | Anotacion | Que prueba |
|-------|-----------|------------|
| Unitario | `@ExtendWith(MockitoExtension.class)` | Logica de negocio aislada |
| Integracion parcial | `@DataJpaTest` | Capa de datos con H2 |
| Integracion parcial | `@WebMvcTest` | Capa web sin servidor |
| Integracion completa | `@SpringBootTest(RANDOM_PORT)` | Toda la aplicacion |

---

## 10. Comparativa de Frameworks

| Framework | Modulo | Foco | Herramientas Clave |
|-----------|--------|------|--------------------|
| **JUnit 5** | junit5_app | Pruebas unitarias | Assertions, lifecycle, parametrizacion, condiciones |
| **Mockito** | app-mockito | Objetos simulados | Mocking, stubbing, verificacion |
| **Mockito Avanzado** | app-mockito 2 | Mocking avanzado | Spies, argument matchers, orden, conteo |
| **Spring Boot Service** | springboot_test_services_test | Capa de servicio | @SpringBootTest, @MockBean |
| **JPA Testing** | springboot_test_data_jpa | Capa de datos | @DataJpaTest, H2, repositorios |
| **MockMvc** | springboot_test_web_controllers | Capa web aislada | MockMvc, jsonPath, status assertions |
| **TestRestTemplate** | springboot_test_TestRestTemplate | Integracion HTTP sincrona | TestRestTemplate, ResponseEntity, JsonNode |
| **WebTestClient** | springboot_test_WebTestClient | Integracion HTTP reactiva | WebTestClient, API fluent, expectBody |

---

## Estructura del Proyecto

```
java-testing-course/
├── junit5_app/                          # JUnit 5 fundamentals
├── app-mockito/                         # Mockito basics
├── app-mockito 2/                       # Mockito advanced
├── springboot_test_services_test/       # Service layer testing
├── springboot_test_data_jpa/            # JPA repository testing
├── springboot_test_web_controllers/     # MockMvc controller testing
├── springboot_test_TestRestTemplate/    # TestRestTemplate integration
└── springboot_test_WebTestClient/       # WebTestClient integration
```

---

## Dependencias Principales

```xml
<!-- JUnit 5 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.6.3</version>
    <scope>test</scope>
</dependency>

<!-- Mockito -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>3.6.28</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>3.6.28</version>
    <scope>test</scope>
</dependency>

<!-- Spring Boot Test (incluye JUnit 5, Mockito, AssertJ, Hamcrest) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- WebTestClient -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <scope>test</scope>
</dependency>

<!-- H2 (base de datos en memoria para tests) -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

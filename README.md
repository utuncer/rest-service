# 🌱 Spring REST Service — Detaylı Rehber

> **Kaynak:** https://spring.io/guides/gs/rest-service  
> **Amaç:** Spring Boot ile sıfırdan bir RESTful Web Service oluşturmak.

---

## 📌 Ne İnşa Ediyoruz?

`GET http://localhost:8080/greeting` adresine gelen isteklere JSON dönen bir servis:

```json
{"id": 1, "content": "Hello, World!"}
```

İsteğe bağlı `name` parametresiyle özelleştirilebilir:

```
http://localhost:8080/greeting?name=Ahmet
→ {"id": 2, "content": "Hello, Ahmet!"}
```

---

## 🗂️ Proje Yapısı

```
src/main/java/com/example/restservice/
├── RestServiceApplication.java   ← Uygulama giriş noktası
├── Greeting.java                 ← Veri modeli (Record)
└── GreetingController.java       ← HTTP isteklerini karşılayan controller
```

---

## 1️⃣ Veri Modeli — `Greeting` Record

```java
package com.example.restservice;

public record Greeting(long id, String content) { }
```

### 🔍 Neden `record` kullanıldı?

| Özellik | Açıklama |
|---|---|
| **Kısa sözdizimi** | `record` ile field, constructor, getter, `equals`, `hashCode`, `toString` otomatik üretilir. Tek satırda bir veri sınıfı tanımlanır. |
| **Immutability (Değişmezlik)** | Record'lar doğası gereği `final`'dır; bir kez yaratılan `Greeting` nesnesi değiştirilemez. REST API'lerde bu tercih edilen bir pratiktir. |
| **JSON serialization** | Jackson kütüphanesi, record'un getter'larını (ör. `id()`, `content()`) okuyarak nesneyi otomatik olarak JSON'a çevirir. |

> **Jackson nedir?**  
> Spring Web Starter ile birlikte gelen JSON işleme kütüphanesidir. `spring-boot-starter-web` bağımlılığı eklenince otomatik olarak classpath'e girer ve `JacksonJsonHttpMessageConverter` devreye girerek Java nesnelerini JSON'a (ve tersine) dönüştürür. Elle `ObjectMapper` yazmaya gerek yoktur.

---

## 2️⃣ Controller — `GreetingController`

```java
package com.example.restservice;

import java.util.concurrent.atomic.AtomicLong;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController                                          // (1)
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong(); // (2)

    @GetMapping("/greeting")                             // (3)
    public Greeting greeting(
        @RequestParam(defaultValue = "World") String name // (4)
    ) {
        return new Greeting(counter.incrementAndGet(), template.formatted(name)); // (5)
    }
}
```

### 🔍 Her yapının açıklaması

---

#### `(1) @RestController`

```java
@RestController
```

**Ne yapar?**  
`@Controller` + `@ResponseBody` anotasyonlarının birleşimidir.

- `@Controller` → Spring'e bu sınıfın HTTP isteklerini karşılayan bir bileşen olduğunu söyler.
- `@ResponseBody` → Metodun döndürdüğü nesneyi, bir view (HTML template) ile render etmek yerine **doğrudan HTTP response body'sine** yazar.

**Neden kullanıldı?**  
Geleneksel MVC controller'larda bir Thymeleaf/JSP view dönerdiniz. REST API'lerde ise istemciye (mobil uygulama, başka bir servis vb.) JSON dönmek istiyoruz. `@RestController` bunu otomatik sağlar.

---

#### `(2) AtomicLong counter`

```java
private final AtomicLong counter = new AtomicLong();
```

**Ne yapar?**  
Thread-safe (çok iş parçacıklı ortamda güvenli) bir sayaç sağlar.

**Neden `AtomicLong`, sıradan `long` değil?**  
Web sunucusu birden fazla isteği aynı anda (paralel olarak) işleyebilir. Eğer iki istek aynı anda `counter++` yapsa, klasik `long`'da **race condition** oluşur ve aynı ID iki kez üretilebilir. `AtomicLong.incrementAndGet()` ise bu işlemi **atomik** (bölünmez) yapar; her isteğe benzersiz bir ID garantilenir.

---

#### `(3) @GetMapping("/greeting")`

```java
@GetMapping("/greeting")
```

**Ne yapar?**  
`HTTP GET /greeting` isteklerini bu metoda yönlendirir.

**Neden `@GetMapping`?**  
`@RequestMapping(method = RequestMethod.GET)` ifadesinin kısa yazımıdır. Diğer HTTP metodları için eşdeğerleri:

| Anotasyon | HTTP Metodu | Tipik Kullanım |
|---|---|---|
| `@GetMapping` | GET | Veri okuma |
| `@PostMapping` | POST | Yeni kayıt oluşturma |
| `@PutMapping` | PUT | Kayıt güncelleme |
| `@DeleteMapping` | DELETE | Kayıt silme |
| `@PatchMapping` | PATCH | Kısmi güncelleme |

---

#### `(4) @RequestParam(defaultValue = "World")`

```java
@RequestParam(defaultValue = "World") String name
```

**Ne yapar?**  
URL'deki query string parametresini (`?name=Ahmet`) metod parametresine bağlar.

- `?name=Ahmet` → `name = "Ahmet"`
- Parametre yoksa → `name = "World"` (defaultValue devreye girer)

**Neden kullanıldı?**  
Kullanıcıya isteğe bağlı özelleştirme imkânı sunarken, parametrenin gönderilmediği durumda uygulamanın çökmemesini sağlar. `required = false` demeye gerek kalmaz; `defaultValue` zaten parametreyi opsiyonel yapar.

---

#### `(5) counter.incrementAndGet()`

```java
return new Greeting(counter.incrementAndGet(), template.formatted(name));
```

**Ne yapar?**  
Sayacı 1 artırır ve yeni değeri döndürür (önce artır, sonra oku). Ardından `template.formatted(name)` ile `"Hello, %s!"` şablonundaki `%s` yerine `name` değeri yerleştirilir.

**Dönüş değeri neden `Greeting` nesnesi?**  
`@RestController` sayesinde Spring, döndürülen `Greeting` nesnesini Jackson aracılığıyla otomatik olarak JSON'a serialize eder. `{"id": 1, "content": "Hello, World!"}` çıktısı bu şekilde üretilir.

---

## 3️⃣ Uygulama Giriş Noktası — `RestServiceApplication`

```java
package com.example.restservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication   // (1)
public class RestServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(RestServiceApplication.class, args); // (2)
    }
}
```

### 🔍 Her yapının açıklaması

---

#### `(1) @SpringBootApplication`

Aslında 3 anotasyonun birleşimidir:

| Alt Anotasyon | Ne Yapar? | Neden Gerekli? |
|---|---|---|
| `@Configuration` | Bu sınıfı Spring'in bean kaynağı olarak işaretler | Spring context'ine config sağlar |
| `@EnableAutoConfiguration` | Classpath'e bakarak otomatik bean konfigürasyonu yapar | `spring-webmvc` varsa `DispatcherServlet` kurar, vb. |
| `@ComponentScan` | `com.example` paketi altındaki tüm `@Controller`, `@Service`, `@Repository` vb. bileşenleri tarar | `GreetingController`'ı bulup Spring'e kaydeder |

**Neden `@SpringBootApplication` tek başına yeterli?**  
Spring Boot'un "opinionated" (iyi pratikleri varsayılan alan) yapısı sayesinde, XML config veya `web.xml` yazmadan tam bir web uygulaması ayağa kalkar.

---

#### `(2) SpringApplication.run()`

```java
SpringApplication.run(RestServiceApplication.class, args);
```

**Ne yapar?**
1. Spring Application Context'i başlatır.
2. Gömülü bir Tomcat sunucusu başlatır (port: 8080).
3. `@ComponentScan` ile bulunan tüm bileşenleri context'e kaydeder.
4. `@EnableAutoConfiguration` ile gerekli bean'leri otomatik oluşturur.

**Neden gömülü Tomcat?**  
Geleneksel Java web uygulamalarında Tomcat'i ayrıca kurmanız, WAR dosyası dağıtmanız gerekirdi. Spring Boot, Tomcat'i JAR içine gömer; `java -jar uygulama.jar` komutuyla tüm uygulama çalışır.

---

## 🔄 İstek Akışı (End-to-End)

```
İstemci
  │
  │  GET /greeting?name=Ahmet
  ▼
DispatcherServlet          ← Spring MVC'nin merkezi yönlendirici
  │
  │  @GetMapping("/greeting") eşleşir
  ▼
GreetingController.greeting("Ahmet")
  │
  │  new Greeting(2, "Hello, Ahmet!")
  ▼
Jackson (JacksonJsonHttpMessageConverter)
  │
  │  {"id": 2, "content": "Hello, Ahmet!"}
  ▼
HTTP Response (200 OK, Content-Type: application/json)
  │
  ▼
İstemci
```

---

## 🛠️ Uygulamayı Çalıştırma

**Maven ile:**
```bash
./mvnw spring-boot:run
```

**Gradle ile:**
```bash
./gradlew bootRun
```

**Test:**
```bash
curl http://localhost:8080/greeting
# {"id":1,"content":"Hello, World!"}

curl http://localhost:8080/greeting?name=Ahmet
# {"id":2,"content":"Hello, Ahmet!"}
```

---

## 📦 Bağımlılık — `pom.xml` (Maven)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Bu tek bağımlılık ile birlikte gelenlerin tam listesi:

| Kütüphane | Görevi |
|---|---|
| Spring MVC | HTTP istek yönlendirme, controller desteği |
| Jackson Databind | Java ↔ JSON dönüşümü |
| Embedded Tomcat | Gömülü web sunucusu |
| Spring Boot AutoConfigure | Otomatik konfigürasyon |

---

## 📝 Özet — Kullanılan Yapılar

| Yapı | Paket/Sınıf | Görevi |
|---|---|---|
| `record` | Java 16+ | Immutable veri taşıyıcı; boilerplate'i ortadan kaldırır |
| `@RestController` | Spring MVC | Controller + JSON yanıt dönüşünü birleştirir |
| `@GetMapping` | Spring MVC | HTTP GET isteğini metoda bağlar |
| `@RequestParam` | Spring MVC | URL query param'ı metod parametresine inject eder |
| `AtomicLong` | java.util.concurrent | Thread-safe sayaç |
| `@SpringBootApplication` | Spring Boot | Otomatik config + component scan başlatır |
| `Jackson` | FasterXML | Nesneyi JSON'a serialize eder (otomatik) |
| `SpringApplication.run()` | Spring Boot | Tüm context'i ve Tomcat'i başlatır |

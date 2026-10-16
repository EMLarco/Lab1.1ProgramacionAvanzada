# Bitácora - Laboratorio 1.1

**Aplicación básica con Spring Boot, endpoints REST, Actuator, perfiles y DevTools**

| Campo | Valor |
|---|---|
| Estudiante | Esteban Larco |
| Correo institucional | emlarco@espe.edu.ec|
| Asignatura | Metodologías de Programación / Desarrollo de Software |
| Docente | Ing. John Javier Cruz Garzón, Mg. |
| Periodo lectivo | 202651 |
| Nivel | 7 |
| Práctica | 1.1 |
| Fecha de ejecución | 16/10/2026 |
| Entorno | Windows 11 · Java 21 (compilación) / Java 25 (runtime) · Spring Boot 4.1.1 · IntelliJ IDEA · Maven Wrapper |
| Proyecto | `com.espe.lab` (Artifact: `lab`) |
| Repositorio | https://github.com/EMLarco/Lab1.1ProgramacionAvanzada |

---

## 1. Resumen de la práctica

Se creó una aplicación Maven con Java 21 usando Spring Boot, integrando las dependencias **Spring Web**, **Spring Boot Actuator** y **Spring Boot DevTools**. Se implementó un endpoint REST que devuelve un JSON con un mensaje y el perfil activo, se gestionaron perfiles de configuración (`default` y `dev`) para cambiar el puerto y el comportamiento de la aplicación, se verificó el estado con Actuator y se comprobó el reinicio automático con DevTools durante el desarrollo.

---

## 2. Configuración aplicada

### `application.properties` (perfil por defecto)

```properties
spring.application.name=lab
server.port=8080
app.mensaje=Aplicación Spring Boot en funcionamiento - ESPE
info.app.perfil=default
management.endpoints.web.exposure.include=health,info
management.info.env.enabled=true
```

### `application-dev.properties` (perfil dev)

```properties
server.port=8081
app.mensaje=Aplicación Spring Boot en entorno de desarrollo - ESPE
info.app.perfil=dev
```

### `SaludoController.java`

```java
package com.espe.lab.controller;

import java.util.LinkedHashMap;
import java.util.Map;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class SaludoController {

    @Value("${app.mensaje:Aplicación Spring Boot en funcionamiento}")
    private String mensaje;

    @Value("${info.app.perfil:default}")
    private String perfil;

    @GetMapping("/saludo")
    public Map<String, String> obtenerSaludo() {
        Map<String, String> respuesta = new LinkedHashMap<>();
        respuesta.put("mensaje", mensaje);
        respuesta.put("perfil", perfil);
        return respuesta;
    }
}
```

**Nota:** La propiedad `management.info.env.enabled=true` es obligatoria en Spring Boot 3+ para que las propiedades `info.*` aparezcan en `/actuator/info`.

---

## 3. Evidencias de ejecución

### 3.1 Perfil `default` (puerto 8080)

| Endpoint | Respuesta |
|---|---|
| `GET /api/saludo` | `{"mensaje":"Aplicación Spring Boot en funcionamiento - ESPE","perfil":"default"}` |
| `GET /actuator/health` | `{"status":"UP"}` |
| `GET /actuator/info` | `{"app":{"perfil":"default"}}` |

Línea del log de arranque:

```text
Tomcat started on port 8080 (http) with context path '/'
Started LabApplication in 2.431 seconds
```

**Observación sobre versiones:** El `pom.xml` fija `release 21` para la compilación (confirmado en la línea `Compiling 2 source files with javac [debug parameters release 21]`), pero el runtime del sistema operativo apunta a Java 25. La aplicación compila y ejecuta correctamente en ambos casos. Para entornos productivos se recomienda alinear `JAVA_HOME` con la versión objetivo del proyecto (Java 21).

### 3.2 Perfil `dev` (puerto 8081)

| Endpoint | Respuesta |
|---|---|
| `GET /api/saludo` | `{"mensaje":"Aplicación Spring Boot en entorno de desarrollo - ESPE","perfil":"dev"}` |
| `GET /actuator/health` | `{"status":"UP"}` |

Líneas del log de arranque:

```text
The following 1 profile is active: "dev"
Tomcat started on port 8081 (http) with context path '/'
Started LabApplication in 2.465 seconds
```

### 3.3 Comparativa de perfiles

| Perfil | Puerto | Mensaje | Fuente de configuración |
|---|---|---|---|
| `default` | 8080 | "Aplicación Spring Boot en funcionamiento - ESPE" | `application.properties` |
| `dev` | 8081 | "Aplicación Spring Boot en entorno de desarrollo - ESPE" | `application-dev.properties` |

**Conclusión parcial:** las propiedades del perfil `dev` sobrescriben las de `application.properties`. Las que no se redefinen (por ejemplo, `management.endpoints.web.exposure.include`) se heredan del perfil por defecto.

---

## 4. Incidencia 1 — Puerto 8080 ya en uso

- **Síntoma observado:** con una instancia de la aplicación en ejecución en el puerto 8080 (Terminal 1), al lanzar una segunda instancia con `.\mvnw.cmd spring-boot:run` en otra terminal (Terminal 2), la aplicación no arranca y el proceso termina con error.

- **Mensaje de error:**

```text
APPLICATION FAILED TO START

Description:

Web server failed to start. Port 8080 was already in use.

Action:

Identify and stop the process that's listening on port 8080 or configure this
application to listen on another port.
```

- **Causa identificada:** existía ya un proceso Tomcat embebido escuchando en el puerto 8080 (primera instancia de `LabApplication`). Spring Boot no permite que dos servidores embebidos usen el mismo puerto en el mismo host.

- **Solución aplicada:** se ejecutó la segunda instancia con el perfil `dev` mediante `.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=dev"`, lo que hace que use el puerto 8081 y no genere conflicto. Alternativa equivalente: detener la primera instancia con `Ctrl+C`.

- **Verificación posterior:**
    - `curl.exe http://localhost:8080/api/saludo` → `{"mensaje":"Aplicación Spring Boot en funcionamiento - ESPE","perfil":"default"}`
    - `curl.exe http://localhost:8081/api/saludo` → `{"mensaje":"Aplicación Spring Boot en entorno de desarrollo - ESPE","perfil":"dev"}`
    - Ambas instancias convivieron sin conflicto tras aplicar la solución.

---

## 5. Incidencia 2 — Error de compilación por import faltante de `@Value`

- **Síntoma observado:** el proyecto no compilaba; el IDE marcaba errores en `SaludoController.java` al copiar el código de ejemplo de la guía.

- **Mensaje de error:** `cannot find symbol: class Value` / `cannot find symbol: method value()`.

- **Causa identificada:** el código de ejemplo de la guía usa la anotación `@Value` sin importar `org.springframework.beans.factory.annotation.Value`; además incluía un `import org.springframework.beans.factory.annotation.Autowired;` que no se utilizaba.

- **Solución aplicada:** agregar `import org.springframework.beans.factory.annotation.Value;` y eliminar el import de `Autowired`.

- **Verificación posterior:** `.\mvnw.cmd clean package -DskipTests` → `BUILD SUCCESS` y el endpoint `/api/saludo` responde el mensaje leído desde `application.properties`.

---

## 6. Registro del reinicio automático con DevTools

| Momento | Acción | Evidencia en el log | Respuesta de `/api/saludo` |
|---|---|---|---|
| Antes | `app.mensaje=Aplicación Spring Boot en funcionamiento - ESPE` | `Started LabApplication in 1.865 seconds` | `{"mensaje":"Aplicación Spring Boot en funcionamiento - ESPE","perfil":"default"}` |
| Cambio | Se editó el valor de `app.mensaje` en `application.properties` y se ejecutó **Build Project** (Ctrl+F9) | — | — |
| Después | DevTools reinició el contexto automáticamente | `Restarting due to 1 class path change (0 additions, 0 deletions, 1 modification)` + `Started LabApplication in X.XXX seconds` ejecutado por el hilo `restartedMain` | `{"mensaje":"Aplicación Spring Boot RECARGADA con DevTools - ESPE","perfil":"default"}` |

**Observaciones:**

- El reinicio se ejecutó en el hilo `restartedMain` **sin detener** la aplicación.
- No fue necesario reiniciar manualmente el proceso.
- DevTools se desactiva automáticamente cuando la aplicación se ejecuta como jar empaquetado (`java -jar`), porque se considera un entorno de producción.

---

## 7. Uso de inteligencia artificial

- **Herramienta utilizada:** [Deepseek]
- **Consultas realizadas:**
    1. Corrección del `import` faltante de `@Value` en `SaludoController` (el código de la guía no incluía `import org.springframework.beans.factory.annotation.Value;`).
    2. Verificación de la propiedad `management.info.env.enabled=true` para publicar `info.*` en `/actuator/info` en Spring Boot 3+.
    3. Apoyo en la redacción de esta bitácora.
- **Verificación realizada:** todas las sugerencias se contrastaron con la documentación oficial de Spring Boot (Actuator → Info endpoint) y se confirmaron con la ejecución real:
    - `curl.exe http://localhost:8080/actuator/info` → `{"app":{"perfil":"default"}}`
    - `curl.exe http://localhost:8081/api/saludo` → `{"mensaje":"Aplicación Spring Boot en entorno de desarrollo - ESPE","perfil":"dev"}`

---

## 8. Conclusiones técnicas

1. Spring Boot permite crear un servicio REST funcional con muy poca configuración, usando Spring Initializr y Maven Wrapper para garantizar versiones consistentes de Maven y del framework.
2. La externalización de la configuración (`application.properties` + `application-{perfil}.properties`) permite cambiar el puerto, los mensajes y el comportamiento de la aplicación sin modificar el código fuente.
3. Spring Boot Actuator expone el estado operativo de la aplicación; en Spring Boot 3+ es necesario `management.info.env.enabled=true` para publicar las propiedades `info.*` en `/actuator/info`.
4. Se respetó el principio de responsabilidad única: `SaludoController` solo se encarga de exponer el endpoint; toda la lógica de configuración queda en archivos `.properties`.
5. Spring Boot DevTools reduce el ciclo de retroalimentación al reiniciar automáticamente el contexto ante cambios en el classpath; debe usarse solo durante el desarrollo.
6. Documentar incidencias con el formato **síntoma → mensaje → causa → solución → verificación** facilita el diagnóstico y evita repetir errores en el futuro.

---

## 9. Recomendaciones

- Exponer solo los endpoints de Actuator necesarios (`health,info`), nunca todos con `*`.
- Mantener toda la configuración variable (puertos, mensajes, perfiles) en archivos `.properties` y no en el código fuente.
- Verificar siempre con `curl` o el navegador el resultado real de cada cambio, en lugar de asumir que la configuración fue aplicada.
- Excluir la carpeta `target/` y los archivos del IDE mediante `.gitignore` y ejecutar `.\mvnw.cmd clean` antes de comprimir el proyecto.
- Usar DevTools únicamente en desarrollo; se desactiva automáticamente con el jar empaquetado (`java -jar`).
- Verificar las recomendaciones de herramientas de IA con la documentación oficial de Spring Boot y con la ejecución real antes de aplicarlas.
- Alinear `JAVA_HOME` con la versión objetivo del proyecto (Java 21) para evitar diferencias entre el entorno de compilación y el de ejecución.

---

## 10. Anexos — Lista de evidencias

| # | Evidencia | Fase |
|---|---|---|
| 1 | Salida de `java -version` mostrando la versión del JDK | Fase 1 |
| 2 | Salida de `.\mvnw.cmd clean package -DskipTests` → `BUILD SUCCESS` | Fase 1 |
| 3 | Log de arranque con `Tomcat started on port 8080` + `Started LabApplication` | Fase 3 |
| 4 | Respuesta de `/api/saludo` en 8080 | Fase 3 |
| 5 | Respuesta de `/actuator/health` en 8080 | Fase 3 |
| 6 | Respuesta de `/actuator/info` en 8080 | Fase 3 |
| 7 | Log con `Restarting due to 1 class path change` (DevTools) | Fase 4 |
| 8 | Respuesta de `/api/saludo` con el mensaje recargado | Fase 4 |
| 9 | Log con `The following 1 profile is active: "dev"` + `port 8081` | Fase 5 |
| 10 | Respuesta de `/api/saludo` en 8081 (perfil dev) | Fase 5 |
| 11 | Captura de `Port 8080 was already in use` | Fase 6 |
| 12 | Verificación posterior (ambos puertos responden) | Fase 6 |
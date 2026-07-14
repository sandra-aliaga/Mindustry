# Informe de Arqueología de Código: Sprint 3 y Sprint 4 (Mindustry)

Repositorio analizado: `sandra-aliaga/Mindustry`  
Método aplicado: reconstrucción retrospectiva (“método del cangrejo”) a partir de historial Git, estructura del código y flujos CI/CD.

---

## 1. Objetivo y enfoque de investigación

Este documento reconstruye decisiones de ingeniería y evolución técnica posteriores a Sprint 2, con foco en:

1. Introducción y consolidación del multijugador online (servidor headless + stack de red).
2. Implementación y maduración del sistema de mods (incluyendo scripting JavaScript sobre JVM con Rhino).
3. Expansión del ecosistema hacia integraciones externas (Steam, Discord, iOS).
4. Estado real de prácticas DevOps en CI/CD para calidad, cobertura y despliegue.

La reconstrucción se basa en evidencia verificable:

- `git log` temático por palabras clave y rutas.
- `git show <hash>` para análisis commit por commit.
- Inspección de clases y paquetes por commit.
- Extracción de workflows en `.github/workflows/`.
- “Capturas” de estructura de directorios mediante `git ls-tree`, `find`, `git worktree`.

---

## 2. Investigación A: Multijugador Online (Servidor Headless y Red)

## 2.1 Contexto histórico: nacimiento de la red y del servidor dedicado

El soporte de red aparece en una secuencia muy concentrada entre fines de 2017 e inicios de 2018:

1. Se crea una API de red mínima (`Net`) basada en providers cliente/servidor.
2. Se conecta la implementación real con Kryonet.
3. Se desacopla el ciclo de vida de conexión en módulos (`NetClient`, `NetServer`).
4. Se habilita ejecución headless con módulo `server` dedicado.

Luego, ya en la etapa de consolidación arquitectónica (2019), se observa:

1. Migración hacia infraestructura de red del ecosistema Arc (`ArcNet`).
2. Refactor posterior para reducir dependencia directa de `arcnet` y reacomodar serializer/transporte.
3. Evolución hasta la arquitectura actual en v8, con `ArcNetProvider` en `core/src/mindustry/net`.

## 2.2 Commits clave y análisis detallado de cambios

### Commit `f6e9710b33f17bbc7ff843f4bb6b1f462eb64553`

- Autor: Anuken
- Fecha: 2017-12-30 11:43:47 -0500
- Mensaje: `Added basic Kryonet support`

Qué cambió exactamente:

1. Se introduce `core/src/io/anuke/mindustry/Net.java` con API de red base.
2. Se define modelo de providers (`ClientProvider`, `ServerProvider`) y dispatch de listeners por tipo de paquete.
3. Se agrega interfaz para registrar clases serializables y envío genérico de objetos.

Fragmento clave:

```java
public static interface ClientProvider {
    public void connect(String ip, String port) throws IOException;
    public void send(Object object);
    public void register(Class<?>... types);
}
```

Justificación arquitectónica:

- Este commit crea un “punto de inversión de control” para transporte de red.
- Permite que la lógica del juego dependa de una abstracción y no de una librería específica, facilitando migraciones posteriores (Kryonet → ArcNet → proveedor actual).

---

### Commit `e24179cd4ce058a27a10c33ea33c809032d9eacf`

- Autor: Anuken
- Fecha: 2017-12-30 12:28:17 -0500
- Mensaje: `Added full Kryonet server/client implementation`

Qué cambió exactamente:

1. Se crea `core/src/io/anuke/mindustry/net/Net.java` con modos TCP/UDP y listeners separados cliente/servidor.
2. Se agregan paquetes de control `Connect` y `Disconnect`.
3. `DesktopLauncher` pasa a instanciar `Client` y `Server` Kryonet reales.

Fragmento clave:

```java
public enum SendMode{
    tcp, udp
}
```

```java
// packets/Connect.java
public class Connect {
    public int id;
    public String addressTCP;
}
```

Justificación arquitectónica:

- Se pasa de una API declarativa a una implementación funcional de multiplayer.
- Introduce handshake y eventos de conexión como eventos de dominio (`Connect`, `Disconnect`) en vez de manejar callbacks de librería directamente en toda la base.

---

### Commit `4a2b2dee722bfd9f693c3db64e748e6bedf2190e`

- Autor: Anuken
- Fecha: 2017-12-30 19:20:20 -0500
- Mensaje: `Added NetClient/NetServer classes`

Qué cambió exactamente:

1. Se agregan `core/src/io/anuke/mindustry/core/NetClient.java` y `NetServer.java`.
2. Se integran como módulos del ciclo de vida del juego.
3. Se centraliza la lógica de mantener/cerrar sesiones según estado de juego.

Fragmento clave:

```java
public void update(){
    if(!Net.client()) return;
    if(!GameState.is(State.menu) && Net.active()){
    }else{
        Net.disconnect();
    }
}
```

Justificación arquitectónica:

- Se separa claramente transporte de red de orquestación de estado de sesión.
- Reduce acoplamiento con UI/launcher y favorece mantenibilidad.

---

### Commit `35b6b41f2402af993cb15d9d9f17f2c60dee426d`

- Autor: Anuken
- Fecha: 2018-01-27 23:42:42 -0500
- Mensaje: `Refactored almost every class, somehow didn't break game yet`

Qué cambió exactamente:

1. Nace submódulo `server` con `server/build.gradle`.
2. Se crea `server/src/io/anuke/mindustry/server/ServerLauncher.java`.
3. Se incorpora `server` en `settings.gradle`.
4. `ServerLauncher` levanta `HeadlessApplication` y configura providers de red para host dedicado.

Fragmento clave:

```java
Net.setClientProvider(new KryoClient());
Net.setServerProvider(new KryoServer());
new HeadlessApplication(new Mindustry());
```

Justificación arquitectónica:

- Habilita despliegue real de servidor sin frontend gráfico.
- Es un punto de inflexión para operación multiplayer productiva (hosting dedicado, scripts de servidor, administración remota).

---

### Commit `20eea3b3856684879d48c23b839bf5d17936305f`

- Autor: Anuken
- Fecha: 2018-01-01 16:09:17 -0500
- Mensaje: `Switched to different Kryonet fork; full Android support`

Qué cambió exactamente:

1. Cambios extensivos en `NetClient`, `NetServer`, `Packets`, `Registrator`.
2. Introducción/ajustes fuertes en serialización de estado (`NetworkIO`).
3. Registro explícito de clases de red y entidades sincronizables.

Fragmentos clave:

```java
public static Class<?>[] getClasses(){
    return new Class<?>[]{
        StreamBegin.class, StreamChunk.class, WorldData.class, SyncPacket.class, ...
    };
}
```

```java
stream.writeInt(fileVersionID);
stream.writeFloat(Timers.time());
stream.writeLong(TimeUtils.millis());
```

Justificación arquitectónica:

- El cambio atiende compatibilidad móvil y estabilidad del protocolo.
- Refuerza contrato de serialización binaria y sincronización de mundo.
- Sienta base de consistencia entre plataformas (desktop/android).

---

### Commit `01e1438382e1f128c2d1f656a1490040fa92fb88`

- Autor: Anuken
- Fecha: 2019-04-17 21:59:26 -0400
- Mensaje: `Switched to ArcNet networking extension`

Qué cambió exactamente:

1. Se agrega `arcModule("extensions:arcnet")` en build de core.
2. Se elimina dependencia Kryonet del módulo `net`.
3. Se crean `ArcNetClient.java`, `ArcNetServer.java`, `PacketSerializer.java`.
4. Se adapta lanzamiento de servidor para nueva capa.

Fragmento clave:

```gradle
compile arcModule("extensions:arcnet")
```

```java
client = new Client(8192, 4096, new PacketSerializer());
server = new Server(4096 * 2, 4096, new PacketSerializer());
```

Justificación arquitectónica:

- Migración orientada a homogeneizar stack con Arc y disminuir deuda de integración externa.
- Mejora control de comportamiento de red en conjunto con el framework propio.

---

### Commit `8b9be6eafe822ca4a39a0a48eb53ec651bc4fe8d`

- Autor: Anuken
- Fecha: 2019-08-23 18:18:14 -0400
- Mensaje: `Removed arcnet as dependency`

Qué cambió exactamente:

1. Se quita `extensions:arcnet` del build.
2. Se eliminan `ArcNetClient.java` y `ArcNetServer.java` del módulo `net`.
3. `PacketSerializer` deja de implementar `NetSerializer` y pasa a integrarse con `MSerializer` (`mnet`).
4. Se actualiza `settings.gradle` para remover módulo ArcNet extension.

Fragmento clave:

```java
public class PacketSerializer implements MSerializer{
```

Justificación arquitectónica:

- No es “retroceso”, sino realineamiento de módulos y responsabilidades.
- Indica una etapa de refactor para desacoplar serializer del transporte ArcNet concreto y consolidar protocolo sobre `mnet`.

---

## 2.3 Estado de arquitectura de red en la base actual (v8)

En `origin/master` se observa red unificada en `core/src/mindustry/net` con `ArcNetProvider` como implementación central.

Fragmento representativo:

```java
public class ArcNetProvider implements NetProvider{
    final Client client;
    final Server server;
}
```

También se mantiene `server/src/mindustry/server/ServerLauncher.java` con ejecución headless y pipeline de carga de mods/contenido antes de aceptar sesiones.

---

## 2.4 Evidencia de estructura de directorios por hitos de red

```text
[worktree e24179cd4c]
core/src/io/anuke/mindustry
core/src/io/anuke/mindustry/net
core/src/io/anuke/mindustry/core
```

```text
[worktree 01e1438382]
core/src/io/anuke/mindustry/net
core/src/io/anuke/mindustry/core
net/src/io/anuke/mindustry/net
server/src/io/anuke/mindustry/server
```

```text
[worktree origin/master]
core/src/mindustry/net
server/src/mindustry/server
```

---

## 3. Investigación B: Soporte para Mods (Rhino JS en JVM)

## 3.1 Contexto histórico

El sistema de extensibilidad madura en varias capas:

1. Carga de mods empaquetados (JAR/ZIP) con metadatos.
2. Integración de parser de contenido.
3. Capa de scripting JavaScript.
4. Migración a fork de Rhino para control fino de compatibilidad.

## 3.2 Commits clave y análisis

### Commit `70ab102d8c2a30191376990bcd1821c65e37ba9d`

- Autor: Anuken
- Fecha: 2019-09-27
- Mensaje: `Mods branch`

Qué cambió exactamente:

1. Se crean `mod/Mod.java` y `mod/Mods.java`.
2. Se implementa detección de `mod.json` / `plugin.json`.
3. Se carga clase principal por `URLClassLoader` y se instancia dinámicamente.
4. En servidor, comandos y reporting empiezan a transicionar de “plugins” a “mods”.

Fragmento clave:

```java
Class<?> main = classLoader.loadClass(meta.main);
return new LoadedMod(jar, zip, (Mod)main.getDeclaredConstructor().newInstance(), meta);
```

Justificación arquitectónica:

- Se formaliza extensibilidad en runtime y se habilita ecosistema de terceros.
- Se desacopla entrega de contenido del ciclo de releases del core.

---

### Commit `d9aa9b6278677ad500336d53dacd7ebcffde5b4f`

- Autor: Anuken
- Fecha: 2019-11-26
- Mensaje: `Desktop scripting support`

Qué cambió exactamente:

1. Se agrega `Scripts.java` para ejecutar JavaScript en mods (etapa inicial basada en GraalJS).
2. Se define API de ejecución de scripts para carga de comportamiento dinámico.

Fragmento clave:

```java
private Context context = Context.newBuilder("js")
    .allowHostClassLookup(...)
    .allowHostAccess(HostAccess.ALL)
    .build();
```

Justificación arquitectónica:

- Permite extensibilidad conductual, no solo declarativa (JSON/contenido estático).
- Acerca el modding a usuarios que no desean compilar Java.

---

### Commit `1eaa1a05aca43a286d4f7338842991d1a4e7c2b3`

- Autor: Anuken
- Fecha: 2020-06-15
- Mensaje: `Switched to rhino fork`

Qué cambió exactamente:

1. Cambio de dependencia:
   - de `org.mozilla:rhino-runtime:1.7.12`
   - a `com.github.Anuken:rhino:$rhinoVersion`
2. Reemplazo de imports `org.mozilla...` por `rhino.*`.
3. Ajuste de contexto de scripting en Android (`AndroidRhinoContext`).

Fragmento clave:

```gradle
api "com.github.Anuken:rhino:$rhinoVersion"
```

```java
import rhino.*;
import rhino.module.*;
import rhino.module.provider.*;
```

Justificación arquitectónica:

- Controlar una variante del engine permite corregir compatibilidad, rendimiento y restricciones por plataforma.
- Reduce dependencia de binarios upstream no adaptados a necesidades concretas del proyecto.

---

### Commit `cf6b7b5aa36daad16b79da4cbb71310f76271650`

- Autor: Anuken
- Fecha: 2025-12-19
- Mensaje: `Updated Rhino version`

Qué cambió exactamente:

1. Se actualiza el hash de `rhinoVersion` en `build.gradle`.

Justificación arquitectónica:

- Mantenimiento evolutivo de runtime de scripts para estabilidad, fixes y compatibilidad con evolución de JVM/plataformas.

## 3.3 Clases actuales relevantes de modding (branch actual)

Ruta principal: `core/src/mindustry/mod/`

Archivos destacados:

1. `Mods.java`: carga, ordenamiento, habilitado/deshabilitado, importación, compatibilidad.
2. `Scripts.java`: ejecución JS, require/module provider, manejo de errores.
3. `ModClassLoader.java`: aislamiento de clases.
4. `ClassMap.java`: mapeo para scripting/mod API.
5. `ContentParser.java`: parse de contenido declarativo.

Fragmento actual de `Scripts.java`:

```java
context = Vars.platform.getScriptContext();
scope = new ImporterTopLevel(context);
new RequireBuilder()
    .setModuleScriptProvider(new SoftCachingModuleScriptProvider(new ScriptModuleProvider()))
    .setSandboxed(true).createRequire(context, scope).install(scope);
```

---

## 3.4 Evidencia de estructura de directorios (mods)

```text
[worktree 70ab102d8c]
core/src/io/anuke/mindustry/mod/Mods.java
core/src/io/anuke/mindustry/mod/Mod.java
```

```text
[worktree d9aa9b6278]
core/src/io/anuke/mindustry/mod/Scripts.java
core/src/io/anuke/mindustry/mod/Mods.java
core/src/io/anuke/mindustry/mod/ModCrashHandler.java
core/src/io/anuke/mindustry/mod/ContentParser.java
```

```text
[worktree origin/master]
core/src/mindustry/mod/Scripts.java
core/src/mindustry/mod/Mods.java
core/src/mindustry/mod/ModClassLoader.java
core/src/mindustry/mod/ClassMap.java
core/src/mindustry/mod/ContentParser.java
```

---

## 4. Investigación C: Expansión de ecosistema (iOS, Steam, Discord)

## 4.1 Steam

### Commit `e482c2c31803b2995f439904565f262bd3aa6820`

- Autor: Anuken
- Fecha: 2019-08-18
- Mensaje: `Steam client init`

Qué cambió exactamente:

1. Se agregan dependencias `steamworks4j` y `steamworks4j-server`.
2. `DesktopPlatform` inicializa `SteamAPI`, ejecuta callbacks periódicos y registra shutdown hook.

Fragmento:

```gradle
compile "com.code-disaster.steamworks4j:steamworks4j:$steamworksVersion"
compile "com.code-disaster.steamworks4j:steamworks4j-server:$steamworksVersion"
```

Justificación arquitectónica:

- Integrar lobby, matchmaking y servicios de plataforma Steam.
- Unificar identidad de jugador y experiencias sociales/workshop.

### Capas actuales Steam en el proyecto

Ruta: `desktop/src/mindustry/desktop/steam/`

1. `SNet.java`: red P2P/lobbies y puente con `NetProvider`.
2. `SWorkshop.java`: publicación/actualización de contenido workshop.
3. `SStats.java`: achievements y estadísticas Steam.
4. `SUser.java`: usuario Steam.

Esto demuestra que Steam no es solo “SDK conectado”, sino un subsistema completo de plataforma.

---

## 4.2 Discord Rich Presence

### Commit `12fad819b5e8ea2196498a6605aa7064e5907e62`

- Autor: BeefEX
- Fecha: 2018-01-02
- Mensaje: `Discord integration, basic rich presence`

Qué cambió exactamente:

1. Se agrega dependencia Discord RPC.
2. Se extiende interfaz de plataforma con callbacks de escena y salida.
3. `DesktopLauncher` publica presencia con estado/mapa.

Justificación arquitectónica:

- Añadir visibilidad social de sesión activa y retención de comunidad.

### Commit `d849a3a87fc43ce9576e89e8eade4276b76a3b41`

- Autor: Daniel Jennings
- Fecha: 2020-01-27
- Mensaje: `Adding Steam Rich Presence support. (#1453)`

Qué cambió exactamente:

1. Refactor de `updateRPC()` para lógica común Discord + Steam.
2. Publicación de estado enriquecido para ambos canales.

Justificación arquitectónica:

- Evita duplicación de lógica de estado y sincroniza UX entre ecosistemas sociales.

---

## 4.3 iOS

### Commit `d8d855217666ded36623dee84b7be9e3bb207c13`

- Autor: Anuken
- Fecha: 2018-04-27
- Mensaje: `Added iOS module`

Qué cambió exactamente:

1. Se agrega módulo `ios`.
2. Se crea `ios/build.gradle` con plugin RoboVM y tareas de empaquetado IPA.
3. Se introduce `IOSLauncher`.
4. Se registra módulo en `settings.gradle`.

Fragmento:

```java
public class IOSLauncher extends IOSApplication.Delegate {
    @Override
    protected IOSApplication createApplication() {
        IOSApplicationConfiguration config = new IOSApplicationConfiguration();
        return new IOSApplication(new Mindustry(), config);
    }
}
```

Justificación arquitectónica:

- Expandir superficie multiplataforma nativa (iPhone/iPad), con launcher y pipeline específicos.

## 4.4 Evidencia de estructura de directorios (ecosistema)

```text
[worktree e482c2c318]
desktop-sdl/src/io/anuke/mindustry/desktopsdl/DesktopPlatform.java
```

```text
[worktree 12fad819b5]
desktop/src/io/anuke/mindustry/desktop/DesktopLauncher.java
core/src/io/anuke/mindustry/io/PlatformFunction.java
```

```text
[worktree d8d8552176]
ios/build.gradle
ios/Info.plist.xml
ios/src/io/anuke/mindustry/IOSLauncher.java
ios/robovm.properties
ios/robovm.xml
```

```text
[worktree origin/master]
desktop/src/mindustry/desktop/steam/SNet.java
desktop/src/mindustry/desktop/steam/SWorkshop.java
desktop/src/mindustry/desktop/steam/SStats.java
ios/src/mindustry/ios/IOSLauncher.java
core/src/mindustry/core/Platform.java
```

---

## 5. Investigación DevOps (CI, Calidad, CD)

## 5.1 [DO-01] Pipeline de calidad y cobertura: estado real

Se analizaron workflows en:

- `origin/master/.github/workflows/`
- `origin/feature/cd-pipeline/.github/workflows/`

### Hallazgos de CI actuales

1. Sí existen pipelines de build/test.
2. No se encontró linter explícito.
3. No se encontró static analysis formal (Checkstyle, PMD, SpotBugs, etc.).
4. No se encontró cobertura (JaCoCo/Codecov).
5. No se encontró umbral de cobertura que rompa pipeline.

Workflow CI de referencia (`origin/master/.github/workflows/ci.yml`):

```yaml
- name: Build
  run: ./gradlew desktop:dist

- name: Test
  run: ./gradlew tests:test
```

Workflow PR (`origin/master/.github/workflows/pr.yml`) con artefacto:

```yaml
- name: Run unit tests
  run: ./gradlew tests:test --stacktrace --rerun

- name: Run unit tests and build JAR
  run: ./gradlew desktop:dist

- name: Upload desktop JAR for testing
  uses: actions/upload-artifact@v4
  with:
    name: Desktop JAR (zipped)
    path: desktop/build/libs/Mindustry.jar
```

Interpretación para el informe académico:

- El pipeline está orientado a verificación funcional (tests + build), pero no a métricas de calidad/cobertura.
- Existe evidencia de publicación de artefactos binarios para pruebas, no de reportes de calidad.

---

## 5.2 [DC-03] CD final: release automatizado y brechas

### Workflow principal de deploy (`origin/master/.github/workflows/deployment.yml`)

Trigger:

```yaml
on:
  push:
    tags:
      - 'v*'
```

Build y artefactos:

```yaml
- name: Create artifacts
  run: |
    ./gradlew desktop:dist server:dist core:depsJar core:mergedJavadoc -Pbuildversion=${RELEASE_VERSION:1}
```

Publicación de release assets:

```yaml
- name: Upload client artifacts
  uses: svenstaro/upload-release-action@v2
  with:
    file: desktop/build/libs/Mindustry.jar

- name: Upload server artifacts
  uses: svenstaro/upload-release-action@v2
  with:
    file: server/build/libs/server-release.jar
```

Actualización de documentación (repo externo):

```yaml
- name: Update docs
  run: |
    git clone --depth=1 https://github.com/MindustryGame/docs.git
    cp -a Mindustry/core/build/javadoc/. docs/
```

### Brechas frente al requerimiento académico DC-03 planteado

1. No hay evidencia de un release específicamente `v1.0.0` (el trigger es genérico `v*`).
2. No hay build Android release (`android:assembleRelease` no aparece en workflows).
3. No hay despliegue explícito con GitHub Pages Actions (`actions/upload-pages-artifact` + `actions/deploy-pages`).
4. Sí hay release automatizado y adjuntos desktop/server/deps.
5. Sí hay despliegue automatizado de documentación, pero a repo externo de docs y no al flujo estándar de Pages del mismo repositorio.

---

## 5.3 Rama académica de CD (`origin/feature/cd-pipeline`)

Commit de creación base:

- `4e0f73a5aadc3bb7f0bd6c28fc2b22451efdf270`
- Autor: KarlaBedregal
- Mensaje: `Add CD workflow`

Evolución posterior:

- `66bdaf9ecb06c9998de40b6cad04b9a7dd9b3c32`
- `ebdb188c03d9b06cfa89c3e188fe6a3094cbd0de`

Qué automatiza:

1. Build `desktop:dist`.
2. Upload artifact JAR.
3. Crea release con tag `build-${{ github.run_number }}`.

Interpretación:

- Este CD de rama sirve como baseline didáctico/experimental para automatizar release de artefacto desktop.
- No cubre APK Android, no versiona por tag semántico final y no despliega Pages.

---

## 6. Comandos Git usados (replicabilidad)

## 6.1 Búsquedas históricas por tema

```bash
git --no-pager log --all --date=iso --pretty=format:'%h|%ad|%an|%s' --grep='kryonet|arcnet|headless|network' -i
git --no-pager log --all --date=iso --pretty=format:'%h|%ad|%an|%s' --grep='mod|rhino|javascript|script' -i
git --no-pager log --all --date=iso --pretty=format:'%h|%ad|%an|%s' --grep='steam|discord|ios|launcher|steamworks' -i
```

## 6.2 Inspección de cambios de commits específicos

```bash
git --no-pager show <hash>
git --no-pager show <hash> -- <ruta1> <ruta2>
git --no-pager show --name-only --pretty=format:'%H|%ad|%an|%s' --date=iso <hash>
```

## 6.3 Estructura por commit sin checkout destructivo

```bash
git --no-pager ls-tree -r --name-only <hash> | grep -E 'net|server|mod|steam|ios'
git worktree add --detach /tmp/mindustry-<tag> <hash>
find /tmp/mindustry-<tag> -maxdepth 5 -type f
git worktree remove /tmp/mindustry-<tag> --force
```

## 6.4 Investigación DevOps

```bash
git --no-pager ls-tree --name-only origin/master:.github/workflows
git --no-pager show origin/master:.github/workflows/ci.yml
git --no-pager show origin/master:.github/workflows/pr.yml
git --no-pager show origin/master:.github/workflows/deployment.yml
git --no-pager grep -nEi 'lint|jacoco|codecov|coverage|threshold|assembleRelease|deploy-pages' origin/master -- .github/workflows
```

---

## 7. Síntesis final para el informe académico

1. **Red y multiplayer**: Sprint 3–4 muestra transición desde API de red mínima hacia stack multiplayer robusto, incluyendo servidor headless y sucesivas refactorizaciones de transporte/serialización.
2. **Modding**: El proyecto evoluciona de carga de mods empaquetados a scripting JS y finalmente consolida Rhino como runtime principal para extensibilidad en JVM.
3. **Ecosistema externo**: Steam, Discord e iOS no son añadidos cosméticos; se implementan como capas concretas con clases dedicadas, hooks de ciclo de vida y dependencias específicas.
4. **DevOps real**: Hay automatización CI/CD funcional, pero con brechas claras respecto al objetivo académico de calidad/cobertura formal y CD final completo multi-artefacto (incluyendo Android + Pages).
5. **Valor metodológico del “método del cangrejo”**: La evidencia en commits muestra decisiones arquitectónicas reactivas y evolutivas típicas de software vivo; permite reconstruir “historias de usuario técnicas” con trazabilidad real en vez de narrativa idealizada.


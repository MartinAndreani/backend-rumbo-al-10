# Ejemplo 1 — JPA Básico (H2 en memoria)

## Objetivos

- Configurar **JPA + Hibernate** sobre **H2 en memoria**.
- Reutilizar la **inicialización de Chinook** (scripts SQL) para tender el **puente JDBC → JPA**.
- Mapear entidades **planas**: `Genre` y `MediaType`.
- Practicar **find** y **persist**, obteniendo correctamente el **EntityManager**.

## 1. Requisitos y estructura

- Java 21, Maven.
- Scripts SQL ya usados en JDBC (adaptados a H2):
  - `src/main/resources/sql/01_chinook_tables.sql`
  - `src/main/resources/sql/02_chinook_data.sql`
  - `src/main/resources/sql/03_chinook_constraints_indexes.sql`
  - `src/main/resources/sql/04_chinook_sequences_h2.sql`

> **Estructura sugerida**
>
> ```bash
> jpa-lab1/
> ├─ pom.xml
> ├─ src/
> │  ├─ main/java/
> │  │  └─ utnfc/isi/back/jpa/
> │  │     ├─ infra/
> │  │     │  └─ DbInit.java
> │  │     ├─ domain/
> │  │     │  ├─ Genre.java
> │  │     │  └─ MediaType.java
> │  │     └─ Main.java
> │  └─ main/resources/
> │     ├─ META-INF/persistence.xml
> │     └─ sql/
> │        ├─ 01_chinook_tables.sql
> │        ├─ 02_chinook_data.sql
> │        ├─ 03_chinook_constraints_indexes.sql
> │        └─ 03_chinook_sequences.sql
> ```

## 2. Configuración - Archivo: `pom.xml`

- API de **Jakarta Persistence**, **Hibernate** como implementación, **H2**, **Lombok** y el **exec-maven-plugin** para correr el main.

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>utnfc.isi.back.jpa</groupId>
  <artifactId>jpa-h2mem-basico</artifactId>
  <version>1.0.0-SNAPSHOT</version>

  <properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <jakarta.persistence.version>3.2.0</jakarta.persistence.version>
    <hibernate.version>6.5.2.Final</hibernate.version>
  </properties>

  <dependencies>
    <!-- API estándar -->
    <dependency>
      <groupId>jakarta.persistence</groupId>
      <artifactId>jakarta.persistence-api</artifactId>
      <version>${jakarta.persistence.version}</version>
    </dependency>

    <!-- Implementación (ORM) -->
    <dependency>
      <groupId>org.hibernate.orm</groupId>
      <artifactId>hibernate-core</artifactId>
      <version>${hibernate.version}</version>
    </dependency>

    <!-- Driver H2 -->
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <version>2.2.224</version>
      <scope>runtime</scope>
    </dependency>

    <!-- Lombok -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>1.18.34</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>3.5.1</version>
        <configuration>
          <mainClass>utnfc.isi.back.jpa.Main</mainClass>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

> Si estás en Windows y tu consola no muestra bien acentos, podés forzar UTF-8 en el run: `mvn -Dfile.encoding=UTF-8 exec:java`.

## 3. Configuración - Archivo: `persistence.xml`

- Usamos **H2 en memoria** con `DB_CLOSE_DELAY=-1` para que la BD viva toda la JVM.
- Dejamos `hibernate.hbm2ddl.auto=validate` para **no sobreescribir** el esquema de Chinook cargado por los scripts.

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence" version="3.0">
  <persistence-unit name="ChinookPU" transaction-type="RESOURCE_LOCAL">
    <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

    <properties>
      <property name="jakarta.persistence.jdbc.driver" value="org.h2.Driver"/>
      <property name="jakarta.persistence.jdbc.url" value="jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1"/>
      <property name="jakarta.persistence.jdbc.user" value="sa"/>
      <property name="jakarta.persistence.jdbc.password" value=""/>

      <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
      <property name="hibernate.hbm2ddl.auto" value="validate"/>

      <property name="hibernate.show_sql" value="true"/>
      <property name="hibernate.format_sql" value="true"/>
    </properties>
  </persistence-unit>
</persistence>
```

> Si tus scripts son estrictamente de PostgreSQL, podés probar H2 con `MODE=PostgreSQL`: `jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1;MODE=PostgreSQL`.

## 4. Infraestructura

En este primer ejemplo obtenemos el **EntityManagerFactory** y el **EntityManager** directamente en `Main`. El `EMFProvider` lo incorporaremos recién en el **Ejemplo 2** para centralizar la obtención del EntityManager.

### 4.1 `DbInit.java` (puente JDBC → JPA)

```java
package utnfc.isi.back.jpa.infra;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.Statement;

public final class DbInit {
    private static final String URL = "jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1";
    private static final String USER = "sa";
    private static final String PASS = "";

    private DbInit() {
    }

    public static void run() throws SQLException, IOException {
        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            exec(conn, "src/main/resources/sql/01_chinook_tables.sql");
            exec(conn, "src/main/resources/sql/02_chinook_data.sql");
            exec(conn, "src/main/resources/sql/03_chinook_constraints_indexes.sql");
            exec(conn, "src/main/resources/sql/04_chinook_sequences.sql");
            alignSequence(conn, "GENRE", "GENRE_ID", "SEQ_GENRE");
            alignSequence(conn, "MEDIA_TYPE", "MEDIA_TYPE_ID", "SEQ_MEDIA_TYPE");
            alignSequence(conn, "ARTIST", "ARTIST_ID", "SEQ_ARTIST");
            alignSequence(conn, "ALBUM", "ALBUM_ID", "SEQ_ALBUM");
            alignSequence(conn, "TRACK", "TRACK_ID", "SEQ_TRACK");
            alignSequence(conn, "PLAY_LIST", "PLAY_LIST_ID", "SEQ_PLAY_LIST");
        }
    }

    private static void exec(Connection conn, String file) throws IOException, SQLException {
        String sql = Files.readString(Path.of(file), StandardCharsets.UTF_8);
        try (Statement st = conn.createStatement()) {
            st.execute(sql);
        }
    }

    private static void alignSequence(Connection conn, String table, String pkColumn, String seqName) throws SQLException {
        long next = 1L;
        try (Statement st = conn.createStatement(); var rs = st.executeQuery("SELECT COALESCE(MAX(" + pkColumn + "), 0) + 1 AS nextval FROM " + table)) {
            if (rs.next()) {
                next = rs.getLong("nextval");
            }
        }
        try (Statement st = conn.createStatement()) {
            st.execute("ALTER SEQUENCE " + seqName + " RESTART WITH " + next);
        }
    }
}
```

## 5. Mapeo - Entidades planas

> Nomenclatura: en la base de datos la convención es usar **UPPER\_SNAKE\_CASE** (`GENRE`, `MEDIA_TYPE`, `GENRE_ID`, `NAME`). En Java, **camelCase**. Mapeamos con `@Table` y `@Column`.

### 5.1 `Genre.java`

```java
package utnfc.back.isi.jpa.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "GENRE")
@Data @NoArgsConstructor @AllArgsConstructor @Builder
@EqualsAndHashCode(of = "id")
public class Genre {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(name = "GENRE_ID")
  private Integer id;

  @Column(name = "NAME", length = 120, nullable = false)
  private String name;
}
```

### 5.2 `MediaType.java`

```java
package utnfc.back.isi.jpa.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "MEDIA_TYPE")
@Data @NoArgsConstructor @AllArgsConstructor @Builder
@EqualsAndHashCode(of = "id")
public class MediaType {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(name = "MEDIA_TYPE_ID")
  private Integer id;

  @Column(name = "NAME", length = 120, nullable = false)
  private String name;
}
```

---

## 6. Archivo: `Main.java` (find + persist)

```java
package utnfc.back.isi.jpa;

import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;
import utnfc.isi.back.jpa.domain.Genre;
import utnfc.isi.back.jpa.domain.MediaType;
import utnfc.isi.back.jpa.infra.DbInit;

public class Main {
  public static void main(String[] args) throws Exception {
    // 0) Inicializamos Chinook en H2 memoria vía JDBC (mismo URL que JPA)
    DbInit.run();

    // 1) Obtenemos EMF y EM directamente
    EntityManagerFactory emf = Persistence.createEntityManagerFactory("ChinookPU");
    EntityManager em = emf.createEntityManager();

    try {
      // 2) find: leer existentes
      Genre g1 = em.find(Genre.class, 1); // por PK
      MediaType m1 = em.find(MediaType.class, 1);
      System.out.println("Genero #1 = " + (g1 != null ? g1.getName() : "(no existe)"));
      System.out.println("MediaType #1 = " + (m1 != null ? m1.getName() : "(no existe)"));

      // 3) persist: crear uno nuevo
      em.getTransaction().begin();
      Genre nuevo = Genre.builder().name("Synthwave").build();
      em.persist(nuevo); // pasa a Managed, INSERT en commit
      em.getTransaction().commit();

      System.out.println("Nuevo Genre id=" + nuevo.getId());

      // 4) verificar con otro find
      Genre check = em.find(Genre.class, nuevo.getId());
      System.out.println("Verificado en DB = " + check.getName());
    } finally {
      em.close();
      emf.close();
    }
  }
}
```

## 7. Ejecución

- Desde la raíz del proyecto:

```bash
mvn -q -Dfile.encoding=UTF-8 exec:java
```

> **Salida esperada (aprox.)**
>
> ```txt
> INFO: HHH000204: Processing PersistenceUnitInfo [name: ChinookPU]
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.Version logVersion
> INFO: HHH000412: Hibernate ORM core version 6.5.2.Final
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.cache.internal.RegionFactoryInitiator initiateService
> INFO: HHH000026: Second-level cache disabled
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.connections.internal.DriverManagerConnectionProviderImpl configure
> WARN: HHH10001002: Using built-in connection pool (not intended for production use)
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.connections.internal.DriverManagerConnectionProviderImpl buildCreator
> INFO: HHH10001005: Loaded JDBC driver class: org.h2.Driver
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.connections.internal.DriverManagerConnectionProviderImpl buildCreator
> INFO: HHH10001012: Connecting with JDBC URL [jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1]
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.connections.internal.DriverManagerConnectionProviderImpl buildCreator
> INFO: HHH10001001: Connection properties: {password=****, user=sa}
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.connections.internal.DriverManagerConnectionProviderImpl buildCreator
> INFO: HHH10001003: Autocommit mode: false
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.connections.internal.> DriverManagerConnectionProviderImpl$PooledConnections <init>
> INFO: HHH10001115: Connection pool size: 20 (min=1)
> sept 14, 2025 11:06:19 P.┬áM. org.hibernate.engine.jdbc.dialect.internal.DialectFactoryImpl constructDialect
> WARN: HHH90000025: H2Dialect does not need to be specified explicitly using 'hibernate.dialect' (remove the property setting and it will be selected by default)
> sept 14, 2025 11:06:21 P.┬áM. org.hibernate.engine.transaction.jta.platform.internal.JtaPlatformInitiator initiateService
> INFO: HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
> sept 14, 2025 11:06:21 P.┬áM. org.hibernate.resource.transaction.backend.jdbc.internal.> DdlTransactionIsolatorNonJtaImpl getIsolatedConnection
> INFO: HHH10001501: Connection obtained from JdbcConnectionAccess [org.hibernate.engine.jdbc.env.internal. JdbcEnvironmentInitiator$ConnectionProviderJdbcConnectionAccess@1dfbbd73] for (non-JTA) DDL execution was not in auto-commit mode; the Connection 'local transaction' will be committed and the Connection will be set into auto-commit mode.
> Hibernate:
>     select
>         g1_0.GENRE_ID,
>         g1_0.NAME
>     from
>         GENRE g1_0
>     where
>         g1_0.GENRE_ID=?
> Hibernate:
>     select
>         mt1_0.MEDIA_TYPE_ID,
>         mt1_0.NAME
>     from
>         MEDIA_TYPE mt1_0
>     where
>         mt1_0.MEDIA_TYPE_ID=?
> Genero #1 = Rock
> MediaType #1 = MPEG audio file
> Hibernate:
>     insert
>     into
>         GENRE
>         (NAME, GENRE_ID)
>     values
>         (?, default)
> Nuevo Genre id=26
> Verificado en DB = Synthwave
> sept 14, 2025 11:06:21 P.┬áM. org.hibernate.engine.jdbc.connections.internal.> DriverManagerConnectionProviderImpl$PoolState stop
> INFO: HHH10001008: Cleaning up connection pool [jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1]
> ```

## 8. Checklist y notas

- H2 memoria: usar **misma URL** en DbInit y en `persistence.xml` (`jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1`).
- `hibernate.hbm2ddl.auto=validate` para no pisar el esquema Chinook.
- Si los scripts son de Postgres, probar `;MODE=PostgreSQL` en la URL.
- Evitar comillas en DDL; mantener **UPPER\_SNAKE\_CASE** y mapear con `@Column`.

## 9. Para pensar (mini desafíos)

- Cambiar el `persist()` por un **update** simple: `find()` un `Genre` existente, modificar `name` y `commit`.
- Agregar una **consulta JPQL**: listar `MediaType` por nombre ascendente (`select m from MediaType m order by m.name`).
- Probar **paginado** con `setFirstResult` y `setMaxResults`.

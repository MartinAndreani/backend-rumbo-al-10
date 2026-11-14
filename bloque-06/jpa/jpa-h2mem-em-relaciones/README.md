# Ejemplo 2 — Relaciones y EntityManager (JPA, H2)

## Objetivos

- Introducir **relaciones** en el mapeo JPA sobre la base **Chinook** (H2 memoria).
- Centralizar la obtención del **EntityManager** mediante `EMFProvider`.
- Practicar `@ManyToOne` y `@OneToMany` (bidireccional) con **métodos de conveniencia** para mantener la asociación.
- Observar el **N+1** y cómo impacta un `toString()` ingenuo; proponer estrategias seguras.

---

## 1. Requisitos y estructura

- Java 21, Maven.
- Misma configuración de dependencias y `persistence.xml` que en **Ejemplo 1** (H2 memoria, `DB_CLOSE_DELAY=-1`, `hbm2ddl.auto=validate`).
- Scripts de inicialización de Chinook + secuencias alineadas (ver Ejemplo 1).

> **Estructura sugerida**
>
> ```txt
> jpa-ejemplo2/
> ├─ pom.xml
> ├─ src/
> │  ├─ main/java/
> │  │  └─ utnfc/isi/back/jpa/
> │  │     ├─ infra/
> │  │     │  ├─ EMFProvider.java
> │  │     │  └─ DbInit.java
> │  │     ├─ domain/
> │  │     │  ├─ Artist.java
> │  │     │  ├─ Album.java
> │  │     │  └─ Track.java
> │  │     └─ Main.java
> │  └─ main/resources/
> │     ├─ META-INF/persistence.xml
> │     └─ sql/
> │        ├─ 01_chinook_tables.sql
> │        ├─ 02_chinook_data.sql
> │        ├─ 03_chinook_constraints_indexes.sql
> │        └─ 03_chinook_sequences.sql
> ```

---

## 2. `EMFProvider`

```java
package utnfc.isi.back.jpa.infra;

import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;

public final class EMFProvider {
  private static final EntityManagerFactory EMF =
      Persistence.createEntityManagerFactory("ChinookPU");

  private EMFProvider() {}

  public static EntityManager em() { return EMF.createEntityManager(); }

  public static void close() { EMF.close(); }
}
```

## 3. `DbInit` (reutilizable)

> Podemos reutilizar el `DbInit` del Ejemplo 1, asegurando que:
>
> - Ejecuta `tablas`, `datos`, `restricciones` y `secuencias`.
> - Alinea las secuencias a `MAX(id)+1` por tabla.

```java
package utnfc.isi.back.jpa.infra;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.sql.*;

public final class DbInit {
  private static final String URL = "jdbc:h2:mem:chinook;DB_CLOSE_DELAY=-1";
  private static final String USER = "sa";
  private static final String PASS = "";

  private DbInit() {}

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
    try (Statement st = conn.createStatement()) { st.execute(sql); }
  }

  private static void alignSequence(Connection conn, String table, String pkColumn, String seqName) throws SQLException {
    long next = 1L;
    try (Statement st = conn.createStatement();
         ResultSet rs = st.executeQuery("SELECT COALESCE(MAX(" + pkColumn + "), 0) + 1 AS nextval FROM " + table)) {
      if (rs.next()) next = rs.getLong("nextval");
    }
    try (Statement st = conn.createStatement()) {
      st.execute("ALTER SEQUENCE " + seqName + " RESTART WITH " + next);
    }
  }
}
```

## 4. Entidades y relaciones

### 4.1 `Artist` (1↔N con `Album`)

```java
package utnfc.isi.back.jpa.domain;

import jakarta.persistence.*;
import lombok.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "ARTIST")
@Data @NoArgsConstructor @AllArgsConstructor
@EqualsAndHashCode(of = "id")
public class Artist {
  @Id
  @SequenceGenerator(name = "artist_seq", sequenceName = "SEQ_ARTIST", allocationSize = 1)
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "artist_seq")
  @Column(name = "ARTIST_ID")
  private Integer id;

  @Column(name = "NAME", length = 120)
  private String name;

  @OneToMany(mappedBy = "artist")
  @ToString.Exclude // evitamos recursión y cargas perezosas en toString
  private List<Album> albums = new ArrayList<>();

  // método de conveniencia
  public void addAlbum(Album a) {
    albums.add(a);
    a.setArtist(this);
  }
}
```

### 4.2 `Album` (N→1 con `Artist` y 1↔N con `Track`)

```java
package utnfc.isi.back.jpa.domain;

import jakarta.persistence.*;
import lombok.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "ALBUM")
@Data @NoArgsConstructor @AllArgsConstructor
@EqualsAndHashCode(of = "id")
public class Album {
  @Id
  @SequenceGenerator(name = "album_seq", sequenceName = "SEQ_ALBUM", allocationSize = 1)
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "album_seq")
  @Column(name = "ALBUM_ID")
  private Integer id;

  @Column(name = "TITLE", length = 160, nullable = false)
  private String title;

  @ManyToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(name = "ARTIST_ID", nullable = false)
  @ToString.Exclude // para no forzar carga LAZY en toString()
  private Artist artist;

  @OneToMany(mappedBy = "album")
  @ToString.Exclude // evita recursión y N+1 en toString
  private List<Track> tracks = new ArrayList<>();

  // conveniencia bidireccional
  public void addTrack(Track t) {
    tracks.add(t);
    t.setAlbum(this);
  }
}
```

### 4.3 `Track` (N→1 con `Album`)

```java
package utnfc.isi.back.jpa.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "TRACK")
@Data @NoArgsConstructor @AllArgsConstructor
@EqualsAndHashCode(of = "id")
public class Track {
  @Id
  @SequenceGenerator(name = "track_seq", sequenceName = "SEQ_TRACK", allocationSize = 1)
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "track_seq")
  @Column(name = "TRACK_ID")
  private Integer id;

  @Column(name = "NAME", length = 200, nullable = false)
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "ALBUM_ID")
  @ToString.Exclude
  private Album album;
}
```

---

## 5. `toString()` seguro y referencias cíclicas

- Evitamos imprimir asociaciones LAZY en `toString()` (anotamos con `@ToString.Exclude`).
- Si queremos un `toString()` “lindo” para `Album`, definimos uno **manual** (sin forzar cargas):

Para mejorar el toString() del Album, podemos implementarlo así:

```java
@Override
public String toString() {
    StringBuilder sb = new StringBuilder();
    sb.append("Album{")
      .append("id=").append(id)
      .append(", title='").append(title).append('\'')
      .append(", artist=").append(artist != null ? artist.getName() : null)
      .append(", tracksCount=").append(tracks != null ? tracks.size() : 0)
      .append("}");

    if (tracks != null && !tracks.isEmpty()) {
        sb.append("\nTracks:\n");
        for (Track t : tracks) {
            sb.append("  ").append(t.toString()).append("\n");
        }
    }
    return sb.toString();
}

```

> `tracks.size()` puede disparar carga LAZY si la colección no está inicializada; usarlo **dentro de transacción**.

Y en la entidad Track algo liviano y controlado para evitar recursividad:

```java
@Override
public String toString() {
    return "Track{" +
           "id=" + id +
           ", name='" + name + '\'' +
           ", millis=" + milliseconds +
           '}';
}

```

---

## 6. `Main` — flujo sugerido

```java
package utnfc.isi.back.jpa;

import jakarta.persistence.EntityManager;
import utnfc.isi.back.jpa.domain.*;
import utnfc.isi.back.jpa.infra.DbInit;
import utnfc.isi.back.jpa.infra.EMFProvider;

public class Main {
  public static void main(String[] args) throws Exception {
    DbInit.run();

    EntityManager em = EMFProvider.em();

    // 1) Crear artista, álbum y un par de tracks
    em.getTransaction().begin();
    Artist artist = new Artist();
    artist.setName("The Monads");

    Album album = new Album();
    album.setTitle("Functional Grooves");
    artist.addAlbum(album); // mantiene ambos lados

    Track t1 = new Track();
    t1.setName("Lambda Jam");
    album.addTrack(t1);

    Track t2 = new Track();
    t2.setName("Stream Flow");
    album.addTrack(t2);

    em.persist(artist); // cascada NO configurada: persistimos explícitamente si fuera necesario
    em.persist(album);
    em.persist(t1);
    em.persist(t2);
    em.getTransaction().commit();

    // 2) Leer un álbum existente y observar cargas LAZY
    em.getTransaction().begin();
    Album a = em.find(Album.class, album.getId());
    System.out.println("Album básico: " + a); // no fuerza cargar artist/tracks por los @ToString.Exclude
    System.out.println("Artist id del álbum: " + (a.getArtist() != null ? a.getArtist().getId() : null)); // puede cargar LAZY
    System.out.println("Cantidad de tracks: " + a.getTracks().size()); // carga LAZY dentro de TX
    em.getTransaction().commit();

    // 3) Consulta segura para evitar N+1 (join fetch)
    em.getTransaction().begin();
    var albums = em.createQuery(
        "select distinct al from Album al " +
        "join fetch al.artist " +
        "left join fetch al.tracks ", Album.class)
      .setMaxResults(5)
      .getResultList();
    albums.forEach(al -> System.out.println("Album+relaciones: " + al.getId() + " - " + al.getTitle() +
        " | artist=" + (al.getArtist()!=null?al.getArtist().getName():null) +
        " | tracks=" + al.getTracks().size()));
    em.getTransaction().commit();

    em.close();
    EMFProvider.close();
  }
}
```

> Si preferís cascada (por ejemplo, `@OneToMany(cascade = CascadeType.PERSIST)` en `Album.tracks`), podrías `persist(album)` y que se persistan los `Track` asociados. Para el curso, es útil **mostrar ambos enfoques**.

---

## 7. Buenas prácticas

- Mantener **métodos de conveniencia** (`addAlbum`, `addTrack`) para relaciones bidireccionales.
- Evitar `toString()` recursivos o que fuercen carga LAZY fuera de transacción.
- Usar `join fetch` para leer gráfico de objetos cuando sea necesario (y `distinct`).
- Controlar el tamaño de colecciones con paginado o límites en consultas.

---

## 8. Mini desafíos

- Agregar `@OneToMany(cascade = CascadeType.PERSIST)` en `Album.tracks` y probar persistir sin `em.persist(t1/t2)`.
- Implementar un `find` por título de álbum con JPQL y paginación.
- Hacer un `detach(a)` y observar el comportamiento al acceder a `a.getTracks()` fuera de la transacción.

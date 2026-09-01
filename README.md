# SamsTrack — Student Attendance Management System (REST API)

A Spring Boot + MySQL REST API for taking and tracking student attendance, with role-based users (admin and faculty) and full CRUD over students, subjects, and attendance records. The API is consumed by a separate React frontend running on `http://localhost:4200`.

Repo: https://github.com/Mr-robot28/SAMSTRACK_API
Postman collection: `SAMSTRACK_API.postman_collection.json` (in this repo)

---

## What it does

Four resource groups, sixteen endpoints:

**User (`/user`) — admin / faculty login and management**
- `POST /user/login-user` — authenticate, returns the user object
- `POST /user/register-user` — create a new user
- `GET  /user/get-user-by-username/{username}` — fetch one user
- `GET  /user/get-all-user` — list all users
- `GET  /user/get-all-admin` — list admins only
- `GET  /user/get-all-faculty` — list faculty only
- `PUT  /user/update-user` — update an existing user
- `DELETE /user/delete-user-by-username?username=...` — delete a user

**Student (`/student`)**
- `POST   /student/add-student`
- `GET    /student/get-all-students`
- `GET    /student/get-student-by-id/{id}`
- `PUT    /student/update-student`
- `DELETE /student/delete-student/{id}`

**Subject (`/subject`)**
- `POST   /subject/add-subject`
- `GET    /subject/get-all-subjects`
- `GET    /subject/get-subject-by-id/{id}`
- `PUT    /subject/update-subject`
- `DELETE /subject/delete-subject/{id}`

**Attendance (`/attendance`)**
- `POST /attendance/take-attendance` — records one attendance session (faculty + subject + date + time + list of present student IDs)
- `GET  /attendance/get-all-attendance-records`
- `GET  /attendance/get-attendance-by-date-subjet/{date}/{subjectId}` — fetch all sessions for a given date + subject

---

## Try it locally

Register a faculty user first:

```http
POST http://localhost:8091/user/register-user
Content-Type: application/json

{
  "username": "faculty01",
  "password": "pass1234",
  "firstName": "Asha",
  "lastName": "Kumar",
  "email": "asha@example.com",
  "role": "faculty"
}
```

Response on success:
```json
1
```
(returned as `ResponseEntity<Integer>` — `1` = created, `3` = username already taken)

Log in:
```http
POST http://localhost:8091/user/login-user
Content-Type: application/json

{ "username": "faculty01", "password": "pass1234" }
```

Add a subject and a few students (CRUD — see the endpoint list above), then take attendance:

```http
POST http://localhost:8091/attendance/take-attendance
Content-Type: application/json

{
  "username": "faculty01",
  "subjectId": 1,
  "date": "2026-09-01",
  "time": "10:30",
  "numberOfStudents": 2,
  "studentIds": [1, 2]
}
```

Response:
```json
{
  "id": "20260901103000123",
  "user": { "username": "faculty01", ... },
  "subject": { "id": 1, "name": "Mathematics" },
  "date": "2026-09-01",
  "time": "10:30",
  "numberOfStudents": 2,
  "students": [ { "id": 1, "name": "..." }, { "id": 2, "name": "..." } ]
}
```

Fetch sessions for that day and subject:
```http
GET http://localhost:8091/attendance/get-attendance-by-date-subjet/2026-09-01/1
```

---

## Architecture

```
Controller  →  Service  →  DAO  →  MySQL
   ↑              ↑          ↑
  CORS,       thin pass-   raw Hibernate
  HTTP        through,     Session +
  shaping     ID-gen       Criteria API
              & dedup
```

- **Controller layer** — handles HTTP, CORS, request mapping. No business logic.
- **Service layer** — thin pass-through to the DAO plus a small amount of business logic: generating the attendance record's `id` (timestamp-based) and deduplicating the result list using a `TreeSet` keyed on `id`.
- **DAO layer** — uses raw Hibernate `Session` + `Criteria` API rather than Spring Data JPA repositories. Every method opens a session, does the work, and closes the session in a `finally` block.
- **Entities** — `User` (PK = username, role-based), `Student` (PK = auto-increment Long), `Subject` (PK = auto-increment Long), `AttendanceRecord` (PK = timestamp string, with `@ManyToOne` to `User` and `Subject`, and `@ManyToMany` to `Student` via a join table `attendance_students`).
- **DTOs** — `AttendanceRecordDTO` and `StudentDTO` exist and are well-shaped, but the controllers return entities directly. The DTOs are not currently wired into the response (see "What I'd add next").
- **Exceptions** — a `GlobalExceptionHandler` (`@RestControllerAdvice`) and an `ErrorResponse` envelope exist, ready to be used. The only `@ExceptionHandler` currently registered is for `DivideByZeroException` (which is never thrown anywhere — vestigial from an earlier iteration).

---

## Key design decisions (and why)

**Raw Hibernate `Session` + `Criteria` API instead of Spring Data JPA repositories.** The project predates my comfort with the repository abstraction, so each DAO owns its session lifecycle explicitly: `openSession()` → work → `close()` in a `finally` block. The upside is full visibility into exactly what SQL gets generated. The downside is a lot of boilerplate and several places where exceptions are silently swallowed by `e.printStackTrace()` instead of being rethrown — a known sharp edge, see "What I'd add next."

**Timestamp-based `AttendanceRecord.id`, not auto-increment.** The id is generated in the service layer as `new SimpleDateFormat("yyyyMMddHHmmssSSS").format(new Date())` — for example `20260901103000123`. This makes records human-readable, sort chronologically without an extra column, and can be reconstructed from the row alone. Trade-off: two attendance saves in the same millisecond would collide, since the id is the `@Id` and not unique-safe. In a single-faculty-takes-attendance use case the collision window is effectively zero, but a high-concurrency deployment would need a sequence or a UUID instead.

**Dedup the read path on the service layer, not the database.** `getAllAttendanceRecords` pulls every row from the DB and then runs the list through a `TreeSet<AttendanceRecord>` keyed on `id`. Hibernate's eager-fetching of the `@ManyToMany` students collection can occasionally surface the same parent row twice when pagination is involved — the `TreeSet` collapses that without forcing a `DISTINCT` into the SQL. Cheap and defensive.

**Role is a plain string on `User`, not an enum or a separate table.** `role` is stored as a free-form string (`"admin"`, `"faculty"`) and filtered with `Restrictions.eq("role", "admin")`. This keeps the schema simple and lets a future "HOD" or "student-self-service" role be added without a migration — but it also means a typo in a controller would silently match nothing rather than failing fast. If the role set stabilizes, this should become an enum or its own table.

**DTOs are defined but not used in the response path.** `AttendanceRecordDTO` and `StudentDTO` are well-shaped (they project out only safe fields, flatten the nested relationships, and avoid serializing Hibernate proxies). The controllers currently return entities directly, which means the full `User` object (including its `password` field!) goes out over the wire when an attendance record is returned. The DTOs are wired into nothing. This is the single most important issue in the codebase — see "What I'd add next."

**`@ManyToMany` for attendance → students, but `@ManyToOne` for attendance → subject and attendance → faculty.** A student is "present in many attendance records" and an attendance record "has many students present" — that's a true many-to-many, so a join table (`attendance_students`) is the right call. A subject is taught across many sessions and a session belongs to one subject — a many-to-one with a foreign key is enough and avoids an unnecessary join table.

**CORS is locked to `http://localhost:4200` on every controller.** The Angular dev server runs there, so every `@RestController` is annotated `@CrossOrigin("http://localhost:4200")`. Some methods re-declare `@CrossOrigin(methods = RequestMethod.POST)` to also whitelist the specific HTTP verb. A production deployment that serves the frontend from a different origin would need a global `WebMvcConfigurer` instead — the current per-controller approach does not scale to a second origin.

**Single global exception handler, currently handling one exception class.** `GlobalExceptionHandler` is registered as `@RestControllerAdvice` and wraps any `DivideByZeroException` in a uniform `ErrorResponse` envelope with `message`, `statusCode`, and `path`. The shape is correct and ready to extend; the `DivideByZeroException` itself is never thrown anywhere in the code, so the handler is effectively dead. Real exceptions right now bubble up as Spring's default 500s with no `path` or `message` projection.

**Hardcoded local credentials in `application.properties`, not environment variables.** `spring.datasource.password=admin` and `username=root` are committed to the repo. Fine for local-only development with a throwaway MySQL install, but a deployment of any kind must move these to `${ENV_VAR:default}` form before this repo can be made public — exactly the same lesson called out in my URL shortener project.

---

## Tech stack

- Java 17
- Spring Boot 2.5.6 (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`)
- Hibernate 5 (via JPA 2.2 — `javax.persistence`, not yet `jakarta.persistence`)
- MySQL 8 (database `attendance` is auto-created on first run via `createDatabaseIfNotExist=true`)
- Hibernate-native `Session` + `Criteria` API for data access (no `JpaRepository` interfaces)
- Maven (with the Maven wrapper — `./mvnw`)
- Frontend (separate repo, not in this one): React on `http://localhost:4200`, which the API's CORS config is locked to

---

## Project structure

```
src/main/java/com/tka/sams/api/
├── SamsTrackApplication.java
├── controller/
│   ├── AttendanceController.java   → /attendance
│   ├── StudentController.java      → /student
│   ├── SubjectController.java      → /subject
│   └── UserController.java         → /user
├── service/
│   ├── AttendanceRecordService.java
│   ├── StudentService.java
│   ├── SubjectService.java
│   └── UserService.java
├── dao/
│   ├── AttendanceRecordDao.java
│   ├── StudentDao.java
│   ├── SubjectDao.java
│   └── UserDao.java
├── entity/
│   ├── AttendanceRecord.java
│   ├── Student.java
│   ├── Subject.java
│   └── User.java
├── model/                          → request shapes + (currently unused) DTOs
│   ├── AttendanceRecordDTO.java
│   ├── AttendanceRecordRequest.java
│   ├── LoginRequest.java
│   └── StudentDTO.java
└── exceptions/
    ├── DivideByZeroException.java
    ├── ErrorResponse.java
    └── GlobalExceptionHandler.java

src/main/resources/
└── application.properties          → DB config + server.port=8091

src/test/java/com/jbk/api/
└── SamsTrackApplicationTests.java  → Spring Boot context-load test
```

---

## Running it locally

Requirements: Java 17, Maven (or use the bundled `./mvnw`), and a running MySQL 8 instance.

**1. Clone and configure the database**

```bash
git clone https://github.com/Mr-robot28/SAMSTRACK_API.git
cd SAMSTRACK_API
```

The default `application.properties` expects:
```
spring.datasource.url=jdbc:mysql://localhost:3306/attendance?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=admin
server.port=8091
```

If your local MySQL has a different username/password, edit `src/main/resources/application.properties` (or override at runtime with `-Dspring.datasource.password=...` on the command line). The `?createDatabaseIfNotExist=true` flag means the `attendance` schema is created automatically on first boot — no need to `CREATE DATABASE` manually.

**2. Run**

From your IDE, run `SamsTrackApplication.main`. Or from the command line:

```bash
./mvnw spring-boot:run
```

The API comes up on `http://localhost:8091`.

**3. Point the Angular frontend at it**

If you also have the Angular frontend, start it on `http://localhost:4200` — the CORS config in every controller is already set up to accept requests from that origin, and the frontend's API base URL should be `http://localhost:8091`.

---

## Testing

A Postman collection is included in this repo (`SAMSTRACK_API.postman_collection.json`) covering the main flows: register a user, log in, CRUD a student, CRUD a subject, take attendance, and fetch attendance by date + subject. Import it into Postman, set the collection's `baseUrl` variable to `http://localhost:8091`, and run it end-to-end against a fresh local database to exercise the whole stack.

There is also a single Spring Boot context-load test (`SamsTrackApplicationTests.contextLoads`) which is useful as a smoke test that the entire wiring (Hibernate mappings, beans, CORS) can be assembled without errors. It does not yet cover the DAOs or controllers — adding a `@DataJpaTest` per entity and a `@WebMvcTest` per controller is the obvious next step (see below).

---

## Deployment notes

- **Database** — any MySQL 8 instance reachable from the app host. The connection string is read from `application.properties`, so the only deployment-time concern is making sure the credentials there match the target environment.
- **App hosting** — built with `./mvnw clean package` → runnable jar at `target/SamsTrack-0.0.1-SNAPSHOT.jar`. A Dockerfile and a cloud-platform target are not included in this repo. The natural next step is to containerize with a multi-stage `eclipse-temurin:17-jdk` build, push the image to a registry, and deploy to Render / Railway / Fly.io on a free tier.
- **Frontend** — lives in a separate repo and is served from `http://localhost:4200` in development. Production would serve it from its own domain, which would require widening the CORS config away from the current single-origin lock.
- **Known limitation** — credentials are committed in `application.properties` and would need to be externalized before this can be deployed to anything publicly addressable.

---

## What I'd add next

- **Wire the DTOs into the response path.** This is the #1 thing. The `AttendanceRecordDTO` exists and is correctly shaped; controllers should return `AttendanceRecordDTO` instead of the raw `AttendanceRecord` entity. Doing so stops the `User.password` field from being serialized into every attendance response and stops Hibernate's lazy-init proxies from being touched during Jackson serialization.
- **Replace the raw Hibernate DAOs with Spring Data JPA repositories** plus a small number of custom `@Query` methods where needed. Removes the open-session / close-session boilerplate, gives free CRUD methods, and makes the code unit-testable with `@DataJpaTest` instead of requiring a full `@SpringBootTest`.
- **Move the DB credentials to environment variables** (`${DB_USERNAME:root}`, `${DB_PASSWORD:admin}` etc.) so the same artifact runs locally and in production without edits.
- **Add a global CORS config via `WebMvcConfigurer`** instead of per-controller `@CrossOrigin` so multi-origin support is a one-line change.
- **Expand `GlobalExceptionHandler`** to cover `EntityNotFoundException`, `MethodArgumentNotValidException`, and a generic `Exception` fallback — the `ErrorResponse` shape is already in place to receive them.
- **Add `@DataJpaTest` and `@WebMvcTest` coverage** for every entity and controller.
- **Promote `User.role` to an enum** (or a `Role` table with a foreign key) once the role set stabilizes.
- **Add a per-student attendance percentage endpoint** (`GET /attendance/percentage/{studentId}`) — the data is all there, it's just a group-by query away.
- **Switch the `AttendanceRecord.id` to a UUID or a DB sequence** if the system is ever used by more than one faculty member at a time. Timestamp-based ids are great for readability and zero-cost generation, but they aren't unique under concurrency.
- **Add Spring Security + JWT** so the `/user/login-user` flow returns a token and every other endpoint is authenticated. Right now any client that knows the URL can call any endpoint.

---

## What I learned building this

- **The right level of abstraction is the one you can debug.** Raw Hibernate `Session` is verbose, but every line of SQL is right there in the stack trace. Spring Data is faster to write but hides the queries — and when something is slow or wrong, "where did this SQL come from?" becomes its own debugging session. Pick the one that matches how comfortable you are reading the lower layer.
- **DTOs are not optional the moment an entity has anything you don't want to ship.** Once a `User` has a `password` field, returning the `User` entity from a controller means that password is now in every JSON response, in every HTTP log, in every browser dev tools network tab. The DTO discipline exists for exactly this reason, and forgetting it once is a lesson you don't forget.
- **A `@RestControllerAdvice` you don't fully wire is just code that looks safe.** Having the `GlobalExceptionHandler` envelope in place made the project *feel* production-ready, but a handler that only matches one never-thrown exception is worse than no handler at all — it gives a false sense of error coverage. Either wire every path or don't add the handler yet.
- **Free-form string fields for "kinds of things" are how typos become silent bugs.** Storing `role` as `"admin"` / `"faculty"` is convenient, but the day someone writes `"Admin"` with a capital A, the query `Restrictions.eq("role", "admin")` returns nothing and there's no error. Enums don't have that failure mode.
- **A timestamp-based primary key is great for one process, terrifying for many.** A single faculty member taking attendance in a classroom has effectively zero collision risk. The same code in a multi-tenant SaaS would occasionally write two records with the same id and lose one. The choice of id strategy has to be informed by the actual concurrency model, not just by "what's easy to read in the database."

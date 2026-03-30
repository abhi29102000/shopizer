# Java Upgrade Implementation Plan — Shopizer

## Current State (Confirmed from Codebase)

| Item | Current Value |
|---|---|
| Java | 11 (`pom.xml` `<java.version>11</java.version>`) |
| Spring Boot | 2.5.12 |
| Dockerfile base | `eclipse-temurin:17-jre-alpine` ← already Java 17 JVM |
| GitHub Actions | `java-version: '17'` ← already Java 17 |
| CircleCI | `shopizerecomm/ci:java11` ← still Java 11 |
| JWT | `io.jsonwebtoken:jjwt:0.8.0` (old monolithic artifact) |
| Swagger | `io.springfox:springfox-swagger2:2.9.2` (abandoned project) |
| Drools | `7.32.0.Final` |
| Infinispan | `9.4.18.Final` |
| Hibernate | Managed by Spring Boot 2.5.12 → 5.4.x |
| MapStruct | `1.3.0.Final` |

> **Key Insight:** The Dockerfile and GitHub Actions already target Java 17, but the Maven compiler still targets Java 11 bytecode. This is a latent inconsistency that must be fixed in Phase 1.

---

## Upgrade Strategy Overview

```
Phase 1:  Java 11 → Java 17          (complete the half-done migration)
Phase 2a: Spring Boot 2.5.12 → 2.7.x (safe incremental step)
Phase 2b: Spring Boot 2.7 → 3.x      (Jakarta namespace migration)
Phase 3:  Java 17 → Java 21          (latest LTS, ZGC, virtual threads)
```

---

## Phase 1 — Java 11 → Java 17

### Compatibility Analysis

| Dependency | Java 17 Compatible? | Action Required |
|---|---|---|
| Spring Boot 2.5.12 | ✅ Yes (with `--add-opens`) | Add JVM flags |
| Hibernate 5.4.x | ✅ Yes | No change |
| `jjwt:0.8.0` | ⚠️ Works but uses internal APIs | Upgrade to `0.11.5` split artifacts |
| `springfox 2.9.2` | ❌ Broken on Spring Boot 2.6+ | Replace with springdoc-openapi |
| Drools 7.32 | ⚠️ Needs `--add-opens` flags | Add JVM args |
| Infinispan 9.4 | ⚠️ Needs `--add-opens` flags | Add JVM args |
| MapStruct 1.3.0 | ✅ Works, 1.5.x preferred | Upgrade recommended |
| `javax.*` packages | ✅ Still present in Java 17 | No change yet (Phase 2b) |

---

### 1.1 Maven Changes — Root `pom.xml`

**Change Java version and upgrade key dependencies:**

```xml
<!-- java version -->
<java.version>17</java.version>
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>

<!-- upgrade MapStruct -->
<org.mapstruct.version>1.5.5.Final</org.mapstruct.version>

<!-- upgrade JWT -->
<jwt.version>0.11.5</jwt.version>
```

**Replace single `jjwt` artifact in `dependencyManagement` with split artifacts:**

```xml
<!-- REMOVE this -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>${jwt.version}</version>
</dependency>

<!-- ADD these -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>${jwt.version}</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>${jwt.version}</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>${jwt.version}</version>
    <scope>runtime</scope>
</dependency>
```

**Replace springfox with springdoc in `dependencyManagement`:**

```xml
<!-- REMOVE these -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${swagger.version}</version>
</dependency>

<!-- ADD this -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>
```

---

### 1.2 Maven Changes — `sm-shop/pom.xml`

**Replace jjwt dependency:**
```xml
<!-- REMOVE -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
</dependency>

<!-- ADD -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
</dependency>
```

**Replace springfox dependencies:**
```xml
<!-- REMOVE -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
</dependency>

<!-- ADD -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
</dependency>
```

**Add JVM flags to Spring Boot Maven plugin for Drools/Infinispan:**
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <jvmArguments>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.util=ALL-UNNAMED
            --add-opens java.base/java.io=ALL-UNNAMED
        </jvmArguments>
    </configuration>
</plugin>
```

---

### 1.3 JWT Code Migration (jjwt 0.8.0 → 0.11.5)

The API changed between these versions. Find all affected files:

```bash
grep -rn "Jwts\.\|SignatureAlgorithm\|Claims" --include="*.java" .
```

Key API changes:

| Old (0.8.0) | New (0.11.5) |
|---|---|
| `Jwts.parser()` | `Jwts.parserBuilder()` |
| `.setSigningKey(String)` | `.setSigningKey(Key)` |
| `Keys` class not available | Use `Keys.hmacShaKeyFor(bytes)` |

Example migration:
```java
// OLD
Jwts.parser().setSigningKey(secret).parseClaimsJws(token)

// NEW
Jwts.parserBuilder()
    .setSigningKey(Keys.hmacShaKeyFor(secret.getBytes()))
    .build()
    .parseClaimsJws(token)
```

---

### 1.4 Springfox → Springdoc Migration

Find and remove all Springfox configuration:

```bash
grep -rn "Docket\|EnableSwagger2\|springfox" --include="*.java" .
```

- Remove all `@EnableSwagger2` annotations
- Remove all `Docket` bean definitions
- Springdoc auto-configures — no replacement bean needed

Add to `sm-shop/src/main/resources/application.properties`:
```properties
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```

> Swagger UI URL changes from `/swagger-ui.html` → `/swagger-ui/index.html`

---

### 1.5 CircleCI Fix

```yaml
# .circleci/config.yml
executors:
  shopizer-ci:
    docker:
      - image: eclipse-temurin:17-jdk-alpine
```

---

### 1.6 Phase 1 Validation

```bash
# 1. Build all modules
mvn clean install -DskipTests

# 2. Verify Java 17 bytecode
javap -verbose sm-shop/target/classes/com/salesmanager/shop/application/ShopApplication.class | grep "major version"
# Expected: major version 61 (= Java 17)

# 3. Run tests
mvn test -pl sm-core,sm-shop --no-transfer-progress

# 4. Start app and verify Swagger
mvn spring-boot:run -pl sm-shop
curl http://localhost:8080/swagger-ui/index.html

# 5. Test auth endpoints
curl -X POST http://localhost:8080/api/v1/private/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin@shopizer.com","password":"password"}'
```

### Phase 1 Risks

| Risk | Severity | Mitigation |
|---|---|---|
| jjwt API breaking change | MEDIUM | Grep all JWT usages before upgrading; test all auth flows |
| Springfox config beans break startup | MEDIUM | Search for `Docket`, `@EnableSwagger2`, remove them all |
| Drools reflection warnings | LOW | Add `--add-opens` flags; warnings are non-fatal |
| Infinispan 9.4 runtime errors on Java 17 | MEDIUM | May need upgrade to Infinispan 13+ if errors occur |

---

## Phase 2a — Spring Boot 2.5.12 → 2.7.x

### 2a.1 Maven Change

```xml
<!-- Root pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
</parent>
```

### 2a.2 Breaking Changes to Handle

**1. Circular bean dependencies** — Spring Boot 2.6+ fails on circular refs by default. If startup fails:
```properties
# application.properties
spring.main.allow-circular-references=true
```

**2. H2 2.x SQL compatibility** — Spring Boot 2.7 bundles H2 2.x which has breaking SQL changes. If H2 dev profile fails:
```properties
spring.datasource.url=jdbc:h2:file:./SALESMANAGER;MODE=LEGACY;DB_CLOSE_ON_EXIT=FALSE
```

**3. Actuator metrics** — `management.metrics.export.*` properties renamed to `management.prometheus.metrics.export.*`. Check your actuator config.

### 2a.3 Phase 2a Validation

```bash
mvn clean install -DskipTests
mvn spring-boot:run -pl sm-shop
# Run full API smoke test
# Test H2 dev profile startup
```

### Phase 2a Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Circular bean dependency failures | MEDIUM | Add `allow-circular-references=true` temporarily |
| H2 2.x SQL syntax changes | LOW | Add `MODE=LEGACY` to H2 URL |
| Dependency version conflicts | LOW | Run `mvn dependency:tree` to inspect |

---

## Phase 2b — Spring Boot 2.7 → 3.x (Jakarta Migration)

This is the highest-risk phase. Spring Boot 3.x requires:
- Java 17 minimum ✅ (done in Phase 1)
- `javax.*` → `jakarta.*` namespace across all source files
- Hibernate 6.x (significant query behavior changes)
- Spring Security 6.x (config API rewrite)

### 2b.1 Maven Change

```xml
<!-- Root pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.12</version>
</parent>
```

### 2b.2 Run OpenRewrite Migration (Automates ~80%)

```bash
mvn org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2
```

Review all changes made by OpenRewrite before committing.

### 2b.3 Manual Jakarta Namespace Migration

Find remaining `javax.*` references after OpenRewrite:

```bash
grep -rn "import javax\." --include="*.java" .
```

Key replacements across all modules:

| Old | New |
|---|---|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.inject.*` | `jakarta.inject.*` |
| `javax.annotation.*` | `jakarta.annotation.*` |
| `javax.mail.*` | `jakarta.mail.*` |

Also update `pom.xml` dependency group/artifact IDs where applicable (e.g. `javax.inject:javax.inject` → `jakarta.inject:jakarta.inject-api`).

### 2b.4 Spring Security 6 Migration

`WebSecurityConfigurerAdapter` is removed in Spring Security 6. Find all usages:

```bash
grep -rn "WebSecurityConfigurerAdapter" --include="*.java" .
```

Migrate from:
```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception { ... }
}
```

To:
```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception { ... }
}
```

### 2b.5 Hibernate 6 Changes

Hibernate 6 (bundled in Spring Boot 3.x) has breaking changes:

- `@Type(type="...")` string-based type references removed → use `@Type(JsonType.class)` style
- `@Column(columnDefinition="...")` behavior may differ
- HQL/JPQL stricter parsing

Run full integration tests to surface Hibernate issues:
```bash
mvn test -pl sm-core,sm-shop --no-transfer-progress
```

### 2b.6 Drools Upgrade (7.x → 8.x)

Drools 7.x is not compatible with Spring Boot 3. Upgrade in `pom.xml`:

```xml
<drools.version>8.44.0.Final</drools.version>
```

Drools 8 has API changes — review all `KieSession`, `KieContainer`, `KieServices` usages:
```bash
grep -rn "KieSession\|KieContainer\|KieServices\|KieBase" --include="*.java" .
```

### 2b.7 Infinispan Upgrade (9.4 → 14.x)

```xml
<infinispan.version>14.0.27.Final</infinispan.version>
<infinispan.tree.version>14.0.27.Final</infinispan.tree.version>
```

Infinispan 14 has configuration API changes. Review cache configuration beans.

### 2b.8 Springdoc Upgrade for Spring Boot 3

```xml
<!-- REMOVE -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>

<!-- ADD -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

### Phase 2b Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Drools 7 → 8 API changes | HIGH | Upgrade incrementally; test all pricing/discount rules |
| Hibernate 6 query behavior changes | HIGH | Run full integration test suite; inspect all HQL queries |
| Spring Security config rewrite | MEDIUM | Use OpenRewrite; test all auth and authorization flows |
| Jakarta namespace in shopizer-commons / canadapost starter | MEDIUM | Check if those starters publish Jakarta-compatible versions |
| H2 2.x in dev profile | LOW | Add `MODE=LEGACY` to H2 URL |

---

## Phase 3 — Java 17 → Java 21

Only start after Phase 2b is stable in production.

### 3.1 Maven Change

```xml
<!-- Root pom.xml -->
<java.version>21</java.version>
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
```

### 3.2 Dockerfile Update

```dockerfile
FROM eclipse-temurin:21-jre-alpine
RUN mkdir /opt/app /files
COPY target/shopizer.jar /opt/app
COPY SALESMANAGER.h2.db /
COPY ./files /files
CMD ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseZGC", \
  "--add-opens", "java.base/java.lang=ALL-UNNAMED", \
  "-jar", "/opt/app/shopizer.jar"]
```

### 3.3 GitHub Actions Update

```yaml
# .github/workflows/ci.yml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: temurin
    cache: maven
```

### 3.4 CircleCI Update

```yaml
executors:
  shopizer-ci:
    docker:
      - image: eclipse-temurin:21-jdk-alpine
```

### 3.5 Optional: Enable Virtual Threads (Spring Boot 3.2+)

Virtual threads (Project Loom) improve throughput for REST API workloads with no code changes:

```properties
# application.properties
spring.threads.virtual.enabled=true
```

### 3.6 Java 21 Considerations

| Feature | Impact on Shopizer |
|---|---|
| Virtual Threads (Loom) | Beneficial — REST API thread-per-request model maps well |
| ZGC (production-ready) | Lower GC pause times vs G1GC; good for API latency |
| Sequenced Collections | Additive only, no breaking changes |
| Pattern matching for switch | Additive only, no breaking changes |
| `--add-opens` flags | Can be removed one by one and tested |

### Phase 3 Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Virtual threads + blocking code | LOW | Monitor thread dumps; avoid `synchronized` blocks in hot paths |
| ZGC memory overhead | LOW | Monitor heap usage; fallback to G1GC if needed |
| Remaining `--add-opens` requirements | LOW | Remove flags one at a time and test |

---

## Infrastructure Changes

### Kubernetes / Colima

No Kubernetes manifest changes required for the Java upgrade. Verify resource limits are appropriate for JVM heap sizing:

```yaml
# deployment.yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

With `MaxRAMPercentage=75.0` and a 1Gi limit, JVM heap = ~768Mi.

Colima on Apple Silicon (ARM64) is fully supported — `eclipse-temurin:17-jre-alpine` and `eclipse-temurin:21-jre-alpine` both publish multi-arch images (`linux/amd64` + `linux/arm64`).

Verify:
```bash
docker buildx imagetools inspect eclipse-temurin:21-jre-alpine | grep -i platform
```

---

## CI/CD Image Tagging Strategy

```yaml
- name: Build and push Docker image
  run: |
    docker build \
      -t shopizerecomm/shopizer:${{ github.sha }} \
      -t shopizerecomm/shopizer:java21 \
      -t shopizerecomm/shopizer:latest \
      sm-shop/
    docker push shopizerecomm/shopizer:${{ github.sha }}
    docker push shopizerecomm/shopizer:java21
    docker push shopizerecomm/shopizer:latest
```

Always tag with SHA in addition to `latest` to enable rollback.

---

## Branching Strategy

```
main
├── feature/java17-upgrade        ← Phase 1
├── feature/springboot-27         ← Phase 2a
├── feature/springboot-3x         ← Phase 2b (Jakarta)
└── feature/java21-upgrade        ← Phase 3
```

Each branch merges to `main` only after:
1. All CI tests pass
2. Docker image built and smoke-tested locally
3. Deployed to staging namespace in Colima and validated

---

## Rollback Strategy

### Kubernetes Rollback

```bash
# Roll back to previous deployment
kubectl rollout undo deployment/shopizer

# Or pin to a specific image SHA
kubectl set image deployment/shopizer \
  shopizer=shopizerecomm/shopizer:<previous-sha>

# Verify rollback
kubectl rollout status deployment/shopizer
```

### Image Rollback

- Never overwrite a SHA-tagged image
- Keep at least 3 previous image tags in the registry
- Tag format: `shopizerecomm/shopizer:<git-sha>` + `shopizerecomm/shopizer:java17` etc.

### Git Rollback

```bash
# Revert a merged phase if issues found in production
git revert -m 1 <merge-commit-sha>
git push origin main
```

---

## Exact Execution Order

### Phase 1 Steps

```
1.  git checkout -b feature/java17-upgrade

2.  pom.xml: java.version 11 → 17

3.  pom.xml: mapstruct 1.3.0.Final → 1.5.5.Final

4.  pom.xml: jwt 0.8.0 → 0.11.5, replace single jjwt with split artifacts

5.  sm-shop/pom.xml: update jjwt dependency to jjwt-api

6.  Find JWT usages:
    grep -rn "Jwts\.\|SignatureAlgorithm\|Claims" --include="*.java" .

7.  Migrate JWT code from 0.8.0 API to 0.11.5 API

8.  pom.xml: remove springfox, add springdoc-openapi-ui:1.7.0

9.  sm-shop/pom.xml: replace springfox with springdoc

10. Find Swagger config beans:
    grep -rn "Docket\|EnableSwagger2" --include="*.java" .

11. Remove all Docket beans and @EnableSwagger2 annotations

12. application.properties: add spring.mvc.pathmatch.matching-strategy=ant_path_matcher

13. sm-shop/pom.xml: add --add-opens JVM args to spring-boot-maven-plugin

14. .circleci/config.yml: update CI image to eclipse-temurin:17-jdk-alpine

15. mvn clean install -DskipTests

16. mvn test -pl sm-core,sm-shop --no-transfer-progress

17. mvn spring-boot:run -pl sm-shop → verify /swagger-ui/index.html + auth

18. git commit, push, open PR → merge to main
```

### Phase 2a Steps

```
1.  git checkout -b feature/springboot-27

2.  pom.xml: spring-boot-starter-parent 2.5.12 → 2.7.18

3.  mvn clean install -DskipTests

4.  Fix compilation errors if any

5.  application.properties: add spring.main.allow-circular-references=true if startup fails

6.  Test H2 dev profile; add MODE=LEGACY to H2 URL if SQL errors

7.  mvn test -pl sm-core,sm-shop --no-transfer-progress

8.  Deploy to staging, run API smoke tests

9.  git commit, push, open PR → merge to main
```

### Phase 2b Steps

```
1.  git checkout -b feature/springboot-3x

2.  Run OpenRewrite recipe:
    mvn org.openrewrite.maven:rewrite-maven-plugin:run \
      -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
      -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2

3.  Review all OpenRewrite changes

4.  pom.xml: spring-boot-starter-parent 2.7.18 → 3.2.12

5.  Check remaining javax.* references:
    grep -rn "import javax\." --include="*.java" .

6.  Manually fix remaining javax.* → jakarta.* imports

7.  Upgrade Drools: drools.version 7.32.0.Final → 8.44.0.Final

8.  Fix Drools API changes (KieSession, KieContainer usages)

9.  Upgrade Infinispan: 9.4.18.Final → 14.0.27.Final

10. Fix Infinispan cache configuration beans

11. Upgrade springdoc: springdoc-openapi-ui:1.7.0 → springdoc-openapi-starter-webmvc-ui:2.3.0

12. Fix Spring Security config (WebSecurityConfigurerAdapter removal)

13. mvn clean install -DskipTests

14. Fix compilation errors iteratively

15. mvn test -pl sm-core,sm-shop --no-transfer-progress

16. Extended regression testing on all API endpoints

17. Deploy to staging, run full API smoke test

18. git commit, push, open PR → merge to main
```

### Phase 3 Steps

```
1.  git checkout -b feature/java21-upgrade

2.  pom.xml: java.version 17 → 21

3.  Dockerfile: eclipse-temurin:17-jre-alpine → 21-jre-alpine

4.  Dockerfile CMD: add -XX:+UseZGC, -XX:MaxRAMPercentage=75.0

5.  GitHub Actions: java-version '17' → '21'

6.  CircleCI: update executor image to eclipse-temurin:21-jdk-alpine

7.  Optional: application.properties: spring.threads.virtual.enabled=true

8.  mvn clean install

9.  mvn test -pl sm-core,sm-shop --no-transfer-progress

10. Load test to validate ZGC behavior

11. Deploy to staging, monitor GC logs and response times

12. git commit, push, open PR → merge to main
```

---

## Testing Strategy

### Per Phase

| Test Type | Tool | When |
|---|---|---|
| Unit tests | JUnit 5 / Mockito | Every commit |
| Integration tests | Spring Boot Test + H2 | Every PR |
| API regression | Postman / curl scripts | After each phase deploy |
| Security tests | Auth endpoint smoke tests | After Phase 1 (JWT change) |
| Load tests | k6 / JMeter | Phase 3 (ZGC validation) |

### Key Endpoints to Smoke Test After Each Phase

```bash
# Auth
POST /api/v1/private/login

# Catalog
GET  /api/v1/products
GET  /api/v1/category

# Cart
POST /api/v1/cart
GET  /api/v1/cart/{code}

# Swagger
GET  /swagger-ui/index.html
GET  /v3/api-docs
```

---

## Summary & Priority

| Phase | Risk | Effort | Priority |
|---|---|---|---|
| Phase 1: Java 17 compiler fix | LOW | Small | **Do immediately** — fixes latent inconsistency |
| Phase 2a: Spring Boot 2.7 | LOW | Small | Do right after Phase 1 stabilizes |
| Phase 2b: Spring Boot 3.x | HIGH | Large | Budget most time; Drools + Jakarta are the hard parts |
| Phase 3: Java 21 | LOW | Small | Straightforward once on Boot 3.x |

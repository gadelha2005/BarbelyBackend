# CLAUDE.md — Barberly API Backend

> Guia de contexto para agentes de desenvolvimento Java/Spring.  
> **Escopo:** Java 21 + Spring Boot 3.3 + MySQL 8.0 + JUnit 5 + Mockito  
> **Local:** `/barbershop-api` no repositório

---

## Visão Geral do Projeto

Backend do **Barberly** — sistema SaaS de agendamentos para barbearias. API REST que gerencia:

- **Usuários** — clientes e profissionais com roles (CLIENT, PROFESSIONAL, ADMIN)
- **Serviços** — corte, barba, etc. com preço e duração
- **Disponibilidade** — blocos de horário por profissional (ex: Seg-Sex 09:00-18:00)
- **Agendamentos** — marca-se um serviço com um profissional, com **detecção automática de conflito** ⚠️
- **Avaliações** — clientes avaliam profissionais após agendamento concluído

**Objetivo técnico:** demonstrar ciclo completo (modelagem → API segura → testes → deploy) em nível profissional.

---

## Stack Técnico

| Camada | Tecnologia | Versão |
|---|---|---|
| Linguagem | Java | 21 (LTS) |
| Framework | Spring Boot | 3.3.0 |
| Segurança | Spring Security + JWT | 0.12.3 (jjwt) |
| ORM | Spring Data JPA + Hibernate | via Boot |
| Banco | MySQL | 8.0 |
| Validação | Bean Validation (@Valid) | via Boot |
| Documentação | Springdoc OpenAPI | 2.1.0 |
| Testes | JUnit 5 + Mockito | via Boot |
| Produtividade | Lombok | via Boot |
| Build | Maven | 3.9+ |

---

## Arquitetura de Pacotes

```
src/main/java/com/barbershop/
├── config/
│   ├── SecurityConfig.java              # Spring Security + JWT
│   └── OpenApiConfig.java               # Swagger UI
├── controller/
│   ├── AuthController.java              # /auth/register, /auth/login
│   ├── AppointmentController.java       # /appointments (CRUD + ações)
│   ├── ProfessionalController.java      # /professionals + /availability
│   ├── ServiceController.java           # /services (CRUD)
│   └── ReviewController.java            # /appointments/{id}/review
├── service/
│   ├── AuthService.java                 # register(), login()
│   ├── AppointmentService.java          # ⚠️ LÓGICA ANTI-CONFLITO aqui
│   ├── ProfessionalService.java         # listagem, perfil, ratings
│   ├── ServiceService.java              # CRUD de serviços
│   ├── AvailabilityService.java         # gerenciar disponibilidade
│   └── ReviewService.java               # validar e salvar reviews
├── repository/
│   ├── AppointmentRepository.java       # ⚠️ Query JPQL anti-conflito
│   ├── UserRepository.java              # findByEmail(), etc
│   ├── AvailabilityRepository.java      # queries de disponibilidade
│   ├── ServiceRepository.java
│   └── ReviewRepository.java
├── entity/
│   ├── User.java                        # Cliente + Profissional + Admin
│   ├── Service.java                     # Serviço (corte, barba, etc)
│   ├── Availability.java                # Blocos de horário
│   ├── Appointment.java                 # Agendamento (regra crítica)
│   └── Review.java                      # Avaliação
├── dto/
│   ├── request/
│   │   ├── RegisterRequest.java
│   │   ├── LoginRequest.java
│   │   ├── AppointmentCreateRequest.java
│   │   ├── AvailabilityRequest.java
│   │   └── ReviewRequest.java
│   └── response/
│       ├── AuthResponse.java
│       ├── AppointmentResponse.java
│       ├── UserResponse.java
│       ├── ServiceResponse.java
│       └── ReviewResponse.java
├── exception/
│   ├── GlobalExceptionHandler.java      # @ControllerAdvice central
│   ├── SlotUnavailableException.java    # 409 — conflito de horário
│   ├── ResourceNotFoundException.java   # 404
│   ├── UnauthorizedException.java       # 403 — role insufficiente
│   ├── PastDateException.java           # 422 — data no passado
│   ├── InvalidAvailabilityException.java # 400 — endTime <= startTime
│   └── ReviewAlreadyExistsException.java # 409 — review duplicada
├── security/
│   ├── JwtService.java                  # gerar token, extrair claims, validar
│   ├── JwtAuthenticationFilter.java     # OncePerRequestFilter
│   └── CustomUserDetailsService.java    # loadUserByUsername()
└── util/
    └── AppointmentUtil.java             # helper para cálculos (ex: endsAt)

src/test/java/com/barbershop/
├── service/
│   ├── AppointmentServiceTest.java      # 4+ cenários obrigatórios
│   ├── AuthServiceTest.java
│   └── ProfessionalServiceTest.java
└── repository/
    ├── AppointmentRepositoryTest.java   # @DataJpaTest
    └── UserRepositoryTest.java
```

---

## Entidades e Relacionamentos

### User (Autenticação e Identificação)

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank private String name;
    @Email @NotBlank @Column(unique = true) private String email;
    @NotBlank private String password;        // BCrypt hash
    
    private String phone;                      // opcional
    @Enumerated(EnumType.STRING) @NotNull private Role role;
    @CreationTimestamp private LocalDateTime createdAt;
}
```

| Campo | Validação | Nota |
|---|---|---|
| `id` | PK autoincrement | gerado |
| `name` | @NotBlank | nome completo |
| `email` | @Email, @NotBlank, unique | login |
| `password` | @NotBlank | armazenado em BCrypt hash |
| `phone` | opcional | contato |
| `role` | CLIENT \| PROFESSIONAL \| ADMIN | não é herança, é composição |
| `createdAt` | gerado | timestamp |

### Service (Serviço Oferecido)

```java
@Entity
@Table(name = "services")
public class Service {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank private String name;           // "Corte masculino"
    private String description;
    @Min(5) private Integer durationMinutes;  // mínimo 5 min
    @DecimalMin("0") private BigDecimal price;
    @Builder.Default private Boolean active = true;  // soft delete
}
```

**Relacionamento:** N:M com User via tabela `professional_services`

### Availability (Blocos de Horário)

```java
@Entity
@Table(name = "availability")
public class Availability {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne @NotNull private User professional;
    @Enumerated(EnumType.STRING) @NotNull private DayOfWeek dayOfWeek;
    @NotNull private LocalTime startTime;
    @NotNull private LocalTime endTime;      // ⚠️ DEVE ser > startTime
}
```

**Validação crítica:** `endTime > startTime` — implementar como `@Constraint` customizado no DTO.

### Appointment (Agendamento — Regra Crítica)

```java
@Entity
@Table(name = "appointments")
public class Appointment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne @NotNull private User client;
    @ManyToOne @NotNull private User professional;
    @ManyToOne @NotNull private Service service;
    
    @Future @NotNull private LocalDateTime scheduledAt;
    private LocalDateTime endsAt;              // calculado: scheduledAt + duration
    
    @Enumerated(EnumType.STRING)
    @Builder.Default
    private AppointmentStatus status = AppointmentStatus.PENDING;
    
    private String notes;
    @CreationTimestamp private LocalDateTime createdAt;
}
```

**⚠️ REGRA DE NEGÓCIO CRÍTICA:**
> Nenhum profissional pode ter dois agendamentos cujos intervalos `[scheduledAt, endsAt]` se sobreponham.

Implementar no `AppointmentService.create()` usando a query JPQL do `AppointmentRepository`.

### Review (Avaliação)

```java
@Entity
@Table(name = "reviews")
public class Review {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToOne @NotNull private Appointment appointment;  // só 1 review por appointment
    @ManyToOne @NotNull private User client;             // derivado do appointment
    @ManyToOne @NotNull private User professional;       // derivado do appointment
    
    @Min(1) @Max(5) @NotNull private Integer rating;
    private String comment;
    @CreationTimestamp private LocalDateTime createdAt;
}
```

**Validação:** cliente só pode avaliar após `appointment.status == COMPLETED`.

### Diagrama ER

```
User (1) ────────────── (N) Appointment [como client]
User (1) ────────────── (N) Appointment [como professional]
User (N) ────────────── (N) Service [tabela: professional_services]
User (1) ────────────── (N) Availability
Appointment (1) ──────── (1) Service
Appointment (1) ──────── (0..1) Review
```

---

## Endpoints da API

### Authentication — `/auth`

| Método | Endpoint | Corpo | Response | HTTP |
|---|---|---|---|---|
| POST | `/auth/register` | `RegisterRequest` | `AuthResponse + JWT` | 201 |
| POST | `/auth/login` | `LoginRequest` | `AuthResponse + JWT` | 200 |

**Fluxo JWT:**
1. Cliente faz login ou registra
2. Servidor gera token HS256 com claims (id, email, role)
3. Cliente armazena em localStorage
4. A cada request, envia `Authorization: Bearer <token>`
5. `JwtAuthenticationFilter` valida antes do controller

---

### Profissionais — `/professionals`

| Método | Endpoint | Auth | Descrição | HTTP |
|---|---|---|---|---|
| GET | `/professionals` | público | Listar com rating médio | 200 |
| GET | `/professionals/{id}` | público | Perfil + serviços + avaliações | 200 |
| GET | `/professionals/{id}/availability` | público | Slots **livres** em uma data | 200 |
| POST | `/professionals/{id}/availability` | PROFESSIONAL | Criar bloco de disponibilidade | 201 |
| DELETE | `/professionals/{id}/availability/{avId}` | PROFESSIONAL | Remover disponibilidade | 204 |

**Query para `/professionals/{id}/availability?date=2025-05-01`:**
```sql
SELECT av FROM Availability av
LEFT JOIN Appointment a ON av.professional = a.professional
WHERE av.professional.id = :id
  AND av.dayOfWeek = :dayOfWeek
  AND (a.id IS NULL OR conflito não existe)
ORDER BY av.startTime
```

---

### Serviços — `/services`

| Método | Endpoint | Auth | Descrição | HTTP |
|---|---|---|---|---|
| GET | `/services` | público | Listar ativos | 200 |
| GET | `/services/{id}` | público | Detalhes | 200 |
| POST | `/services` | ADMIN | Criar | 201 |
| PUT | `/services/{id}` | ADMIN | Atualizar | 200 |
| DELETE | `/services/{id}` | ADMIN | Soft delete (active=false) | 204 |

---

### Agendamentos — `/appointments`

| Método | Endpoint | Auth | Descrição | HTTP |
|---|---|---|---|---|
| POST | `/appointments` | CLIENT | Criar agendamento | 201 |
| GET | `/appointments` | autenticado | Meus agendamentos (paginado) | 200 |
| GET | `/appointments/{id}` | autenticado | Detalhes | 200 |
| PUT | `/appointments/{id}/confirm` | PROFESSIONAL | Confirmar (PENDING → CONFIRMED) | 200 |
| PUT | `/appointments/{id}/cancel` | CLIENT ou PROFESSIONAL | Cancelar | 200 |
| PUT | `/appointments/{id}/complete` | PROFESSIONAL | Marcar concluído (CONFIRMED → COMPLETED) | 200 |
| POST | `/appointments/{id}/review` | CLIENT | Avaliar (apenas se COMPLETED) | 201 |

**Transições de status:**
```
PENDING ──(PROFESSIONAL confirma)──> CONFIRMED ──(PROFESSIONAL completa)──> COMPLETED
   ↓ (qualquer um cancela)            ↓ (qualquer um cancela)
CANCELLED                            CANCELLED
```

---

## Exceções e Tratamento de Erros

**Padrão:** todas as exceções são capturadas por `GlobalExceptionHandler` (um `@ControllerAdvice` central).

### Mapa de Exceções

| Exceção | HTTP | Cenário | Lançada em |
|---|---|---|---|
| `SlotUnavailableException` | **409** | Horário ocupado pelo profissional | `AppointmentService.create()` |
| `ResourceNotFoundException` | **404** | Entidade não encontrada | `*Service.findById()` |
| `UnauthorizedException` | **403** | Ação não permitida (role) | `AppointmentService.confirm/cancel/complete()` |
| `PastDateException` | **422** | `scheduledAt` no passado | `AppointmentService.create()` |
| `InvalidAvailabilityException` | **400** | `endTime <= startTime` | `AvailabilityService.create()` |
| `ReviewAlreadyExistsException` | **409** | Appointment já avaliado | `ReviewService.create()` |
| `MethodArgumentNotValidException` | **400** | `@Valid` falhou | Spring (automático) |
| `AuthenticationException` | **401** | Token inválido/expirado | Spring Security |

### Formato padrão de erro

```json
{
  "timestamp": "2025-04-20T10:30:00Z",
  "status": 409,
  "error": "SLOT_UNAVAILABLE",
  "message": "O profissional já possui agendamento no horário solicitado",
  "path": "/appointments"
}
```

**Para erros de validação (@Valid):**
```json
{
  "timestamp": "2025-04-20T10:30:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Validação falhou",
  "details": [
    {
      "field": "scheduledAt",
      "message": "deve ser uma data futura"
    }
  ],
  "path": "/appointments"
}
```

---

## Query JPQL Anti-Conflito ⚠️

**CRÍTICA:** esta query determina se um agendamento conflita com outro para o mesmo profissional.

```java
// AppointmentRepository.java

@Query("""
    SELECT COUNT(a) > 0 FROM Appointment a
    WHERE a.professional.id = :professionalId
      AND a.status NOT IN ('CANCELLED')
      AND a.scheduledAt < :endsAt
      AND a.endsAt > :scheduledAt
""")
boolean existsConflict(
    @Param("professionalId") Long professionalId,
    @Param("scheduledAt") LocalDateTime scheduledAt,
    @Param("endsAt") LocalDateTime endsAt
);
```

**Lógica:** dois intervalos se sobrepõem se:
- `start1 < end2` **AND** `end1 > start2`

**Uso no Service:**
```java
if (appointmentRepository.existsConflict(professional.getId(), scheduled, endsAt)) {
    throw new SlotUnavailableException("Conflito de horário");
}
```

---

## Fluxo de uma Request HTTP

```
┌─────────────────────────────────────────────┐
│ 1. Cliente envia: POST /appointments        │
│    Authorization: Bearer <JWT>              │
│    Body: { professional, service, ...}      │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 2. JwtAuthenticationFilter                  │
│    - extrai token do header                 │
│    - valida assinatura                      │
│    - carrega User no SecurityContext        │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 3. AppointmentController.create()           │
│    - recebe @RequestBody                    │
│    - chama @Valid (valida DTO)              │
│    - delega para service                    │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 4. AppointmentService.create()              │
│    - calcula endsAt                         │
│    - valida regras de negócio               │
│    - **checa conflito com query JPQL**      │
│    - lança exceção ou segue                 │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 5. AppointmentRepository.save()             │
│    - executa INSERT no banco                │
│    - retorna entity salva                   │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 6. Entity → DTO (mapper)                    │
│    - converte Appointment para              │
│      AppointmentResponse                    │
│    - NUNCA retornar Entity diretamente      │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 7. HTTP 201 Created                         │
│    Content-Type: application/json           │
│    Body: AppointmentResponse                │
└─────────────────────────────────────────────┘
```

**Se alguma exceção ocorrer:**
→ `GlobalExceptionHandler` captura → formata resposta → retorna com status apropriado

---

## Segurança

### JWT (Spring Security + jjwt)

- **Algoritmo:** HS256 (HMAC com SHA-256)
- **Secret:** mínimo **256 bits** (32 caracteres)
- **Expiração:** sugestão 24h para tokens de acesso
- **Claims:** `id`, `email`, `role`
- **Armazenamento:** variável de ambiente `JWT_SECRET`

**Nunca hardcoded!**

```properties
# application.properties
jwt.secret=${JWT_SECRET}
jwt.expiration=86400000  # 24h em ms
```

### Autenticação e Autorização

```java
// SecurityConfig.java

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf().disable()
        .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .and()
        .authorizeHttpRequests()
            .requestMatchers("/auth/**", "/services/**", "/professionals/**").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        .and()
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
        .exceptionHandling()
            .authenticationEntryPoint((req, res, ex) -> {
                res.setStatus(401);
                res.getWriter().write("Unauthorized");
            });
    
    return http.build();
}
```

### Passwords

- **Encoding:** BCrypt (Spring Security faz automaticamente com `PasswordEncoder`)
- **Força:** mínimo 10 rounds (padrão do Spring)

```java
// No AuthService
user.setPassword(passwordEncoder.encode(password));
```

### CORS

Em produção (Vercel), configurar no `SecurityConfig`:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(Arrays.asList(
        "http://localhost:3000",                          // dev
        "https://barbershop-frontend.vercel.app"         // prod
    ));
    config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(Arrays.asList("*"));
    config.setAllowCredentials(true);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

---

## Docker e Desenvolvimento Local

### Dockerfile (Multi-stage)

```dockerfile
# Fase 1: Build
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src src/
RUN mvn clean package -DskipTests

# Fase 2: Runtime (JRE apenas)
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Resultado:** imagem reduzida de ~800 MB para ~200 MB.

### docker-compose.yml (Desenvolvimento)

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: barbershop_db
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build: ./barbershop-api
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/barbershop_db
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: apppass
      JWT_SECRET: ${JWT_SECRET:-dev-secret-key-change-in-production}
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      SPRING_JPA_SHOW_SQL: "true"
    depends_on:
      mysql:
        condition: service_healthy

volumes:
  mysql-data:
```

### Comandos Úteis

```bash
# Build + subir tudo
docker-compose up --build

# Subir em background
docker-compose up -d

# Logs em tempo real
docker-compose logs -f backend

# Acessar MySQL dentro do container
docker exec -it <container_id> mysql -u root -p

# Parar e limpar volumes
docker-compose down -v

# Rebuild sem cache
docker-compose build --no-cache
```

---

## Deploy — Railway (Backend + MySQL)

### Passo a Passo

1. Criar conta em `railway.app`
2. New Project → Empty Project
3. Add Service → Database → MySQL (Railway provisiona automaticamente)
4. Add Service → GitHub Repo → selecionar `barbershop-api`
5. Railway detecta `Dockerfile` e usa automaticamente
6. Em Settings do serviço backend, configurar variáveis de ambiente

### Variáveis de Ambiente — Railway

| Variável | Valor |
|---|---|
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://${{MYSQL_HOST}}:${{MYSQL_PORT}}/${{MYSQL_DATABASE}}` |
| `SPRING_DATASOURCE_USERNAME` | `${{MYSQL_USER}}` |
| `SPRING_DATASOURCE_PASSWORD` | `${{MYSQL_PASSWORD}}` |
| `SPRING_JPA_HIBERNATE_DDL_AUTO` | `update` |
| `JWT_SECRET` | chave segura gerada aleatoriamente |
| `SERVER_PORT` | `8080` |

> As variáveis `${{MYSQL_*}}` são preenchidas automaticamente pelo Railway. Não copiar manualmente.

### Resultado

- URL pública: `https://barbershop-api.railway.app`
- Deploy automático a cada push no `main`
- Rollback com um clique no painel

### Verificar Deploy

```bash
curl https://barbershop-api.railway.app/swagger-ui/index.html
```

Deve carregar a documentação Swagger.

---

## Fases de Implementação

### Fase 1 — Setup e Modelagem de Dados

**Objetivo:** Projeto Spring Boot funcional com todas as entidades mapeadas.

**Passos:**
1. Criar projeto via `start.spring.io` com dependências listadas no `pom.xml`
2. Configurar `application.yml` (datasource, JPA, porta)
3. Criar arquivo `docker-compose.yml`
4. Mapear `User`, `Service`, `Availability`, `Appointment`, `Review` com `@Entity`, `@Table`, `@Column`
5. Subir `docker-compose up` e verificar schema criado

**Definition of Done:**
- ✅ `docker-compose up` sobe sem erro
- ✅ Tabelas existem no MySQL
- ✅ Projeto compila sem erros
- ✅ `spring.jpa.show-sql=true` mostra queries de criação

**Checklist:**
- [ ] `pom.xml` atualizado
- [ ] `application.properties` configurado
- [ ] `docker-compose.yml` criado e testado
- [ ] Todas as 5 entidades mapeadas
- [ ] Banco criado com sucesso

---

### Fase 2 — Autenticação e Segurança JWT

**Objetivo:** Endpoints `/auth/register` e `/auth/login` funcionando com JWT.

**Passos:**
1. Criar `JwtService` (gerar, validar, extrair token)
2. Criar `JwtAuthenticationFilter extends OncePerRequestFilter`
3. Configurar `SecurityConfig` (httpBasic off, csrf off, regras por rota)
4. Criar `AuthController` com endpoints
5. Criar `AuthService` (register, login com BCrypt)
6. Criar DTOs: `RegisterRequest`, `LoginRequest`, `AuthResponse`
7. Testar via Postman/Swagger

**Definition of Done:**
- ✅ `POST /auth/register` → 201 + JWT
- ✅ `POST /auth/login` → 200 + JWT
- ✅ `GET /services` sem token → 401
- ✅ `GET /services` com token válido → 200
- ✅ Token expirado → 401
- ✅ JWT_SECRET carregado via env var

**Checklist:**
- [ ] `JwtService` implementado
- [ ] `JwtAuthenticationFilter` funcionando
- [ ] `SecurityConfig` com regras corretas
- [ ] `AuthController` testado
- [ ] Swagger documentando endpoints

---

### Fase 3 — CRUD de Serviços e Profissionais

**Objetivo:** Endpoints de Services e Professionals com CRUD completo.

**Passos:**
1. Criar `ServiceRepository`, `ServiceService`, `ServiceController`
2. Criar DTOs: `ServiceRequest`, `ServiceResponse`
3. Implementar validações no DTO
4. Criar `ProfessionalController` com listagem e detalhes
5. Implementar `AvailabilityRequest` com `@Constraint` customizado (endTime > startTime)
6. Criar `GlobalExceptionHandler` com `@ControllerAdvice`

**Definition of Done:**
- ✅ `GET /services` (sem auth)
- ✅ `POST /services` (ADMIN only)
- ✅ `PUT /services/{id}` (ADMIN only)
- ✅ `DELETE /services/{id}` (soft delete)
- ✅ `AvailabilityRequest` com endTime <= startTime → 400
- ✅ Entidades **nunca** retornadas diretamente (sempre DTO)
- ✅ Swagger documenta todos os endpoints

**Checklist:**
- [ ] `ServiceService` com lógica de soft delete
- [ ] `ServiceController` com todas as rotas
- [ ] `ProfessionalController` com listagem
- [ ] `AvailabilityService` implementado
- [ ] DTOs de request e response criados
- [ ] `GlobalExceptionHandler` capturando exceções

---

### Fase 4 — Lógica de Agendamentos (Anti-Conflito) ⚠️

**Objetivo:** Agendamento com detecção de conflito e transição de status.

**Passos:**
1. Criar `AppointmentRepository` com query customizada de conflito
2. Implementar a query JPQL (vide seção "Query JPQL Anti-Conflito")
3. Criar `AppointmentService.create()` — calcular endsAt, checar conflito, salvar
4. Implementar `confirm()`, `cancel()`, `complete()` com validação de role
5. Criar `AppointmentController`
6. Lançar exceções apropriadas:
   - `SlotUnavailableException` (409) se conflito
   - `UnauthorizedException` (403) se role insuficiente
   - `PastDateException` (422) se data no passado
7. Mapear exceções no `GlobalExceptionHandler`

**Definition of Done:**
- ✅ Dois agendamentos no mesmo horário → 409
- ✅ Agendamento no passado → 422
- ✅ CLIENT tenta confirmar → 403
- ✅ Transição PENDING → CONFIRMED → COMPLETED funciona
- ✅ CANCELLED não cria novos conflitos

**Checklist:**
- [ ] `AppointmentRepository.existsConflict()` implementada
- [ ] `AppointmentService.create()` com lógica anti-conflito
- [ ] Métodos de transição de status (confirm, cancel, complete)
- [ ] Exceções testadas isoladamente
- [ ] `AppointmentController` com todos os endpoints

---

### Fase 5 — Queries Avançadas, Reviews e Disponibilidade

**Objetivo:** Query de disponibilidade, ranking por rating, Reviews.

**Passos:**
1. Implementar query de disponibilidade (LEFT JOIN)
   - Dado `professionalId` + `date`, retornar blocos livres
2. Implementar ranking (AVG rating no repository)
3. Criar `ReviewService` (validar appointment.status == COMPLETED)
4. Criar `ReviewController` (POST `/appointments/{id}/review`)
5. Adicionar paginação nos endpoints (Pageable)

**Definition of Done:**
- ✅ `GET /professionals/{id}/availability?date=2025-05-01` retorna slots livres
- ✅ `GET /professionals` ordenado por rating médio
- ✅ Cliente não consegue avaliar PENDING (deve ser COMPLETED)
- ✅ `GET /appointments?page=0&size=10` com paginação

**Checklist:**
- [ ] Query de disponibilidade com LEFT JOIN
- [ ] Rating calculado corretamente
- [ ] `ReviewService` validando status
- [ ] Paginação em endpoints de listagem
- [ ] Endpoint de review funcionando

---

### Fase 6 — Testes Automatizados

**Objetivo:** Cobertura de testes (JUnit 5 + Mockito).

**Testes obrigatórios:**

**`AppointmentServiceTest`:**
- [ ] `create()` com slot disponível → deve salvar
- [ ] `create()` com conflito → `SlotUnavailableException`
- [ ] `cancel()` por não-dono → `UnauthorizedException`
- [ ] `create()` com data no passado → `PastDateException`

**`AppointmentRepositoryTest` (@DataJpaTest):**
- [ ] `existsConflict()` retorna true em conflito
- [ ] `existsConflict()` retorna false sem conflito

**`AuthServiceTest`:**
- [ ] `register()` com email duplicado → exceção
- [ ] `login()` com credencial inválida → exceção

**Definition of Done:**
- ✅ `mvn test` passa sem falhas
- ✅ AppointmentService com 4+ cenários
- ✅ Cada exceção tem pelo menos 1 teste
- ✅ Coverage >= 70%

**Checklist:**
- [ ] Testes de Service com Mockito
- [ ] Testes de Repository com @DataJpaTest
- [ ] Testes de exceções
- [ ] `mvn test` passando

---

## Convenções do Projeto

### ✅ Padrões Obrigatórios

1. **DTOs sempre, Entities nunca**
   - ❌ `return appointment;`
   - ✅ `return appointmentMapper.toResponse(appointment);`

2. **Sem lógica de negócio nos Controllers**
   - ❌ Controller checa conflito
   - ✅ Controller delega para Service

3. **Sem try-catch nos Controllers**
   - ❌ try { ... } catch { ... }
   - ✅ Deixe exceção subir → GlobalExceptionHandler

4. **Soft delete, não delete real**
   - ❌ `DELETE FROM services WHERE id = ?`
   - ✅ `UPDATE services SET active = false WHERE id = ?`

5. **Variáveis sensíveis em .env / environment variables**
   - ❌ `String SECRET = "abc123";`
   - ✅ `String secret = System.getenv("JWT_SECRET");`

6. **Validação em camadas**
   - DTO: `@NotBlank`, `@Email`, etc
   - Service: regras de negócio
   - Repository: queries customizadas

7. **Naming clara**
   - Métodos: `create`, `findById`, `update`, `delete`
   - Exceções: `SlotUnavailableException`, não `BadRequest`
   - DTOs: `AppointmentCreateRequest`, `AppointmentResponse`

---

## Dependências (pom.xml)

Vide arquivo `pom.xml` na raiz do projeto. Resumo:

```xml
<!-- Core -->
spring-boot-starter-web
spring-boot-starter-data-jpa
spring-boot-starter-security
spring-boot-starter-validation

<!-- Database -->
mysql-connector-j (8.0.33)
h2 (scope: test)

<!-- JWT -->
jjwt-api, jjwt-impl, jjwt-jackson (0.12.3)

<!-- Documentation -->
springdoc-openapi-starter-webmvc-ui (2.1.0)

<!-- Productivity -->
lombok
spring-boot-devtools

<!-- Testing -->
spring-boot-starter-test
spring-security-test
mockito-core
mockito-junit-jupiter
```

---

## application.properties (Configuração)

```properties
# Server
server.port=8080

# Database
spring.datasource.url=jdbc:mysql://localhost:3306/barbershop_db
spring.datasource.username=appuser
spring.datasource.password=apppass
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.jpa.properties.hibernate.format_sql=true

# JWT
jwt.secret=${JWT_SECRET:dev-secret-change-in-production}
jwt.expiration=86400000

# Logging
logging.level.root=INFO
logging.level.com.barbershop=DEBUG
```

---

## Swagger / OpenAPI

Acesse em desenvolvimento:
```
http://localhost:8080/swagger-ui/index.html
```

Em produção (Railway):
```
https://barbershop-api.railway.app/swagger-ui/index.html
```

A documentação é gerada automaticamente pela `springdoc-openapi`. Adicione `@Operation`, `@ApiResponses` nos controllers para melhorar.

---

## Checklist de Implementação Completa

### Backend
- [ ] Fase 1: Entidades modeladas
- [ ] Fase 2: Autenticação JWT funcionando
- [ ] Fase 3: CRUD de Services e Professionals
- [ ] Fase 4: Lógica anti-conflito de Appointments
- [ ] Fase 5: Queries avançadas e Reviews
- [ ] Fase 6: Testes automatizados (coverage >= 70%)
- [ ] `pom.xml` finalizado
- [ ] `application.properties` configurado
- [ ] `docker-compose.yml` testado localmente
- [ ] Railway configurado e deployado
- [ ] Swagger acessível em produção
- [ ] README com instruções

---

## Melhorias Futuras

| Melhoria | Complexidade |
|---|---|
| Notificações por email (Spring Mail) | Baixa |
| Refresh Token (`/auth/refresh`) | Baixa |
| Cache de disponibilidade (Redis) | Média |
| Notificações WhatsApp (Twilio) | Média |
| Pagamento online (Stripe) | Alta |
| WebSockets (atualizações real-time) | Alta |
| Multi-tenancy | Alta |

---

## Referências Rápidas

### Entidades Principais
- **User** — autenticação e identificação
- **Service** — o que se oferece
- **Availability** — quando o profissional trabalha
- **Appointment** — o que está agendado ⚠️
- **Review** — feedback

### Queries Mais Usadas
- `existsConflict()` — valida agendamento
- `findByEmail()` — login
- disponibilidade com LEFT JOIN — slots livres

### Exceções Mais Usadas
- `SlotUnavailableException` (409)
- `UnauthorizedException` (403)
- `ResourceNotFoundException` (404)
- `PastDateException` (422)

### Status de Agendamento
- `PENDING` — criado, aguardando confirmação
- `CONFIRMED` — profissional confirmou
- `COMPLETED` — serviço realizado
- `CANCELLED` — cancelado


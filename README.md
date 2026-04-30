# proj_spring_april

> **Prinsip: Simple → Transparan → Maintainable → Scalable**
> Jangan tambah sesuatu yang belum dibutuhkan. Tambahkan saat pain-nya terasa nyata.
> Baca code-nya, langsung paham apa yang terjadi. Tidak ada magic.

---

# BAGIAN 1 — TECH STACK & ARSITEKTUR

## Keputusan Final

| Aspek | Keputusan |
|---|---|
| Java | **21 LTS** |
| Lombok | Minimal: `@Slf4j` + `@RequiredArgsConstructor` saja |
| Deployment | VPS / VM sederhana |
| Auth | Multi-user, role-based (DB-backed) |
| Caching | Skip dulu |
| File Upload | Local disk (simple) |
| Testing | Minimal pragmatis — smoke + security test saja |

## Stack Lengkap — Bangun Sekarang

| # | Tech | Fungsi |
|---|---|---|
| 1 | Spring Boot 3.x (Java 21) | Backend framework |
| 2 | PostgreSQL | Database |
| 3 | Spring Data JPA + HikariCP | ORM + connection pool |
| 4 | Flyway | Database migration & versioning |
| 5 | Docker Compose | Dev environment (PostgreSQL saja) |
| 6 | Java Records + Manual Mapping | DTO transparan, immutable |
| 7 | Lombok minimal | `@Slf4j` + `@RequiredArgsConstructor` |
| 8 | Jakarta Validation | `@Valid`, `@NotBlank` — cegah data kotor |
| 9 | JPA Auditing + BaseEntity | `created_at`, `updated_at`, `created_by` — semua entity tinggal `extends` |
| 10 | Soft Delete (Hibernate `@SoftDelete`) | Data tidak hilang permanen, cukup annotation di BaseEntity |
| 11 | UUID Public ID | ID aman untuk URL — hindari expose auto-increment |
| 12 | `@ConfigurationProperties` | Config type-safe, 1 class per domain — ganti `@Value` yang bertebaran |
| 13 | Spring Security (Session + DB Roles) | Auth + role-based access |
| 14 | Thymeleaf + Layout Dialect | SSR template engine + master layout |
| 15 | Tailwind CSS + DaisyUI | Styling cepat |
| 16 | Alpine.js | Interaktivitas ringan (modal, dropdown) |
| 17 | Spring Boot DevTools | Live reload |
| 18 | `@ControllerAdvice` + ProblemDetail | Global error handling (RFC 7807) |
| 19 | Custom error pages | `404.html`, `500.html` untuk user |
| 20 | MDC Logging + AOP | `transactionId` per request + audit log tanpa polusi bisnis logic |
| 21 | Spring Actuator | `/actuator/health` untuk monitoring |
| 22 | File upload (local disk) | Simple upload ke `app.upload.dir` |
| 23 | Spring Profiles | `dev` / `prod` config |
| 24 | Response Compression | `server.compression.enabled=true` — hemat bandwidth, 1 baris config |

## Tambah Nanti (Ketika Dibutuhkan)

| Tech | Trigger |
|---|---|
| Zipkin + Micrometer | Ketika punya 2+ service |
| Full unit/integration test | Ketika ada bug berulang atau logic kompleks |
| Dockerfile multi-stage | Menjelang deployment ke VPS |
| CI/CD (GitHub Actions) | Ketika tim > 2 orang |
| Caching (Caffeine) | Ketika ada query lambat dan sering dipanggil |
| Rate Limiting (Bucket4j) | Ketika app sudah public-facing |

## Dependencies (pom.xml)

```xml
<!-- Core -->
spring-boot-starter-web
spring-boot-starter-data-jpa
spring-boot-starter-validation
spring-boot-starter-security
spring-boot-starter-thymeleaf
spring-boot-starter-actuator

<!-- Database -->
postgresql (runtime)
flyway-core
flyway-database-postgresql

<!-- Template -->
thymeleaf-layout-dialect

<!-- Dev -->
spring-boot-devtools (dev only)
lombok (optional, compile only)

<!-- Test (minimal) -->
spring-boot-starter-test
spring-security-test
```

---

# BAGIAN 1.5 — SETUP SEKALI, SAYANG KALAU TIDAK DARI AWAL

Fitur-fitur berikut **setup < 15 menit** tapi **sangat menyakitkan kalau di-retrofit belakangan**.

## 1. BaseEntity — Extend Sekali, Audit Selamanya

Semua entity tinggal `extends BaseEntity` → otomatis dapat audit fields + soft delete + UUID.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // UUID untuk public-facing (URL, API) — jangan expose auto-increment
    @Column(name = "public_id", unique = true, nullable = false, updatable = false)
    private String publicId;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    // Soft delete — data tidak hilang, hanya ditandai
    @Column(name = "deleted", nullable = false)
    private boolean deleted = false;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @PrePersist
    public void prePersist() {
        if (this.publicId == null) {
            this.publicId = UUID.randomUUID().toString().substring(0, 8);
        }
    }
}
```

```java
// Config — aktifkan auditing + siapa current user
@Configuration
@EnableJpaAuditing
public class AuditConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .map(Authentication::getName);
    }
}
```

```java
// Semua entity cukup extends — selesai
@Entity
@Table(name = "users")
public class User extends BaseEntity {
    // field spesifik User saja, tanpa audit fields
}
```

**Kenapa dari awal?** Kalau ditambah nanti: harus alter SEMUA tabel, update SEMUA entity, migrasi SEMUA URL dari `/users/1` ke `/users/a1b2c3d4`.

## 2. UUID Public ID — Jangan Expose Auto-Increment

```
❌ /users/1          → attacker tahu: ada 1 user, bisa coba /users/2, /users/3
✅ /users/a1b2c3d4   → aman, tidak bisa ditebak
```

- Internal tetap pakai `Long id` (cepat untuk JOIN, indexing)
- Public URL pakai `publicId` (UUID 8 char)
- Lookup: `userRepository.findByPublicId(publicId)`

**Kenapa dari awal?** Kalau retrofit: SEMUA URL, template Thymeleaf, link, dan bookmark user berubah.

## 3. Soft Delete — Data Tidak Pernah Hilang

```java
// Di Service — override delete menjadi soft delete
@Transactional
public void delete(String publicId) {
    User user = userRepository.findByPublicIdAndDeletedFalse(publicId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    user.setDeleted(true);
    user.setDeletedAt(LocalDateTime.now());
    userRepository.save(user);
    log.info("User soft-deleted: publicId={}", publicId);
}

// Di Repository — semua query default filter deleted=false
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByPublicIdAndDeletedFalse(String publicId);
    Page<User> findAllByDeletedFalse(Pageable pageable);
}
```

**Kenapa dari awal?** Kalau retrofit: data yang sudah di-delete fisik **hilang selamanya**, harus alter semua tabel + rewrite semua query.

## 4. @ConfigurationProperties — Ganti @Value yang Bertebaran

```java
// ❌ TANPA — @Value bertebaran di mana-mana, susah track
@Value("${app.upload.dir}") private String uploadDir;      // di FileService
@Value("${app.upload.max-size}") private long maxSize;      // di FileService
@Value("${app.upload.allowed-types}") private String types;  // di FileService
@Value("${app.name}") private String appName;                // di SomeController

// ✅ DENGAN — 1 class, type-safe, validated
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
    @NotBlank String name,
    Upload upload
) {
    public record Upload(
        @NotBlank String dir,
        @Min(1) long maxSize,
        List<String> allowedTypes
    ) {}
}

// application.yml
// app:
//   name: "MyApp"
//   upload:
//     dir: "./uploads"
//     max-size: 10485760
//     allowed-types: ["image/jpeg", "image/png", "application/pdf"]

// Pakai di mana saja — inject 1 object
@RequiredArgsConstructor
public class FileUploadService {
    private final AppProperties appProperties;
    // appProperties.upload().dir()
    // appProperties.upload().maxSize()
}
```

**Kenapa dari awal?** Kalau retrofit: harus cari semua `@Value` yang tersebar di puluhan file, lalu pindahkan ke 1 class. Rawan miss.

## 5. Response Compression — 1 Baris Config

```yaml
# application.yml
server:
  compression:
    enabled: true
    mime-types: text/html,text/css,application/javascript,application/json
    min-response-size: 1024
```

HTML/CSS/JS yang dikirim ke browser dikompres otomatis. **Hemat 60-80% bandwidth.** Tidak ada alasan untuk tidak mengaktifkan ini.

---

# BAGIAN 2 — PROJECT STRUCTURE

## Package-by-Feature

```
com.yourapp
├── common/
│   ├── config/              # SecurityConfig, WebConfig, AuditConfig
│   ├── entity/              # BaseEntity (abstract)
│   ├── exception/           # GlobalExceptionHandler, custom exceptions
│   ├── logging/             # MDC filter, AOP logging aspect
│   ├── properties/          # AppProperties (@ConfigurationProperties)
│   └── upload/              # FileUploadService, FileUploadController
├── auth/
│   ├── AuthController.java
│   ├── AuthService.java
│   └── dto/
│       ├── LoginRequest.java
│       └── RegisterRequest.java
├── user/
│   ├── User.java            # @Entity extends BaseEntity
│   ├── Role.java            # @Entity extends BaseEntity
│   ├── UserRepository.java
│   ├── UserService.java
│   ├── UserController.java
│   └── dto/
│       ├── UserRequest.java
│       └── UserResponse.java
└── [feature]/               # Tambah folder baru per fitur
    ├── [Feature].java       # extends BaseEntity
    ├── [Feature]Repository.java
    ├── [Feature]Service.java
    ├── [Feature]Controller.java
    └── dto/
```

## Database Schema (Flyway V1)

```sql
-- V1__init_schema.sql

-- Permissions — MODULE_ACTION pattern
CREATE TABLE permissions (
    id          BIGSERIAL PRIMARY KEY,
    public_id   VARCHAR(8) NOT NULL UNIQUE,
    module      VARCHAR(50) NOT NULL,             -- 'USER', 'PRODUCT', 'REPORT'
    action      VARCHAR(50) NOT NULL,             -- 'CREATE', 'READ', 'UPDATE', 'DELETE'
    name        VARCHAR(100) NOT NULL UNIQUE,     -- 'USER_CREATE', 'USER_READ', dst
    description VARCHAR(255),
    deleted     BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at  TIMESTAMP,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by  VARCHAR(100)
);

-- Roles — grouping of permissions
CREATE TABLE roles (
    id          BIGSERIAL PRIMARY KEY,
    public_id   VARCHAR(8) NOT NULL UNIQUE,
    name        VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(255),
    is_default  BOOLEAN NOT NULL DEFAULT FALSE,   -- auto-assign ke user baru
    deleted     BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at  TIMESTAMP,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by  VARCHAR(100)
);

CREATE TABLE role_permissions (
    role_id       BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    public_id   VARCHAR(8) NOT NULL UNIQUE,
    username    VARCHAR(100) UNIQUE,              -- nullable: SSO user bisa tanpa username
    email       VARCHAR(255) NOT NULL UNIQUE,
    password    VARCHAR(255),                     -- nullable: SSO user tidak punya password
    auth_method VARCHAR(20) NOT NULL DEFAULT 'LOCAL', -- 'LOCAL', 'GOOGLE', 'GITHUB', dll
    enabled     BOOLEAN NOT NULL DEFAULT TRUE,
    deleted     BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at  TIMESTAMP,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by  VARCHAR(100)
);

CREATE TABLE user_roles (
    user_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

-- Indexes
CREATE INDEX idx_users_deleted ON users(deleted);
CREATE INDEX idx_roles_deleted ON roles(deleted);
CREATE INDEX idx_permissions_deleted ON permissions(deleted);
CREATE INDEX idx_permissions_module ON permissions(module);

-- === SEED DATA ===

-- Permissions (MODULE_ACTION pattern)
INSERT INTO permissions (public_id, module, action, name, description) VALUES
    (substr(gen_random_uuid()::text,1,8), 'USER',      'CREATE', 'USER_CREATE',      'Membuat user baru'),
    (substr(gen_random_uuid()::text,1,8), 'USER',      'READ',   'USER_READ',        'Melihat daftar & detail user'),
    (substr(gen_random_uuid()::text,1,8), 'USER',      'UPDATE', 'USER_UPDATE',      'Mengubah data user'),
    (substr(gen_random_uuid()::text,1,8), 'USER',      'DELETE', 'USER_DELETE',      'Menghapus user'),
    (substr(gen_random_uuid()::text,1,8), 'PROFILE',   'READ',   'PROFILE_READ',     'Melihat profil sendiri'),
    (substr(gen_random_uuid()::text,1,8), 'PROFILE',   'UPDATE', 'PROFILE_UPDATE',   'Mengubah profil sendiri'),
    (substr(gen_random_uuid()::text,1,8), 'DASHBOARD', 'READ',   'DASHBOARD_READ',   'Akses dashboard');

-- Roles
INSERT INTO roles (public_id, name, description, is_default) VALUES
    (substr(gen_random_uuid()::text,1,8), 'ADMIN', 'Full access',           FALSE),
    (substr(gen_random_uuid()::text,1,8), 'STAFF', 'Operational access',    FALSE),
    (substr(gen_random_uuid()::text,1,8), 'USER',  'Basic user access',     TRUE);

-- ADMIN = semua permission
INSERT INTO role_permissions (role_id, permission_id)
    SELECT r.id, p.id FROM roles r, permissions p WHERE r.name = 'ADMIN';

-- STAFF = read + update, tanpa delete/create user
INSERT INTO role_permissions (role_id, permission_id)
    SELECT r.id, p.id FROM roles r, permissions p
    WHERE r.name = 'STAFF' AND p.name IN (
        'USER_READ', 'USER_UPDATE', 'DASHBOARD_READ', 'PROFILE_READ', 'PROFILE_UPDATE'
    );

-- USER = hanya profil sendiri + dashboard
INSERT INTO role_permissions (role_id, permission_id)
    SELECT r.id, p.id FROM roles r, permissions p
    WHERE r.name = 'USER' AND p.name IN (
        'PROFILE_READ', 'PROFILE_UPDATE', 'DASHBOARD_READ'
    );
```

### RBAC Model — Cara Kerja

```
User ──→ Roles ──→ Permissions (MODULE_ACTION)
         (siapa)    (boleh apa)

ADMIN → USER_CREATE, USER_READ, USER_UPDATE, USER_DELETE, PROFILE_*, DASHBOARD_*
STAFF → USER_READ, USER_UPDATE, PROFILE_*, DASHBOARD_*
USER  → PROFILE_*, DASHBOARD_*
```

### Menambah Fitur Baru — Cukup Insert

```sql
-- V3__add_product_permissions.sql (contoh nanti)
INSERT INTO permissions (public_id, module, action, name, description) VALUES
    (substr(gen_random_uuid()::text,1,8), 'PRODUCT', 'CREATE', 'PRODUCT_CREATE', 'Membuat produk'),
    (substr(gen_random_uuid()::text,1,8), 'PRODUCT', 'READ',   'PRODUCT_READ',   'Melihat produk');

-- Assign ke ADMIN semua, STAFF hanya read
INSERT INTO role_permissions (role_id, permission_id)
    SELECT r.id, p.id FROM roles r, permissions p
    WHERE r.name = 'ADMIN' AND p.module = 'PRODUCT';
```

### Penggunaan di Code

```java
// Controller — check permission, bukan role
@PreAuthorize("hasAuthority('USER_DELETE')")
@PostMapping("/{publicId}/delete")
public String deleteUser(@PathVariable String publicId) { ... }
```

---

### Dynamic Menu System — Berdasarkan Permission

#### Schema

```sql
-- Di V1__init_schema.sql (tambahkan setelah user_roles)

CREATE TABLE menus (
    id              BIGSERIAL PRIMARY KEY,
    public_id       VARCHAR(8) NOT NULL UNIQUE,
    parent_id       BIGINT REFERENCES menus(id),       -- NULL = top-level menu
    title           VARCHAR(100) NOT NULL,              -- 'User Management'
    icon            VARCHAR(50),                        -- DaisyUI/icon class: 'users', 'home'
    url             VARCHAR(255),                       -- '/users' (NULL jika parent group)
    permission_name VARCHAR(100) REFERENCES permissions(name), -- link ke permission
    sort_order      INT NOT NULL DEFAULT 0,             -- urutan tampil
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    deleted         BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at      TIMESTAMP,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(100)
);

CREATE INDEX idx_menus_parent ON menus(parent_id);
CREATE INDEX idx_menus_sort ON menus(sort_order);
CREATE INDEX idx_menus_permission ON menus(permission_name);
```

#### Seed Menu Data

```sql
-- Top-level menus (parent_id = NULL)
INSERT INTO menus (public_id, parent_id, title, icon, url, permission_name, sort_order) VALUES
    (substr(gen_random_uuid()::text,1,8), NULL, 'Dashboard',       'home',     '/dashboard',  'DASHBOARD_READ', 1),
    (substr(gen_random_uuid()::text,1,8), NULL, 'User Management', 'users',    NULL,           NULL,             2),
    (substr(gen_random_uuid()::text,1,8), NULL, 'My Profile',      'user',     '/profile',    'PROFILE_READ',   9);

-- Sub-menus: User Management
INSERT INTO menus (public_id, parent_id, title, icon, url, permission_name, sort_order) VALUES
    (substr(gen_random_uuid()::text,1,8),
     (SELECT id FROM menus WHERE title = 'User Management'),
     'All Users', 'list', '/users', 'USER_READ', 1),
    (substr(gen_random_uuid()::text,1,8),
     (SELECT id FROM menus WHERE title = 'User Management'),
     'Add User', 'plus', '/users/new', 'USER_CREATE', 2);
```

#### Cara Kerja

```
User login → load permissions → query menus yang permission-nya dimiliki user

ADMIN melihat:                    USER melihat:
├── Dashboard                     ├── Dashboard
├── User Management               └── My Profile
│   ├── All Users
│   └── Add User
└── My Profile

Sidebar SAMA, template SAMA — data berbeda berdasarkan permission.
```

#### Entity

```java
@Entity
@Table(name = "menus")
@Getter @Setter @NoArgsConstructor
public class Menu extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Menu parent;

    @OneToMany(mappedBy = "parent", fetch = FetchType.LAZY)
    @OrderBy("sortOrder ASC")
    private List<Menu> children = new ArrayList<>();

    @Column(nullable = false, length = 100)
    private String title;

    @Column(length = 50)
    private String icon;

    @Column(length = 255)
    private String url;

    @Column(name = "permission_name", length = 100)
    private String permissionName;

    @Column(name = "sort_order")
    private int sortOrder;

    @Column(name = "is_active")
    private boolean isActive = true;
}
```

#### MenuService

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MenuService {

    private final MenuRepository menuRepository;

    @Transactional(readOnly = true)
    public List<MenuResponse> getMenusForUser(Set<String> userPermissions) {
        // Ambil semua top-level menu yang active
        List<Menu> topMenus = menuRepository
                .findByParentIsNullAndDeletedFalseAndIsActiveTrueOrderBySortOrder();

        return topMenus.stream()
                .map(menu -> buildMenuTree(menu, userPermissions))
                .filter(Objects::nonNull)   // buang menu yang tidak boleh diakses
                .toList();
    }

    private MenuResponse buildMenuTree(Menu menu, Set<String> permissions) {
        // Filter children berdasarkan permission
        List<MenuResponse> visibleChildren = menu.getChildren().stream()
                .filter(child -> child.isActive() && !child.isDeleted())
                .filter(child -> child.getPermissionName() == null
                        || permissions.contains(child.getPermissionName()))
                .map(child -> buildMenuTree(child, permissions))
                .filter(Objects::nonNull)
                .toList();

        // Parent tanpa permission = tampil jika punya children yang visible
        if (menu.getPermissionName() == null && menu.getUrl() == null) {
            return visibleChildren.isEmpty() ? null
                    : MenuResponse.of(menu, visibleChildren);
        }

        // Menu dengan permission = tampil jika user punya permission
        if (menu.getPermissionName() != null
                && !permissions.contains(menu.getPermissionName())) {
            return null;
        }

        return MenuResponse.of(menu, visibleChildren);
    }
}
```

#### DTO

```java
public record MenuResponse(
    String title,
    String icon,
    String url,
    List<MenuResponse> children
) {
    public static MenuResponse of(Menu menu, List<MenuResponse> children) {
        return new MenuResponse(menu.getTitle(), menu.getIcon(), menu.getUrl(), children);
    }

    public boolean hasChildren() {
        return children != null && !children.isEmpty();
    }
}
```

#### Thymeleaf Sidebar (Dynamic)

```html
<!-- fragments/sidebar.html -->
<nav>
    <ul class="menu">
        <li th:each="menu : ${menus}">
            <!-- Menu TANPA children = link biasa -->
            <a th:if="${!menu.hasChildren()}"
               th:href="${menu.url}"
               th:classappend="${currentUrl == menu.url} ? 'active'">
                <i th:class="'icon-' + ${menu.icon}"></i>
                <span th:text="${menu.title}">Menu</span>
            </a>

            <!-- Menu DENGAN children = collapsible group -->
            <details th:if="${menu.hasChildren()}" open>
                <summary>
                    <i th:class="'icon-' + ${menu.icon}"></i>
                    <span th:text="${menu.title}">Group</span>
                </summary>
                <ul>
                    <li th:each="child : ${menu.children}">
                        <a th:href="${child.url}"
                           th:classappend="${currentUrl == child.url} ? 'active'">
                            <i th:class="'icon-' + ${child.icon}"></i>
                            <span th:text="${child.title}">Child</span>
                        </a>
                    </li>
                </ul>
            </details>
        </li>
    </ul>
</nav>
```

#### Controller — Inject Menu ke Setiap Page

```java
// Gunakan @ControllerAdvice agar menu tersedia di SEMUA template
@ControllerAdvice
@RequiredArgsConstructor
public class MenuAdvice {

    private final MenuService menuService;

    @ModelAttribute("menus")
    public List<MenuResponse> populateMenus(@AuthenticationPrincipal AppUserDetails user) {
        if (user == null) return List.of();
        return menuService.getMenusForUser(user.getUser().getAllPermissions());
    }

    @ModelAttribute("currentUrl")
    public String populateCurrentUrl(HttpServletRequest request) {
        return request.getRequestURI();
    }
}
```

#### Menambah Menu Baru — Cukup Insert

```sql
-- V3__add_product_menus.sql
-- 1. Tambah permission (kalau belum)
-- 2. Tambah menu — SELESAI, template tidak berubah

INSERT INTO menus (public_id, parent_id, title, icon, url, permission_name, sort_order) VALUES
    (substr(gen_random_uuid()::text,1,8), NULL, 'Products', 'box', NULL, NULL, 3);

INSERT INTO menus (public_id, parent_id, title, icon, url, permission_name, sort_order) VALUES
    (substr(gen_random_uuid()::text,1,8),
     (SELECT id FROM menus WHERE title = 'Products'),
     'All Products', 'list', '/products', 'PRODUCT_READ', 1),
    (substr(gen_random_uuid()::text,1,8),
     (SELECT id FROM menus WHERE title = 'Products'),
     'Add Product', 'plus', '/products/new', 'PRODUCT_CREATE', 2);
```

**Hasil: sidebar otomatis punya menu "Products" — tanpa ubah HTML satu baris pun.**

---

## Environment Configuration

```
src/main/resources/
├── application.yml           # Shared config + compression + app properties
├── application-dev.yml       # Dev: localhost DB, debug logging
└── application-prod.yml      # Prod: env vars, optimized settings
```

Sensitive values → **Environment Variables**, bukan hardcode.

---

# BAGIAN 3 — CODING CONVENTIONS

## Data Flow — Dari Request Sampai Database

```
Browser → Controller → Service → Repository → Database
                         ↕
                    Mapping di sini
                  (DTO ↔ Entity)
```

**Aturan:**
- DTO **tidak pernah** masuk ke Repository layer
- Entity **tidak pernah** keluar ke Controller layer
- Mapping selalu terjadi **di Service**

## 3.1 Controller — Tipis, Hanya Routing

```java
@Controller
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public String listUsers(Model model, @RequestParam(defaultValue = "0") int page) {
        model.addAttribute("users", userService.findAll(page));
        return "user/list";
    }

    @PostMapping
    public String createUser(@Valid @ModelAttribute UserRequest request,
                             BindingResult result, RedirectAttributes redirect) {
        if (result.hasErrors()) return "user/form";
        userService.create(request);
        redirect.addFlashAttribute("success", "User berhasil dibuat");
        return "redirect:/users";
    }
}
```

**Aturan:** Controller hanya inject Service (bukan Repository). Return template name atau `redirect:`.

## 3.2 Service — Semua Logic di Sini

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public UserResponse create(UserRequest request) {
        if (userRepository.existsByUsername(request.username())) {
            throw new DuplicateResourceException("Username sudah digunakan");
        }

        User user = new User();
        user.setUsername(request.username());
        user.setEmail(request.email());
        user.setPassword(passwordEncoder.encode(request.password()));

        User saved = userRepository.save(user);
        log.info("User created: username={}", saved.getUsername());

        return UserResponse.from(saved);
    }
}
```

**Aturan:**
- `@Transactional` di method tulis, `@Transactional(readOnly = true)` di method baca
- Throw custom exception, jangan return null
- Log **setelah** aksi berhasil, 1 event = 1 baris

## 3.3 DTO — Java Records + Static Factory

```java
// Request DTO — validasi langsung di field
public record UserRequest(
    @NotBlank(message = "Username wajib diisi")
    @Size(min = 3, max = 100) String username,

    @NotBlank @Email String email,

    @NotBlank @Size(min = 8) String password
) {}

// Response DTO — static factory method untuk mapping
public record UserResponse(Long id, String username, String email,
                           Set<String> roles, LocalDateTime createdAt) {

    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(), user.getUsername(), user.getEmail(),
            user.getRoles().stream().map(Role::getName).collect(Collectors.toSet()),
            user.getCreatedAt()
        );
    }

    public static Page<UserResponse> from(Page<User> users) {
        return users.map(UserResponse::from);
    }
}
```

## 3.4 Entity — Extends BaseEntity

```java
// Entity cukup extends BaseEntity — audit fields sudah gratis
@Entity
@Table(name = "users")
@Getter @Setter @NoArgsConstructor
public class User extends BaseEntity {

    @Column(nullable = false, unique = true, length = 100)
    private String username;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    private boolean enabled = true;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles = new HashSet<>();

    // id, publicId, createdAt, updatedAt, createdBy,
    // deleted, deletedAt → semua dari BaseEntity
}
```

**Lombok di Entity:**

| Annotation | Boleh? | Alasan |
|---|---|---|
| `@Getter` `@Setter` `@NoArgsConstructor` | ✅ | Entity butuh ini untuk JPA |
| `@Data` | ❌ | `equals/hashCode` konflik dengan JPA proxy |
| `@Builder` | ❌ | Menyembunyikan konstruksi, kurang transparan |
| `@ToString` | ⚠️ | Exclude relasi, bisa trigger lazy loading |

## 3.5 Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Public-facing lookup — pakai publicId, bukan id
    Optional<User> findByPublicIdAndDeletedFalse(String publicId);

    // Internal lookup
    Optional<User> findByUsernameAndDeletedFalse(String username);
    boolean existsByUsername(String username);

    // List — selalu filter soft-deleted
    Page<User> findAllByDeletedFalse(Pageable pageable);

    // Method name > 3 kata → pakai @Query
    @Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.username = :username AND u.deleted = false")
    Optional<User> findByUsernameWithRoles(@Param("username") String username);
}
```

**Aturan:** Return `Optional<T>` untuk single result. `JOIN FETCH` untuk relasi yang pasti dibutuhkan. Selalu filter `deleted = false`.

## 3.6 Exception — Custom + Handler Terpusat

```java
// Custom exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) { super(message); }
}

// Global handler
@ControllerAdvice @Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public String handleNotFound(ResourceNotFoundException ex, Model model) {
        log.warn("Resource not found: {}", ex.getMessage());
        model.addAttribute("message", ex.getMessage());
        return "error/404";
    }

    @ExceptionHandler(Exception.class)
    public String handleGeneral(Exception ex, Model model) {
        log.error("Unexpected error", ex);
        model.addAttribute("message", "Terjadi kesalahan sistem");
        return "error/500";
    }
}
```

**Aturan:** Jangan try-catch di service, biarkan bubble up ke `@ControllerAdvice`.

## 3.7 URL Pattern

| Aksi | HTTP | URL | Method |
|---|---|---|---|
| List | GET | `/users` | `listUsers()` |
| Form buat | GET | `/users/new` | `showCreateForm()` |
| Simpan | POST | `/users` | `createUser()` |
| Detail | GET | `/users/{id}` | `showUser()` |
| Form edit | GET | `/users/{id}/edit` | `showEditForm()` |
| Update | POST | `/users/{id}` | `updateUser()` |
| Hapus | POST | `/users/{id}/delete` | `deleteUser()` |

Pakai `POST` untuk delete — HTML form hanya support GET dan POST.

## 3.8 Naming Convention

| Jenis | Pattern | Contoh |
|---|---|---|
| Entity | `[Noun]` | `User.java` |
| Repository | `[Noun]Repository` | `UserRepository.java` |
| Service | `[Noun]Service` | `UserService.java` |
| Controller | `[Noun]Controller` | `UserController.java` |
| Request DTO | `[Noun]Request` | `UserRequest.java` |
| Response DTO | `[Noun]Response` | `UserResponse.java` |

## 3.9 Quick Reference

| ✅ Do | ❌ Don't |
|---|---|
| Logic di Service | Logic di Controller |
| Return `Optional` | Return `null` |
| Custom exception | Return error code / boolean |
| `@Valid` di DTO | Validasi manual di service |
| `log.info("msg: {}", var)` | `log.info("msg: " + var)` |
| `JOIN FETCH` untuk relasi | Akses lazy relation di controller |
| Manual mapping (explicit) | Auto-mapper (implicit) |
| Throw → ControllerAdvice | Try-catch di setiap method |
| `@Transactional` di service | `@Transactional` di controller |
| `POST` + redirect (PRG) | `POST` tanpa redirect |

---

# BAGIAN 4 — LOGGING STRATEGY

## Aturan: Apa yang Di-log vs Tidak

### ✅ LOG

| Apa | Level | Contoh |
|---|---|---|
| Data created/updated/deleted | `INFO` | `"User created: username=john"` |
| Keputusan bisnis | `INFO` | `"Order cancelled: overdue 7 days"` |
| Auth event | `INFO` | `"Login success: user=sigit"` |
| Slow execution > 500ms | `WARN` | `"SLOW createUser() → 1205ms"` |
| Client error | `WARN` | `"Resource not found: User id=99"` |
| Unexpected exception | `ERROR` | Full stack trace di log |

### ❌ JANGAN LOG

- Read/GET biasa (bukan event)
- "Entering method" / "Exiting method" di production
- Setiap step dalam proses
- Data sensitif (password, token)
- Di dalam loop
- Full object dump

## MDC — Trace Setiap Request

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MdcFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        MDC.put("txId", UUID.randomUUID().toString().substring(0, 8));
        MDC.put("user", request.getUserPrincipal() != null
                ? request.getUserPrincipal().getName() : "anonymous");
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

**Hasil:** Setiap log punya `txId` → `grep a1b2c3d4 app.log` = semua log 1 request.

## AOP — Hanya Log Slow + Error

```java
@Aspect @Component @Slf4j
public class RequestLoggingAspect {

    @Around("@within(org.springframework.stereotype.Controller)")
    public Object logRequest(ProceedingJoinPoint jp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = jp.proceed();
            long duration = System.currentTimeMillis() - start;
            if (duration > 500) {
                log.warn("SLOW {} → {}ms", jp.getSignature().toShortString(), duration);
            } else {
                log.debug("{} → {}ms", jp.getSignature().toShortString(), duration);
            }
            return result;
        } catch (Exception ex) {
            log.error("FAILED {} → {}ms: {}", jp.getSignature().toShortString(),
                      System.currentTimeMillis() - start, ex.getMessage());
            throw ex;
        }
    }
}
```

**Produksi:** Normal request = silent. Hanya slow + error yang muncul.

## Log Level Per Environment

### Dev (`application-dev.yml`)
```yaml
logging:
  level:
    root: WARN
    com.yourapp: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
```

### Prod (`application-prod.yml`)
```yaml
logging:
  level:
    root: WARN
    com.yourapp: INFO
    org.springframework.security: WARN
    org.hibernate: WARN
    org.flywaydb: INFO
```

> **PENTING:** Di production, Hibernate SQL = OFF. Ini penyebab #1 log spam.

## Logback Config (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- DEV: console dengan warna -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss} %highlight(%-5level) [%X{txId:-no-tx}] [%X{user:-system}] %cyan(%-20.20logger{0}) - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="WARN"><appender-ref ref="CONSOLE"/></root>
        <logger name="com.yourapp" level="DEBUG"/>
    </springProfile>

    <!-- PROD: file + rotation + error terpisah -->
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/app.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <maxFileSize>50MB</maxFileSize>
                <maxHistory>7</maxHistory>
                <totalSizeCap>500MB</totalSizeCap>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%X{txId:-no-tx}] [%X{user:-system}] %-30.30logger{0} - %msg%n</pattern>
            </encoder>
        </appender>

        <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/error.log</file>
            <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
                <level>ERROR</level>
            </filter>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <fileNamePattern>logs/error.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <maxFileSize>20MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>200MB</totalSizeCap>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%X{txId:-no-tx}] [%X{user:-system}] %logger{36} - %msg%n%ex</pattern>
            </encoder>
        </appender>

        <root level="WARN">
            <appender-ref ref="FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
        <logger name="com.yourapp" level="INFO"/>
        <logger name="org.flywaydb" level="INFO"/>
    </springProfile>
</configuration>
```

| Fitur | Efek |
|---|---|
| `totalSizeCap=500MB` | Disk VPS tidak penuh |
| `maxHistory=7` | Log lama otomatis dihapus |
| `.gz` compression | Hemat 80% disk |
| Error file terpisah | `tail -f error.log` → langsung lihat masalah |

---

# BAGIAN 5 — CHECKLIST IMPLEMENTASI

| # | Task | Estimasi |
|---|---|---|
| 1 | Init Spring Boot project (Java 21, Maven, dependencies) | 15 menit |
| 2 | `application.yml` + profiles (dev/prod) | 15 menit |
| 3 | `docker-compose.yml` (PostgreSQL) | 10 menit |
| 4 | Flyway V1 — schema users, roles, user_roles | 10 menit |
| 5 | Entity classes (User, Role) + JPA Auditing | 30 menit |
| 6 | Spring Security config (Session + DB roles + BCrypt) | 45 menit |
| 7 | Auth flow — login, register, logout | 45 menit |
| 8 | Thymeleaf layout + base template + error pages | 30 menit |
| 9 | Tailwind + DaisyUI + Alpine.js integration | 20 menit |
| 10 | Global Exception Handler + MDC + AOP logging | 30 menit |
| 11 | File upload (local disk) | 20 menit |
| 12 | Smoke Test + Security Test | 15 menit |
| | **Total estimasi** | **~4.5 jam** |

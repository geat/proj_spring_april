# proj_spring_april
Kesimpulan Tech Stack untuk Rapid Development
Kesimpulan Lengkap Tech Stack untuk Rapid Development
1. Core Framework & Database

Spring Boot: Fondasi backend utama yang tangguh dan modern.

PostgreSQL: Database relasional yang solid dan stabil.

Spring Data JPA: Menangani interaksi database sekaligus menyediakan fitur pagination bawaan.

Flyway: Mengelola migrasi dan versioning skema database secara aman untuk menghindari konflik.

Docker Compose (Spring Boot 3.1+): Menyederhanakan development dengan menyalakan/mematikan container database (seperti PostgreSQL atau Zipkin) secara otomatis.

2. Data Handling & Security

Java Records + Manual Mapping / Static Factory: Pendekatan transparan 100% untuk DTO. Mengganti auto-mapper (seperti MapStruct) dengan pemetaan eksplisit untuk mencegah bug atau error yang tersembunyi (silent failure).

Lombok: Mengeliminasi penulisan boilerplate Java (seperti getter/setter dan logger) secara efisien.

Spring Security (Session-Based): Memanfaatkan autentikasi stateful bawaan untuk arsitektur SSR yang jauh lebih cepat dikonfigurasi dan lebih aman dari eksploitasi client-side dibandingkan JWT.

3. Frontend & Developer Experience (DX)

Thymeleaf + Layout Dialect: Template engine andalan dengan kapabilitas master layout agar struktur HTML (seperti header/sidebar) tidak berulang.

Tailwind CSS + DaisyUI: Mempercepat styling menggunakan komponen UI siap pakai tanpa perlu pusing memikirkan nama class CSS manual.

Alpine.js: Memberikan interaktivitas sisi klien (seperti modal dan dropdown) yang sangat ringan tanpa kerumitan state management atau proses build JS.

Spring Boot DevTools: Mempercepat siklus coding dengan fitur Live Reload setiap ada perubahan file.

4. Observability & Error Handling (Pemantauan & Pelacakan)

Global Error Handling: Sentralisasi penangkapan error menggunakan @ControllerAdvice. Memanfaatkan ProblemDetail (standar RFC 7807) untuk respons API yang konsisten, dan custom template Thymeleaf (404.html, 500.html) untuk tampilan user.

Clear Logging (MDC + AOP): Menggunakan Mapped Diagnostic Context (MDC) untuk menyisipkan transactionId di setiap baris log Logback, dibantu dengan Spring AOP untuk mencatat audit log (request/response time) tanpa mengotori logika bisnis.

Distributed Tracing (Micrometer + Zipkin): Menggunakan Micrometer Tracing untuk menyuntikkan traceId secara otomatis, divisualisasikan menggunakan Zipkin (via Docker) untuk melacak bottleneck performa dan transparansi eksekusi query dari ujung ke ujung.

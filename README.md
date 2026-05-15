# Tauri Masaüstü Uygulama - Backend Plan

## Context

Mevcut scheduler projesi (C# + OR-Tools) ders programı çözücü olarak çalışıyor. Şimdi bir masaüstü uygulama gerekiyor: okul yetkilisi temel verileri girer, atamaları yapar, sonra birden fazla ders programı oluşturur. Scheduler entegrasyonu şimdilik kapsam dışı.

## Kullanım Akışı (UI sayfaları)

1. **Temel veri sayfaları**: Dersler, Sınıflar, Derslikler, Öğretmenler (basit CRUD listeleri)
2. **Sınıf-Ders atama**: Her sınıfa okutulan dersler eklenir (henüz öğretmen/derslik atamadan)
3. **Öğretmen atama**: Her sınıf-ders ikilisine öğretmen atanır
4. **Derslik atama**: Her sınıf-ders ikilisine derslik atanır
5. **Ortak dersler**: Lesson group'lar tanımlanır
6. **Okul ders saatleri**: 7×100 grid'de hangi (gün, saat) hücreleri aktif seçilir (pazartesi 4 saat, salı 7 saat olabilir)
7. **Ders programları**: Liste sayfası — yeni program oluştur veya mevcut olanı aç
8. **Ders programı detay sayfası**: 
   - Her sınıf için tablo görünümü (engelli saat, manuel atama)
   - Her öğretmen için tablo görünümü (engelli saat, manuel atama)
   - Her derslik için tablo görünümü
   - Global/teacher/subject config ayarları
   - "Yerleştir" butonu (scheduler entegrasyonu sonra)

## Silme Stratejisi
- **Soft snapshot**: Eski ders programı sonuçları denormalize saklanır (isimler dahil)
- Temel veri silindiğinde eski sonuçlar bozulmaz; aktif schedule'da kullanılıyorsa RESTRICT ile uyarı

## Proje Yapısı

```
D:\dev\scheduler-app\
├── src-tauri/
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── build.rs
│   ├── migrations/
│   │   └── 001_initial_schema.sql
│   └── src/
│       ├── main.rs
│       ├── lib.rs
│       ├── error.rs
│       ├── db/mod.rs
│       ├── models/
│       │   ├── mod.rs
│       │   ├── school_settings.rs
│       │   ├── school_hours.rs
│       │   ├── class.rs
│       │   ├── teacher.rs
│       │   ├── subject.rs
│       │   ├── classroom.rs
│       │   ├── class_lesson.rs
│       │   ├── lesson_group.rs
│       │   ├── schedule.rs
│       │   └── timeslot.rs
│       ├── commands/
│       │   ├── mod.rs
│       │   ├── school_settings.rs
│       │   ├── school_hours.rs
│       │   ├── classes.rs
│       │   ├── teachers.rs
│       │   ├── subjects.rs
│       │   ├── classrooms.rs
│       │   ├── class_lessons.rs
│       │   ├── lesson_groups.rs
│       │   ├── schedules.rs
│       │   └── schedule_views.rs
│       └── validation/
│           ├── mod.rs
│           ├── pattern.rs
│           └── timeslot.rs
├── src/                  -- React frontend
├── package.json
└── vite.config.ts
```

## Veritabanı Şeması

### Entity-Relationship Diyagramı

```
              TEMEL VERİ (Global)
              ==================

school_settings (1 satır)
school_hours    (set of day_index, period_index)

subjects ──< teachers_subjects   >── teachers
   |                                    |
   |───< classrooms_subjects >── classrooms
   |
   |───< class_lessons >── classes ──[homeroom_teacher_id]── teachers
            |                  |
            |                  └─< class_available_hours
            |
            └─< class_lesson_nosplit_hours  (permanent property)

lesson_groups (subject_id) ──< lesson_group_classes >── classes


              DERS PROGRAMI (Per-Schedule)
              ============================

schedules
   |
   ├──  schedule_global_config (1:1)
   ├──< schedule_teacher_configs
   ├──< schedule_subject_configs
   ├──< schedule_teacher_blocked_hours
   ├──< schedule_class_blocked_hours       (sınıf bazlı engelli)
   ├──< schedule_classroom_blocked_hours   (derslik bazlı engelli)
   ├──< schedule_class_lesson_blocked_hours
   ├──< schedule_class_lesson_assigned_hours (manuel atama)
   └──< schedule_results                   (çözüm sonuçları snapshot)
```

---

### TEMEL VERİ TABLOLARI

#### 1. school_settings (singleton, id=1)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | 1 | 1 | - | evet (CHECK id=1) |
| school_name | TEXT | evet | 0 | 200 | '' | hayır |
| max_days | INTEGER | evet | 1 | 7 | 5 | hayır |
| max_periods_per_day | INTEGER | evet | 1 | 99 | 10 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

**Not**: `max_days` ve `max_periods_per_day` UI grid boyutu için kullanılır. Gerçek aktif saatler `school_hours` tablosunda.

#### 2. school_hours (aktif okul saatleri)
| Alan | Tip | Zorunlu | Min | Max | Unique |
|------|-----|---------|-----|-----|--------|
| id | INTEGER PK | evet | auto | - | evet |
| day_index | INTEGER | evet | 0 | 6 | combo |
| period_index | INTEGER | evet | 0 | 99 | combo |

**UNIQUE(day_index, period_index)**
UI: 7×N grid, kullanıcı aktif hücreleri seçer. Her aktif hücre bir satır.

#### 3. subjects (dersler)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | evet |
| code | TEXT | hayır | 1 | 20 | NULL | evet (NULL hariç) |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 4. teachers (öğretmenler)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | hayır |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 5. classrooms (derslikler)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | evet |
| code | TEXT | hayır | 1 | 20 | NULL | evet (NULL hariç) |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 6. classes (sınıflar)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique | Not |
|------|-----|---------|-----|-----|---------|--------|-----|
| id | INTEGER PK | evet | auto | - | autoincrement | evet | |
| name | TEXT | evet | 1 | 50 | - | evet | "5-A" |
| homeroom_teacher_id | INTEGER FK | hayır | - | - | NULL | hayır | → teachers, SET NULL (sınıf öğretmeni) |
| sort_order | INTEGER | evet | - | - | 0 | hayır | |
| created_at | TEXT | evet | - | - | now() | hayır | |
| updated_at | TEXT | evet | - | - | now() | hayır | |

#### 7. teachers_subjects (öğretmenin verebileceği dersler)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | autoincrement |
| teacher_id | INTEGER FK | evet | → teachers, CASCADE |
| subject_id | INTEGER FK | evet | → subjects, CASCADE |

**UNIQUE(teacher_id, subject_id)**

#### 8. classrooms_subjects (derslikte yapılabilecek dersler)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | autoincrement |
| classroom_id | INTEGER FK | evet | → classrooms, CASCADE |
| subject_id | INTEGER FK | evet | → subjects, CASCADE |

**UNIQUE(classroom_id, subject_id)**

#### 9. class_lessons (sınıf-ders atamaları)
| Alan | Tip | Zorunlu | Min | Max | Default | Not |
|------|-----|---------|-----|-----|---------|-----|
| id | INTEGER PK | evet | auto | - | autoincrement | |
| class_id | INTEGER FK | evet | - | - | - | → classes, CASCADE |
| subject_id | INTEGER FK | evet | - | - | - | → subjects, RESTRICT |
| teacher_id | INTEGER FK | hayır | - | - | NULL | → teachers, SET NULL |
| classroom_id | INTEGER FK | hayır | - | - | NULL | → classrooms, SET NULL |
| hours_per_week | INTEGER | evet | 1 | 40 | - | haftalık ders süresi |
| pattern | TEXT (JSON) | evet | - | - | '[]' | "[2,2,2]" |
| lesson_group_tag | TEXT | hayır | 1 | 50 | NULL | split grup etiketi |
| sort_order | INTEGER | evet | - | - | 0 | |
| created_at | TEXT | evet | - | - | now() | |
| updated_at | TEXT | evet | - | - | now() | |

**UNIQUE(class_id, subject_id, teacher_id)**

**Validasyon (Rust):**
- `teacher_id` verilmişse → `teachers_subjects` kaydı olmalı
- `classroom_id` verilmişse → `classrooms_subjects` kaydı olmalı
- `pattern`: JSON array, pozitif int (1..max_periods_per_day), toplam = `hours_per_week`. `[]` = otomatik

#### 10. class_lesson_nosplit_hours (bölünmeyen ders saatleri — permanent)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | autoincrement |
| class_lesson_id | INTEGER FK | evet | → class_lessons, CASCADE |
| day_index | INTEGER | evet | 0..6 |
| period_index | INTEGER | evet | 0..99 |

**UNIQUE(class_lesson_id, day_index, period_index)**

#### 11. class_available_hours (sınıfın müsait saatleri)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | |
| class_id | INTEGER FK | evet | → classes, CASCADE |
| day_index | INTEGER | evet | 0..6 |
| period_index | INTEGER | evet | 0..99 |

**UNIQUE(class_id, day_index, period_index)**
Yeni sınıf oluşturulunca `school_hours` ile aynı slotlar otomatik doldurulur.

#### 12. lesson_groups (senkron ders grupları)
| Alan | Tip | Zorunlu | Unique | Not |
|------|-----|---------|--------|-----|
| id | INTEGER PK | evet | evet | |
| subject_id | INTEGER FK | evet | evet | → subjects, RESTRICT |
| name | TEXT | evet | hayır | max 100 char, default '' |
| created_at | TEXT | evet | hayır | now() |
| updated_at | TEXT | evet | hayır | now() |

#### 13. lesson_group_classes (junction)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | |
| lesson_group_id | INTEGER FK | evet | → lesson_groups, CASCADE |
| class_id | INTEGER FK | evet | → classes, CASCADE |

**UNIQUE(lesson_group_id, class_id)**

---

### DERS PROGRAMI TABLOLARI (Per-Schedule)

#### 14. schedules
| Alan | Tip | Zorunlu | Min | Max | Default |
|------|-----|---------|-----|-----|---------|
| id | INTEGER PK | evet | auto | - | autoincrement |
| name | TEXT | evet | 1 | 100 | - |
| status | TEXT | evet | - | - | 'draft' |
| created_at | TEXT | evet | - | - | now() |
| updated_at | TEXT | evet | - | - | now() |

**status**: 'draft' | 'solved' | 'archived'

#### 15. schedule_global_config (1:1 per schedule)
| Alan | Tip | Zorunlu | Min | Max | Default |
|------|-----|---------|-----|-----|---------|
| id | INTEGER PK | evet | auto | - | - |
| schedule_id | INTEGER FK | evet | - | - | UNIQUE, → schedules, CASCADE |
| teacher_max_gap | INTEGER | hayır | 0 | 10 | NULL |
| teacher_daily_max_lesson_hour | INTEGER | hayır | 1 | 12 | NULL |
| pattern_min_gap | INTEGER | hayır | 0 | 5 | NULL |
| split_class_same_teacher_lessons | INTEGER | evet | 0 | 1 | 0 |
| split_class_lesson_groups | INTEGER | evet | 0 | 1 | 0 |
| split_two_hour_blocks | INTEGER | evet | 0 | 1 | 0 |

#### 16. schedule_teacher_configs
| Alan | Tip | Zorunlu | Min | Max |
|------|-----|---------|-----|-----|
| id | INTEGER PK | evet | auto | - |
| schedule_id | INTEGER FK | evet | - | - |
| teacher_id | INTEGER FK | evet | - | - |
| teacher_max_gap | INTEGER | hayır | 0 | 10 |
| teacher_daily_max_lesson_hour | INTEGER | hayır | 1 | 12 |
| split_class_same_teacher_lessons | INTEGER | hayır | 0 | 1 |

**UNIQUE(schedule_id, teacher_id)**

#### 17. schedule_subject_configs
| Alan | Tip | Zorunlu | Min | Max |
|------|-----|---------|-----|-----|
| id | INTEGER PK | evet | auto | - |
| schedule_id | INTEGER FK | evet | - | - |
| subject_id | INTEGER FK | evet | - | - |
| pattern_min_gap | INTEGER | hayır | 0 | 5 |
| split_two_hour_blocks | INTEGER | hayır | 0 | 1 |

**UNIQUE(schedule_id, subject_id)**

#### 18. schedule_teacher_blocked_hours
| Alan | Tip | Zorunlu |
|------|-----|---------|
| id | INTEGER PK | evet |
| schedule_id | INTEGER FK | evet |
| teacher_id | INTEGER FK | evet |
| day_index | INTEGER | evet |
| period_index | INTEGER | evet |

**UNIQUE(schedule_id, teacher_id, day_index, period_index)**

#### 19. schedule_class_blocked_hours (sınıf bazlı engelli saatler)
| Alan | Tip |
|------|-----|
| id | INTEGER PK |
| schedule_id | INTEGER FK |
| class_id | INTEGER FK |
| day_index | INTEGER |
| period_index | INTEGER |

**UNIQUE(schedule_id, class_id, day_index, period_index)**

#### 20. schedule_classroom_blocked_hours
| Alan | Tip |
|------|-----|
| id | INTEGER PK |
| schedule_id | INTEGER FK |
| classroom_id | INTEGER FK |
| day_index | INTEGER |
| period_index | INTEGER |

**UNIQUE(schedule_id, classroom_id, day_index, period_index)**

#### 21. schedule_class_lesson_blocked_hours (class_lesson bazlı engelli)
| Alan | Tip |
|------|-----|
| id | INTEGER PK |
| schedule_id | INTEGER FK |
| class_lesson_id | INTEGER FK |
| day_index | INTEGER |
| period_index | INTEGER |

**UNIQUE(schedule_id, class_lesson_id, day_index, period_index)**

#### 22. schedule_class_lesson_assigned_hours (manuel atamalar)
Aynı yapı. **UNIQUE(schedule_id, class_lesson_id, day_index, period_index)**

#### 23. schedule_results (çözüm sonuçları — denormalize snapshot)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | |
| schedule_id | INTEGER FK | evet | → schedules, CASCADE |
| class_lesson_id | INTEGER | evet | referans (FK DEĞİL) |
| class_id | INTEGER | evet | referans |
| class_name | TEXT | evet | snapshot |
| subject_id | INTEGER | evet | referans |
| subject_name | TEXT | evet | snapshot |
| subject_code | TEXT | hayır | snapshot |
| teacher_id | INTEGER | hayır | referans |
| teacher_name | TEXT | hayır | snapshot |
| classroom_id | INTEGER | hayır | referans |
| classroom_name | TEXT | hayır | snapshot |
| classroom_code | TEXT | hayır | snapshot |
| hours_per_week | INTEGER | evet | |
| lesson_hours | TEXT | evet | JSON: [[(day,period),...],...] per pattern block |

---

### Silme Davranışları

| Silinen | class_lessons | Schedule kısıtları | schedule_results |
|---|---|---|---|
| class | CASCADE | CASCADE | Bozulmaz |
| teacher | SET NULL | CASCADE (teacher_config/blocked) | Bozulmaz |
| subject | RESTRICT | CASCADE | Bozulmaz |
| classroom | SET NULL | CASCADE (classroom_blocked) | Bozulmaz |
| class_lesson | - | CASCADE | Bozulmaz |
| schedule | - | CASCADE | CASCADE |

---

## Tauri Commands (Endpoint'ler)

### Temel Veri CRUD

| Command | Açıklama |
|---------|----------|
| `get_school_settings()` / `update_school_settings(input)` | Singleton |
| `list_school_hours()` / `set_school_hours(slots[])` | Toplu set (7×N grid) |
| `list/get/create/update/delete_subjects` | + code alanı |
| `list/get/create/update/delete_teachers` | |
| `list/get/create/update/delete_classrooms` | + code alanı |
| `list/get/create/update/delete_classes` | + homeroom_teacher_id; create'de available hours oto-doldur |
| `set_teacher_subjects(teacher_id, subject_ids[])` | Toplu set |
| `set_classroom_subjects(classroom_id, subject_ids[])` | Toplu set |
| `set_class_available_hours(class_id, slots[])` | Toplu set |

### Atama Sayfaları

| Command | UI Sayfası | Açıklama |
|---------|------------|----------|
| `list_class_subject_assignments()` | Sayfa 2 | Sınıf-ders matrisi |
| `set_class_subjects(class_id, [{subject_id, hours_per_week, pattern}])` | Sayfa 2 | Sınıfa toplu ders ekle (öğretmen/derslik boş) |
| `set_class_lesson_teacher(class_lesson_id, teacher_id)` | Sayfa 3 | Öğretmen ata |
| `set_class_lesson_classroom(class_lesson_id, classroom_id)` | Sayfa 4 | Derslik ata |
| `update_class_lesson(id, input)` | Detay | Tüm alanları güncelle (pattern, hours dahil) |
| `set_class_lesson_nosplit_hours(class_lesson_id, slots[])` | Detay | Bölünmeyen saatler |
| `list/get/create/update/delete_lesson_groups` | Sayfa 5 | + class_ids |

### Ders Programı CRUD

| Command |
|---------|
| `list/get/create/update/delete_schedules` |
| `get_schedule_global_config(schedule_id)` / `update_schedule_global_config(schedule_id, input)` |
| `set_schedule_teacher_config(schedule_id, teacher_id, input)` (upsert) |
| `set_schedule_subject_config(schedule_id, subject_id, input)` (upsert) |
| `set_schedule_teacher_blocked_hours(schedule_id, teacher_id, slots[])` |
| `set_schedule_class_blocked_hours(schedule_id, class_id, slots[])` |
| `set_schedule_classroom_blocked_hours(schedule_id, classroom_id, slots[])` |
| `set_schedule_class_lesson_blocked_hours(schedule_id, class_lesson_id, slots[])` |
| `set_schedule_class_lesson_assigned_hours(schedule_id, class_lesson_id, slots[])` |
| `save_schedule_results(schedule_id, results[])` | Çözüm sonrası |
| `get_schedule_results(schedule_id)` | Görüntüleme |

### Ders Programı Detay Sayfası — View Endpoint'leri

| Command | Dönüş |
|---------|-------|
| `get_schedule_view_for_class(schedule_id, class_id)` | Grid: her (day, period) için cell durumu (off-hour, blocked, assigned-lesson info, free) + bu sınıfın class_lesson listesi |
| `get_schedule_view_for_teacher(schedule_id, teacher_id)` | Grid: aynı yapı + bu öğretmenin verdiği class_lesson'lar |
| `get_schedule_view_for_classroom(schedule_id, classroom_id)` | Grid: aynı yapı + bu dersliğe atanmış class_lesson'lar |

**Cell durumları**:
- `off`: school_hours dışı veya class/teacher/classroom available değil
- `blocked`: ilgili blocked_hours tablosunda
- `assigned`: assigned_hours veya schedule_results'ta var (class_lesson detayı dahil)
- `free`: müsait

---

## Uygulama Adımları

### Adım 1: Proje setup
```bash
cd D:\dev
npm create tauri-app@latest scheduler-app
# React + TypeScript template
```

### Adım 2: Cargo bağımlılıkları
```toml
[dependencies]
tauri = { version = "2", features = ["devtools"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite"] }
tokio = { version = "1", features = ["full"] }
thiserror = "2"
```

### Adım 3: Migration SQL (`migrations/001_initial_schema.sql`)
- PRAGMA journal_mode=WAL, foreign_keys=ON
- 23 tablo (yukarıdaki sıra)
- Singleton INSERT: school_settings(id=1)
- updated_at trigger'ları

### Adım 4: DB katmanı (`db/mod.rs`)
- SqlitePool: `{app_data_dir}/scheduler.db`
- `sqlx::migrate!()`
- Tauri managed state

### Adım 5: Error handling (`error.rs`)
- `AppError`: DbError, ValidationError, NotFound, Conflict, ForeignKeyViolation
- `impl Serialize` (Tauri IPC)

### Adım 6: Model structs (`models/*.rs`)
- `#[derive(Serialize, Deserialize, sqlx::FromRow)]`
- Input/Output DTO'ları

### Adım 7: Validasyon (`validation/`)
- `pattern.rs`: JSON parse, sum check
- `timeslot.rs`: school_settings range check
- `class_lesson.rs`: teacher_subject + classroom_subject uyumluluk

### Adım 8: Tauri Commands
- Yukarıdaki tüm endpoint'ler
- Toplu set işlemleri transaction içinde
- Schedule view'lar JOIN'lerle assemble edilir

### Adım 9: React Frontend (iskelet)
- Vite + React + TypeScript + React Router
- Sayfa iskeletleri:
  - `/` Dashboard
  - `/subjects`, `/teachers`, `/classrooms`, `/classes`
  - `/assignments/class-subject`
  - `/assignments/teachers`
  - `/assignments/classrooms`
  - `/lesson-groups`
  - `/school-hours`
  - `/schedules`
  - `/schedules/:id` (detay, içinde class/teacher/classroom tab'leri)

## Doğrulama

1. `cargo build` derleme hatasız
2. `cargo tauri dev` uygulama açılır, DB oluşur (migration çalışır)
3. Her CRUD command React'tan invoke ile test edilir
4. FK kısıtları: silinme davranışları doğrulanır
5. Validasyon hataları: geçersiz pattern/range için error döner
6. Schedule view: temel veri varken doğru grid oluşur

## Referans Dosyalar
- `D:\dev\scheduler\Models.cs` — C# domain model
- `D:\dev\scheduler\data\fevzipasa.json` — Test verisi
- `D:\dev\scheduler\Scheduler.cs` — Solver veri kullanımı

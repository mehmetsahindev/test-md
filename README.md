# Tauri Masaüstü Uygulama - Backend Plan

## Context

Mevcut scheduler projesi (C# + OR-Tools) ders programı çözücü olarak çalışıyor. Şimdi bir masaüstü uygulama gerekiyor: okul yetkilisi temel verileri girer (dersler, sınıflar, öğretmenler, derslikler, atamalar), sonra birden fazla ders programı oluşturabilir. Her ders programı kendi config ve kısıtlarına sahiptir. Scheduler entegrasyonu şimdilik kapsam dışı.

**Kullanım akışı:**
1. Temel verileri gir: Dersler, Sınıflar, Derslikler, Öğretmenler
2. Atamaları yap: Sınıf→Ders, Öğretmen→Ders, Derslik→Ders
3. Ortak dersleri (lesson_groups) gir
4. Okul ders saatlerini ayarla
5. Ders programı oluştur → config ayarla → kısıtları gir → "Yerleştir" → sonuçları kaydet

**Silme stratejisi:** Sonuç snapshot — ders programı sonuçları denormalize saklanır (isimler dahil). Temel veri silinebilir, eski programlar bozulmaz.

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
│       ├── db/
│       │   └── mod.rs
│       ├── models/
│       │   ├── mod.rs
│       │   ├── school_settings.rs
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
│       │   ├── classes.rs
│       │   ├── teachers.rs
│       │   ├── subjects.rs
│       │   ├── classrooms.rs
│       │   ├── class_lessons.rs
│       │   ├── lesson_groups.rs
│       │   └── schedules.rs
│       └── validation/
│           ├── mod.rs
│           ├── pattern.rs
│           └── timeslot.rs
├── src/
│   ├── App.tsx
│   └── ...
├── package.json
├── index.html
└── vite.config.ts
```

## Veritabanı Şeması

### Entity-Relationship Diyagramı

```
                        TEMEL VERİ (Global)
                        ==================

subjects ──< teachers_subjects >── teachers
    |                                  |
    |──< classrooms_subjects >── classrooms
    |
    |──< class_lessons >── classes
    |        |                 |
    |        |-- teacher_id FK-┘ (teachers)
    |        |-- classroom_id FK (classrooms)
    |        |
    |──< lesson_groups          classes ──< class_available_hours
             |
             |──< lesson_group_classes >── classes


                   DERS PROGRAMI (Per-Schedule)
                   ============================

schedules
    |
    |──  schedule_global_config (1:1)
    |──< schedule_teacher_configs
    |──< schedule_subject_configs
    |──< schedule_teacher_blocked_hours
    |──< schedule_class_lesson_blocked_hours
    |──< schedule_class_lesson_nosplit_hours
    |──< schedule_class_lesson_assigned_hours
    |──< schedule_results (denormalized snapshot)
```

---

### TEMEL VERİ TABLOLARI

#### 1. school_settings (singleton, id=1)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique | Not |
|------|-----|---------|-----|-----|---------|--------|-----|
| id | INTEGER PK | evet | 1 | 1 | - | evet | CHECK(id=1) |
| school_name | TEXT | evet | 0 | 200 | '' | hayır | |
| days_per_week | INTEGER | evet | 1 | 7 | 5 | hayır | |
| hours_per_day | INTEGER | evet | 1 | 12 | 7 | hayır | |
| created_at | TEXT | evet | - | - | now() | hayır | ISO 8601 |
| updated_at | TEXT | evet | - | - | now() | hayır | trigger |

#### 2. classes (sınıflar)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 50 | - | evet |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 3. teachers (öğretmenler)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | hayır |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 4. subjects (dersler — soyut kavram: Matematik, Türkçe)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | evet |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 5. classrooms (derslikler)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | evet |
| sort_order | INTEGER | evet | - | - | 0 | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

#### 6. teachers_subjects (öğretmenin verebileceği dersler)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | autoincrement |
| teacher_id | INTEGER FK | evet | → teachers, ON DELETE CASCADE |
| subject_id | INTEGER FK | evet | → subjects, ON DELETE CASCADE |

**UNIQUE(teacher_id, subject_id)**

#### 7. classrooms_subjects (derslikte yapılabilecek dersler)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | autoincrement |
| classroom_id | INTEGER FK | evet | → classrooms, ON DELETE CASCADE |
| subject_id | INTEGER FK | evet | → subjects, ON DELETE CASCADE |

**UNIQUE(classroom_id, subject_id)**

#### 8. class_lessons (sınıf-ders atamaları)
| Alan | Tip | Zorunlu | Min | Max | Default | Not |
|------|-----|---------|-----|-----|---------|-----|
| id | INTEGER PK | evet | auto | - | autoincrement | |
| class_id | INTEGER FK | evet | - | - | - | → classes, CASCADE |
| subject_id | INTEGER FK | evet | - | - | - | → subjects, RESTRICT |
| teacher_id | INTEGER FK | hayır | - | - | NULL | → teachers, SET NULL |
| classroom_id | INTEGER FK | hayır | - | - | NULL | → classrooms, SET NULL |
| hours_per_week | INTEGER | evet | 1 | 40 | - | toplam haftalık saat |
| pattern | TEXT (JSON) | evet | - | - | '[]' | [2,2,2] formatı |
| lesson_group_tag | TEXT | hayır | 1 | 50 | NULL | split grup etiketi |
| sort_order | INTEGER | evet | - | - | 0 | |
| created_at | TEXT | evet | - | - | now() | |
| updated_at | TEXT | evet | - | - | now() | |

**UNIQUE(class_id, subject_id, teacher_id)**

**Validasyon (Rust, class_lesson oluşturulurken):**
- `teacher_id` verilmişse → `teachers_subjects` tablosunda `(teacher_id, subject_id)` kaydı olmalı
- `classroom_id` verilmişse → `classrooms_subjects` tablosunda `(classroom_id, subject_id)` kaydı olmalı
- `pattern`: JSON array, her eleman pozitif int (1..hours_per_day), toplam = hours_per_week. `[]` = otomatik

#### 9. class_available_hours (sınıf müsait saatler)
| Alan | Tip | Zorunlu | Range | Not |
|------|-----|---------|-------|-----|
| id | INTEGER PK | evet | auto | |
| class_id | INTEGER FK | evet | - | → classes, CASCADE |
| day_index | INTEGER | evet | 0..6 | App: 0..(days_per_week-1) |
| period_index | INTEGER | evet | 0..11 | App: 0..(hours_per_day-1) |

**UNIQUE(class_id, day_index, period_index)**
Yeni sınıf oluşturulunca tüm slotlar otomatik doldurulur.

#### 10. lesson_groups (senkron ders grupları)
| Alan | Tip | Zorunlu | Unique | Not |
|------|-----|---------|--------|-----|
| id | INTEGER PK | evet | evet | |
| subject_id | INTEGER FK | evet | evet | → subjects, RESTRICT |
| name | TEXT | evet | hayır | max 100 char, default '' |
| created_at | TEXT | evet | hayır | now() |
| updated_at | TEXT | evet | hayır | now() |

#### 11. lesson_group_classes (junction)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | |
| lesson_group_id | INTEGER FK | evet | → lesson_groups, CASCADE |
| class_id | INTEGER FK | evet | → classes, CASCADE |

**UNIQUE(lesson_group_id, class_id)**

---

### DERS PROGRAMI TABLOLARI (Per-Schedule)

#### 12. schedules (ders programları)
| Alan | Tip | Zorunlu | Min | Max | Default | Unique |
|------|-----|---------|-----|-----|---------|--------|
| id | INTEGER PK | evet | auto | - | autoincrement | evet |
| name | TEXT | evet | 1 | 100 | - | hayır |
| status | TEXT | evet | - | - | 'draft' | hayır |
| created_at | TEXT | evet | - | - | now() | hayır |
| updated_at | TEXT | evet | - | - | now() | hayır |

**status**: 'draft' | 'solved' | 'archived'

#### 13. schedule_global_config (ders programı genel config — singleton per schedule)
| Alan | Tip | Zorunlu | Min | Max | Default | Not |
|------|-----|---------|-----|-----|---------|-----|
| id | INTEGER PK | evet | auto | - | - | |
| schedule_id | INTEGER FK | evet | - | - | - | UNIQUE, → schedules, CASCADE |
| teacher_max_gap | INTEGER | hayır | 0 | 10 | NULL | NULL = kısıt yok |
| teacher_daily_max_lesson_hour | INTEGER | hayır | 1 | 12 | NULL | NULL = limit yok |
| pattern_min_gap | INTEGER | hayır | 0 | 5 | NULL | NULL = kısıt yok |
| split_class_same_teacher_lessons | INTEGER | evet | 0 | 1 | 0 | bool |
| split_class_lesson_groups | INTEGER | evet | 0 | 1 | 0 | bool |
| split_two_hour_blocks | INTEGER | evet | 0 | 1 | 0 | bool |

#### 14. schedule_teacher_configs (ders programı — öğretmen override)
| Alan | Tip | Zorunlu | Min | Max | Not |
|------|-----|---------|-----|-----|-----|
| id | INTEGER PK | evet | auto | - | |
| schedule_id | INTEGER FK | evet | - | - | → schedules, CASCADE |
| teacher_id | INTEGER FK | evet | - | - | → teachers, CASCADE |
| teacher_max_gap | INTEGER | hayır | 0 | 10 | NULL = global kullan |
| teacher_daily_max_lesson_hour | INTEGER | hayır | 1 | 12 | NULL = global kullan |
| split_class_same_teacher_lessons | INTEGER | hayır | 0 | 1 | NULL = global kullan |

**UNIQUE(schedule_id, teacher_id)**

#### 15. schedule_subject_configs (ders programı — ders override)
| Alan | Tip | Zorunlu | Min | Max | Not |
|------|-----|---------|-----|-----|-----|
| id | INTEGER PK | evet | auto | - | |
| schedule_id | INTEGER FK | evet | - | - | → schedules, CASCADE |
| subject_id | INTEGER FK | evet | - | - | → subjects, CASCADE |
| pattern_min_gap | INTEGER | hayır | 0 | 5 | NULL = global kullan |
| split_two_hour_blocks | INTEGER | hayır | 0 | 1 | NULL = global kullan |

**UNIQUE(schedule_id, subject_id)**

#### 16. schedule_teacher_blocked_hours (ders programında öğretmen bloklu saatler)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | |
| schedule_id | INTEGER FK | evet | → schedules, CASCADE |
| teacher_id | INTEGER FK | evet | → teachers, CASCADE |
| day_index | INTEGER | evet | 0..6 |
| period_index | INTEGER | evet | 0..11 |

**UNIQUE(schedule_id, teacher_id, day_index, period_index)**

#### 17. schedule_class_lesson_blocked_hours
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | |
| schedule_id | INTEGER FK | evet | → schedules, CASCADE |
| class_lesson_id | INTEGER FK | evet | → class_lessons, CASCADE |
| day_index | INTEGER | evet | 0..6 |
| period_index | INTEGER | evet | 0..11 |

**UNIQUE(schedule_id, class_lesson_id, day_index, period_index)**

#### 18. schedule_class_lesson_nosplit_hours (aynı yapı)
**UNIQUE(schedule_id, class_lesson_id, day_index, period_index)**

#### 19. schedule_class_lesson_assigned_hours (aynı yapı)
**UNIQUE(schedule_id, class_lesson_id, day_index, period_index)**

#### 20. schedule_results (çözüm snapshot — denormalize)
| Alan | Tip | Zorunlu | Not |
|------|-----|---------|-----|
| id | INTEGER PK | evet | autoincrement |
| schedule_id | INTEGER FK | evet | → schedules, CASCADE |
| class_lesson_id | INTEGER | evet | referans, FK DEĞİL (silinmiş olabilir) |
| class_id | INTEGER | evet | referans |
| class_name | TEXT | evet | snapshot |
| subject_id | INTEGER | evet | referans |
| subject_name | TEXT | evet | snapshot |
| teacher_id | INTEGER | hayır | referans |
| teacher_name | TEXT | hayır | snapshot |
| classroom_id | INTEGER | hayır | referans |
| classroom_name | TEXT | hayır | snapshot |
| hours_per_week | INTEGER | evet | |
| lesson_hours | TEXT | evet | JSON: [[0,1],[200,201,202]] (slot listeleri per pattern block) |

---

### Silme Davranışları Özeti

| Silinen veri | class_lessons | schedule config/hours | schedule_results |
|---|---|---|---|
| class silinirse | CASCADE (silinir) | CASCADE (silinir) | Bozulmaz (snapshot) |
| teacher silinirse | SET NULL | CASCADE (teacher config silinir) | Bozulmaz (snapshot) |
| subject silinirse | RESTRICT (atama varsa silinemez) | CASCADE | Bozulmaz (snapshot) |
| classroom silinirse | SET NULL | - | Bozulmaz (snapshot) |
| class_lesson silinirse | - | CASCADE (kısıtlar silinir) | Bozulmaz (snapshot) |
| schedule silinirse | - | CASCADE (tümü silinir) | CASCADE (silinir) |

---

## Uygulama Adımları

### Adım 1: Tauri v2 + React + Vite projesi oluştur
- `D:\dev\scheduler-app` dizininde `npm create tauri-app@latest`
- React + TypeScript template seç

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

### Adım 3: Migration SQL dosyası
- `migrations/001_initial_schema.sql` — yukarıdaki 20 tablo
- PRAGMA journal_mode = WAL, foreign_keys = ON
- Singleton satırları INSERT (school_settings, schedule_global_config)
- updated_at trigger'ları

### Adım 4: DB katmanı (`db/mod.rs`)
- SqlitePool oluştur: `{app_data_dir}/scheduler.db`
- `sqlx::migrate!()` ile migration çalıştır
- Pool'u Tauri managed state olarak kaydet

### Adım 5: Error handling (`error.rs`)
- `AppError` enum: DbError, ValidationError, NotFound, Conflict
- `impl Serialize for AppError` (Tauri IPC uyumlu)

### Adım 6: Model structs (`models/*.rs`)
- Her tablo için Rust struct
- `#[derive(Serialize, Deserialize, sqlx::FromRow)]`
- Input/output DTO'ları ayrı

### Adım 7: Validasyon (`validation/`)
- `pattern.rs`: JSON parse, pozitif int, toplam check
- `timeslot.rs`: day/period school_settings'e göre range check
- `class_lesson.rs`: teacher_subject ve classroom_subject uyumluluk check

### Adım 8: Tauri Commands — Temel CRUD
Her entity için: `list`, `get`, `create`, `update`, `delete`

| Command | Entity | Özel davranış |
|---------|--------|---------------|
| `get/update_school_settings` | school_settings | Singleton, sadece get+update |
| `list/get/create/update/delete_classes` | classes | Create'de tüm available hours oto-dolsun |
| `list/get/create/update/delete_teachers` | teachers | |
| `list/get/create/update/delete_subjects` | subjects | Delete'de RESTRICT (class_lesson varsa hata) |
| `list/get/create/update/delete_classrooms` | classrooms | |
| `set_teacher_subjects` | teachers_subjects | teacher_id + subject_id[] toplu set |
| `set_classroom_subjects` | classrooms_subjects | classroom_id + subject_id[] toplu set |
| `list/get/create/update/delete_class_lessons` | class_lessons | teacher/classroom uyumluluk validasyonu |
| `set_class_available_hours` | class_available_hours | class_id + slots[] toplu set |
| `list/get/create/update/delete_lesson_groups` | lesson_groups + classes | class_id listesi ile birlikte |

### Adım 9: Tauri Commands — Schedule CRUD
| Command | Açıklama |
|---------|----------|
| `list/get/create/update/delete_schedules` | Ders programı CRUD |
| `get/update_schedule_global_config` | Per-schedule config |
| `set_schedule_teacher_configs` | Per-schedule öğretmen override |
| `set_schedule_subject_configs` | Per-schedule ders override |
| `set_schedule_teacher_blocked_hours` | Toplu set |
| `set_schedule_class_lesson_blocked_hours` | Toplu set |
| `set_schedule_class_lesson_nosplit_hours` | Toplu set |
| `set_schedule_class_lesson_assigned_hours` | Toplu set |
| `save_schedule_results` | Çözüm sonuçlarını snapshot olarak kaydet |
| `get_schedule_results` | Kayıtlı sonuçları getir |

### Adım 10: React Frontend (minimal)
- Vite + React + TypeScript
- Basit sayfa yapısı: her entity için liste/form
- `@tauri-apps/api` invoke çağrıları

## Doğrulama

1. `cargo build` — derleme hatası yok
2. `cargo tauri dev` — uygulama açılır, DB oluşur
3. Her CRUD command React'tan test edilir
4. FK kısıtları: bağımlı kayıt silindiğinde doğru davranış
5. Validasyon: geçersiz pattern, range dışı değerler hata döner

## Referans Dosyalar
- `D:\dev\scheduler\Models.cs` — C# domain model
- `D:\dev\scheduler\data\fevzipasa.json` — Test verisi
- `D:\dev\scheduler\Scheduler.cs` — Solver'ın veriyi kullanımı

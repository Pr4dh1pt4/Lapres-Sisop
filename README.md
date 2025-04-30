# Lapres-Sisop


## 🧠 Penjelasan Fungsi

# 2️⃣ A
Anthony ingin semuanya serba instan dengan hanya satu perintah. Dengan satu perintah saja, Anthony bisa:
- Mendownload file ZIP yang berisi data film-film Netflix favoritnya.
- Mengekstrak file ZIP tersebut ke dalam folder yang sudah terorganisir.
- Menghapus file ZIP yang sudah tidak diperlukan lagi, supaya tidak memenuhi penyimpanan.
  
### 📄 `log_message(const char* message)`
Mencatat pesan ke file `log.txt` disertai timestamp (jam-menit-detik). Digunakan untuk mencatat proses penting seperti pengelompokan film.
  
  ```c
    void log_message(const char* message) {
          FILE* log = fopen("log.txt", "a");
          if (!log) return;
      
          time_t now = time(NULL);
          struct tm *t = localtime(&now);
          fprintf(log, "[%02d:%02d:%02d] %s\n", t->tm_hour, t->tm_min, t->tm_sec, message);
          fclose(log);
      }
  ```
- Fungsi ini mencatat pesan log ke dalam file log.txt dengan timestamp (jam:menit:detik).
- Fungsi ini membantu dalam debugging dan pelacakan aktivitas program (misal: saat proses pengelompokan film berlangsung).

### 📄 `parse_csv_line(char* line, char** fields, int max_fields)`
Memecah satu baris CSV menjadi array `fields`. Mengabaikan tanda kutip dan menghapus karakter newline. Digunakan oleh proses parsing CSV.

  ```c
    void parse_csv_line(char* line, char** fields, int max_fields) {
        int in_quotes = 0;
        int field_index = 0;
        char* start = line;
    
        for (char* p = line; *p && field_index < max_fields; p++) {
            if (*p == '"') {
                in_quotes = !in_quotes; 
            } else if (*p == ',' && !in_quotes) {
                *p = '\0';
                fields[field_index++] = start;
                start = p + 1;
            }
        }
        
        if (field_index < max_fields) {
            fields[field_index++] = start;
        }
    
        for (int i = 0; i < field_index; i++) {
            fields[i][strcspn(fields[i], "\r\n")] = 0;
        }
    }
  ```
- Memecah satu baris CSV menjadi array fields berdasarkan koma, memperhatikan kutipan ganda (").
- Fungsi ini digunakan untuk mengurai data dalam file CSV, seperti Title, Director, Country, dan Year.

### 📄 `int extract_zip()`

  ```c
    int extract_zip() {
        struct zip *archive;
        struct zip_file *file;
        struct zip_stat stat;
        char buf[100];
        int err;
    
        mkdir(DATA_FOLDER, 0777);
    
        archive = zip_open(ZIP_FILE, 0, &err);
        if (!archive) {
            zip_error_to_str(buf, sizeof(buf), err, errno);
            fprintf(stderr, "Error opening zip: %s\n", buf);
            return -1;
        }
    
        int entries = zip_get_num_entries(archive, 0);
        for (int i = 0; i < entries; i++) {
            if (zip_stat_index(archive, i, 0, &stat) == -1) continue;
    
            char path[256];
            snprintf(path, sizeof(path), "%s/%s", DATA_FOLDER, stat.name);
    
            char *dir = strdup(path);
            char *last_slash = strrchr(dir, '/');
            if (last_slash) {
                *last_slash = '\0';
                mkdir(dir, 0777);
            }
            free(dir);
    
            file = zip_fopen_index(archive, i, 0);
            if (!file) continue;
    
            FILE *out = fopen(path, "wb");
            if (!out) {
                zip_fclose(file);
                continue;
            }
    
            int bytes_read;
            while ((bytes_read = zip_fread(file, buf, sizeof(buf))) > 0) {
                fwrite(buf, 1, bytes_read, out);
            }
    
            fclose(out);
            zip_fclose(file);
        }
    
        zip_close(archive);
        return 0;
    }
  ```
- Membuka file ZIP `netflixData.zip`
- Mengekstrak semua isinya ke folder `data/`
- Membuat folder jika belum ada
- Menggunakan pustaka `libzip`
- Return `0` jika sukses, `-1` jika gagal

---

### 📄 `void delete_zip()`
- Fungsi ini akan menghapus file ZIP setelah diekstrak untuk menghemat penyimpanan.

  ```c
    void delete_zip() {
        remove(ZIP_FILE);
    }
  ```
---

### 📄 `void* menu_download(void* arg)`
  ```c
    void* menu_download(void* arg) {
        printf("Ekstrak isi ZIP lokal...\n");
        if (extract_zip() == 0) {
            printf("Berhasil diekstrak ke folder %s.\n", DATA_FOLDER);
            delete_zip();
            printf("ZIP dihapus untuk menghemat penyimpanan.\n");
        } else {
            printf("Gagal mengekstrak ZIP.\n");
        }
        pthread_exit(NULL);
    }
  ```
- Menjalankan fungsi ekstraksi ZIP (`extract_zip`)
- Menghapus ZIP setelah selesai (`delete_zip`)
- Menampilkan status ke layar

---

# 2️⃣ B
Anthony ingin mengelompokkan film-filmnya dengan dua cara yang sangat mudah:
1. Berdasarkan huruf pertama dari judul film.
2. Berdasarkan tahun rilis (release year).

### 📄 `void* group_by_abjad(void* arg)`
- Fungsi ini mengelompokkan film berdasarkan huruf pertama judul. Hasil disimpan dalam folder judul/ sesuai abjad atau simbol (#).
  
  ```c
    void* group_by_abjad(void* arg) {
        FILE* f = fopen(CSV_FILE, "r");
        if (!f) pthread_exit(NULL);
    
        mkdir("judul", 0777);
        char line[1024];
        fgets(line, sizeof(line), f);
    
        while (fgets(line, sizeof(line), f)) {
            char* fields[10];
            parse_csv_line(line, fields, 10);
            char* title = fields[0];
            char* director = fields[1];
            char* country = fields[2];
            char* year_str = fields[3];
            if (!title || !year_str || !director) continue;
    
            year_str[strcspn(year_str, "\r\n")] = 0;
    
            char logmsg[256];
            snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Abjad: sedang mengelompokkan untuk film %s", title);
            log_message(logmsg);
    
            char fname[20];
            char first = title[0];
            if (isalpha(first)) {
                snprintf(fname, sizeof(fname), "judul/%c.txt", toupper(first));
            } else if (isdigit(first)) {
                snprintf(fname, sizeof(fname), "judul/%c.txt", first);
            } else {
                snprintf(fname, sizeof(fname), "judul/#.txt");
            }
    
            FILE* out = fopen(fname, "a");
            if (out) {
                fprintf(out, "%s - %s - %s\n", title, year_str, director);
                fclose(out);
            }
        }
        fclose(f);
        pthread_exit(NULL);
    }
  ```
Thread untuk mengelompokkan film berdasarkan huruf awal judul:
- Folder output: `judul/`
- Nama file: `A.txt`, `B.txt`, dst. Jika bukan huruf/angka → `#.txt`
- Format isi: `Judul - Tahun - Sutradara`
- Logging untuk setiap film

---

### 📄 `void* group_by_year(void* arg)`
- Fungsi ini mengelompokkan film berdasarkan tahun rilis dan menyimpannya ke dalam folder tahun/<year>.txt.
  
  ```c
    void* group_by_year(void* arg) {
        FILE* f = fopen(CSV_FILE, "r");
        if (!f) pthread_exit(NULL);
    
        mkdir("tahun", 0777);
        char line[1024];
        fgets(line, sizeof(line), f);
    
        while (fgets(line, sizeof(line), f)) {
            char* fields[10];
            parse_csv_line(line, fields, 10);
            char* title = fields[0];
            char* director = fields[1];
            char* country = fields[2];
            char* year_str = fields[3];        
            if (!title || !year_str || !director) continue;
    
            year_str[strcspn(year_str, "\r\n")] = 0;
    
            char logmsg[256];
            snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Tahun: sedang mengelompokkan untuk film %s", title);
            log_message(logmsg);
    
            char fname[64];
            snprintf(fname, sizeof(fname), "tahun/%s.txt", year_str);
    
            FILE* out = fopen(fname, "a");
            if (out) {
                fprintf(out, "%s - %s - %s\n", title, year_str, director);
                fclose(out);
            }
        }
        fclose(f);
        pthread_exit(NULL);
    }
  ```
Thread untuk mengelompokkan film berdasarkan tahun:
- Folder output: `tahun/`
- Nama file: `1999.txt`, `2010.txt`, dst.
- Format isi: `Judul - Tahun - Sutradara`
- Logging untuk setiap film

---

### 📄 `void* menu_group(void* arg)`

  ```c
    void* menu_group(void* arg) {
        pthread_t t1, t2;
        pthread_create(&t1, NULL, group_by_abjad, NULL);
        pthread_create(&t2, NULL, group_by_year, NULL);
        pthread_join(t1, NULL);
        pthread_join(t2, NULL);
        printf("Film berhasil dikelompokkan berdasarkan abjad dan tahun.\n");
        pthread_exit(NULL);
    }
  ```
- Fungsi ini membuat dua thread: `group_by_abjad` dan `group_by_year`
- Menunggu hingga keduanya selesai (`pthread_join`)
- Memberi pesan status ke pengguna

---

# 2️⃣ C
Anthony ingin mengetahui statistik lebih mendalam tentang film-film yang dia koleksi. Misalnya, dia ingin tahu berapa banyak film yang dirilis *sebelum tahun 2000* dan *setelah tahun 2000*

### 📄 `void* generate_report(void* arg)`
- Fungsi ini digunakan untuk membuat laporan statistik jumlah film berdasarkan Country, dibedakan antara film sebelum tahun 2000 dan setelah tahun 2000.
  
  ```c
    void* generate_report(void* arg) {
        FILE* f = fopen(CSV_FILE, "r");
        if (!f) pthread_exit(NULL);
    
        typedef struct {
            char country[100];
            int before_2000;
            int after_2000;
        } CountryStat;
    
        CountryStat stats[100] = {0};
        int stat_count = 0;
    
        char line[1024];
        fgets(line, sizeof(line), f);
    
        while (fgets(line, sizeof(line), f)) {
            char* fields[10];
            parse_csv_line(line, fields, 10);
            char* title = fields[0];
            char* director = fields[1];
            char* country = fields[2];
            char* year_str = fields[3];  
            if (!title || !director || !country || !year_str) continue;
    
            country[strcspn(country, "\r\n")] = 0;
            year_str[strcspn(year_str, "\r\n")] = 0;
    
            int year = atoi(year_str);
            int found = 0;
    
            for (int i = 0; i < stat_count; i++) {
                if (strcmp(stats[i].country, country) == 0) {
                    found = 1;
                    if (year < 2000) stats[i].before_2000++;
                    else if(year > 2000) stats[i].after_2000++;
                    break;
                }
            }
    
            if (!found && stat_count < 100) {
                strcpy(stats[stat_count].country, country);
                if (year < 2000) stats[stat_count].before_2000 = 1;
                else if (year > 2000) stats[stat_count].after_2000 = 1;
                stat_count++;
            }
        }
        fclose(f);
    
        time_t now = time(NULL);
        struct tm *t = localtime(&now);
        char report_name[50];
        strftime(report_name, sizeof(report_name), "report_%d%m%Y.txt", t);
    
        FILE* report = fopen(report_name, "w");
        if (report) {
            for (int i = 0; i < stat_count; i++) {
                fprintf(report, "%d. Negara: %s\n", i+1, stats[i].country);
                fprintf(report, "Film sebelum 2000: %d\n", stats[i].before_2000);
                fprintf(report, "Film setelah 2000: %d\n\n", stats[i].after_2000);
            }
            fclose(report);
            printf("Report berhasil dibuat: %s\n", report_name);
        } else {
            printf("Gagal membuat report\n");
        }
    
        pthread_exit(NULL);
    }
  ```
Thread yang menghasilkan laporan statistik:
- Statistik jumlah film sebelum dan sesudah tahun 2000
- Dikelompokkan berdasarkan negara
- Disimpan di file `report_DDMMYYYY.txt`
- Format:


### 📄 `void* menu_report(void* arg)`
  ```c
    void* menu_report(void* arg) {
        pthread_t tid;
        pthread_create(&tid, NULL, generate_report, NULL);
        pthread_join(tid, NULL);
        pthread_exit(NULL);
    }
  ```

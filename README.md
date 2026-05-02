# Modul 3 - Deployment Multi-Node dengan Ansible

## Deskripsi

Repository ini berisi hasil pengerjaan Modul 3 (Ansible) untuk deployment multi-node. Setup mencakup instalasi Docker Engine pada setiap node (Praktikan 1) dan deployment backend service (Praktikan 2).

---

## Arsitektur

```
C09 (Ansible Controller)
   |
   | SSH
   |
   +-- node1 (backend target)  --> Express.js + PostgreSQL (port 3000)
   |
   +-- node2 (frontend target) --> (menunggu Praktikan 3)
```

---

## Struktur Folder

```
ansible/
├── inventory.yml
├── playbook.yml
├── group_vars/
│   └── backend.yml
└── roles/
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    └── backend/
        ├── files/
        │   ├── index.js
        │   └── package.json
        ├── tasks/
        │   └── main.yml
        └── templates/
            ├── .env.j2
            ├── Dockerfile.j2
            └── docker-compose.yml.j2
```

---

## Requirement

Pastikan environment berikut tersedia:

* 1 VM controller: `C09`
* 2 VM target: `node1`, `node2`
* Ansible terinstall di `C09`
* SSH key authentication sudah aktif (tanpa password)

---

## Setup SSH (WAJIB)

Dari dalam `C09`:

```bash
ssh ubuntu@192.168.1.14
ssh ubuntu@192.168.1.15
```

Jika tidak diminta password, SSH sudah benar.

---

## Konfigurasi Inventory

File: `inventory.yml`

```yaml
all:
  children:
    docker_nodes:
      hosts:
        node1:
          ansible_host: 192.168.1.14
          ansible_user: ubuntu
        node2:
          ansible_host: 192.168.1.15
          ansible_user: ubuntu

    backend:
      hosts:
        node1:

    frontend:
      hosts:
        node2:
```

---

## Cara Menjalankan

### 1. Test koneksi

```bash
ansible all -i inventory.yml -m ping
```

Expected:

```
node1 | SUCCESS
node2 | SUCCESS
```

### 2. Jalankan playbook

```bash
ansible-playbook -i inventory.yml playbook.yml
```

---

## Praktikan 1 - Docker Setup

### Deskripsi

Praktikan 1 bertanggung jawab atas setup environment multi-node dan instalasi Docker Engine pada setiap node menggunakan Ansible.

### Cakupan

* Instalasi Docker CE, Docker CLI, containerd, dan Docker Compose plugin pada `node1` dan `node2`.
* Konfigurasi firewall (UFW) dengan port 22 terbuka.
* Start dan enable Docker service.

### Verifikasi Docker

Masuk ke masing-masing node:

```bash
ssh ubuntu@192.168.1.14
docker run hello-world
```

```bash
ssh ubuntu@192.168.1.15
docker run hello-world
```

Expected output:

```
Hello from Docker!
```

### Catatan

* Docker Compose menggunakan versi terbaru dengan perintah `docker compose` (bukan `docker-compose`).
* Jika muncul error permission, jalankan `sudo usermod -aG docker ubuntu` lalu login ulang.

---

## Praktikan 2 - Backend Deployment

### Deskripsi

Praktikan 2 bertanggung jawab atas deployment backend service berbasis Node.js (Express) dan PostgreSQL pada `node1` menggunakan Docker Compose yang diorkestrasi oleh Ansible.

### Cakupan

* Role Ansible `backend` yang melakukan:
  * Membuka port backend (3000) pada firewall.
  * Membuat direktori deployment `/opt/backend`.
  * Deploy template Dockerfile, `.env`, dan `docker-compose.yml` menggunakan Jinja2.
  * Deploy source code aplikasi (`index.js`, `package.json`).
  * Menjalankan `docker compose up -d --build`.
  * Health check menggunakan module `uri` Ansible pada endpoint `/health`.

### Konfigurasi Variabel

File: `group_vars/backend.yml`

```yaml
db_name: "app_database"
db_username: "db_admin"
db_password: "SuperSecretPassword123"
backend_port: 3000
jwt_secret: "my_very_secure_jwt_secret_key"
```

### Komponen yang Di-deploy

| Komponen   | Image             | Port | Keterangan                     |
|------------|-------------------|------|--------------------------------|
| PostgreSQL | postgres:15-alpine| 5432 | Database internal, tidak exposed|
| Backend    | node:18-alpine    | 3000 | Express.js API                 |

### Endpoint API

| Method | Path        | Deskripsi                          |
|--------|-------------|------------------------------------|
| GET    | `/health`   | Health check, return status backend|
| GET    | `/`         | Informasi versi API                |
| POST   | `/register` | Registrasi user baru               |
| GET    | `/users`    | Daftar seluruh user                |

### Verifikasi Backend Manual

#### 1. Health Check

```bash
curl http://192.168.1.14:3000/health
```

Expected:

```json
{"status":"ok","message":"Backend is healthy"}
```

#### 2. Register User

```bash
curl -X POST http://192.168.1.14:3000/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"testuser@example.com","password":"password123"}'
```

Expected:

```json
{
  "message": "User registered successfully",
  "user": {
    "id": 1,
    "username": "testuser",
    "email": "testuser@example.com",
    "created_at": "2026-05-02T11:43:56.029Z"
  },
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

#### 3. Lihat Daftar User

```bash
curl http://192.168.1.14:3000/users
```

---

## Untuk Praktikan 3 (LANJUT DARI SINI)

Praktikan 3 bertanggung jawab atas deployment frontend pada `node2`.

### 1. Role baru: frontend

Buat folder dengan struktur berikut:

```
roles/frontend/
├── files/
│   └── (file statis frontend jika ada)
├── tasks/
│   └── main.yml
└── templates/
    ├── config.js.j2
    ├── Dockerfile.j2
    └── docker-compose.yml.j2
```

### 2. Gunakan group frontend

Node frontend sudah disiapkan di inventory:

```yaml
frontend:
  hosts:
    node2:
```

### 3. Variabel yang perlu disiapkan

Buat file `group_vars/frontend.yml` dengan isi:

```yaml
frontend_port: <port_frontend>
backend_url: "http://{{ hostvars['node1']['ansible_host'] }}:{{ hostvars['node1']['backend_port'] | default(3000) }}"
```

Catatan: `backend_url` wajib menggunakan Jinja2 template yang merujuk ke variabel dari Ansible inventory. Tidak boleh hardcode IP address. Jika terdapat isu konektivitas akibat double layer virtualization (misalnya WSL), sesuaikan dengan IP yang dapat dijangkau oleh `node2`.

### 4. Buat Dockerfile frontend

Gunakan base image `nginx`. Dockerfile harus meng-copy file konfigurasi nginx dan file statis frontend ke dalam container.

### 5. Buat template Jinja2

Buat minimal dua template:

* `config.js.j2` - Konfigurasi frontend yang berisi `backend_url` agar frontend mengetahui alamat backend API.
* `docker-compose.yml.j2` - Definisi service frontend dengan port mapping sesuai variabel `frontend_port`.

### 6. Buat task utama

File `tasks/main.yml` harus memuat task berikut:

1. Buka port frontend pada firewall menggunakan module `ufw`:

```yaml
- name: Buka port firewall untuk frontend ({{ frontend_port }})
  ufw:
    rule: allow
    port: "{{ frontend_port }}"
    proto: tcp
```

2. Buat direktori deployment, deploy semua template dan file yang diperlukan.

3. Jalankan container menggunakan `docker compose up -d --build`.

4. Lakukan health check menggunakan module `uri`:

```yaml
- name: Tunggu frontend siap (Healthcheck)
  uri:
    url: "http://localhost:{{ frontend_port }}"
    status_code: 200
  register: result
  until: result.status == 200
  retries: 6
  delay: 10
```

### 7. Update playbook.yml

Tambahkan play berikut pada `playbook.yml`:

```yaml
- hosts: frontend
  become: yes
  roles:
    - frontend
```

---

## Catatan Penting

### Docker Compose

Menggunakan versi terbaru:

```
docker compose
```

Bukan `docker-compose`.

### Permission Docker

Jika muncul error permission:

```bash
sudo usermod -aG docker ubuntu
```

Lalu login ulang.

### Firewall

Port yang dibuka per node:

| Node  | Port | Keterangan |
|-------|------|------------|
| node1 | 22   | SSH        |
| node1 | 3000 | Backend    |
| node2 | 22   | SSH        |
| node2 | (menunggu Praktikan 3) | Frontend |

---

## Status

* Infrastruktur siap
* Docker berjalan di kedua node
* Backend ter-deploy dan terverifikasi di node1
* Menunggu deploy frontend di node2

---

## Penutup

Environment ini sudah siap digunakan untuk deployment frontend oleh Praktikan 3. Backend API berjalan di `node1:3000` dan dapat diakses dari `node2` melalui jaringan internal.

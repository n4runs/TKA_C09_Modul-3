# Praktikan 1 (Docker Setup dengan Ansible)

## 👤 Deskripsi

Repository ini berisi hasil pengerjaan **Praktikan 1** pada Modul 3 (Ansible), yaitu setup environment multi-node menggunakan Ansible dan instalasi Docker Engine pada setiap node.

Setup ini akan menjadi **fondasi utama** untuk Praktikan 2 (backend) dan Praktikan 3 (frontend).

---

## 🧩 Arsitektur

```
C09 (Ansible Controller)
   ↓ SSH
node1 (backend target)
node2 (frontend target)
```

---

## 📁 Struktur Folder

```
ansible/
├── inventory.yml
├── playbook.yml
└── roles/
    └── docker/
        └── tasks/
            └── main.yml
```

---

## ⚙️ Requirement

Pastikan environment berikut tersedia:

* 1 VM controller: `C09`
* 2 VM target: `node1`, `node2`
* Ansible terinstall di `C09`
* SSH key authentication sudah aktif (tanpa password)

---

## 🔑 Setup SSH (WAJIB)

Dari dalam `C09`:

```bash
ssh ubuntu@192.168.1.14
ssh ubuntu@192.168.1.15
```

Jika tidak diminta password:

> ✅ SSH sudah benar

---

## 📄 Konfigurasi Inventory

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

## ▶️ Cara Menjalankan

```
kalo mau ngelanjutin dari gptku
https://chatgpt.com/share/69f47b75-1058-8323-98fd-76b028401c95
```
### 1. Test koneksi

```bash
ansible all -i inventory.yml -m ping
```

Expected:

```
node1 | SUCCESS
node2 | SUCCESS
```

---

### 2. Jalankan playbook

```bash
ansible-playbook -i inventory.yml playbook.yml
```

---

## 🐳 Verifikasi Docker

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

---

## ⚠️ Catatan Penting

### 1. Docker Compose

Menggunakan versi terbaru:

```
docker compose
```

(BUKAN `docker-compose`)

---

### 2. Permission Docker

Jika muncul error permission:

```bash
sudo usermod -aG docker ubuntu
```

Lalu login ulang.

---

### 3. Firewall

Hanya port 22 yang dibuka sesuai requirement praktikum.

---

## 🎯 Status

✔ Infrastruktur siap
✔ Docker berjalan
✔ Siap deploy backend

---

# Praktikan 2 (Backend Deployment)

## Deskripsi

Praktikan 2 bertanggung jawab atas deployment backend service berbasis Node.js (Express) dan PostgreSQL pada `node1` menggunakan Docker Compose yang diorkestrasi oleh Ansible.

---

## Struktur Folder (Update)

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

## Konfigurasi Variabel

File: `group_vars/backend.yml`

```yaml
db_name: "app_database"
db_username: "db_admin"
db_password: "SuperSecretPassword123"
backend_port: 3000
jwt_secret: "my_very_secure_jwt_secret_key"
```

---

## Cakupan Role Backend

Role `backend` menjalankan task berikut secara berurutan:

1. Membuka port backend (3000) pada firewall menggunakan module `ufw`.
2. Membuat direktori deployment `/opt/backend`.
3. Deploy template `Dockerfile.j2`, `.env.j2`, dan `docker-compose.yml.j2` menggunakan Jinja2.
4. Deploy source code aplikasi (`index.js`, `package.json`).
5. Menjalankan `docker compose up -d --build` di dalam `/opt/backend`.
6. Health check menggunakan module `uri` Ansible pada endpoint `/health`.

---

## Komponen yang Di-deploy

| Komponen   | Image              | Port | Keterangan                      |
|------------|--------------------|------|---------------------------------|
| PostgreSQL | postgres:15-alpine | 5432 | Database internal, tidak exposed|
| Backend    | node:18-alpine     | 3000 | Express.js API                  |

---

## Endpoint API

| Method | Path        | Deskripsi                           |
|--------|-------------|-------------------------------------|
| GET    | `/health`   | Health check, return status backend |
| GET    | `/`         | Informasi versi API                 |
| POST   | `/register` | Registrasi user baru                |
| GET    | `/users`    | Daftar seluruh user                 |

---

## Cara Menjalankan

### 1. Jalankan seluruh playbook (Docker + Backend)

```bash
ansible-playbook -i inventory.yml playbook.yml
```

Command ini menjalankan semua play dalam `playbook.yml` secara berurutan:
- Play pertama (`docker_nodes`) menginstall Docker pada `node1` dan `node2`.
- Play kedua (`backend`) men-deploy backend service pada `node1`.

### 2. Jalankan khusus role backend saja

Jika Docker sudah terinstall dan hanya ingin menjalankan ulang deployment backend:

```bash
ansible-playbook -i inventory.yml playbook.yml --limit backend
```

Command ini membatasi eksekusi hanya pada host yang termasuk dalam group `backend` (yaitu `node1`). Play `docker_nodes` akan di-skip untuk host yang tidak masuk filter. Play `backend` akan:
- Membuka port 3000 di firewall `node1`.
- Menyalin semua file dan template ke `/opt/backend`.
- Menjalankan `docker compose up -d --build` untuk membangun dan menjalankan container backend dan PostgreSQL.
- Menunggu hingga endpoint `http://localhost:3000/health` merespons HTTP 200 (maksimum 6 retry, interval 10 detik).

### 3. Jalankan dengan mode verbose

Untuk melihat detail output setiap task:

```bash
ansible-playbook -i inventory.yml playbook.yml --limit backend -v
```

---

## Verifikasi Backend Manual

### 1. Health Check

```bash
curl http://192.168.1.14:3000/health
```

Expected:

```json
{"status":"ok","message":"Backend is healthy"}
```

### 2. Register User

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

### 3. Lihat Daftar User

```bash
curl http://192.168.1.14:3000/users
```

Expected:

```json
{
  "users": [
    {
      "id": 1,
      "username": "testuser",
      "email": "testuser@example.com",
      "created_at": "2026-05-02T11:43:56.029Z"
    }
  ]
}
```

---

## Status

- Infrastruktur siap
- Docker berjalan di kedua node
- Backend ter-deploy dan terverifikasi di node1
- Menunggu deploy frontend di node2

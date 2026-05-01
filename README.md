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

## 🚀 Untuk Praktikan 2 (LANJUT DARI SINI)

Kamu akan mengerjakan:

### 📌 1. Role baru: backend

Buat folder:

```
roles/backend/
```

---

### 📌 2. Gunakan group `backend`

Node backend sudah disiapkan di:

```yaml
backend:
  hosts:
    node1
```

---

### 📌 3. Variabel yang perlu disiapkan

Gunakan `group_vars/backend.yml`:

```yaml
db_name:
db_username:
db_password:
backend_port:
jwt_secret:
```

---

### 📌 4. Yang perlu kamu buat

* Dockerfile backend
* Template `.env` (Jinja2)
* Template `docker-compose.yml`
* Task untuk:

  * membuka port backend
  * menjalankan container
  * health check (`uri`)

---

### 📌 5. Update playbook.yml

Tambahkan:

```yaml
- hosts: backend
  become: yes
  roles:
    - backend
```

---

## 🧠 Tips dari Praktikan 1

* Jangan ubah role docker (sudah stabil)
* Test backend tanpa frontend dulu
* Gunakan `ansible --limit backend` saat debugging
* Pastikan endpoint `/health` bisa diakses sebelum lanjut

---

## 🎯 Status

✔ Infrastruktur siap
✔ Docker berjalan
✔ Siap deploy backend

---

## 📌 Penutup

Environment ini sudah siap digunakan untuk:

* Deployment backend (Praktikan 2)
* Deployment frontend (Praktikan 3)

Silakan lanjutkan implementasi sesuai modul.

---

Good luck 🚀

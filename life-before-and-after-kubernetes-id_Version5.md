# Life Before and After Kubernetes: Pengalaman (Nggak) Sakti Deploy Aplikasi 🚀

**Penulis:** Kami, engineer yang juga masih suka pusing  
**Untuk:** Teman-teman yang penasaran, "Sebahagia apa sih hidup dengan Kubernetes?"

---

## Halo, Teman-Teman!

Jujur aja, dulu kami mikir, "Deploy aplikasi? Ya tinggal SSH ke server, copy file, terus jalanin, kan?"  
Tapi makin ke sini, hidup nggak sesimpel itu. Apalagi pas aplikasi mulai rame, traffic naik, tiba-tiba harus scaling, atau—yang paling nyebelin—servernya crash tengah malam.  
Nah, di artikel ini, kami mau cerita pengalaman kami coba dua cara: **manual (ala jadul)** dan **pakai Kubernetes (ala sultan automation)**.  
Mudah-mudahan setelah baca dan praktik, kalian bisa bilang, "Ooo, pantesan DevOps pada heboh sama K8s!" 😂

---

## Bagian 1: Hidup Manual, Rasanya Kayak LDR Jarak Jauh

### 1️⃣ Siapkan Aplikasi:  
Bayangin kita masak mie instan. Bahannya?  
- Satu file Go sederhana:  

```go
// app.go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        hostname, _ := os.Hostname()
        fmt.Fprintf(w, "Halo dari %s\n", hostname)
    })
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    http.ListenAndServe(":"+port, nil)
}
```

Masak dulu di dapur lokal:  
```bash
go build -o myapp app.go
```

---

### 2️⃣ Bikin VM:  
Kayak nyari kontrakan, kita pilih salah satu tempat tinggal.  
Contohnya di Google Cloud (GCP).  
- Buka GCP, bikin VM (Ubuntu 22.04, e2-micro, allow HTTP), tunggu alamat rumah (IP) keluar.

---

### 3️⃣ Belanja Bumbu:  
SSH ke VM (kontrakan baru kita):  
```bash
gcloud compute ssh <NAMA-VM> --zone=<ZONE>
sudo apt update
sudo apt install -y golang nginx
```

---

### 4️⃣ Kirim Makanan ke Kontrakan:  
Dari rumah (laptop), kita lempar mie instan (biner):  
```bash
scp -i ~/.ssh/google_compute_engine myapp <username>@<EXTERNAL_IP_VM>:/home/<username>/
```

---

### 5️⃣ Masak & Sajikan:  
Di VM, kita masak dua porsi sekaligus:  
```bash
PORT=8081 nohup ./myapp > app1.log 2>&1 &
PORT=8082 nohup ./myapp > app2.log 2>&1 &
```
Cek, sudah matang belum:  
```bash
curl http://localhost:8081
curl http://localhost:8082
```

---

### 6️⃣ Bagi-Bagi ke Teman Lewat Nginx (Load Balancer Manual):  
Kita set Nginx biar kalau ada tamu, bisa pilih porsi mana aja.  
```nginx
upstream myapp_backend {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    location / {
        proxy_pass http://myapp_backend;
    }
}
```
Aktifkan:  
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```
Tamu tinggal datang ke:  
http://<EXTERNAL_IP_VM>

---

### 7️⃣ Drama Dimulai: Scaling, Upgrade, Crash, Config (Hands-on!)

#### **A. Scaling (Tambah Porsi Manual)**
Misal, tiba-tiba harus tambah instance karena ada promo!

```bash
PORT=8083 nohup ./myapp > app3.log 2>&1 &
```
Edit Nginx agar kenal porsi baru:
```nginx
upstream myapp_backend {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
}
```
Restart Nginx:
```bash
sudo nginx -t
sudo systemctl restart nginx
```
**Cek:**  
Akses berulang ke http://<EXTERNAL_IP_VM> dan lihat hostname berbeda-beda.

---

#### **B. Simulasi Crash (Bakar Dapur Satu Panci)**
Matikan satu aplikasi:
```bash
ps aux | grep myapp
kill <PID_app1>
```
**Cek:**  
Akses ke http://<EXTERNAL_IP_VM> tetap jalan, tapi load balancing cuma ke dua porsi.

**Recovery?**  
Harus jalankan ulang manual:
```bash
PORT=8081 nohup ./myapp > app1.log 2>&1 &
```

---

#### **C. Upgrade (Ganti Resep Manual)**
Misal, ingin ubah pesan jadi "Halo Dunia!"

1. Edit app.go:
    ```go
    fmt.Fprintf(w, "Halo Dunia dari %s\n", hostname)
    ```
2. Build ulang:
    ```bash
    go build -o myapp app.go
    ```
3. Stop semua aplikasi:
    ```bash
    pkill myapp
    ```
4. Jalankan ulang:
    ```bash
    PORT=8081 nohup ./myapp > app1.log 2>&1 &
    PORT=8082 nohup ./myapp > app2.log 2>&1 &
    PORT=8083 nohup ./myapp > app3.log 2>&1 &
    ```
5. Cek hasil upgrade:
    ```bash
    curl http://localhost:8081
    ```

---

#### **D. Config & Secret (Cara Manual dan Berisiko)**
1. Tambah file `.env`:
    ```
    GREETING=Rahasia Dunia
    ```
2. Edit app.go supaya baca ENV:
    ```go
    greeting := os.Getenv("GREETING")
    fmt.Fprintf(w, "%s dari %s\n", greeting, hostname)
    ```
3. Jalankan:
    ```bash
    GREETING="Rahasia Dunia" PORT=8081 nohup ./myapp > app1.log 2>&1 &
    ```
**Risiko:**  
Kalau file .env ke-upload ke GitHub, rahasia bocor!  
Kalau lupa set ENV, greeting jadi kosong.

---

### Curhat:
Rasanya?  
- Capek! Banyak kerjaan yang sama, gampang lupa.
- Kalau pelanggan (traffic) tiba-tiba rame, bisa-bisa dapur kebakaran.
- Semua harus dipegang sendiri.  
- Tapi... buat belajar dasar, seru juga! Mirip belajar masak dari awal.

---

## Bagian 2: Hidup Modern—Kubernetes, Serasa Punya Dapur Otomatis!

### 1️⃣ Masak Sekali, Bisa Dimakan di Mana Saja (Dockerize)  
Biar nggak perlu install ulang bumbu tiap pindah dapur.

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY app.go .
RUN go build -o myapp app.go

FROM alpine:latest
WORKDIR /app
COPY --from=build /src/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

Build & kirim ke Docker Hub:
```bash
docker build -t <dockerhub-username>/myapp:v1 .
docker login
docker push <dockerhub-username>/myapp:v1
```

---

### 2️⃣ Punya Dapur Sendiri (Minikube)

```bash
minikube start
```

---

### 3️⃣ Kasih Resep ke Robot Dapur (Kubernetes YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: <dockerhub-username>/myapp:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```
Apply:
```bash
kubectl apply -f deployment.yaml
```
Lihat hasil masakan robot:
```bash
kubectl get pods
kubectl get service
minikube service myapp-svc --url
```

---

### 4️⃣ Drama Dimulai: Scaling, Upgrade, Crash, Secret (Hands-on ala K8s)

#### **A. Scaling Otomatis**
```bash
kubectl scale deployment myapp --replicas=5
kubectl get pods
```
Cek aplikasi:
```bash
curl $(minikube ip):30080
```
Tiap request bisa dapat instance berbeda!

---

#### **B. Crash Test (Bakar 1 Pod)**
Matikan satu pod:
```bash
kubectl get pods
kubectl delete pod <nama-salah-satu-pod>
```
Kubernetes langsung masak ulang!  
Cek:
```bash
kubectl get pods
```
Pod baru langsung muncul, nggak perlu panik.

---

#### **C. Upgrade (Rolling Update Otomatis)**
1. Edit app.go, build & push image baru (misal `v2`):
    ```go
    fmt.Fprintf(w, "Halo Dunia dari %s\n", hostname)
    ```
    ```bash
    docker build -t <dockerhub-username>/myapp:v2 .
    docker push <dockerhub-username>/myapp:v2
    ```
2. Edit deployment.yaml, ganti image jadi `v2`, lalu apply:
    ```yaml
        image: <dockerhub-username>/myapp:v2
    ```
    ```bash
    kubectl apply -f deployment.yaml
    ```
3. Lihat rollout:
    ```bash
    kubectl rollout status deployment/myapp
    ```
4. Cek hasil upgrade:
    ```bash
    curl $(minikube ip):30080
    ```

---

#### **D. Config & Secret (Aman dan Terkontrol)**
1. Buat ConfigMap:
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myapp-config
    data:
      GREETING: "Rahasia Dunia"
    ```
    ```bash
    kubectl apply -f configmap.yaml
    ```
2. Tambahkan ke deployment.yaml:
    ```yaml
            env:
            - name: GREETING
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: GREETING
    ```
    ```bash
    kubectl apply -f deployment.yaml
    kubectl rollout restart deployment myapp
    ```
3. Buat Secret:
    ```bash
    kubectl create secret generic myapp-secret --from-literal=API_KEY=ini_rahasia_banget
    ```
4. Tambahkan ke deployment.yaml:
    ```yaml
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: API_KEY
    ```
5. Cek dari dalam pod:
    ```bash
    kubectl exec -it <nama-pod> -- printenv | grep API_KEY
    ```

---

### 5️⃣ Cek Dapur, Nonton CCTV (Monitoring)

```bash
kubectl get pods
kubectl logs <nama-pod>
```
Bisa juga pasang Prometheus, Grafana, atau tools lain.  
Kalau mau pindah dapur (cloud lain), cukup bawa file YAML-nya.

---

### Ngobrol-ngobrol:  
Rasanya?  
- Hidup jadi lebih tenang, scaling tanpa drama.
- Kalau dapur rame, robot langsung tambah chef.
- Belajar Kubernetes memang butuh waktu, tapi setelah ngerti, serasa punya kru MasterChef pribadi.
- Tapi ya, buat masak mie instan doang, bawa seluruh robot ini kadang overkill juga... 😂

---

## Kesimpulan Ngopi

| Aspek         | Manual (Tradisional)           | Kubernetes (Modern)         |
|---------------|-------------------------------|----------------------------|
| Deploy        | SSH, copy file, run           | kubectl apply -f ...        |
| Scaling       | Jalankan process, edit nginx  | Ubah replicas, auto scale   |
| Recovery      | Restart manual                | Otomatis, self-healing      |
| Upgrade       | Stop, copy, jalanin ulang     | Rolling update, zero downtime|
| Config/Secret | ENV/file manual, rawan lupa   | ConfigMap/Secret, aman      |
| Monitoring    | Cek log/process manual        | kubectl logs/status, add-on |
| Portability   | Tiap VM beda-beda             | Bisa di mana saja           |

---

## Kapan Pakai Manual, Kapan Pakai Kubernetes?

- **Manual:**  
  Kalau cuma buat belajar, aplikasi kecil, atau sekadar nostalgia.  
  Kayak masak mie di kosan—cepat, murah, tapi capek sendiri kalau undangan buka puasa rame-rame.

- **Kubernetes:**  
  Kalau aplikasi makin rame, butuh scaling, nggak mau mati gaya pas traffic naik, atau pengen automation.  
  Serasa punya dapur profesional—semua bisa jalan otomatis, kita tinggal atur resepnya.

---

## Penutup

Setelah ngoprek dua dunia ini, kami sadar:  
Kubernetes itu keren, tapi bukan buat semua masalah.  
Mulai dari yang manual dulu, biar paham rasa capeknya—baru nanti saat butuh, pindah ke Kubernetes bakal lebih terasa nikmatnya.

Semoga cerita kami bisa bikin teman-teman makin semangat belajar,  
dan kalau ada yang mau sharing pengalaman, yuk ngobrol di kolom komentar! 🚀☕

---

_Share kalau menurutmu cerita ini bermanfaat!_
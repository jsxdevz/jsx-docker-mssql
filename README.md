# 🐳 MSSQL Server 2017 — Docker

<div align="center">
  <img src="https://img.shields.io/badge/Microsoft%20SQL%20Server-CC2927?style=for-the-badge&logo=microsoft%20sql%20server&logoColor=white" alt="MSSQL Server" />
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
</div>

---

## 📋 ภาพรวม

รัน Microsoft SQL Server 2017 บน Docker โดยใช้ named volume `mssql_data` เก็บข้อมูลทั้งหมด

- **Container:** `mssql17`
- **Port:** `1433`
- **Volume:** `mssql_data` → mount ที่ `/var/opt/mssql` ใน container
- **Timezone:** Asia/Bangkok

---

## ⚙️ ไฟล์ตั้งค่า

### `.env`

```env
MSSQL_SERVER_IMAGE=mcr.microsoft.com/mssql/server:2017-latest
MSSQL_CONTAINER_NAME=mssql17
MSSQL_SA_PASWORD=🔑🔑🔑🔑🔑🔑🔑🔑
MSSQL_PID=Developer
MSSQL_TZ=Asia/Bangkok
MSSQL_PORT=1433
```

### `docker-compose.yaml`

```yaml
version: "3.9"

services:
  MSSQL_SERVER:
    image: ${MSSQL_SERVER_IMAGE}
    container_name: ${MSSQL_CONTAINER_NAME}
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: ${MSSQL_SA_PASSWORD}
      MSSQL_PID: ${MSSQL_PID}
      TZ: ${MSSQL_TZ}
    user: "root"
    ports:
      - "${MSSQL_PORT}:1433"
    volumes:
      - mssql_data:/var/opt/mssql
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P '${MSSQL_SA_PASSWORD}' -Q 'SELECT 1' || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 40s

volumes:
  mssql_data:
    external: true
```

---

## 🔌 ข้อมูลการเชื่อมต่อ

| Field    | Value        |
|----------|--------------|
| Server   | `localhost,1433` |
| Username | `SA`         |
| Password | `🔑🔑🔑🔑🔑🔑🔑🔑` |

---

## 💾 Backup — สำรองข้อมูลจาก Linux Server มาเก็บที่ Windows

> ใช้เมื่อ: ต้องการดึงข้อมูลจาก Linux Server มาเก็บไว้ที่เครื่อง Windows

### ขั้นตอน

**ขั้นที่ 1 — บน Linux Server:** pack ข้อมูลใน volume ให้เป็นไฟล์ `.tar.gz`

``` powershell
sudo tar -czvf  [tar_path] -C [data_path]
```

```bash
# ตัวอย่าง
sudo tar -czvf /root/mssql_volume.tar.gz -C /var/lib/docker/volumes/mssql_data/_data .
```

> `-C <path> .` = เข้าไปใน path นั้นก่อน แล้วเก็บทุกอย่างข้างใน  
> ผลลัพธ์ใน tar จะเป็น `./data/`, `./log/` ฯลฯ — ไม่มี folder ครอบด้านนอก

**ขั้นที่ 2 — บน Windows (PowerShell):** ดึงไฟล์จาก Server มาที่เครื่อง
``` powershell
scp [linux_user]@[linux_ip]:[linux_path] "[windows path]"
```

```powershell
# ตัวอย่าง
scp root@10.182.4.16:/root/mssql_volume.tar.gz "D:\"
```

---

## 📦 Backup — สำรองข้อมูลจากเครื่อง Windows Local

> ใช้เมื่อ: มีข้อมูลอยู่ใน folder `mssql_volume\` บนเครื่อง Windows แล้ว และต้องการ pack เป็นไฟล์
``` powershell
tar -czvf "[tar_path]" -C "[data_path]" .
```

```powershell
# ตัวอย่าง
tar -czvf "D:\mssql_volume.tar.gz" -C "D:\mssql_volume" .
```

---

## 🔄 Restore — กู้คืนข้อมูลเข้า Docker Volume

> ใช้เมื่อ: ต้องการนำไฟล์ `mssql_volume.tar.gz` กลับเข้า volume `mssql_data`

### แนวคิด

Docker named volume ไม่สามารถเข้าถึงได้โดยตรงจาก Windows  
วิธีแก้คือ สร้าง container Ubuntu ชั่วคราวที่ mount volume ไว้ แล้วใช้มันเป็นตัวกลางในการ copy และ extract ไฟล์

```
[Windows] ──cp──▶ [temp-ubuntu container] ──extract──▶ [mssql_data volume]
```

### ขั้นตอน

**ขั้นที่ 1 — หยุด SQL Server container**

```powershell
docker compose down
```

**ขั้นที่ 2 — สร้าง Ubuntu container ชั่วคราว** ที่ mount volume `mssql_data` ไว้ที่ `/data`

```powershell
docker run -d --rm -v mssql_data:/data --name temp-ubuntu ubuntu sleep infinity
```

> `sleep infinity` = ให้ container รันค้างไว้เพื่อใช้งานต่อ  
> `--rm` = ลบ container อัตโนมัติเมื่อหยุด

**ขั้นที่ 3 — ลบข้อมูลเก่าใน volume ออกให้หมด**

```powershell
docker exec temp-ubuntu bash -c "rm -rf /data/*"
```

**ขั้นที่ 4 — Copy ไฟล์ `.tar.gz` เข้า container**

```powershell
docker cp "[tar_path]" temp-ubuntu:/data/
```


```powershell
# ตัวอย่าง
docker cp "D:\mssql_volume.tar.gz" temp-ubuntu:/data/
```

**ขั้นที่ 5 — Extract ไฟล์เข้า volume**

```powershell
docker exec temp-ubuntu bash -c "tar -xzvf /data/mssql_volume.tar.gz -C /data && rm /data/mssql_volume.tar.gz"
```

> Extract เสร็จแล้วลบไฟล์ `.tar.gz` ออกเพื่อไม่ให้ค้างใน volume

**ขั้นที่ 6 — หยุด Ubuntu container** (จะถูกลบอัตโนมัติเพราะมี `--rm`)

```powershell
docker stop temp-ubuntu
```

**ขั้นที่ 7 — เริ่ม SQL Server**

```powershell
docker compose up -d
```

**ขั้นที่ 8 — ตรวจสอบว่า container ทำงานปกติ**

```powershell
docker ps
docker logs mssql17 --tail 20
```

---

## 🆕 ติดตั้งครั้งแรก (Fresh Install)

1. สร้าง Docker Volume:

   ```powershell
   docker volume create mssql_data
   ```

2. (ถ้ามีไฟล์ backup) ทำ Restore ตามขั้นตอนด้านบนก่อน

3. รัน SQL Server:

   ```powershell
   docker compose up -d
   ```

---

## ❓ Troubleshooting

| ปัญหา | วิธีตรวจสอบ |
|-------|------------|
| Container ไม่ขึ้น | `docker logs mssql17` |
| เชื่อมต่อไม่ได้ | `docker ps` — ดูว่า port 1433 map อยู่ |
| Volume หาย | `docker volume ls` |
| รหัสผ่านไม่ผ่าน | ต้องมีอย่างน้อย 8 ตัว, มีตัวใหญ่ + ตัวเลข + อักขระพิเศษ |

---

<div align="center">
  <p><em>อัปเดตล่าสุด: 21 เมษายน 2569 (April 21, 2026)</em></p>
  <p>Made with ❤️ by jsx</p>
</div>

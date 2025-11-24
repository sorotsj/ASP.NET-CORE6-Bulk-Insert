# PWA React + SQLite (WASM) + Offline GPS Guide

บทสรุปแนวทางลงมือทำ PWA ด้วย React ที่บันทึกพิกัด GPS แบบออฟไลน์ด้วย SQLite (ผ่าน WebAssembly) แล้วซิงค์เมื่อออนไลน์อีกครั้ง ตัวอย่างนี้ใช้ `sql.js` (WASM), Service Worker และ IndexedDB เพื่อเก็บแบ็กอัปฐานข้อมูลในเครื่อง

## Stack หลัก
- React + TypeScript
- Service Worker + Workbox (precaching + runtime cache + Background Sync)
- `sql.js` / `@sqlite.org/sqlite-wasm` สำหรับรัน SQLite ในเบราว์เซอร์
- IndexedDB สำหรับเก็บ snapshot DB (ไฟล์ `.sqlite`) และคิวงานระหว่างออฟไลน์
- Geolocation API (`navigator.geolocation`) สำหรับอ่านพิกัด

## โครงสร้างไฟล์ที่แนะนำ
```
src/
  db/sqlite.ts         // โหลด/เปิด SQLite (sql.js) + backup/restore IndexedDB
  db/offlineQueue.ts   // คิวบันทึกตำแหน่งระหว่างออฟไลน์
  hooks/useGeolocation.ts
  sync/syncService.ts  // ดึงคิว -> sync server หรือประมวลผลต่อ
  service-worker.ts    // ใช้ Workbox ทำ precache + Background Sync queue
public/manifest.json   // PWA manifest
public/service-worker.js (build ออกมาจาก service-worker.ts)
```

## ขั้นตอนตั้งต้น
1. สร้างแอปด้วย Vite/CRA แล้วเปิด PWA support (เช่น Vite + `vite-plugin-pwa` หรือ Workbox CLI)
2. ติดตั้งไลบรารีหลัก
   ```bash
   npm i sql.js idb workbox-window
   npm i -D workbox-build workbox-cli typescript
   ```
3. เปิดใช้ HTTPS บน dev server เพื่อขอ permission GPS ได้ง่ายขึ้น (เช่น `vite --host --https`)
4. ทดสอบขอ location permission ขณะออนไลน์ก่อนเข้าสู่โหมดออฟไลน์

## ตัวอย่างโค้ดหลัก
### 1) โมดูล SQLite (sql.js + IndexedDB backup)
```ts
// src/db/sqlite.ts
import initSqlJs, { Database, SqlJsStatic } from 'sql.js';
import { openDB } from 'idb';

const DB_NAME = 'offline-gps-db';
const DB_KEY = 'sqlite-binary';

let sqlite: SqlJsStatic | null = null;
let db: Database | null = null;

export async function openDatabase() {
  if (!sqlite) {
    sqlite = await initSqlJs({ locateFile: (file) => `/sql-wasm.wasm` });
  }
  if (!db) {
    const snapshot = await loadSnapshot();
    db = snapshot ? new sqlite!.Database(snapshot) : new sqlite!.Database();
    db.run(`CREATE TABLE IF NOT EXISTS gps_logs(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      lat REAL, lng REAL, accuracy REAL,
      created_at TEXT
    );`);
  }
  return db!;
}

export async function persistSnapshot() {
  if (!db) return;
  const binary = db.export();
  const store = await openDB(DB_NAME, 1, { upgrade(db) { db.createObjectStore('snapshots'); } });
  await store.put('snapshots', binary, DB_KEY);
}

async function loadSnapshot() {
  const store = await openDB(DB_NAME, 1, { upgrade(db) { db.createObjectStore('snapshots'); } });
  return store.get('snapshots', DB_KEY);
}
```

### 2) Hook จับ GPS และบันทึกลง SQLite/คิว
```ts
// src/hooks/useGeolocation.ts
import { useEffect, useRef, useState } from 'react';
import { enqueueOfflineLog } from '../db/offlineQueue';
import { openDatabase, persistSnapshot } from '../db/sqlite';

export function useGeolocation(enable = true) {
  const [error, setError] = useState<string | null>(null);
  const watchId = useRef<number | null>(null);

  useEffect(() => {
    if (!enable || !navigator.geolocation) return;

    watchId.current = navigator.geolocation.watchPosition(async (pos) => {
      try {
        const { latitude, longitude, accuracy } = pos.coords;
        const db = await openDatabase();
        db.run('INSERT INTO gps_logs(lat, lng, accuracy, created_at) VALUES (?, ?, ?, ?);', [
          latitude,
          longitude,
          accuracy,
          new Date().toISOString(),
        ]);
        await persistSnapshot();
      } catch (err) {
        // ออฟไลน์หรือเขียน DB ไม่ได้ ให้ดันเข้าคิวไว้ก่อน
        await enqueueOfflineLog(pos);
      }
    }, (err) => setError(err.message), { enableHighAccuracy: true });

    return () => {
      if (watchId.current !== null) navigator.geolocation.clearWatch(watchId.current);
    };
  }, [enable]);

  return { error };
}
```

### 3) คิวตำแหน่งออฟไลน์ (IndexedDB)
```ts
// src/db/offlineQueue.ts
import { openDB } from 'idb';
const DB_NAME = 'offline-gps-queue';

export async function enqueueOfflineLog(position: GeolocationPosition) {
  const db = await openDB(DB_NAME, 1, { upgrade(db) { db.createObjectStore('queue', { autoIncrement: true }); } });
  await db.add('queue', position);
}

export async function drainQueue(handler: (p: GeolocationPosition) => Promise<void>) {
  const db = await openDB(DB_NAME, 1, { upgrade(db) { db.createObjectStore('queue', { autoIncrement: true }); } });
  const tx = db.transaction('queue', 'readwrite');
  const store = tx.objectStore('queue');
  let cursor = await store.openCursor();
  while (cursor) {
    await handler(cursor.value);
    await cursor.delete();
    cursor = await cursor.continue();
  }
  await tx.done;
}
```

### 4) Background Sync (Workbox) 
```ts
// service-worker.ts (ตัวอย่างย่อ)
import { registerRoute } from 'workbox-routing';
import { BackgroundSyncPlugin } from 'workbox-background-sync';
import { Queue } from 'workbox-background-sync';

const syncQueue = new Queue('gps-sync');

self.addEventListener('sync', (event) => {
  if (event.tag === 'gps-sync') {
    event.waitUntil(syncQueue.replayRequests());
  }
});

// ใช้ Background Sync ส่งคิวไป API เมื่อออนไลน์
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/gps'),
  new workbox.strategies.NetworkOnly({ plugins: [syncQueue] }),
  'POST'
);
```

### 5) Routine ซิงค์เมื่อออนไลน์
```ts
// src/sync/syncService.ts
import { drainQueue } from '../db/offlineQueue';
import { openDatabase, persistSnapshot } from '../db/sqlite';

export async function syncOfflineLogs() {
  const db = await openDatabase();
  await drainQueue(async (pos) => {
    const { latitude, longitude, accuracy } = pos.coords;
    db.run('INSERT INTO gps_logs(lat, lng, accuracy, created_at) VALUES (?, ?, ?, ?);', [
      latitude,
      longitude,
      accuracy,
      new Date().toISOString(),
    ]);
  });
  await persistSnapshot();
}

window.addEventListener('online', () => {
  syncOfflineLogs();
  navigator.serviceWorker?.ready.then((reg) => reg.sync.register('gps-sync')).catch(() => {/* ignore */});
});
```

## เคล็ดลับเพิ่มเติม
- ใช้ React Query/RTK Query เพื่อ cache + retry request อัตโนมัติ แล้วผูกกับ IndexedDB persistence
- เพิ่ม UI แสดงสถานะออฟไลน์, สรุปจำนวนคิวที่รอซิงค์ และปุ่ม “ซิงค์ทันที”
- ตั้งค่า `background_color`, `theme_color`, icons ใน `manifest.json` และทดสอบ Lighthouse PWA score
- หากต้องการรัน background location จริงขณะแอปปิดหน้าจอ ให้พิจารณาแพ็กด้วย Capacitor แล้วใช้ปลั๊กอิน Background Geolocation

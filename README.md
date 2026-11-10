
# gobak-sodor-online.{
  "name": "gobak-sodor-3d-multiplayer",
  "version": "1.0.0",
  "private": true,
  "description": "Gobak Sodor 3D cartoon multiplayer browser game for up to 20 players",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.21.2",
    "socket.io": "^4.8.1"
  }
}
# Gobak Sodor 3D Multiplayer (maks. 20 pemain)

Game browser 3D kartun sederhana menggunakan Three.js, Node.js, Express, dan Socket.IO.

## Menjalankan di komputer sendiri
1. Pasang Node.js versi 18 atau lebih baru.
2. Ekstrak ZIP ini.
3. Buka terminal pada folder proyek.
4. Jalankan `npm install`.
5. Jalankan `npm start`.
6. Buka `http://localhost:3000`.

Untuk bermain bersama lewat jaringan lokal, pemain lain membuka alamat IP komputer server dengan port `3000`.

## Agar bisa dimainkan online dari internet
Proyek ini sudah memiliki server multiplayer, tetapi belum di-hosting. Deploy folder proyek ke layanan hosting Node.js yang mendukung WebSocket (misalnya Render, Railway, atau VPS), lalu bagikan URL HTTPS kepada pemain. Pastikan port menggunakan `process.env.PORT`.

## Kontrol
- Komputer: WASD atau tombol panah.
- HP: tombol arah di dalam arena.
- Maksimal 20 pemain per room.
- Masuk room yang sama menggunakan kode room.
- Tim dibagi otomatis dan seimbang.

## Catatan versi awal
Ini adalah prototipe awal yang dapat dikembangkan. Server menangani room, tim, posisi, sentuhan, skor, dan durasi ronde. Aturan gobak sodor tradisional dapat disesuaikan lebih lanjut, termasuk jalur penjaga, pergantian peran antarronde, animasi, dan sistem anti-cheat.

const express = require("express");
const http = require("http");
const { Server } = require("socket.io");
const crypto = require("crypto");

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });
const PORT = process.env.PORT || 3000;
app.use(express.static("public"));

const rooms = new Map();
const MAX_PLAYERS = 20;
const FIELD = { x: 8, z: 14 };
const clamp = (v, lo, hi) => Math.max(lo, Math.min(hi, v));

function makeRoom(code) {
  return {
    code, players: {}, status: "waiting", scores: { red: 0, blue: 0 },
    roundEnds: 0, lastTag: 0, created: Date.now()
  };
}
function publicRoom(room) {
  return {
    code: room.code, status: room.status, scores: room.scores,
    players: Object.values(room.players).map(p => ({
      id: p.id, name: p.name, team: p.team, x: p.x, z: p.z,
      yaw: p.yaw, ready: p.ready, tagged: p.tagged
    }))
  };
}
function assignTeam(room) {
  const red = Object.values(room.players).filter(p => p.team === "red").length;
  const blue = Object.values(room.players).filter(p => p.team === "blue").length;
  return red <= blue ? "red" : "blue";
}
function emitRoom(room) { io.to(room.code).emit("state", publicRoom(room)); }

io.on("connection", socket => {
  socket.on("join", ({ code, name }) => {
    code = String(code || "").toUpperCase().replace(/[^A-Z0-9]/g, "").slice(0, 8);
    name = String(name || "Pemain").trim().slice(0, 16) || "Pemain";
    if (!code) code = crypto.randomBytes(3).toString("hex").toUpperCase();
    let room = rooms.get(code);
    if (!room) { room = makeRoom(code); rooms.set(code, room); }
    if (Object.keys(room.players).length >= MAX_PLAYERS) {
      socket.emit("joinError", "Room penuh (maksimal 20 pemain)."); return;
    }
    if (room.status === "playing") {
      socket.emit("joinError", "Permainan sedang berlangsung. Coba room lain."); return;
    }
    const team = assignTeam(room);
    const player = {
      id: socket.id, name, team, x: team === "red" ? -2.8 : 2.8,
      z: -6.2 + Math.random() * 1.2, yaw: 0, ready: false, tagged: false
    };
    room.players[socket.id] = player;
    socket.join(code);
    socket.data.room = code;
    socket.emit("joined", { id: socket.id, code, team });
    emitRoom(room);
  });

  socket.on("ready", () => {
    const room = rooms.get(socket.data.room); if (!room) return;
    const p = room.players[socket.id]; if (!p) return;
    p.ready = true; emitRoom(room);
  });

  socket.on("startGame", () => {
    const room = rooms.get(socket.data.room); if (!room) return;
    const players = Object.values(room.players);
    if (players.length < 2) { socket.emit("notice", "Butuh minimal 2 pemain."); return; }
    if (room.status !== "waiting") return;
    room.status = "playing"; room.roundEnds = Date.now() + 120000;
    room.scores = { red: 0, blue: 0 };
    players.forEach(p => { p.tagged = false; p.x = p.team === "red" ? -2.8 : 2.8; p.z = p.team === "red" ? -6 : 6; });
    emitRoom(room);
    io.to(room.code).emit("notice", "Mulai! Penyerang berusaha menyeberang, penjaga menghadang!");
  });

  socket.on("move", ({ x, z, yaw }) => {
    const room = rooms.get(socket.data.room); if (!room || room.status !== "playing") return;
    const p = room.players[socket.id]; if (!p || p.tagged) return;
    p.x = clamp(Number(x) || 0, -FIELD.x/2 + .35, FIELD.x/2 - .35);
    p.z = clamp(Number(z) || 0, -FIELD.z/2 + .35, FIELD.z/2 - .35);
    p.yaw = Number(yaw) || 0;
    // Catching: only the guarding team can tag an opponent at close range.
    const now = Date.now();
    if (now - room.lastTag > 700) {
      for (const other of Object.values(room.players)) {
        if (other.team !== p.team && !other.tagged &&
            Math.hypot(other.x - p.x, other.z - p.z) < 0.72) {
          other.tagged = true; room.lastTag = now;
          io.to(room.code).emit("notice", `${other.name} tersentuh penjaga!`);
          break;
        }
      }
    }
    // A successful crossing scores for the attacking side and returns the player to start.
    if ((p.team === "blue" && p.z < -6.5) || (p.team === "red" && p.z > 6.5)) {
      room.scores[p.team]++; p.z = p.team === "blue" ? 6.2 : -6.2;
      p.x = p.team === "blue" ? 2.8 : -2.8;
      io.to(room.code).emit("notice", `${p.name} berhasil menyeberang! +1 poin untuk tim ${p.team === "red" ? "Merah" : "Biru"}.`);
    }
  });

  socket.on("resetPlayer", () => {
    const room = rooms.get(socket.data.room); if (!room) return;
    const p = room.players[socket.id]; if (p) {
      p.tagged = false; p.x = p.team === "red" ? -2.8 : 2.8;
      p.z = p.team === "red" ? -6.2 : 6.2; emitRoom(room);
    }
  });

  socket.on("disconnect", () => {
    const code = socket.data.room; const room = rooms.get(code); if (!room) return;
    delete room.players[socket.id];
    if (!Object.keys(room.players).length) rooms.delete(code);
    else emitRoom(room);
  });
});

setInterval(() => {
  for (const room of rooms.values()) {
    if (room.status === "playing" && Date.now() >= room.roundEnds) {
      room.status = "waiting";
      io.to(room.code).emit("notice", `Waktu habis! Skor Merah ${room.scores.red} - Biru ${room.scores.blue}.`);
      emitRoom(room);
    }
  }
}, 1000);

server.listen(PORT, () => console.log(`Gobak Sodor berjalan di http://localhost:${PORT}`));

# Pemilu Web3

**Tally pemilu Indonesia yang bisa diaudit siapa pun.**
Setiap hasil TPS ditandatangani saksi, dicatat secara append-only, dan di-hash
on-chain — sehingga siapa pun bisa menghitung ulang total nasional tanpa harus
mempercayai pihak yang menerbitkannya.

> **Status:** desain + spesifikasi. Kontrak dan dashboard dijelaskan di bawah dan
> menjadi langkah berikutnya di roadmap. Belum ada prototipe yang berjalan.

---

## Masalahnya

Hasil pemilu Indonesia tetap berstatus tidak resmi selama berminggu-minggu
setelah hari pencoblosan. Pada Pemilu 2024 (sekitar 204 juta pemilih terdaftar,
sekitar 820 ribu TPS), rekapitulasi resmi memakan waktu kira-kira satu bulan.
Jeda itu diisi dengan tangkapan layar, klaim kemenangan sepihak, dan hoaks —
bukan karena hitungannya salah, tapi karena publik tidak punya cara untuk
**memeriksanya**.

Titik rawan yang diakui bukanlah basis datanya. Titik rawan adalah saat angka
bergerak naik: **TPS → kecamatan → kabupaten → nasional**. Angka yang berubah di
salah satu lompatan itu sulit dibuktikan dan mudah dituduhkan.

Upaya yang sudah ada (Kawal Suara, quick count) membuktikan bahwa **kecepatan**
bisa dicapai. Yang belum ada adalah **integritas yang bisa diverifikasi orang
asing**.

## Apa itu proyek ini

Sebuah tally yang:

- **Append-only** — koreksi adalah entri baru, tidak pernah menimpa.
- **Ditandatangani di sumber** — petugas KPU plus saksi, bukan satu pengunggah.
- **Di-anchor** — rolling hash ditulis ke chain publik, sehingga sejarah tidak
  bisa ditulis ulang oleh pihak yang memiliki datanya.
- **Bisa dihitung ulang** — verifier publik menurunkan total dari record mentah.

## Apa yang bukan

- **Bukan e-voting.** Surat suara kertas dan hitung manual di TPS tetap seperti
  sekarang. Ini batasan terpenting: kerahasiaan bilik suara adalah **fitur
  keamanan**, bukan keterbatasan warisan.
- **Bukan hasil resmi.** Rekapitulasi manual KPU tetap menjadi acuan. Proyek ini
  menghasilkan *tally yang bisa diaudit*, berlabel belum resmi, dipublikasikan
  berdampingan.
- **Bukan solusi politik uang.** Lihat [Yang Bukan Tujuan Proyek](#yang-bukan-tujuan-proyek).

## Prinsip desain

> **Tidak ada yang melaporkan total. Kontrak yang menghitungnya.**

Tidak ada `setNationalTotal()` di mana pun dalam desain ini. Angka nasional
adalah nilai turunan, fungsi dari kumpulan record. Kamu tidak bisa mengubah nilai
turunan tanpa mengubah record-nya, dan setiap record ditandatangani serta
dirantai.

## Cara kerjanya

```
  TPS                 Ingestor                 Ledger (EVM)            Publik
  ───────────────────────────────────────────────────────────────────────────
  hitung C-hasil ───►  hash foto        ───►  submitResult()   ───►  dashboard
  ditandatangani       + angka               ├ cek invariant       ├ tally live
  KPU + 2 saksi        + tanda tangan saksi  ├ revert jika salah   └ verifikasi TPS
                                             ├ tally += suara
                                             └ rollingRoot = H(prev, rec)
```

1. TPS menghitung surat suara kertas dan mengisi formulir C-hasil, ditandatangani
   basah seperti sekarang.
2. Petugas KPU dan tiap saksi memotret **formulir yang sama** lewat aplikasi
   ingestor — offline-first, tersinkron saat ada sinyal. Tiap salinan adalah
   pengamatan independen.
3. Ingestor meng-hash foto dan mengirim angka beserta tanda tangan saksi. Record
   yang tidak valid akan revert.
4. Dashboard menampilkan tally berjalan, selalu dengan **penyebut kelengkapan
   pelaporan** (contoh: `42.318 / 820.000 TPS melapor — 5,2%`).
5. Siapa pun bisa mengunduh record mentah, menghitung ulang total, dan
   membandingkannya dengan rolling root on-chain.

## Komponen

| Komponen | Tanggung jawab |
| --- | --- |
| `contracts/ElectionLedger.sol` | Cek invariant, tally on-chain, rolling audit root, role gating |
| `scripts/deploy.js` | Deploy ke testnet |
| `scripts/seed.js` | Memuat record contoh dan menghasilkan `data/seed.json` |
| `web/` | Dashboard statis + verifier per TPS, membaca chain langsung via `viem` |
| `data/seed.json` | Leaf set publik yang dipakai verifier |

## Skema record

```solidity
struct Result {
    bytes32 tpsId;
    uint32  dpt;              // pemilih terdaftar di TPS ini
    uint32  ballotsIssued;    // surat suara yang dikeluarkan
    uint32  ballotsValid;
    uint32  ballotsInvalid;
    uint32  abstain;          // hasil yang sah, bukan sisa
    uint16[] votes;           // per paslon / partai
    bytes32 photoHash;        // keccak256 foto C-hasil yang ditandatangani
}
```

`abstain` adalah field kelas satu. Pemilu punya **plafon** (pemilih eligible),
tidak pernah punya target. Desain apa pun yang menuntut partisipasi mencapai
angka persis harus memalsukan suara untuk mencapainya — dan itu justru definisi
kecurangan yang diklaim ingin dicegah.

## Invariant yang ditegakkan on-chain

```solidity
ballotsValid + ballotsInvalid + abstain == ballotsIssued
ballotsIssued <= dpt
sum(votes) == ballotsValid
```

Record yang melanggar salah satunya akan **revert saat penulisan** dan tidak
pernah masuk ledger. Inilah alasan terkuat memakai ledger alih-alih database:
aturannya ditegakkan oleh kode, bukan oleh kejujuran operator.

Rolling root membuat revisi terlihat:

```solidity
rollingRoot = keccak256(abi.encode(rollingRoot, recordHash));
```

Mengirim ulang satu TPS akan menambah versi baru. Riwayat koreksinya tetap bisa
dibaca — dan itu terbaca sebagai transparansi, bukan sebagai kecurangan.

> `shortcut:` hash chain memberi integritas kronologis tapi inclusion proof-nya
> O(n). Upgrade ke Merkle tree / MMR kalau ukuran proof per record jadi penting
> pada skala nasional.

## Kenapa ledger, bukan database?

Database bisa membuktikan **apa** yang tersimpan sekarang. Ledger bisa
membuktikan bahwa **tidak ada yang berubah sejak waktu tertentu** — klaim yang
tidak bisa dibuat oleh pihak yang justru sedang kita verifikasi. Itu satu-satunya
justifikasi. Semua bagian lain dari sistem ini (tanda tangan, hashing, penyimpanan
append-only) bisa jalan dengan infrastruktur biasa.

## Data

`data/seed.json` bersifat **sintetis** dan diberi label demikian. Tujuannya agar
seluruh pipeline bisa didemokan end to end. Ganti dengan data C-hasil asli
sebelum membuat klaim apa pun soal akurasi.

## Yang Bukan Tujuan Proyek

Ditulis dengan sengaja — batasan adalah bagian dari desain.

- **Politik uang.** Pertahanan utamanya justru karena surat suara di bilik
  tertutup tidak bisa dibuktikan ke siapa pun. Sistem apa pun yang memberi bukti
  terverifikasi akan dipakai pembeli suara untuk menagih sebelum membayar. Voting
  yang bisa dilacak memperburuk ini, bukan memperbaikinya.
- **Voting jarak jauh / internet.** Alasan yang sama, ditambah malware di sisi
  klien dan kesenjangan digital.
- **Kebenaran data sumber.** Kalau formulirnya sudah diisi salah, ledger
  menyimpan kesalahan itu secara permanen dan dengan tanda tangan yang sah.
  Garbage in, gospel out.
- **Operasi tanpa kepercayaan.** Kepercayaan tidak hilang; ia berpindah — ke
  penerbit kredensial, pemegang kunci, dan penulis kontrak. Ini pergeseran, bukan
  penghapusan.

## Roadmap

- [x] Perumusan masalah, threat model, non-goals
- [ ] `ElectionLedger.sol` + test invariant
- [ ] Multi-tanda tangan saksi (`k`-of-`n` per record)
- [ ] Rolling root + Merkle inclusion proof
- [ ] Generasi `seed.json` dari data contoh
- [ ] Dashboard: tally live dengan kelengkapan pelaporan
- [ ] Halaman verifier per TPS
- [ ] Deploy ke testnet publik + tx hash anchor di README ini
- [ ] Rolling root di-anchor ke chain publik secara berkala

## Tech stack

Solidity `^0.8.24` · Hardhat · `viem` · frontend statis (tanpa backend;
dashboard membaca state chain langsung).

## Lisensi

MIT

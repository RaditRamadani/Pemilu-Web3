# Pemilu Web3

**Rekapitulasi pemilu Indonesia yang berjalan di atas ledger.**
Record hasil TPS disimpan on-chain, total nasional dihitung oleh kontrak, dan
aturan validasi ditegakkan saat penulisan.

> **Status:** desain + spesifikasi. Kontrak dan dashboard dijelaskan di bawah dan
> menjadi langkah berikutnya di roadmap. Belum ada prototipe yang berjalan.

---

## Apa itu proyek ini

Pemilu Web3 adalah rekapitulasi suara pemilu Indonesia yang dijalankan di atas
blockchain. Di sini blockchain bukan lapisan tambahan di atas sistem lama, tapi
tempat record hasil TPS disimpan, tempat total nasional dihitung, dan tempat
aturan validasi ditegakkan.

Yang tetap sama dengan pemilu hari ini: surat suara kertas, bilik tertutup, dan
hitung manual di TPS. Yang baru: begitu TPS selesai menghitung, hasilnya
ditandatangani petugas KPU dan saksi lalu masuk ke ledger sebagai record
permanen. Setelah itu tidak ada satu pihak pun yang bisa mengubahnya tanpa
meninggalkan jejak, dan total nasional bukan angka yang dilaporkan siapa pun,
melainkan hasil penjumlahan kontrak atas record yang ada.

Hasilnya, pemilu bisa dipantau berjalan dan diverifikasi siapa saja, tanpa
menunggu rekapitulasi resmi selesai dan tanpa harus mempercayai satu institusi.

## Masalahnya

Hasil pemilu Indonesia tetap berstatus tidak resmi selama berminggu-minggu
setelah hari pencoblosan. Pada Pemilu 2024 (sekitar 204 juta pemilih terdaftar,
sekitar 820 ribu TPS), rekapitulasi resmi memakan waktu kira-kira satu bulan.
Jeda itu diisi dengan tangkapan layar, klaim kemenangan sepihak, dan hoaks,
bukan karena hitungannya salah, tapi karena publik tidak punya cara untuk
**memeriksanya**.

Titik rawan yang diakui bukanlah basis datanya. Titik rawan adalah saat angka
bergerak naik: **TPS, kecamatan, kabupaten, nasional**. Angka yang berubah di
salah satu lompatan itu sulit dibuktikan dan mudah dituduhkan.

Upaya yang sudah ada (Kawal Suara, quick count) membuktikan bahwa **kecepatan**
bisa dicapai. Yang belum ada adalah **integritas yang bisa diverifikasi orang
asing**.

## Peran blockchain

Blockchain di proyek ini mengerjakan enam hal, dan semuanya adalah fungsi inti,
bukan hiasan.

**1. System of record.** Record angka tiap TPS disimpan di storage kontrak, bukan
di database KPU. Salinan yang dipegang semua node tidak bisa dihapus sepihak.

**2. Total dihitung kontrak.** Tidak ada fungsi `setNationalTotal()`. Angka
nasional adalah state turunan dari record, jadi mengubahnya berarti mengubah
record yang bertanda tangan.

**3. Aturan ditegakkan saat penulisan.** Invariant dicek di dalam `submitResult()`.
Record yang tidak konsisten akan revert dan tidak pernah masuk ledger. Operator
tidak bisa memilih untuk mengabaikan aturan.

**4. Kontrol akses on-chain.** Peran KPU, saksi, dan Bawaslu adalah state kontrak.
Menambah atau mencabut peran selalu tercatat dan tidak bisa dibatalkan diam-diam.

**5. Rekapitulasi berjenjang sebagai state transition.** Naik dari TPS ke
kecamatan, kabupaten, lalu nasional adalah transisi state yang meninggalkan jejak
dan bisa dibandingkan dengan level di bawahnya.

**6. Jejak sengketa.** Keberatan saksi, hash bukti, dan penyelesaiannya dicatat
permanen, sehingga proses keberatan tidak hanya hidup di berita.

## Arsitektur on-chain

| Data | Lokasi | Alasan |
| --- | --- | --- |
| Angka hasil TPS | on-chain | menjadi system of record, tiap perubahan punya jejak |
| Tally nasional | on-chain | nilai turunan, tidak pernah dilaporkan manual |
| Peran dan kunci saksi | on-chain | kontrol akses tidak bisa diubah diam-diam |
| Metadata TPS dan daerah | on-chain | dibutuhkan untuk rekapitulasi berjenjang |
| Foto C-hasil | off-chain | ukuran besar, on-chain cukup hash-nya |
| Hash foto dan tanda tangan | on-chain | pengikat antara record dan bukti fisiknya |

Pemisahan ini disengaja. Angka dan aturan harus on-chain karena di situlah
sengketa terjadi. Foto tetap off-chain karena menyimpannya on-chain mahal dan
tidak menambah jaminan apa pun di luar hash-nya.

## Alur kerja

```
  TPS                 Ingestor                  Ledger (EVM)             Publik
  ------------------------------------------------------------------------------
  hitung C-hasil --->  hash foto          --->  submitResult()   --->  dashboard
  ditandatangani       + angka                 + cek invariant          + tally live
  KPU + 2 saksi        + tanda tangan saksi    + tally += suara         + verifikasi
                                               + rekap berjenjang         per TPS
```

1. TPS menghitung surat suara kertas dan mengisi formulir C-hasil, ditandatangani
   basah seperti sekarang.
2. Petugas KPU dan tiap saksi memotret formulir yang sama lewat aplikasi
   ingestor, offline-first, tersinkron saat ada sinyal. Tiap salinan adalah
   pengamatan independen.
3. Ingestor meng-hash foto dan mengirim angka beserta tanda tangan saksi.
   Kontrak menolak record yang tidak valid atau tanda tangannya kurang.
4. Dashboard menampilkan tally berjalan, selalu dengan penyebut kelengkapan
   pelaporan (contoh: `42.318 / 820.000 TPS melapor, 5,2%`).
5. Setelah satu wilayah lengkap, rekapitulasi tingkat atasnya difinalisasi
   on-chain. Siapa pun bisa membandingkan angka tiap level dengan level di
   bawahnya.

## Skema record

```solidity
struct Result {
    bytes32 tpsId;
    uint8   regionLevel;      // 1 = kecamatan, 2 = kabupaten, 3 = provinsi
    bytes32 regionId;
    uint32  dpt;              // pemilih terdaftar di TPS ini
    uint32  ballotsIssued;    // surat suara yang dikeluarkan
    uint32  ballotsValid;
    uint32  ballotsInvalid;
    uint32  abstain;          // hasil yang sah, bukan sisa
    uint16[] votes;           // per paslon atau partai
    bytes32 photoHash;        // keccak256 foto C-hasil yang ditandatangani
}
```

`abstain` adalah field kelas satu. Pemilu punya plafon (pemilih eligible), tidak
pernah punya target. Desain apa pun yang menuntut partisipasi mencapai angka
persis harus memalsukan suara untuk mencapainya, dan itu justru definisi
kecurangan yang diklaim ingin dicegah.

## Invariant yang ditegakkan on-chain

```solidity
ballotsValid + ballotsInvalid + abstain == ballotsIssued
ballotsIssued <= dpt
sum(votes) == ballotsValid
```

Record yang melanggar salah satunya akan revert dan tidak pernah masuk ledger.
Inilah yang membedakan ledger dari database: di database, aturan adalah
kesepakatan yang bisa dilanggar operator. Di kontrak, aturan adalah kode yang
harus dilalui operator.

Kontrak juga menolak record yang tanda tangan saksinya kurang dari ambang
`k`-of-`n`, sehingga tidak ada satu orang yang bisa menulis hasil sendirian.

## Rekapitulasi berjenjang

```
  TPS               Kecamatan          Kabupaten          Nasional
  submitResult  ->  finalizeRecap  ->  finalizeRecap  ->  finalizeRecap
  (per TPS)         (agregasi)         (agregasi)         (agregasi)
```

Tiap level adalah state kontrak yang dihitung dari level di bawahnya. Kalau angka
kecamatan tidak sama dengan jumlah TPS di dalamnya, transisi itu gagal. Ini
menutup titik rawan klasik, yaitu perubahan angka saat naik level, karena
lompatannya sekarang harus lolos pemeriksaan kode.

## Interface kontrak

```solidity
function submitResult(Result calldata r, bytes[] calldata witnessSigs) external;
function finalizeRecap(uint8 level, bytes32 regionId) external;
function tallyOf(uint16 candidate) external view returns (uint256);
function regionTotal(bytes32 regionId) external view returns (uint32, uint32, uint32);
function rollingRoot() external view returns (bytes32);
```

Kontrak dipecah tiga supaya tanggung jawabnya jelas:

| Kontrak | Tanggung jawab |
| --- | --- |
| `ElectionLedger.sol` | Record TPS, invariant, tally, rolling root |
| `Recapitulation.sol` | Agregasi berjenjang dan finalisasi wilayah |
| `WitnessRegistry.sol` | Peran, kunci saksi, ambang tanda tangan |

## Manfaat

**1. Hasil bisa dipantau tanpa menunggu sebulan.** Tally berjalan sejak TPS
pertama melapor, dan selalu ditampilkan bersama persentase TPS yang sudah masuk,
jadi angka awal tidak bisa dibaca sebagai hasil akhir.

**2. Perubahan angka jadi bukti, bukan tuduhan.** Kalau angka berbeda antara
tingkat TPS dan tingkat kecamatan, selisihnya terlihat dan bisa ditunjuk.

**3. Verifikasi tidak perlu izin.** Siapa pun bisa membaca state kontrak dan
menghitung ulang total nasional sendiri. Tidak perlu akun, tidak perlu menjadi
lembaga, tidak perlu percaya KPU.

**4. Koreksi justru menambah kepercayaan.** Karena koreksi tercatat sebagai entri
baru, riwayat revisi terbuka. Yang di sistem lain tampak seperti manipulasi, di
sini tampak seperti transparansi.

**5. Data tidak valid tidak bisa masuk sama sekali.** Invariant ditegakkan saat
penulisan, bukan dicek belakangan. Ini juga melindungi penyelenggara, karena
kesalahan hitung administratif tertangkap di sumber sebelum menjadi isu nasional.

**6. Tidak butuh perangkat baru.** Pencatatan memakai HP saksi yang sudah ada di
TPS. Tidak ada pengadaan perangkat nasional, tidak ada instalasi di 820 ribu
titik.

**7. Sengketa punya satu sumber kebenaran.** KPU, Bawaslu, saksi, dan publik
membaca record yang sama. Sengketa bergeser dari "versi siapa yang benar" menjadi
"record ini cocok atau tidak dengan hash dan tanda tangannya".

## Batasan dan risiko

Bagian ini bagian dari desain, bukan catatan kaki.

- **Gas.** Menyimpan record lengkap on-chain mahal di mainnet dengan 820 ribu
  TPS. Untuk demokrasi skala nasional, desain ini butuh L2 atau permissioned
  chain. Biaya pastinya harus diukur setelah kontraknya ada.
- **Liveness.** Kalau chain berhenti di hari pemilu, itu krisis. Kertas tetap
  jalan tanpa listrik, chain tidak. Harus ada mode degradasi.
- **Manajemen kunci.** Tiap titik TPS butuh kunci. Bocor satu berarti record
  palsu dengan tanda tangan sah. Ini program keamanan operasional, bukan fitur
  software.
- **Kebenaran data sumber.** Kalau formulir C-hasil sudah diisi salah, ledger
  menyimpan kesalahan itu secara permanen dan dengan tanda tangan yang sah.
  Garbage in, gospel out.
- **Status hukum.** Rekapitulasi manual KPU tetap menjadi acuan sampai ada dasar
  hukum yang mengakui ledger sebagai system of record.
- **Fork.** Kalau validator pecah, muncul dua sejarah. Aturan penyelesaiannya
  belum ada di mana pun.

## Yang Bukan Tujuan Proyek

- **Politik uang.** Pertahanan utamanya justru karena surat suara di bilik
  tertutup tidak bisa dibuktikan ke siapa pun. Sistem apa pun yang memberi bukti
  terverifikasi akan dipakai pembeli suara untuk menagih sebelum membayar. Voting
  yang bisa dilacak memperburuk ini, bukan memperbaikinya.
- **Voting jarak jauh atau lewat internet.** Alasan yang sama, ditambah malware
  di sisi klien dan kesenjangan digital.
- **E-voting.** Surat suara kertas dan hitung manual di TPS tetap dipertahankan.
- **Operasi tanpa kepercayaan.** Kepercayaan tidak hilang, ia berpindah ke
  penerbit kredensial, pemegang kunci, dan penulis kontrak.

## Roadmap

- [x] Perumusan masalah, threat model, non-goals
- [ ] `WitnessRegistry.sol` dan `ElectionLedger.sol` dengan cek invariant
- [ ] Multi-tanda tangan saksi (`k`-of-`n` per record)
- [ ] `Recapitulation.sol` dan agregasi berjenjang
- [ ] Test invariant: revert pada record tidak valid
- [ ] `seed.json` dari data contoh, satu provinsi
- [ ] Dashboard: tally live dengan kelengkapan pelaporan
- [ ] Halaman verifier per TPS
- [ ] Deploy ke testnet publik dan tx hash anchor di README ini

## Tech stack

Solidity `^0.8.24`, Hardhat, `viem`, frontend statis tanpa backend karena
dashboard membaca state chain langsung.

## Lisensi

MIT

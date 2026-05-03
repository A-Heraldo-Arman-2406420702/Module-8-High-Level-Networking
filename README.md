# Reflection - Module 8: Rust gRPC

**1. What are the key differences between unary, server streaming, and bi-directional streaming RPC methods, and in what scenarios would each be most suitable?**

* **Unary:** Klien mengirim satu *request* dan menunggu satu *response* dari server. Cocok untuk operasi standar seperti mengambil data pengguna, menyimpan entri ke database, atau memproses satu pembayaran.
* **Server Streaming:** Klien mengirim satu *request*, namun server merespons dengan aliran data (*stream*) yang berisi banyak pesan secara bertahap. Cocok untuk mengambil data berjumlah besar (seperti riwayat transaksi) agar tidak membebani memori sekaligus, atau untuk pembaruan *real-time* satu arah seperti *feed* harga saham.
* **Bi-directional Streaming:** Klien dan server saling mengirim aliran pesan secara independen dan asinkron melalui satu koneksi. Sangat cocok untuk aplikasi *chat*, sistem *multiplayer game*, atau analitik data *real-time* dua arah.

**2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?**

* **Data Encryption:** gRPC berjalan di atas HTTP/2, sehingga penggunaan TLS/SSL sangat diwajibkan untuk mengenkripsi transmisi data agar tidak bisa disadap (*man-in-the-middle attacks*).
* **Authentication:** Validasi identitas pengguna (misal menggunakan JWT atau *bearer token*) harus diimplementasikan dengan mengekstrak data dari `Metadata` (mirip dengan HTTP Headers) pada setiap *request* gRPC. Interceptor dapat digunakan untuk melakukan ini secara terpusat.
* **Authorization:** Setelah diautentikasi, sistem harus memastikan pengguna memiliki izin (RBAC/ABAC) untuk mengakses metode RPC tertentu.

**3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?**

* **Concurrency & State Management:** Mengelola koneksi klien yang banyak secara bersamaan (*concurrent*) dan melacak status klien (siapa yang *online*, *room* mana yang diikuti) membutuhkan struktur data yang aman untuk konkurensi (seperti `Arc<Mutex<...>>` atau *actor model*).
* **Network Instability:** Menangani koneksi yang terputus secara tiba-tiba (*drop*), implementasi *reconnection logic*, dan membersihkan *resource* yang ditinggalkan agar tidak terjadi *memory leak*.
* **Backpressure:** Jika klien atau server mengirim pesan lebih cepat daripada yang bisa diproses, hal ini bisa membebani memori. Manajemen *buffer* dan *channel* (seperti `mpsc`) harus diatur batasnya.

**4. What are the advantages and disadvantages of using `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?**

* **Advantages:** Sangat memudahkan integrasi antara arsitektur *channel* berbasis MPSC (`mpsc::channel`) milik Tokio dengan antarmuka *stream* yang dibutuhkan oleh Tonic. Memungkinkan kita untuk memisahkan *logic* pemrosesan pesan (pada *background task*) dengan pengiriman *response*.
* **Disadvantages:** Ada sedikit *overhead* dalam hal alokasi dan sinkronisasi antrean *channel*. Selain itu, jika batas memori *channel* (*bounded channel*) penuh dan tidak dikelola dengan hati-hati, proses *sender* bisa terblokir atau pesan bisa hilang.

**5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?**

* Memisahkan definisi Protobuf (`.proto`) ke dalam *crate* atau modul terpisah agar mudah dibagikan antara klien dan server.
* Menerapkan pola *Clean Architecture* atau *Hexagonal Architecture*. Lapis transport (gRPC) hanya bertugas menerima *request* dan meneruskannya ke *Domain Layer* atau *Use Case Layer* independen, sehingga *core logic* aplikasi tidak terikat langsung dengan *framework* Tonic.
* Menggunakan *Dependency Injection* melalui struktur *trait* Rust untuk menyuntikkan *database repository* atau *service* eksternal ke dalam struct *service* gRPC.

**6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?**

* **Validasi Input:** Memastikan `amount` lebih besar dari 0 dan format `user_id` valid.
* **Idempotency:** Menambahkan `idempotency_key` pada parameter *request* untuk mencegah pembayaran ganda (ter-*charge* dua kali) jika terjadi *network retry*.
* **Database & Transaksi:** Menyimpan status pembayaran ke database (seperti PostgreSQL) dalam lingkup transaksi ACID sebelum merespons sukses.
* **Integrasi Third-Party:** Memanggil API dari *payment gateway* eksternal (seperti Midtrans atau Stripe) dan menangani kegagalannya dengan implementasi `gRPC Status Error` (seperti `Status::internal` atau `Status::invalid_argument`).

**7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?**

* Memaksa sistem untuk menggunakan kontrak komunikasi yang ketat dan didefinisikan dengan jelas sejak awal melalui file `.proto`.
* Mempermudah interkoneksi antar *microservices* yang ditulis dalam berbagai bahasa pemrograman karena alat *code generation* Protobuf tersedia untuk hampir semua bahasa modern.
* **Kelemahan Interoperabilitas:** Tidak dapat langsung dipanggil oleh *browser web* (memerlukan *proxy* seperti gRPC-Web) dan lebih sulit di-*debug* secara manual menggunakan *tools* sederhana seperti `curl` dibandingkan dengan REST JSON.

**8. What are the advantages and disadvantages of HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?**

* **Advantages:** HTTP/2 mendukung *multiplexing* (mengirim banyak *request/response* secara bersamaan dalam satu koneksi TCP), kompresi *header* (HPACK) untuk efisiensi *bandwidth*, dan format biner yang lebih cepat diproses oleh mesin.
* **Disadvantages:** Lebih kompleks untuk diimplementasikan dan di-*debug* daripada komunikasi teks biasa (HTTP/1.1). WebSocket memberikan komunikasi dua arah yang sangat baik untuk *browser*, namun tidak memiliki kontrak skema dan tipe data bawaan yang sekuat gRPC.

**9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?**

* REST APIs beroperasi sepenuhnya *stateless* dan satu arah (Klien → Server → Klien). Untuk komunikasi *real-time*, REST membutuhkan teknik *long-polling* atau *polling* berkala yang boros *resource* dan memiliki latensi tinggi.
* Bi-directional streaming gRPC mempertahankan satu koneksi yang terus terbuka. Klien dan server bisa melempar data satu sama lain seketika saat ada *event* baru secara asinkron (*push-based*), sehingga jauh lebih *responsive* dan hemat *resource* jaringan.

**10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?**

* **Schema-based (Protobuf):** Menjamin keamanan tipe data (*type-safety*) saat kompilasi, ukurannya (*payload*) lebih padat/kecil (karena berbentuk biner), parsing lebih cepat, dan mendukung *backward/forward compatibility* melalui penomoran urutan *field*.
* **Schema-less (JSON):** Jauh lebih fleksibel untuk berubah (*schema evolution*) tanpa perlu kompilasi ulang, *human-readable* (mudah dibaca manusia), namun ukuran *payload*-nya lebih besar, lebih lambat di-*parsing*, dan rentan terhadap kesalahan tipe data (*runtime errors*) karena struktur datanya implisit.

# Modul 8 - High Level Networking

## Reflection

### 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?
- Unary RPC: Client mengirimkan satu request ke server dan mendapatkan satu response kembali. Cocok untuk operasi CRUD seperti mengambil data profil pengguna, mengirimkan form, memproses pembayaran, dan lainnya
- Server Streaming RPC: Client mengirimkan satu request dan server membalas dengan stream pesan yang dikirimkan secara berurutan. Client membaca dari stream ini hingga tidak ada lagi pesan. Cocok untuk kasus di mana data terlalu besar untuk dikirim sekaligus (misal riwayat transaksi yang panjang)
- Bi-directional Streaming RPC: Client dan server sama-sama membaca dan menulis stream pesan melalui koneksi yang sama secara independen. Keduanya dapat membaca dan menulis secara bersamaan. Cocok untuk komunikasi real-time dua arah seperti aplikasi chat atau game multiplayer (misal game multiplayer FPS seperti Valorant)

### 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption
- Authentication: Menerapkan sistem seperti JSON Web Tokens (JWT) atau Mutual TLS (mTLS) untuk memverifikasi identitas client. Dalam gRPC, token biasanya dimasukkan ke dalam metadata request dan divalidasi menggunakan Interceptor
- Authorization: Setelah client terautentikasi, sistem harus memastikan client tersebut memiliki hak akses (Role-Based Access Control) untuk memanggil metode RPC tertentu atau mengakses data spesifik
- Data encryption: Saluran komunikasi bisa diamankan dengan Transport Layer Security (TLS) untuk melindungi data yang sedang dikirim dari ancaman seperti packet sniffing

### 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?
- Melacak client yang terhubung dan sinkronisasi data antar client memerlukan penggunaan struktur data yang bisa dijalankan dalam concurrency seperti `Arc<RwLock<T>>` atau `Mutex`. Jika manajemen lock buruk, maka ada kemungkinan terjadi deadlock
- Client bisa saja terputus tanpa mengirim sinyal terminate sehingga server harus mendeteksi koneksi zombie untuk menghindari kebocoran memori (resource leaks)
- Jika satu client lambat menerima pesan, channel queue bisa penuh. Akibatnya, client lain bisa ikut lambat menerima pesan juga. Solusinya adalah dengan mengatur laju aliran pesan antar task-task asynchronous

### 4. What are the advantages and disadvantages of using the `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?
- Keuntungan: 
    - Memudahkan integrasi dengan ekosistem Tokio, terutama dalam mendelegasikan tugas produksi pesan ke background task terpisah melalui `mpsc::channel`
    - Memisahkan logika bisnis dari struktur respons gRPC sehingga membuat kode lebih bersih
- Kekurangan: 
    - Menambahkan lapisan abstraksi tambahan seperti saluran dan buffer
    - Harus mengalokasikan memori untuk channel capacity dan mengatur logika backpressure ketika saluran mpsc penuh (bounded channel)

### 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?
- Memisahkan definisi interface (file `.proto` dan kode hasil generate tonic) ke dalam crate atau modul tersendiri (Separation of Concerns)
- Tidak meletakkan logika utama langsung di dalam fungsi implementasi RPC. Implementasi RPC hanya bertugas untuk melakukan formatting request dan response. Operasi utama yang berisi logika bisnis dilakukan oleh modul service atau use-case
- Menggunakan Trait pada Rust untuk mendefinisikan interface ke database atau external service. Hal ini mempermudah penggantian komponen dan pengujian menggunakan mock

### 6. In the `MyPaymentService` implementation, what additional steps might be necessary to handle more complex payment processing logic?
Untuk meng-handle logika pemrosesan payment yang lebih kompleks, kita bisa:
- Melakukan validasi request seperti memeriksa ketersediaan saldo, format ID pengguna, dan batas minimum/maksimum transaksi
- Menyambungkan ke database yang persisten/tidak volatile seperti database PostgreSQL atau external service seperti Supabase. Hal ini agar data terkait transaksi bisa dicatat
- Menerapkan prinsip ACID (Atomicity, Consistency, Isolation, Durability) pada database
- Memastikan bahwa duplicate request tidak akan secara tidak sengaja tidak akan menagih pengguna dua kali
- Menangani error seperti mengembalikan kode status gRPC yang tepat (misal `INVALID_ARGUMENT`, `UNAUTHENTICATED`, dan lainnya) alih-alih `panic` atau mengembalikan response palsu.

### 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?
- Pesan dalam binary sehingga lebih kecil dari JSON, XML, atau pesan lainnya berbasis teks. Selain itu, parsing bisa dilakukan lebih cepat sehingga mengurangi latency antar service pada microservices
- Komunikasi service dengan bahasa berbeda menjadi lebih mudah karena abstraksi Protocol Buffers (protobuf) yang tidak bergantung pada suatu bahasa tertentu dan menjamin tipe data tetap akan sinkron
- Service yang menggunakan gRPC tidak dapat diakses secara langsung dari browser atau client HTTP/1 biasa tanpa alat tambahan seperti gRPC-Web atau API Gateway

### 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?
- Keuntungan:
    - Mendukung Multiplexing yang memungkinkan banyak request dan response dikirim secara bersamaan melalui satu koneksi TCP tanpa blocking
    - Mendukung header compression (`HPACK`) yang menghemat bandwidth
    - Mendukung server push yang memungkinkan server mengirimkan data atau aset ke client sebelum client meminta (melakukan request) terhadap data tersebut
- Kekurangan:
    - Lebih sulit di-debug secara manual menggunakan alat seperti `curl` atau network browser panel karena protokolnya berupa binary bukan teks biasa
    - Load balancing HTTP/2 lebih kompleks dan sering kali membutuhkan mekanisme pada Layer 7 (Application layer).

### 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?
REST (HTTP/1.1) bersifat stateless dan client-based. Untuk mendapatkan data real-time, client harus melakukan polling (bertanya berulang kali ke server) atau long-polling yang boros koneksi, CPU, dan menciptakan latency (jeda) sehingga kurang responsif dibanding gRPC yang menggunakan HTTP/2.

gRPC Bidirectional mempertahankan koneksi yang persisten dan responsif. Client dan server bereaksi secara asynchronous. Server dapat melakukan push update ke client begitu ada perubahan data seperti pesan baru masuk sehingga aplikasi terasa instan. Sumber daya yang digunakan juga lebih efisien dibandingkan membuat request HTTP baru terus-menerus seperti pada HTTP/1.1.

### 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?
- Pada gRPC (Protobuf), skema ditentukan secara ketat di file `.proto` sebelum kode ditulis. Akibatnya:
    - Type safety dijamin di tingkat kontrak karena dipastikan oleh skema sehingga meminimalisir bug saat runtime akibat tipe data yang tidak sesuai
    - Data dikirim menggunakan binary oleh kedua pihak sehingga lebih kecil dan cepat dibanding data dalam text
    - Skema harus dikompilasi terlebih dahulu untuk bahasa pemrograman tertentu. Ini menambah langkah saat development, tetapi integritas data menjadi lebih baik
- JSON bersifat fleksibel dan tidak memerlukan skema formal agar dapat berfungsi. Akibatnya:
    - Struktur data bisa diubah kapan saja tanpa perlu melakukan kompilasi ulang pada interface. Ini memudahkan pada tahap awal pengembangan (prototyping) dimana struktur data sering berubah
    - Mudah dibaca oleh manusia tanpa alat tambahan sehingga memudahkan proses debugging secara langsung melalui browser atau alat seperti `curl`
    - Setiap pesan JSON harus membawa nama field (key) setiap saat. Akibatnya, proses parsing teks menjadi objek di memori memakan lebih banyak sumber daya CPU dibandingkan format binary pada gRPC
    - Kesalahan struktur data sering kali baru terdeteksi saat aplikasi sedang berjalan (runtime) karena tidak ada skema yang ketat. Developer harus menulis kode validasi tambahan untuk memastikan data yang diterima benar
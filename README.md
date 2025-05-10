# REFLECTION

1. **Perbedaan Mendasar Antara Unary, Server Streaming, dan Bidirectional Streaming RPC**  
   - **Unary RPC** adalah pola komunikasi paling sederhana dimana client mengirim satu request dan menerima satu response secara synchronous, mirip dengan fungsi tradisional. Contoh penggunaan: saat mengambil data profil pengguna dari database dimana Anda butuh respon segera dan lengkap.  
   - **Server Streaming RPC** memungkinkan server mengirim sequence of messages sebagai respon dari single request client. Contoh nyata: sistem monitoring real-time dimana server terus mengirim update metrik performa setiap 5 detik melalui stream yang sama.  
   - **Bidirectional Streaming RPC** membuka kanal komunikasi dua arah yang sepenuhnya asynchronous, dimana client dan server bisa saling mengirim message secara independen. Contoh aplikasi: sistem chat dimana kedua pihak bisa mengirim pesan kapan saja tanpa perlu request-response cycle.  

2. **Pertimbangan Keamanan Utama untuk gRPC di Rust**  
   - **Autentikasi**: Implementasikan mutual TLS (mTLS) untuk verifikasi kedua sisi, atau gunakan JWT dengan signature verification di interceptor. Contoh di Rust:  
     ```rust
     // Contoh interceptor untuk validasi JWT
     async fn check_auth(req: Request<()>) -> Result<Request<()>, Status> {
         let token = req.metadata().get("auth-token").ok_or(Status::unauthenticated("no token"))?;
         verify_jwt(token.to_str()?).await?;
         Ok(req)
     }
     ```  
   - **Enkripsi**: Selalu gunakan TLS 1.3 untuk transport encryption, dan pertimbangkan field-level encryption untuk data sensitif seperti nomor kartu kredit.  
   - **Hardening**: Tambahkan rate limiting (misal menggunakan governor crate) untuk prevent DDoS, dan lakukan exhaustive validation pada semua protobuf message.  

3. **Challenge Kompleks Bidirectional Streaming**  
   - **Flow Control**: Gunakan teknik seperti sliding window protocol untuk menghindari satu pihak membanjiri channel. Contoh: implementasikan backpressure dengan tokio's semaphore.  
   - **Connection Resiliency**: Tambahkan heartbeat mechanism (ping/pong setiap 30 detik) untuk mendeteksi dead connection secara proactive.  
   - **State Management**: Pada aplikasi chat, gunakan Arc<Mutex<HashMap<UserId, Sender>>> untuk manage connected clients secara thread-safe.  

4. **ReceiverStream: Analisis Mendalam**  
   Keunggulan utama adalah kemudahan integrasi dengan sistem channel existing:  
   ```rust
   let (tx, rx) = mpsc::channel(32);
   let receiver_stream = ReceiverStream::new(rx);
   ```  
   Namun memiliki beberapa limitation:  
   - Tidak ada built-in error propagation - harus handle error manual via separate channel  
   - Overhead pembuatan task tambahan untuk polling stream  
   - Buffer size harus di-tune manual berdasarkan use case  

5. **Arsitektur Modular untuk Codebase gRPC**  
   Struktur direkomendasikan:  
   ```
   /src
   ├── proto/           # File .proto dan generated code
   ├── services/        # Implementasi handler gRPC 
   │   ├── auth/        # Terpisah per domain bisnis
   │   └── payments/
   ├── core/            # Business logic murni
   ├── adapters/        # Integrasi eksternal (DB, dll)
   └── middleware/      # Interceptors shared
   ```  
   Gunakan trait untuk abstract core logic:  
   ```rust
   pub trait PaymentProcessor {
       async fn process(&self, amount: f64) -> Result<Receipt>;
   }
   ```

6. **Ekstensi Payment Service**  
   Untuk production-grade system:  
   - Tambahkan idempotency key untuk handle duplicate request  
   - Implementasikan saga pattern untuk transaksi distributed  
   - Gunakan circuit breaker (misal menggunakan tower) saat call payment gateway  
   - Audit trail dengan immutable event logging  

7. **Dampak Arsitektural gRPC**  
   Keunggulan performa (throughput 5-10x REST) harus diimbangi dengan:  
   - Kebutuhan code generator (prost/tonic-build)  
   - Operasional yang lebih kompleks (require HTTP/2 LB)  
   - Interoperabilitas dengan web via grpc-web proxy  

8. **HTTP/2 vs HTTP/1.1: Perbandingan Teknis**  
   - **Header Compression**: HTTP/2 menggunakan HPACK binary compression vs plaintext di HTTP/1.1  
   - **Multiplexing**: 100+ concurrent streams dalam satu TCP connection vs 6 parallel connections di HTTP/1.1  
   - **Prioritization**: Bisa set priority untuk critical resources  

9. **Real-World Performance Benchmark**  
   Pada jaringan latency 50ms:  
   - REST API membutuhkan 3 RTT (TCP handshake + TLS + request) = ~150ms minimum  
   - gRPC dengan connection reuse hanya 1 RTT = 50ms untuk request pertama, 0ms untuk subsequent calls  

10. **Protobuf vs JSON: Tradeoff Nyata**  
    **Protobuf**:  
    - Ukuran payload 70-80% lebih kecil  
    - Serialization speed 5x lebih cepat  
    - Strict schema mencegah bugs schema drift  
    **JSON**:  
    - Human-readable untuk debugging  
    - Tidak perlu pre-compile step  
    - Native support di browser dan tools seperti curl  

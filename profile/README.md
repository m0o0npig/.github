# Góc nhìn **rõ ràng – thực tế – theo đúng kiến trúc backend modern** để hiểu các bottleneck khi cố đẩy **1 server** lên mức **~1.000.000 request/second** 👋

<!--

**Here are some ideas to get you started:**

🙋‍♀️ A short introduction - what is your organization all about?
🌈 Contribution guidelines - how can the community get involved?
👩‍💻 Useful resources - where can the community find your docs? Is there anything else the community should know?
🍿 Fun facts - what does your team eat for breakfast?
🧙 Remember, you can do mighty things with the power of [Markdown](https://docs.github.com/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
-->


---

# 1) Các bottleneck chính trong 1 server web-scale

## **1. Network throughput (băng thông mạng)**

* Dữ liệu trả về (HTML, JSON, pre-rendered content, images nhỏ…) nhân với lượng req/s sẽ nhanh chóng đẩy server đến giới hạn NIC.
* Lưu lượng mạng phải tính luôn:

  * Response payload (thường là 1–20 KB/tùy trang)
  * HTTP headers
  * TLS overhead
* Đây thường là **bottleneck số 1** cho workload kiểu “serve content”.

---

## **2. Packet Per Second (PPS) / Kernel Networking**

* PPS còn quan trọng hơn băng thông.
* 1M req/s nghĩa là hàng triệu packets/s → kernel network stack có thể không xử lý kịp:

  * Interrupt handling
  * Context switching
  * Socket operations
  * Accept() / epoll() / io_uring
* PPS limit thường choke **trước** khi bạn đạt 100 Gbps.

---

## **3. TLS compute cost**

Ngay cả với CPU mạnh + AES-NI:

* TLS handshake → cực nặng.
* TLS session resumption giảm tải nhưng vẫn có overhead.
* Với request ngắn (tiny request), **TLS trở thành bottleneck trầm trọng**.

Nhiều hệ thống high-RPS phải:

* Terminate TLS ở reverse proxy (nginx/envoy)
* Hoặc dùng **Cloudflare / CDN** để offload TLS hoàn toàn.

---

## **4. Memory bandwidth + CPU cache**

Nếu server phải:

* Render page
* Làm logic app
* Chạm vào DB
* Deserialize JSON
  → RAM + L3 cache coherence trở thành giới hạn.

Nhưng với hệ thống đọc bài kiểu Medium/Substack:

* Trang đã được cache sẵn
  → CPU/RAM không phải bottleneck lớn nhất.

---

## **5. Disk I/O (không phải bottleneck chính nếu kiến trúc đúng)**

* Static content → nên nằm ở ramdisk hoặc CDN
* DB → nên không đặt trong cùng server
* Medium/Substack load theo hướng **cache-first**, không phải truy vấn DB mỗi request.

Nếu còn phải đụng DB trên mỗi request → Không server nào đạt 1M req/s.

---

## **6. Application framework overhead**

Thứ tự hiệu năng:

```
C / Rust server (nginx, envoy, h2o) > Go / Elixir / Java
>> Node.js / Python
```

Một web framework chậm có thể giảm throughput **10–50 lần**.

Để đạt 1M req/s, application layer gần như luôn cần:

* Viết với Go, Rust, Elixir hoặc C-level optimization
* Hoặc bypass bằng cách dùng reverse proxy + static content cache

---

## **7. Lock contention / thread scheduling**

High parallelism có thể gây deadlock hoặc lock storm:

* Python GIL
* Shared memory locks
* DB connection pool
* Thread pool bottleneck

1M req/s đòi hỏi kiến trúc **lock-free hoặc low-lock**.

---

## **8. Syscall overhead**

Mỗi request gây syscall:

* read()
* write()
* sendfile()
* epoll_wait()

Khi lên hàng triệu ops/s:
→ syscall overhead cực lớn
→ phải dùng **io_uring** hoặc **sendfile** để giảm syscall.

---

# 2) Vậy bottleneck chính khi muốn đạt **1M req/s** là gì?

### **→ Network (Bandwidth + PPS) là bottleneck số 1.**

Để đạt 1M req/s thực tế, workload phải:

* Response cực nhỏ (dưới 1 KB)
* Static cached content
* TLS offloaded hoặc session reuse 100%
* Kernel networking tuned (IRQ pinning, RPS/RFS, zero-copy)
* NIC 25/50/100 Gbps

**Không phải CPU, không phải disk.
Chính là network + kernel networking.**

---

# 3) Kiến trúc để 1 server đạt 1M req/s

### **A. Offload tối đa:**

* CDN → cache toàn bộ bài viết
* Edge → xử lý TLS + compression
* Server chỉ nhận request cache miss (thường rất ít)

=> Medium/Substack làm đúng mô hình này.

**Một server thật sự không cần handle 1M req/s**
→ hệ thống CDN edge handle gần hết.

---

### **B. Nếu buộc “1 server thực sự xử lý 1M req/s” thì:**

1. NIC 100 Gbps

2. Kernel tuned:

   * irqbalance off
   * pin IRQ vào dedicated cores
   * TCP pacing
   * Increase backlog, somaxconn
   * RSS, RFS, XDP if possible

3. Web server zero-copy:

   * nginx with sendfile + reuseport
   * hoặc h2o (C++)

4. App không động đến DB

5. Static content + cache in RAM

6. TLS offload

7. Response nhỏ (0.5–2 KB)

---

# **Tóm tắt nhanh**

| Bottleneck                | Quan trọng | Ghi chú                                  |
| ------------------------- | ---------- | ---------------------------------------- |
| **Network bandwidth**     | ★★★★★      | Bị choke đầu tiên                        |
| **Packet Per Second**     | ★★★★★      | Thường giới hạn trước cả bandwidth       |
| **TLS overhead**          | ★★★★☆      | Handshake = tử thần                      |
| **Kernel networking**     | ★★★★☆      | syscalls, context switching              |
| **CPU**                   | ★★★☆☆      | Không phải bottleneck nếu static content |
| **Disk I/O**              | ★★☆☆☆      | Gần như không phải vấn đề                |
| **App logic / framework** | ★★★☆☆      | Quan trọng nếu dynamic                   |

---

Nếu muốn, tôi có thể viết thêm:

✅ Benchmark thực tế về nginx, envoy, h2o để đạt 1M req/s
✅ Kiến trúc tối ưu kernel cho 1M req/s
✅ Kiến trúc giống Medium/Substack (heavy CDN, light origin)

Bạn muốn hướng nào?


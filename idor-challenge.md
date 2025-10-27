# Hack The Boo CTF - Web Challenge Write-up

## Challenge Information

**Challenge:** The Gate of Broken Names  
**Category:** Web Security  
**Difficulty:** Easy  
**Description:** 
> Among the ruins of Briarfold, Mira uncovers a gate of tangled brambles and forgotten sigils. Every name carved into its stone has been reversed, letters twisted, meanings erased. When she steps through, the ground blurs—the village ahead is hers, yet wrong: signs rewritten, faces familiar but altered, her own past twisted. Tracing the pattern through spectral threads of lies and illusion, she forces the true gate open—not by key, but by unraveling the false paths the Hollow King left behind.

---

## Initial Reconnaissance

### 1. Truy cập challenge

Challenge cung cấp Docker spawned và source code. Khi khởi động, nhận được địa chỉ IP và port để truy cập:

![Start](./idor_images/Start.png)

Giao diện ban đầu là trang đăng nhập với 2 chức năng chính: **Login** và **Create Account**.

![Homepage](./idor_images/Homepage.png)

### 2. Thu thập thông tin ban đầu

- View page source không có thông tin hữu ích
- Sử dụng `gobuster` để quét thư mục/endpoints:

```bash
gobuster dir -u http://209.38.234.15:31546 -w directory-list-lowercase-2.3-medium.txt -x txt,log,html,js,php,py
```

![Gobuster](./idor_images/gobuster.png)

Không phát hiện gì đặc biệt. Có vẻ cần tạo tài khoản và đăng nhập để khám phá thêm.

---

## Post-Authentication Analysis

### 1. Đăng nhập và khám phá Dashboard

Sau khi tạo tài khoản và đăng nhập, tôi được đưa đến trang **Dashboard**. Ứng dụng cho phép người dùng quản lý các ghi chú (gọi là "Chronicles").

Tại đây có danh sách **"Public Chronicles"**.

![Page1](./idor_images/page1.png)

### 2. Phân tích chức năng

**URL Public Notes:**
```
http://209.38.234.15:31546/public-notes?page=1
```

- Thử SQL Injection ở tham số `page` với các payload đơn giản (`'`, `1=1`, v.v.) → không có phản ứng

**Tạo Chronicle:**
- Form gồm "Title" và "Content"
- Thử payload XSS cơ bản vào cả hai trường → đã bị filter

---

## IDOR Vulnerability Discovery

### 1. Phát hiện điểm khai thác

Sau khi tạo note thành công, trình duyệt chuyển hướng đến URL:
```
http://209.38.234.15:31546/note?id=211
```

![211](./idor_images/211.png)

Tham số `id` trong URL → nghi ngờ **IDOR vulnerability**.

### 2. Kiểm tra lỗ hổng

**Thử với `id=1`:**
```
http://209.38.234.15:31546/note?id=1
```

![1](./idor_images/1.png)

→ Trả về public note (không có gì đặc biệt)

**Thử với `id=2`:**
```
http://209.38.234.15:31546/note?id=2
```

![2](./idor_images/2.png)
→ **Thành công!** Lấy được private note của user khác mà lẽ ra không có quyền xem.

**Kết luận:** Bất kỳ user nào đã đăng nhập đều có thể đọc tất cả note trong hệ thống, chỉ cần đoán đúng `id`.

---

## Exploitation

### Khai thác với Burp Suite Intruder

Vì note của tôi có `id=211`, nên có ít nhất 211 note trong database. Flag có thể nằm trong các private note từ `id=1` đến `id=210`.

**Các bước:**

1. **Bắt request** `GET /api/notes/211` trong Burp Suite, gửi sang tab **Intruder**

> **Chú ý:** Phải bắt `GET /api/notes/211`, không được bắt gọi `GET /note?id=211`, vì `/api/notes/211` mới là gọi hiển thị note.

![intercept](./idor_images/intercept.png)

2. **Tab Positions:**
   - Clear tất cả vị trí
   - Chọn số `211` trong `/api/notes/211` và nhấn "Add §"
   - Request trông như sau:

```http
GET /api/notes/§211§ HTTP/1.1
Host: 209.38.234.15:31546
... (các header khác)
```

3. **Tab Payloads:**
   - Payload type: **Numbers**
   - From: **1**
   - To: **211**
   - Step: **1**

4. Nhấn **Start Attack** và đợi quá trình khai thác hoàn thành.

![intruder](./idor_images/intruder.png)

### Phát hiện Flag

Kiểm tra toàn bộ 211 gói tin vừa được bắt. Sử dụng mũi tên di chuyển lên xuống cho nhanh, sẽ phát hiện flag nằm ở 1 trong 211 gói tin này. Flag sẽ nằm random trong 1 gói nào đó - điều này mỗi IP sẽ khác nhau, vị trí flag sẽ không cố định.

![flag](./idor_images/flag.png)
---

## Solution

**Flag:**
```
HTB{br0k3n_n4m3s_r3v3rs3d_4nd_r3st0r3d_<hashcode>}
```

Phần hashcode mỗi người khi dùng máy có IP khác nhau sẽ cho ra 1 hash code khác nhau. Ví dụ của tôi là:

```
HTB{br0k3n_n4m3s_r3v3rs3d_4nd_r3st0r3d_3c9593bcde968834f7cab3777de789da}
```

Đây là một cách hay của HTB để chống copy flag, đảm bảo tính công bằng cho người chơi.

---

## Key Takeaways

### Bài học rút ra:

1. **Luôn kiểm tra tham số `id` trên URL** - đặc biệt sau khi tạo tài nguyên mới
2. **IDOR xảy ra khi** server cho phép truy cập tài nguyên chỉ dựa vào ID mà không kiểm tra quyền
3. **Cách phòng chống:** Server phải xác thực user hiện tại có quyền truy cập tài nguyên hay không

### Tools & References:

- [Burp Suite Documentation](https://portswigger.net/burp/documentation)
- [OWASP: IDOR](https://owasp.org/www-community/vulnerabilities/Insecure_Direct_Object_Reference)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)

---

**Duration:** 30 phút  
**Date:** 25/10/2024
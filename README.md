# Báo Cáo Dự Án Cuối Kỳ (Nâng Cao): Triển Khai và Đánh Giá Toàn Diện Bảo Mật Web Server với FortiWeb
**Môn học:** An Ninh Mạng

## Mục Tiêu

Báo cáo này trình bày chi tiết quá trình triển khai và đánh giá hiệu quả của Tường lửa Ứng dụng Web (WAF) FortiWeb 100D trong việc bảo vệ một máy chủ Web mục tiêu chạy ứng dụng DVWA. Kịch bản nâng cao này bao gồm việc cấu hình bảo vệ cơ bản, mô phỏng và phòng chống một loạt các cuộc tấn công phổ biến và nghiêm trọng (SQL Injection, Cross-Site Scripting - XSS, Command Injection, File Inclusion, File Upload, Denial of Service - DoS). Đồng thời, kịch bản cũng bao gồm việc sử dụng tính năng quét lỗ hổng và các công cụ giám sát, báo cáo của FortiWeb để đánh giá toàn diện tình trạng an ninh của ứng dụng web được bảo vệ.

## Môi Trường Thực Hiện

* **Thiết bị WAF:** FortiWeb 100D (Giả định đã được cấu hình IP cơ bản và có thể truy cập giao diện quản lý).
* **Máy Client/Attacker:** Máy tính PC cài đặt trình duyệt web, XAMPP (để sử dụng ApacheBench `ab`), và có khả năng tạo/chỉnh sửa file text đơn giản.
* **Máy chủ Web:** Máy chủ cài đặt DVWA (IP: `172.16.4.241`).
* **Mô hình mạng:** Theo mô hình trong tài liệu lab, FortiWeb hoạt động ở chế độ Reverse Proxy.
* **Địa chỉ truy cập:**
    * Giao diện quản lý FortiWeb: `https://192.168.1.xxx` (thay xxx bằng IP thực tế).
    * Ứng dụng Web DVWA (qua FortiWeb): `http://dvwa.hcmute.com` (Giả định DNS đã được cấu hình để phân giải tên miền này về Virtual IP của FortiWeb trên `port2`, ví dụ `10.1.1.252`).

## Các Bước Thực Hiện Chi Tiết

### Giai đoạn 1: Cấu hình Cơ bản và Làm quen Giao diện

1.  **Truy cập và Khám phá Giao diện:**
    * Mở trình duyệt trên máy PC, truy cập `https://192.168.1.xxx`.
    * Đăng nhập bằng tài khoản quản trị (vd: `admin`, không mật khẩu hoặc mật khẩu mặc định nếu có).
    * Dành thời gian điều hướng qua các menu chính để làm quen: `System`, `FortiView`, `User`, `Policy`, `Server Objects`, `Application Delivery`, `Web Protection`, `Bot Mitigation`, `API Protection`, `DoS Protection`, `Log & Report`, `Monitor`.

2.  **Cấu hình Server Pool (Real Server):**
    * Điều hướng đến `Server Objects > Server > Server Pool`.
    * Nhấn `Create New`.
    * Cấu hình các thông số:
        * `Name`: `DVWA`
        * `Type`: `Reverse Proxy`
        * `Single Server/Server Balance`: `Single Server`
    * Trong mục `Server Pool Member`, nhấn `Create New`.
    * Cấu hình Real Server:
        * `Status`: `Enable`
        * `Server Type`: `IP`
        * `Domain`: `172.16.4.241` (IP của máy chủ DVWA)
        * `Port`: `80`
    * Nhấn `OK` để lưu Real Server.
    * Nhấn `OK` để lưu Server Pool.

3.  **Cấu hình Virtual Server (VIP):**
    * Điều hướng đến `Server Objects > Server > Virtual Server`.
    * Nhấn `Create New`.
    * Đặt `Name`: `DVWA-VIP`.
    * Nhấn `OK`.
    * Trong danh sách Virtual Server Item của `DVWA-VIP`, nhấn `Create New`.
    * Cấu hình:
        * Check vào ô `Use Interface IP`.
        * `Interface`: Chọn `port2` (cổng kết nối mạng chứa Web Server hoặc cổng mà người dùng sẽ truy cập vào).
    * Nhấn `OK`.

4.  **Cấu hình X-Forwarded-For (XFF):**
    * **Mục đích:** Khi FortiWeb hoạt động ở chế độ Reverse Proxy, IP nguồn mà Web Server nhìn thấy là IP của FortiWeb. XFF giúp chèn IP gốc của Client vào HTTP Header để Web Server có thể ghi nhận.
    * Điều hướng đến `Server Objects > X-Forwarded-For`.
    * Nhấn `Create New`.
    * Cấu hình Profile XFF:
        * `Name`: `XFF-Rule`
        * Bật (check) các tùy chọn: `Add X-Forwarded-For`, `Add X-Forwarded-Proto`, `Add X-Real-IP` (tùy chọn theo yêu cầu).
    * Trong mục `Trusted X-Header Sources`, nhấn `Create New`:
        * Nhập IP của Virtual Server (`10.1.1.252`). Nhấn `OK`.
    * Nhấn `Create New` lần nữa:
        * Nhập IP của Real Server (`172.16.4.241`). Nhấn `OK`.
    * Nhấn `OK` để lưu Profile XFF.

5.  **Kiểm tra Truy cập Ban đầu:**
    * Từ trình duyệt máy PC, truy cập `http://dvwa.hcmute.com`.
    * **Kết quả mong đợi:** Trang đăng nhập của DVWA hiển thị thành công.

### Giai đoạn 2: Tấn công và Phòng chống SQL Injection

1.  **Mô phỏng Tấn công SQLi (Chưa bảo vệ):**
    * Truy cập `http://dvwa.hcmute.com`, đăng nhập vào DVWA (vd: `admin`/`password`).
    * Trong menu bên trái, chọn `DVWA Security`, đặt mức `Security Level` thành `low`. Nhấn `Submit`.
    * Chọn mục `SQL Injection`.
    * Trong ô `User ID`, nhập payload: `' or '1'='1' union select version(), database()#`
    * Nhấn `Submit`.
    * **Kết quả mong đợi:** Thông tin về phiên bản MySQL và tên database (`dvwa`) được hiển thị. Cuộc tấn công thành công.

2.  **Cấu hình Phòng chống SQLi:**
    * **Tạo Signature Policy:**
        * Điều hướng đến `Web Protection > Known Attacks > Signatures`.
        * Nhấn `Create New`.
        * `Name`: `SQLi-Policy`
        * Trong danh sách `Main Class Name`:
            * Tìm đến `SQL Injection`, `SQL Injection (Extended)`, `Generic Attacks`, `Generic Attacks (Extended)`.
            * Đảm bảo cột `Status` được bật (Enable).
            * Đặt `Action` thành `Alert & Deny`.
            * Các signature khác có thể để `Disable` hoặc `Alert`.
        * Nhấn `OK`.
    * **Tạo Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Nhấn `Create New`.
        * `Name`: `Base-Protection-Profile`
        * Trong phần `Inline Standard Protection`:
            * `Signatures`: Chọn `SQLi-Policy` vừa tạo.
        * Trong phần `HTTP Header Security`:
            * `X-Forwarded-For`: Chọn `XFF-Rule` đã tạo.
        * Nhấn `OK`.
    * **Tạo Server Policy:**
        * Điều hướng đến `Policy > Server Policy`.
        * Nhấn `Create New`.
        * `Name`: `Policy_Protect_DVWA`
        * `Deployment Mode`: `Reverse Proxy` (thường mặc định)
        * `Virtual Server`: Chọn `DVWA-VIP`.
        * `Server Pool`: Chọn `DVWA`.
        * `HTTP Service`: Chọn `HTTP` (hoặc tạo mới nếu chưa có).
        * `Web Protection Profile`: Chọn `Base-Protection-Profile`.
        * Nhấn `OK`.

3.  **Kiểm tra sau khi Bảo vệ:**
    * Lặp lại bước mô phỏng tấn công SQLi (Bước 1 của Giai đoạn 2).
    * **Kết quả mong đợi:** FortiWeb chặn yêu cầu, trình duyệt hiển thị trang "Web Page Blocked!".
    * **Kiểm tra Log:**
        * Điều hướng đến `Log & Report > Log Access > Attack`.
        * Tìm log mới nhất. Quan sát thông tin: `Date/Time`, `Policy` (`Policy_Protect_DVWA`), `Source` (IP máy PC), `Destination` (IP Real Server), `Main Type` (`SQL Injection`), `Action` (`Deny`), `Signature` (ID của signature đã kích hoạt).

### Giai đoạn 3: Tấn công và Phòng chống XSS

1.  **Mô phỏng Tấn công XSS (Đã có bảo vệ SQLi, chưa có XSS):**
    * **Reflected XSS:**
        * Trong DVWA (vẫn ở mức `low`), chọn mục `XSS (Reflected)`.
        * Trong ô `What's your name?`, nhập payload: `<script>alert('XSS Reflected Test')</script>`
        * Nhấn `Submit`.
        * **Kết quả mong đợi:** Hộp thoại alert với nội dung "XSS Reflected Test" xuất hiện. Tấn công thành công.
    * **Stored XSS:**
        * Chọn mục `XSS (Stored)`.
        * Trong ô `Name`, nhập `TestName`.
        * Trong ô `Message`, nhập payload: `<script>alert(document.cookie)</script>`
        * Nhấn `Sign Guestbook`.
        * Tải lại trang hoặc điều hướng đi rồi quay lại trang `XSS (Stored)`.
        * **Kết quả mong đợi:** Hộp thoại alert chứa thông tin cookie của người dùng (bao gồm cả session ID) xuất hiện. Tấn công thành công.

2.  **Cấu hình Phòng chống XSS:**
    * **Tạo Signature Policy:**
        * Điều hướng đến `Web Protection > Known Attacks > Signatures`.
        * Nhấn `Create New`.
        * `Name`: `XSS-Policy`
        * Trong danh sách `Main Class Name`:
            * Tìm đến `Cross Site Scripting`, `Cross Site Scripting (Extended)`.
            * Đảm bảo `Status` là `Enable` và `Action` là `Alert & Deny`.
            * Các signature khác đặt là `Disable`.
        * Nhấn `OK`.
    * **Cập nhật Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Trong phần `Inline Standard Protection`:
            * `Signatures`: **Thay đổi** từ `SQLi-Policy` thành `XSS-Policy`. *(Lưu ý: Để bảo vệ cả hai, bạn cần tạo một Signature Policy gộp cả SQLi và XSS, hoặc sử dụng các tính năng Profile nâng cao nếu có. Trong kịch bản này, chúng ta tạm thời chỉ bật XSS để kiểm tra theo cấu trúc lab).*
        * Nhấn `OK`.

3.  **Kiểm tra sau khi Bảo vệ:**
    * Lặp lại cả hai bước mô phỏng tấn công Reflected và Stored XSS (Bước 1 của Giai đoạn 3).
    * **Kết quả mong đợi:** FortiWeb chặn các yêu cầu chứa payload XSS, hiển thị trang "Web Page Blocked!".
    * **Kiểm tra Log:**
        * Điều hướng đến `Log & Report > Log Access > Attack`.
        * Tìm các log mới nhất liên quan đến XSS bị chặn.

### Giai đoạn 4: Tấn công và Phòng chống Command Injection (Mới)

1.  **Mô phỏng Tấn công Command Injection (Chưa bảo vệ Command Inj):**
    * Trong DVWA (level `low`), chọn mục `Command Injection`.
    * Trong ô IP address, nhập payload: `127.0.0.1 && ls -la` (để liệt kê file thư mục hiện tại) hoặc `127.0.0.1; cat /etc/passwd` (để đọc file passwd).
    * Nhấn `Submit`.
    * **Kết quả mong đợi:** Output của lệnh `ls -la` hoặc nội dung file `/etc/passwd` được hiển thị. Tấn công thành công.
2.  **Cấu hình Phòng chống Command Injection (Cập nhật Profile):**
    * **Kiểm tra/Tạo Signature Policy Gộp:**
        * Đảm bảo Signature Policy đang áp dụng cho `Base-Protection-Profile` (ví dụ: `Combined-SQLi-XSS-Policy`) đã bao gồm các signature cho `OS Commanding` hoặc `Command Injection`. Nếu chưa, hãy tạo một policy gộp mới (`Combined-All-Policy`) bao gồm cả SQLi, XSS, và Command Injection signatures (Action `Alert & Deny`).
    * **Cập nhật Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Trong phần `Inline Standard Protection > Signatures`: Chọn policy gộp (`Combined-All-Policy` hoặc tên tương ứng).
        * Nhấn `OK`.
3.  **Kiểm tra sau khi Bảo vệ:** Thử lại tấn công Command Injection với các payload ở Bước 1.
    * **Kết quả mong đợi:** Bị chặn bởi FortiWeb. Kiểm tra log Attack, tìm các log liên quan đến `OS Commanding` hoặc `Command Injection`.

### Giai đoạn 5: Tấn công và Phòng chống File Inclusion (Mới)

1.  **Mô phỏng Tấn công File Inclusion (Chưa bảo vệ File Incl):**
    * Trong DVWA (level `low`), chọn mục `File Inclusion`.
    * URL ban đầu có thể là `http://dvwa.hcmute.com/vulnerabilities/fi/?page=include.php`.
    * Thay đổi tham số `page` trong URL thành một payload LFI, ví dụ: `http://dvwa.hcmute.com/vulnerabilities/fi/?page=../../../../etc/passwd`
    * Truy cập URL đã sửa đổi.
    * **Kết quả mong đợi:** Nội dung file `/etc/passwd` của server được hiển thị trên trang web. Tấn công LFI thành công.
2.  **Cấu hình Phòng chống File Inclusion (Cập nhật Profile):**
    * **Kiểm tra/Tạo Signature Policy Gộp:**
        * Đảm bảo Signature Policy đang áp dụng (`Combined-All-Policy`) đã bao gồm các signature cho `Directory Traversal`, `File Inclusion`, `Path Traversal` (Action `Alert & Deny`). Nếu chưa, cập nhật policy gộp này.
    * **Cập nhật Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Đảm bảo `Inline Standard Protection > Signatures` đang sử dụng policy gộp (`Combined-All-Policy`) đã cập nhật.
        * Nhấn `OK`.
3.  **Kiểm tra sau khi Bảo vệ:** Thử lại tấn công File Inclusion bằng cách truy cập URL với payload LFI.
    * **Kết quả mong đợi:** Bị chặn bởi FortiWeb. Kiểm tra log Attack, tìm các log liên quan đến `Directory Traversal` hoặc `File Inclusion`.

### Giai đoạn 6: Tấn công và Phòng chống File Upload (Mới)

1.  **Mô phỏng Tấn công File Upload (Chưa bảo vệ File Upload):**
    * **Chuẩn bị file shell:** Tạo một file text tên `shell.php` với nội dung đơn giản sau:
        ```php
        <?php echo "<pre>"; system($_GET['cmd']); echo "</pre>"; ?>
        ```
    * **Upload file:**
        * Trong DVWA (level `low`), chọn mục `File Upload`.
        * Nhấn `Browse...`, chọn file `shell.php` vừa tạo.
        * Nhấn `Upload`.
        * **Kết quả mong đợi ban đầu:** File được upload thành công, trang web thường hiển thị đường dẫn tới file đã upload (ví dụ: `../../hackable/uploads/shell.php`).
    * **Thực thi shell:**
        * Truy cập đường dẫn tới file shell qua trình duyệt, thêm tham số `cmd` để thực thi lệnh, ví dụ: `http://dvwa.hcmute.com/hackable/uploads/shell.php?cmd=ls -la` hoặc `http://dvwa.hcmute.com/hackable/uploads/shell.php?cmd=id`
        * **Kết quả mong đợi:** Output của lệnh (danh sách file hoặc user id) được hiển thị trên trang. Tấn công thành công, đã chiếm được quyền thực thi lệnh trên server.
2.  **Cấu hình Phòng chống File Upload (Cập nhật Profile):**
    * **Kiểm tra/Tạo Signature Policy Gộp:**
        * Đảm bảo Signature Policy đang áp dụng (`Combined-All-Policy`) đã bao gồm các signature liên quan đến `Malicious File Upload`, `Web Shell` (Action `Alert & Deny`). Nếu chưa, cập nhật policy gộp này.
    * **(Tùy chọn) Cấu hình File Upload Restriction:**
        * Khám phá mục `Input Validation > File Upload Restriction` trong FortiWeb (nếu có và muốn tìm hiểu sâu hơn). Có thể tạo rule để giới hạn loại file (vd: chỉ cho phép ảnh), kích thước file.
    * **Cập nhật Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Đảm bảo `Inline Standard Protection > Signatures` đang sử dụng policy gộp (`Combined-All-Policy`) đã cập nhật.
        * Nhấn `OK`.
3.  **Kiểm tra sau khi Bảo vệ:**
    * **Thử Upload lại:** Lặp lại bước upload file `shell.php`.
    * **Kết quả mong đợi:** Việc upload có thể bị chặn ngay bởi FortiWeb (nếu signature phát hiện), hoặc upload thành công nhưng khi truy cập để thực thi shell (bước thực thi shell) sẽ bị chặn.
    * **Kiểm tra Log:** Kiểm tra log Attack, tìm các log liên quan đến `Malicious File Upload`, `Web Shell` hoặc các signature tương ứng.

### Giai đoạn 7: Tấn công và Phòng chống DoS

1.  **Mô phỏng Tấn công DoS (Chưa bảo vệ DoS):**
    * **Chuẩn bị:** Mở `XAMPP Control Panel`, nhấn nút `Shell`.
    * **Thực hiện:** Chạy lệnh sau trong cửa sổ shell vừa mở:
        ```bash
        ab -c 100 -n 2000 [http://dvwa.hcmute.com/login.php](http://dvwa.hcmute.com/login.php)
        ```
        *(Giải thích: `-c 100`: 100 kết nối đồng thời, `-n 2000`: tổng cộng 2000 yêu cầu. Bạn có thể điều chỉnh các giá trị này)*.
    * **Quan sát:** Theo dõi output của lệnh `ab`. Ghi lại các thông số như `Complete requests`, `Failed requests`, `Requests per second`.
    * **Kết quả mong đợi:** Phần lớn hoặc toàn bộ request thành công (`Failed requests: 0`), tốc độ `Requests per second` tương đối ổn định.

2.  **Cấu hình Phòng chống DoS:**
    * **Tạo HTTP Access Limit Rule:**
        * Điều hướng đến `DoS Protection > Application > HTTP Access Limit`.
        * Nhấn `Create New`.
        * `Name`: `DoS-Access-Limit`
        * `HTTP Request Limit/sec (Standalone IP)`: `200` (hoặc giá trị phù hợp)
        * `HTTP Request Limit/sec (Shared IP)`: `200` (hoặc giá trị phù hợp)
        * `Action`: `Block Period`
        * `Block Period`: `600` (giây)
        * `Severity`: `High`
        * Nhấn `OK`.
    * **Tạo HTTP Flood Prevention Rule:** (Thường dùng trong mode Transparent, nhưng cấu hình để hoàn chỉnh)
        * Điều hướng đến `DoS Protection > Application > HTTP Flood Prevention`.
        * Nhấn `Create New`.
        * `Name`: `DoS-Flood-Prevent`
        * `HTTP Request Limit/sec`: `200`
        * `Action`: `Block Period`
        * `Block Period`: `600`
        * `Severity`: `High`
        * Nhấn `OK`.
    * **Tạo DoS Protection Policy:**
        * Điều hướng đến `DoS Protection > DoS Protection Policy`.
        * Nhấn `Create New`.
        * `Name`: `DoS-Policy`
        * `HTTP Access Limit`: Chọn `DoS-Access-Limit`.
        * `HTTP Flood Prevention`: Chọn `DoS-Flood-Prevent`.
        * Nhấn `OK`.
    * **Cập nhật Web Protection Profile (Kết hợp các bảo vệ):**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Trong phần `Inline Standard Protection`:
            * `Signatures`: Chọn lại `SQLi-Policy` **HOẶC** tạo một Signature Policy mới (`Combined-Signatures`) bao gồm cả SQLi và XSS signatures rồi chọn nó. *(Để đảm bảo bảo vệ toàn diện, nên tạo policy gộp)*.
        * Trong phần `DoS Protection`:
            * `DoS Protection Policy`: Chọn `DoS-Policy`.
        * Nhấn `OK`.

3.  **Kiểm tra sau khi Bảo vệ:**
    * Chạy lại lệnh `ab` như ở Bước 1 của Giai đoạn 4.
    * **Quan sát:** Lệnh `ab` sẽ chạy một lúc rồi báo lỗi, ví dụ: `apr_socket_recv: Connection reset by peer (104)` hoặc tương tự.
    * **Kết quả mong đợi:** Số lượng `Complete requests` giảm đáng kể, `Failed requests` tăng lên, `Requests per second` giảm mạnh sau khi ngưỡng bị vượt qua.
    * **Kiểm tra Log:**
        * Điều hướng đến `Log & Report > Log Access > Attack` hoặc `Event`.
        * Tìm các log liên quan đến `HTTP Access Limit Violation` hoặc `DoS Protection`.

### Giai đoạn 8: Quét Lỗ hổng Bảo mật

1.  **Cấu hình và Chạy Scan:**
    * **Tạo Scan Profile:**
        * Điều hướng đến `Web Vulnerability Scan > Scan Profile`.
        * Nhấn `Create New`.
        * `Name`: `DVWA-FullScan`
        * `Scan Target`: Nhập `dvwa.hcmute.com`.
        * `Scan Template`: Chọn `Full Audit`.
        * Nhấn `OK`.
    * **Tạo và Chạy Scan Policy:**
        * Điều hướng đến `Web Vulnerability Scan > Web Vulnerability Scan Policy`.
        * Nhấn `Create New`.
        * `Name`: `DVWA-ScanNow-Policy`
        * `Type`: Chọn `Run Now`.
        * `Profile`: Chọn `DVWA-FullScan`.
        * `Report Format`: Chọn `HTML`.
        * Nhấn `OK`.
    * **Theo dõi:** Quan sát cột `Status` của policy vừa tạo, trạng thái sẽ chuyển từ `Starting` -> `Scanning` -> `Done`. Quá trình này có thể mất vài phút đến vài chục phút tùy thuộc vào ứng dụng web.

2.  **Xem Kết quả Scan:**
    * Sau khi trạng thái là `Done`, điều hướng đến `Web Vulnerability Scan > Scan History`.
    * Tìm đến dòng tương ứng với `DVWA-ScanNow-Policy`.
    * Nhấn vào biểu tượng `Download` ở cột `Action`.
    * Mở file báo cáo (`.html`) vừa tải về bằng trình duyệt.
    * **Kết quả mong đợi:** Báo cáo chi tiết các lỗ hổng bảo mật (theo mức độ High, Medium, Low) được tìm thấy trên trang `dvwa.hcmute.com`, bao gồm mô tả lỗ hổng, URL bị ảnh hưởng, và đôi khi có gợi ý khắc phục.

### Giai đoạn 9: Giám sát và Báo cáo

1.  **Giám sát Trạng thái Hệ thống:**
    * Điều hướng đến `System > Status > Status`.
    * Quan sát các widget:
        * `System Resources`: CPU, Memory usage.
        * `System Information`: Trạng thái HA, phiên bản Firmware, Uptime.
        * `Throughput`: Biểu đồ lưu lượng mạng đi qua FortiWeb.
        * `Attack Event History`: Thống kê số lượng tấn công theo mức độ nghiêm trọng.
        * `Attack Log Widget`: Các log tấn công gần nhất.
        * `HTTP Transactions`: Số lượng giao dịch HTTP.

2.  **Sử dụng FortiView:**
    * Điều hướng đến `FortiView`.
    * Khám phá các tab khác nhau như `Threats`, `Sources`, `Destinations`, `Policies`, `Applications`.
    * Sử dụng các bộ lọc (Filter) và thay đổi khoảng thời gian (Time Interval) để xem dữ liệu trực quan về các cuộc tấn công, nguồn gốc, đích đến, và chính sách nào đã được áp dụng.

3.  **Phân tích Log Chi tiết:**
    * Điều hướng đến `Log & Report > Log Access`.
    * Xem lại các log trong `Attack`, `Traffic`, `Event`.
    * Click vào một dòng log để xem chi tiết (Log Details), bao gồm HTTP Headers, Parameters, Cookies, thông tin về Signature ID, Severity Level, Action Taken.

4.  **Xuất Báo cáo:**
    * Điều hướng đến `Log & Report > Report`.
    * Xem danh sách các mẫu báo cáo (`Report Templates`) có sẵn.
    * Chọn một mẫu phù hợp (ví dụ: `Security Analysis Report`, `Attack Event Summary`).
    * Nhấn `Run Now` (hoặc cấu hình `Schedule` nếu muốn chạy định kỳ).
    * Sau khi báo cáo được tạo (`Status` là `Done`), nhấn `Download` để tải về (thường ở định dạng PDF).
    * Xem xét nội dung báo cáo tổng hợp về tình hình an ninh trong khoảng thời gian đã chọn.

## Kết Luận (Nâng Cao)

Kịch bản thực hành nâng cao này đã chứng minh một cách toàn diện khả năng của FortiWeb trong việc bảo vệ ứng dụng web trước một loạt các mối đe dọa nghiêm trọng và đa dạng. Từ các lỗ hổng kinh điển như SQL Injection và XSS, đến các tấn công nguy hiểm hơn như Command Injection, File Inclusion và File Upload, FortiWeb đều thể hiện khả năng phát hiện và ngăn chặn hiệu quả thông qua cơ chế signature-based và các chính sách bảo vệ khác. Việc triển khai thành công các biện pháp chống tấn công DoS cũng đảm bảo tính sẵn sàng cho ứng dụng.

Kết quả quét lỗ hổng và việc giám sát liên tục qua FortiView, Log và Report đã cung cấp bằng chứng xác thực về hiệu quả của các chính sách bảo mật được áp dụng. Mặc dù WAF không thay thế hoàn toàn việc vá lỗi và lập trình an toàn ở tầng ứng dụng, nhưng nó đóng vai trò là một lớp phòng thủ vững chắc, cực kỳ quan trọng để giảm thiểu rủi ro và bảo vệ tài sản số của tổ chức trước các cuộc tấn công ngày càng tinh vi. FortiWeb là một giải pháp WAF mạnh mẽ, linh hoạt và hiệu quả.

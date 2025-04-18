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

*(Thực hiện tương tự Giai đoạn 1 của kịch bản trước)*

1.  **Truy cập và Khám phá Giao diện:** Đăng nhập và làm quen với các menu chính.
2.  **Cấu hình Server Pool (Real Server):** Tạo Server Pool `DVWA` với Real Server là `172.16.4.241:80`.
3.  **Cấu hình Virtual Server (VIP):** Tạo Virtual Server `DVWA-VIP` sử dụng IP Interface `port2`.
4.  **Cấu hình X-Forwarded-For (XFF):** Tạo Profile `XFF-Rule` và cấu hình `Trusted X-Header Sources`.
5.  **Kiểm tra Truy cập Ban đầu:** Truy cập `http://dvwa.hcmute.com`, đảm bảo trang DVWA hiển thị.

### Giai đoạn 2: Tấn công và Phòng chống SQL Injection

*(Thực hiện tương tự Giai đoạn 2 của kịch bản trước)*

1.  **Mô phỏng Tấn công SQLi (Chưa bảo vệ):** Đăng nhập DVWA (đặt Security Level `low`), vào mục SQL Injection, thử payload `' or '1'='1' union select version(), database()#`. **Kết quả mong đợi:** Tấn công thành công, thấy thông tin DB.
2.  **Cấu hình Phòng chống SQLi:**
    * Tạo Signature Policy `SQLi-Policy` (Action: `Alert & Deny` cho SQL Injection & Generic Attacks).
    * Tạo Web Protection Profile `Base-Protection-Profile` (Sử dụng Signature `SQLi-Policy` và XFF `XFF-Rule`).
    * Tạo Server Policy `Policy_Protect_DVWA` (Áp dụng `Base-Protection-Profile` cho `DVWA-VIP` và `DVWA` Server Pool).
3.  **Kiểm tra sau khi Bảo vệ:** Thử lại tấn công SQLi. **Kết quả mong đợi:** Bị chặn bởi FortiWeb. Kiểm tra log Attack.

### Giai đoạn 3: Tấn công và Phòng chống XSS

*(Thực hiện tương tự Giai đoạn 3 của kịch bản trước, nhưng sẽ cập nhật Profile để gộp bảo vệ)*

1.  **Mô phỏng Tấn công XSS (Chưa bảo vệ XSS):**
    * **Reflected:** Vào mục XSS (Reflected) (level `low`), thử payload `<script>alert('XSS Reflected Test')</script>`. **Kết quả mong đợi:** Alert box hiện ra.
    * **Stored:** Vào mục XSS (Stored) (level `low`), thử payload `<script>alert(document.cookie)</script>`. **Kết quả mong đợi:** Alert box chứa cookie hiện ra sau khi submit/reload.
2.  **Cấu hình Phòng chống XSS (Cập nhật Profile):**
    * **Tạo Signature Policy:** Tạo Signature Policy `XSS-Policy` (Action: `Alert & Deny` cho Cross Site Scripting).
    * **Tạo Signature Policy Gộp (Khuyến nghị):**
        * Điều hướng đến `Web Protection > Known Attacks > Signatures`.
        * Nhấn `Create New`. Đặt tên `Combined-SQLi-XSS-Policy`.
        * Kích hoạt (`Enable`, Action `Alert & Deny`) các signatures cho `SQL Injection`, `SQL Injection (Extended)`, `Generic Attacks`, `Generic Attacks (Extended)`, `Cross Site Scripting`, `Cross Site Scripting (Extended)`.
        * Nhấn `OK`.
    * **Cập nhật Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Trong phần `Inline Standard Protection`:
            * `Signatures`: Thay đổi thành `Combined-SQLi-XSS-Policy` (hoặc policy gộp bạn vừa tạo).
        * Nhấn `OK`.
3.  **Kiểm tra sau khi Bảo vệ:** Thử lại cả tấn công Reflected và Stored XSS. **Kết quả mong đợi:** Bị chặn bởi FortiWeb. Kiểm tra log Attack.

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

### Giai đoạn 7: Tấn công và Phòng chống DoS (Đã tích hợp bảo vệ khác)

*(Thực hiện tương tự Giai đoạn 4 của kịch bản trước, nhưng đảm bảo Profile đã gộp các bảo vệ)*

1.  **Mô phỏng Tấn công DoS:** Chạy lệnh `ab -c 100 -n 2000 http://dvwa.hcmute.com/login.php`. Quan sát kết quả ban đầu (nếu chưa bật DoS protection trong profile).
2.  **Cấu hình Phòng chống DoS (Cập nhật Profile):**
    * Tạo `DoS-Access-Limit` Rule, `DoS-Flood-Prevent` Rule, `DoS-Policy` như kịch bản trước.
    * **Cập nhật Web Protection Profile:**
        * Điều hướng đến `Policy > Web Protection Profile > Inline Protection Profile`.
        * Chọn và **Edit** Profile `Base-Protection-Profile`.
        * Đảm bảo `Inline Standard Protection > Signatures` đang sử dụng policy gộp (`Combined-All-Policy`).
        * Trong phần `DoS Protection`: Chọn `DoS-Policy`.
        * Nhấn `OK`.
3.  **Kiểm tra sau khi Bảo vệ:** Chạy lại lệnh `ab`. **Kết quả mong đợi:** Tấn công bị giới hạn/chặn. Kiểm tra log Attack/Event liên quan đến DoS.

### Giai đoạn 8: Quét Lỗ hổng Bảo mật (Sau khi áp dụng bảo vệ)

*(Thực hiện tương tự Giai đoạn 5 của kịch bản trước)*

1.  **Cấu hình và Chạy Scan:** Tạo Scan Profile `DVWA-FullScan`, tạo Scan Policy `DVWA-ScanNow-Policy` (Type `Run Now`, Profile `DVWA-FullScan`, Format `HTML`), chạy và đợi hoàn thành.
2.  **Xem Kết quả:** Download và xem báo cáo scan từ `Scan History`. **Kết quả mong đợi:** Báo cáo có thể vẫn phát hiện một số lỗ hổng ở tầng ứng dụng (vì WAF không sửa code gốc), nhưng cũng có thể cho thấy một số tấn công không thực hiện được do bị WAF chặn ở tầng ngoài. Phân tích sự khác biệt nếu bạn đã từng scan trước khi bật WAF.

### Giai đoạn 9: Giám sát và Báo cáo (Toàn diện)

*(Thực hiện tương tự Giai đoạn 6 của kịch bản trước, nhưng bao quát hơn)*

1.  **Giám sát Trạng thái Hệ thống:** Sử dụng `System > Status > Status` để xem tổng quan tài nguyên, throughput, và các sự kiện tấn công mới nhất (bao gồm cả các loại tấn công mới được thêm vào).
2.  **Sử dụng FortiView:** Khám phá sâu hơn các biểu đồ, áp dụng filter cho các loại tấn công cụ thể (SQLi, XSS, OS Commanding, Directory Traversal, DoS...) để thấy rõ hoạt động bảo vệ của WAF.
3.  **Phân tích Log Chi tiết:** Vào `Log & Report > Log Access > Attack`, xem xét kỹ lưỡng các log của tất cả các loại tấn công đã thực hiện, hiểu rõ thông tin chi tiết mà FortiWeb cung cấp.
4.  **Xuất Báo cáo:** Tạo và xem xét các báo cáo tổng hợp (`Security Analysis`, `Attack Event Summary`), đảm bảo chúng phản ánh đầy đủ các hoạt động tấn công và phòng thủ đã diễn ra trong toàn bộ kịch bản nâng cao.

## Kết Luận (Nâng Cao)

Kịch bản thực hành nâng cao này đã chứng minh một cách toàn diện khả năng của FortiWeb trong việc bảo vệ ứng dụng web trước một loạt các mối đe dọa nghiêm trọng và đa dạng. Từ các lỗ hổng kinh điển như SQL Injection và XSS, đến các tấn công nguy hiểm hơn như Command Injection, File Inclusion và File Upload, FortiWeb đều thể hiện khả năng phát hiện và ngăn chặn hiệu quả thông qua cơ chế signature-based và các chính sách bảo vệ khác. Việc triển khai thành công các biện pháp chống tấn công DoS cũng đảm bảo tính sẵn sàng cho ứng dụng.

Kết quả quét lỗ hổng và việc giám sát liên tục qua FortiView, Log và Report đã cung cấp bằng chứng xác thực về hiệu quả của các chính sách bảo mật được áp dụng. Mặc dù WAF không thay thế hoàn toàn việc vá lỗi và lập trình an toàn ở tầng ứng dụng, nhưng nó đóng vai trò là một lớp phòng thủ vững chắc, cực kỳ quan trọng để giảm thiểu rủi ro và bảo vệ tài sản số của tổ chức trước các cuộc tấn công ngày càng tinh vi. FortiWeb là một giải pháp WAF mạnh mẽ, linh hoạt và hiệu quả.

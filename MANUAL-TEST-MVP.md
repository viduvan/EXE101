# Kịch bản test thủ công — MVP Web LaborLink

Mục tiêu: đi tay hết các luồng cần có để coi MVP web là **demo được**. Mỗi case ghi rõ user story/UC, tài khoản dùng, các bước bấm và kết quả mong đợi.

Phạm vi: chỉ web (không có mobile). Không có quản lý hợp đồng; quy trình dừng ở bước **Offer → Đã tuyển**.

---

## 0. Chuẩn bị

1. Chạy hệ thống và seed dữ liệu:
   ```bash
   docker compose up -d --build
   make demo-seed
   ```
2. Các địa chỉ:
   - Web: http://localhost:3000
   - Mailpit (xem email xác minh, quên mật khẩu, thông báo): http://localhost:8025
3. **Mở 2 trình duyệt** (hoặc 1 cửa sổ thường + 1 cửa sổ ẩn danh) để đăng nhập **nhà tuyển dụng** và **ứng viên** cùng lúc. Các luồng hai phía (chat, phỏng vấn, offer) cần kiểm tra cả hai bên.
4. Mở web bằng `localhost`, không dùng IP LAN, thì nút "Lấy vị trí hiện tại" mới hoạt động.
5. Nếu dữ liệu đã bị thao tác nhiều, chạy lại `make demo-seed` (seed cập nhật theo email/tiêu đề, không nhân bản). `make demo-reset` xóa sạch rồi seed lại.

### Tài khoản demo (mật khẩu chung: `DemoPass123!`)

| Tài khoản | Vai trò | Dùng để test |
|---|---|---|
| `admin@demo.local` | Admin | Toàn bộ khu `/admin` |
| `employer@demo.local` | NTD đã xác minh, gói PLUS | Tin tuyển dụng, pipeline, phỏng vấn, offer, ghim tin, thanh toán, analytics |
| `pending@demo.local` | NTD chờ duyệt | Admin duyệt doanh nghiệp |
| `cafe@demo.local` | NTD Chuỗi cà phê Sáng | Có sẵn 1 tin tạm dừng |
| `candidate@demo.local` | Ứng viên | Có lời mời chờ phản hồi, 3 việc đã lưu, gợi ý việc |
| `candidate.mai@demo.local` | Ứng viên | Có lịch phỏng vấn chờ xác nhận |
| `candidate.minh@demo.local` | Ứng viên | Có Offer chờ phản hồi, hội thoại cần NTD xử lý (AI chuyển tiếp) |
| `candidate.ngoc@demo.local` | Ứng viên | Có Offer chờ phản hồi (Demo Shop) |
| `candidate.tuan@demo.local` | Ứng viên | Đã nhận việc, có 1 đơn đã rút |

> Mẹo: seed có sẵn trạng thái cho case "phản hồi" (xác nhận phỏng vấn, nhận/từ chối Offer, nhận lời mời). Nhưng mỗi trạng thái chỉ phản hồi được **một lần**. Muốn test lại thì seed lại, hoặc tự tạo đơn mới theo luồng E2E ở mục 9.

### Quy ước kết quả

- Mọi thông báo đều bằng **tiếng Việt**, hiện dạng **toast** (góc màn hình). Không được lộ mã lỗi thô như `invalid_transition`.
- Ô nhập sai phải **viền đỏ và có dòng lỗi ngay dưới ô**, và không được gửi request.
- Ghi kết quả từng case: ✅ Đạt / ❌ Lỗi (mô tả + ảnh chụp) / ⏭ Bỏ qua.

---

## 1. Tài khoản & xác thực (SYS-AUTH-01, SYS-AUTH-02)

### AUTH-01 · Đăng ký ứng viên + xác minh email — **P0**
**User story:** Là học sinh/sinh viên, tôi muốn tự tạo tài khoản để tìm việc.
**Tài khoản:** email mới, ví dụ `test.sv1@demo.local`.

1. Vào `/register`, chọn vai trò **Ứng viên**.
2. Bấm **Tạo tài khoản** khi để trống → các ô báo lỗi đỏ.
3. Nhập **Mật khẩu** và **Nhập lại mật khẩu** khác nhau → báo không khớp.
4. Nhập đủ và đúng, bấm **Tạo tài khoản** → hiện "Đăng ký thành công".
5. Mở Mailpit (localhost:8025), lấy mã hoặc link xác minh. Nhập vào ô **Mã xác minh**, bấm **Xác minh email** → "Xác minh thành công".
6. Đăng ký lại bằng cùng email → báo email đã tồn tại.

**Kỳ vọng:** Sau khi xác minh, đăng nhập vào được `/candidate/dashboard`.

### AUTH-02 · Đăng ký nhà tuyển dụng — **P0**
**User story:** Là chủ hộ kinh doanh, tôi muốn tạo tài khoản để đăng tin tuyển dụng.

1. Vào `/register`, chọn vai trò **Nhà tuyển dụng**, điền đủ thông tin rồi bấm **Tạo tài khoản**.
2. Thử đăng nhập trước khi xác minh email → bị chặn hoặc được nhắc xác minh.
3. Xác minh qua Mailpit rồi đăng nhập → vào được `/employer/dashboard`.

### AUTH-03 · Đăng nhập & điều hướng theo vai trò — **P0**
**UC:** Hệ thống đưa mỗi vai trò về đúng khu làm việc.

1. Vào `/login`, bấm đăng nhập khi để trống → lỗi tại ô.
2. Nhập sai mật khẩu, và thử email không tồn tại → toast lỗi tiếng Việt.
3. Đăng nhập lần lượt `candidate@`, `employer@`, `admin@`.

**Kỳ vọng:** Ứng viên về `/candidate/dashboard`, NTD về `/employer/dashboard`, admin về `/admin`.

### AUTH-04 · Phân quyền truy cập — **P0**
1. Đăng nhập `candidate@`, gõ thẳng URL `/admin` và `/employer/jobs` → bị chặn hoặc chuyển hướng.
2. Đăng xuất (bấm **Mở menu tài khoản** → Đăng xuất), sau đó vào `/candidate/dashboard` → bị đưa về `/login`.

### AUTH-05 · Quên mật khẩu — P1
1. Ở `/login`, vào quên mật khẩu (`/forgot-password`), nhập email `candidate@demo.local`, bấm **Gửi liên kết**.
2. Mở Mailpit, bấm link → nhập **Mật khẩu mới** và **Nhập lại mật khẩu mới**, bấm **Cập nhật mật khẩu**.
3. Đăng nhập bằng mật khẩu mới → được. Đăng nhập bằng mật khẩu cũ → bị từ chối.

> Làm xong nhớ đổi lại `DemoPass123!` hoặc seed lại.

### AUTH-06 · Đổi mật khẩu trong cài đặt — P1
**Cài đặt → tab Bảo mật:** nhập sai **Mật khẩu hiện tại** → lỗi. Nhập đúng → đổi thành công, đăng nhập lại bằng mật khẩu mới.

---

## 2. Hồ sơ (SYS-EMP-01, SYS-CAN-01/02/03)

### PROF-01 · Hồ sơ doanh nghiệp — **P0**
**User story:** Là NTD, tôi muốn hoàn thiện hồ sơ cơ sở kinh doanh để ứng viên tin tưởng.
**Tài khoản:** `employer@`

1. Vào **Cài đặt** → tab hồ sơ (hoặc `/employer/profile`).
2. Xóa **Tên doanh nghiệp / hộ kinh doanh**, bấm **Lưu thay đổi** → lỗi đỏ.
3. Sửa **Mô tả doanh nghiệp** và **Số điện thoại**, bấm **Lưu thay đổi** → toast thành công.
4. Tải lại trang (F5) → dữ liệu vẫn còn.

### PROF-02 · Gửi xác minh doanh nghiệp — **P0**
**User story:** Là NTD mới, tôi gửi giấy phép kinh doanh để được admin xác minh.
**Tài khoản:** NTD vừa đăng ký ở AUTH-02.

1. Vào hồ sơ doanh nghiệp, bấm **Gửi xác minh** khi chưa tải giấy phép → báo bắt buộc.
2. Tải file sai định dạng hoặc quá lớn → báo lỗi.
3. Tải file hợp lệ, bấm **Gửi xác minh** → trạng thái "Đang chờ duyệt".
4. Tiếp tục ở ADM-01 để admin duyệt. Sau đó quay lại → hiện huy hiệu "Đã xác minh".

### PROF-03 · Hồ sơ ứng viên + quyền riêng tư — **P0**
**User story:** Là ứng viên, tôi khai hồ sơ tìm việc và chọn thông tin liên hệ nào được lộ.
**Tài khoản:** `candidate@`

1. Vào **Cài đặt** → tab **Hồ sơ cá nhân** (hoặc `/candidate/profile`).
2. Điền **Họ và tên**, **Vị trí mong muốn**, **Số năm kinh nghiệm**, **Kỹ năng**, **Tỉnh hoặc thành phố**, **Số điện thoại liên hệ**, **Liên kết mạng xã hội**.
3. Chỉnh trạng thái sẵn sàng tìm việc và các tùy chọn ẩn/hiện email, số điện thoại.
4. Bấm **Lưu thay đổi**, F5 → dữ liệu còn nguyên.

---

## 3. Tin tuyển dụng (SYS-EMP-03 → 07, SYS-ADM-03)

### JOB-01 · Tạo tin nháp thủ công — **P0**
**User story:** Là NTD, tôi tạo tin tuyển dụng.
**Tài khoản:** `employer@`

1. Menu **Tin tuyển dụng** (`/employer/jobs`), bấm **Tạo bản nháp** khi để trống → lỗi tại ô.
2. Nhập **Tiêu đề công việc**, **Danh mục ngành nghề**, **Lương tối thiểu** lớn hơn **Lương tối đa** → báo lỗi.
3. Đặt **Hạn nộp hồ sơ** ở ngày đã qua → báo lỗi.
4. Nhập hợp lệ, bấm **Tạo bản nháp** → toast "Đã tạo bản nháp …". Tin xuất hiện ở tab **Bản nháp**.

### JOB-02 · Tạo tin bằng AI — **P0** (điểm nhấn demo)
**User story:** Là NTD, tôi mô tả ngắn vị trí cần tuyển và AI điền giúp các trường.

1. Mở một tin nháp (`/employer/jobs/<id>`).
2. Ô **Mô tả vị trí cần tuyển**: nhập chữ quá ngắn, ví dụ "ngắn", rồi bấm **Tạo gợi ý** → báo cần mô tả dài hơn.
3. Nhập mô tả đầy đủ, ví dụ "Cần 2 nhân viên pha chế ca tối tại quận 1, lương 25k/giờ, ưu tiên sinh viên", bấm **Tạo gợi ý**.
4. Hộp **Xem trước gợi ý AI** hiện ra. Bấm **Áp dụng N trường**.

**Kỳ vọng:**
- AI chỉ điền vào các trường đang trống.
- Tin **không tự đăng**; chỉ khi bấm **Lưu thay đổi** thì dữ liệu mới được lưu.
- Nếu AI lỗi thì hiện toast tiếng Việt, không hiện mã lỗi thô.

### JOB-03 · Chỉnh sửa tin: ca làm, kỹ năng, bản đồ — **P0**
1. Trong trang tin, bấm **Thêm ca làm**. Đặt giờ kết thúc trước giờ bắt đầu → lỗi "Giờ kết thúc phải sau giờ bắt đầu."
2. Chọn một kỹ năng trong **Gợi ý kỹ năng** (ví dụ "Pha chế").
3. Bấm **Lấy vị trí hiện tại** (hoặc bấm vào bản đồ để đặt pin) → toast "Đã đặt pin…".
4. Bấm **Lưu thay đổi** → "Tin tuyển dụng đã được lưu." F5 → dữ liệu còn.

### JOB-04 · Gửi duyệt → Admin duyệt / từ chối — **P0**
**UC:** Tin phải được admin duyệt mới hiện công khai. Mặc định hệ thống **bật kiểm duyệt**; xem ở `/admin/settings` → "Kiểm duyệt tin trước khi đăng".

1. **NTD:** trong tin nháp đầy đủ, bấm **Gửi duyệt** → xác nhận **Gửi duyệt** → trang hiện "Tin đang chờ quản trị viên duyệt".
2. **Ứng viên / khách:** tìm tin này ở `/jobs` → **không thấy**.
3. **Admin:** vào **Tin tuyển dụng** (`/admin/jobs?status=PENDING_REVIEW`), bấm tên tin → **Duyệt tin** → toast "Đã duyệt tin…".
4. **Khách:** tìm lại ở `/jobs` → giờ đã thấy tin.
5. **Nhánh từ chối:** gửi duyệt một tin khác. Admin bấm **Từ chối**, để trống lý do rồi bấm **Từ chối tin** → báo bắt buộc. Nhập lý do, ví dụ "Mức lương chưa rõ ràng", rồi bấm lại **Từ chối tin**.
6. **NTD:** mở tin đó → thấy "Tin chưa được duyệt." và "Lý do: …", và vẫn sửa được.

### JOB-05 · Vòng đời tin — **P0**
**UC:** NTD tạm dừng, đăng lại, đóng và mở lại tin.
**Tài khoản:** `employer@`, dùng một tin **Đang tuyển**.

Trên trang chi tiết tin, lần lượt bấm và xác nhận các nút sau. Mỗi lần có toast "… thành công.":

| Bấm | Trạng thái mới | Khách còn thấy trên `/jobs`? |
|---|---|---|
| **Tạm dừng** | Tạm dừng | Không |
| **Đăng lại** | Đang tuyển | Có |
| **Đóng tin** | Đã đóng (tab **Đã đóng**) | Không |
| **Mở lại tin** | Đang tuyển | Có |

Thêm: tin nháp thiếu tiêu đề mà bấm **Đăng tin** → toast "Vui lòng bổ sung trước khi đăng: tiêu đề."

### JOB-06 · Xóa tin nháp — P1
Ở `/employer/jobs`, tab **Bản nháp**, bấm **Xóa** một tin nháp và xác nhận → tin biến mất.

### JOB-07 · Ghim tin (gói PLUS) — P1
**User story:** Là NTD trả phí, tôi ghim tin để tin nổi lên đầu kết quả tìm kiếm.

1. `employer@` → mở tin đang tuyển → vào trang quảng bá (`/employer/jobs/<id>/promotion`) → hiện "Tin hiện chưa được ghim."
2. Chọn **Bắt đầu ghim** và **Kết thúc ghim**, bấm **Ghim tin** → "Đã lên lịch ghim tin tuyển dụng."
3. Thử ghim thêm tin thứ hai → báo "Mỗi nhà tuyển dụng chỉ được có tối đa 1 lượt ghim…".
4. Khách tìm việc ở `/jobs` → tin được ghim nằm trên đầu.

> Tài khoản `employer@` đã có sẵn 1 tin ghim. Nếu bước 2 bị chặn vì hạn mức, admin hủy lượt ghim cũ trước (ADM-05).

---

## 4. Tìm việc & ứng tuyển (SYS-CAN-04 → 09)

### SEARCH-01 · Tìm việc công khai + bộ lọc — **P0**
**User story:** Là ứng viên (kể cả chưa đăng nhập), tôi tìm việc theo từ khóa, nơi làm và mức lương.

1. Chưa đăng nhập, vào `/jobs`. Nhập **Tên công việc hoặc kỹ năng**, ví dụ "pha chế", bấm **Tìm kiếm**.
2. Lọc theo tỉnh/thành, **Hình thức** (toàn thời gian / bán thời gian / linh hoạt), danh mục.
3. Bấm **Mức lương**, chọn **Theo tháng**, nhập **Từ** lớn hơn **Đến** → báo lỗi. Nhập hợp lệ, bấm **Áp dụng** → hiện chip lọc, bấm chip để bỏ lọc.
4. Chuyển trang (phân trang). F5 → giữ nguyên bộ lọc trên URL.
5. Nhập từ khóa vô nghĩa → "Không tìm thấy công việc".

### SEARCH-02 · Việc làm trên bản đồ & tìm gần — **P0**
1. `candidate@` → menu **Việc làm** (`/candidate/jobs`).
2. Chuyển sang chế độ **Bản đồ** → chỉ hiện các tin có tọa độ (TP HCM, Hà Nội, Đà Nẵng).
3. Đặt **Khoảng cách tối đa (km)** → danh sách thu hẹp theo vị trí trong hồ sơ hoặc vị trí hiện tại.

### SEARCH-03 · Gợi ý việc làm phù hợp — P1
**Tài khoản:** `candidate@`, đã có sẵn lịch sử xem.

1. Mục **Tổng quan** (`/candidate/dashboard`) có khối việc gợi ý, kèm lý do gợi ý.
2. Xem và lưu một tin, sau đó quay lại dashboard → tin tương tự được đẩy lên với lý do dựa trên hành vi.

### SEARCH-04 · Lưu việc — P1
1. Trên thẻ tin hoặc trang chi tiết, bấm **Lưu việc làm** → nút đổi thành **Đã lưu**.
2. Menu **Việc đã lưu** → tin có trong danh sách. Số đếm trên dashboard tăng.
3. Bấm **Bỏ lưu** → xác nhận ở hộp "Bỏ lưu việc làm?" → tin biến mất.

### APPLY-01 · Ứng tuyển — **P0**
**User story:** Là ứng viên, tôi ứng tuyển kèm lời nhắn cho NTD.
**Tài khoản:** `candidate@`, chọn một tin chưa ứng tuyển.

1. Mở tin → bấm **Ứng tuyển** → hộp **Ứng tuyển** hiện ra.
2. Nhập lời nhắn quá dài → báo lỗi độ dài.
3. Nhập lời nhắn hợp lệ, bấm **Gửi đơn ứng tuyển** → toast thành công.
4. Menu **Đơn ứng tuyển** → đơn mới có trạng thái "Đã ứng tuyển".
5. Mở lại tin đó → không ứng tuyển lần hai được (hiện thông báo đã ứng tuyển).

**Kiểm tra phía NTD:** NTD sở hữu tin nhận thông báo (chuông), và ứng viên xuất hiện trong pipeline.

### APPLY-02 · Rút đơn — P1
1. **Đơn ứng tuyển** → bấm **Rút đơn** trên một đơn → hộp "Rút đơn ứng tuyển?".
2. Nhập **Lý do rút đơn** (không bắt buộc, tối đa 1000 ký tự), bấm **Xác nhận rút đơn** → toast "Đã rút đơn".
3. **NTD** mở pipeline → thẻ ghi "Ứng viên đã rút đơn" và **không còn** nút chuyển trạng thái.
4. Nếu NTD vẫn đang mở trang cũ và bấm chuyển trạng thái → toast "Không thể chuyển đơn ứng tuyển sang trạng thái này." Đây là backend chặn chuyển trạng thái sai.

### APPLY-03 · Báo cáo tin vi phạm — P1
1. `candidate@` mở một tin → **Báo cáo tin** → bấm **Gửi báo cáo** khi để trống → lỗi.
2. Chọn **Lý do**, nhập mô tả, bấm **Gửi báo cáo** → "Đã gửi báo cáo…".
3. Báo cáo lại tin đó → "Bạn đã báo cáo tin tuyển dụng này…".
4. Tiếp tục ở ADM-04.

---

## 5. Quy trình tuyển dụng — luồng chính (SYS-EMP-08 → 13, SYS-CAN-08/10/11)

> Đây là **xương sống của demo**. Cần 2 trình duyệt: A = `employer@`, B = ứng viên.
> Trạng thái đơn: `Đã ứng tuyển → Đã liên hệ → Đang phỏng vấn → Đã gửi offer → Đã tuyển`. Có thể **Từ chối** ở các bước giữa.

### HIRE-01 · Pipeline ứng viên — **P0**
**User story:** Là NTD, tôi xem và chuyển trạng thái ứng viên theo từng bước.

1. A: **Tin tuyển dụng** → mở một tin có ứng viên → trang ứng viên (`/employer/jobs/<id>/applicants`).
2. Kiểm tra các tab đếm số: **Tất cả**, **Đã ứng tuyển**, **Đang phỏng vấn**, **Đã gửi offer**, **Đã tuyển**…
3. Bấm **Bảng** để xem dạng Kanban, bấm vào thẻ ứng viên để mở chi tiết, **Đóng**, rồi quay lại **Danh sách**.
4. Trên thẻ ứng viên "Đã ứng tuyển", bấm **Chuyển Đã liên hệ** → toast "Đã chuyển sang “Đã liên hệ”."
5. Bấm **Chuyển Đang phỏng vấn** → giờ thẻ hiện nút **Đặt phỏng vấn** và **Gửi offer**.
6. F5 → trạng thái vẫn giữ.
7. B: **Đơn ứng tuyển** → đơn hiện "Đang phỏng vấn", có dòng thời gian các bước. B cũng nhận thông báo.

**Kỳ vọng:** Chỉ hiện các nút hợp lệ với trạng thái hiện tại.

### HIRE-02 · Đánh giá & ghi chú ứng viên — P1
A: mở chi tiết ứng viên (`/employer/applicants/<id>`) → thêm ghi chú nội bộ, xem phần đánh giá điểm mạnh/yếu do AI tạo.
Seed chỉ có sẵn phần đánh giá AI cho `candidate@`, `candidate.mai@` và `candidate.minh@`.
Kiểm tra thêm: ứng viên **không** thấy được ghi chú nội bộ.

### HIRE-03 · Tìm & mời ứng viên — **P0**
**User story:** Là NTD, tôi chủ động tìm ứng viên hợp với tin và mời họ ứng tuyển.

**Cách 1: từ đề xuất theo tin**
1. A: trang ứng viên của tin → mục đề xuất (`/employer/jobs/<id>/applicants/recommendations`).
2. Nhập **Kỹ năng**, bấm **Đề xuất ứng viên** → danh sách xếp theo điểm phù hợp.
3. Bấm **Mời ứng tuyển** → hộp "Xác nhận lời mời". Nhập **Lời nhắn cho ứng viên** dài hơn 1000 ký tự → lỗi. Nhập hợp lệ, bấm **Gửi lời mời**.
4. Người vừa mời không còn nút **Mời ứng tuyển**.

**Cách 2: từ menu Ứng viên**
Vào `/employer/candidates`, bấm mời mà chưa chọn tin → báo bắt buộc chọn tin. Nếu ứng viên đã ứng tuyển tin đó → hiện thông báo. Mời lại người đã mời → báo đã mời.

### HIRE-04 · Ứng viên nhận / từ chối lời mời — **P0**
**Tài khoản B:** `candidate@`, đã có sẵn lời mời chờ phản hồi, hoặc ứng viên vừa được mời ở HIRE-03.

1. B: menu **Lời mời** → xem lời nhắn và tin đi kèm.
2. Bấm **Nhận lời & ứng tuyển** → xác nhận → lời mời chuyển trạng thái, và **Đơn ứng tuyển** có đơn mới.
3. **Nhánh từ chối** (lời mời khác): bấm **Từ chối** → hộp "Từ chối lời mời?". Bấm **Quay lại** để hủy, rồi bấm lại **Từ chối** → **Xác nhận từ chối** → **không** tạo đơn ứng tuyển.
4. A nhận được thông báo phản hồi.

### HIRE-05 · Đặt lịch phỏng vấn — **P0**
**User story:** Là NTD, tôi hẹn lịch phỏng vấn; ứng viên xác nhận tham gia và thêm vào lịch cá nhân.

1. A: trên thẻ ứng viên "Đang phỏng vấn", bấm **Đặt phỏng vấn** → hộp "Đặt lịch phỏng vấn".
2. Chọn **Thời gian** trong quá khứ, bấm **Lưu thay đổi** → lỗi.
3. Chọn thời lượng (15/30/45/60/90 phút) và **Hình thức** (trực tiếp hoặc online).
4. Nhập **Liên kết phòng họp** sai định dạng → lỗi. Để trống thì hệ thống tự tạo link Jitsi.
5. Bấm **Lưu thay đổi** → thành công.
6. B: menu **Phỏng vấn** → thấy lịch, bấm được link phòng họp, bấm **Tải file lịch (.ics)** → file tải về mở được bằng Google Calendar/Outlook.
7. A: menu **Phỏng vấn** → **Sửa / đổi lịch** → thử thời lượng 10 hoặc 481 → lỗi. Đổi giờ hợp lệ, bấm **Lưu / đổi lịch**.
8. B: thấy giờ mới → bấm **Xác nhận tham gia** → xác nhận → trạng thái "Đã xác nhận".
9. A: **Hủy lịch** → nhập **Lý do hủy** → **Xác nhận** → B thấy lịch bị hủy, không còn nút .ics hay xác nhận.

> Có sẵn: đăng nhập `candidate.mai@` để test ngay bước 8 (lịch chờ xác nhận).

### HIRE-06 · Gửi Offer → Ứng viên nhận — **P0**
**User story:** Là NTD, tôi gửi đề nghị tuyển dụng (lương, ngày bắt đầu); ứng viên chấp nhận và được ghi nhận "Đã tuyển".

1. A: trên thẻ ứng viên "Đang phỏng vấn", bấm **Gửi offer** → hộp "Gửi offer".
2. Nhập **Mức lương** = 0, để trống **Ngày bắt đầu** → lỗi. Chọn ngày bắt đầu trong quá khứ → lỗi.
3. Nhập lương 9.500.000, chọn ngày trong tương lai, nhập **Ghi chú offer**, bấm **Lưu thay đổi**.
4. A: thẻ hiện nhãn **"Đã gửi offer"**, tab **Đã gửi offer** tăng 1.
5. B: **Đơn ứng tuyển** → nhãn **"Đã nhận đề nghị"**, thấy lương, ngày bắt đầu và ghi chú → bấm **Chấp nhận** → **Chấp nhận đề nghị**.
6. A: tab **Đã tuyển** tăng 1. Hai phía hiển thị trạng thái khớp nhau.

> Có sẵn: `candidate.minh@` hoặc `candidate.ngoc@` đang có Offer chờ phản hồi.

### HIRE-07 · Ứng viên từ chối Offer — **P0**
1. Làm như HIRE-06 đến bước 4 (hoặc đăng nhập `candidate.trang@`).
2. B: bấm **Từ chối** → hộp "Từ chối đề nghị?" → bấm **Xác nhận từ chối** khi chưa nhập lý do → lỗi bắt buộc.
3. Nhập **Lý do từ chối**, ví dụ "Mức lương chưa phù hợp", rồi xác nhận.
4. A: đơn hiện trạng thái offer bị từ chối, kèm lý do. Trạng thái này **khác** với "rút đơn".

---

## 6. Nhắn tin, AI trả lời hộ & thông báo (SYS-EMP-14/15/16, SYS-CAN-12/14)

### MSG-01 · Chat hai chiều realtime — **P0**
**User story:** NTD và ứng viên nhắn tin trực tiếp với nhau quanh một đơn ứng tuyển.

1. A và B mở cùng một hội thoại: menu **Tin nhắn**, hoặc từ đơn ứng tuyển.
2. B gõ vào ô **Nội dung tin nhắn**, bấm **Gửi** → A thấy tin mới **ngay lập tức**, không cần F5.
3. A trả lời → B thấy ngay.
4. Kiểm tra biểu tượng tin nhắn trên header: có **badge số chưa đọc**, mở dropdown được, và bấm vào thì mở cửa sổ chat nổi mà không rời trang.
5. Trang `/messages`: tìm kiếm hội thoại, lọc chưa đọc, dòng đang chọn được tô sáng, có panel thông tin đơn/tin ở bên.
6. Cuộn lên trong hội thoại dài → tải thêm tin cũ, vị trí cuộn không bị nhảy.

### MSG-02 · AI trả lời hộ có kiểm soát — **P0** (điểm nhấn demo)
**User story:** AI trả lời các câu hỏi thường gặp dựa trên câu trả lời mẫu đã duyệt. Câu nào ngoài phạm vi thì AI chuyển cho NTD, NTD duyệt bản nháp rồi mới gửi.

1. **Chuẩn bị:** A vào **Cài đặt** → tab **Câu trả lời mẫu**, xem hoặc thêm FAQ, ví dụ về đồng phục hay giờ làm.
2. B (`candidate@`): trong hội thoại, hỏi một câu **có trong FAQ** → AI trả lời, tin gắn nhãn "Trợ lý AI".
3. B hỏi câu **ngoài phạm vi hoặc nhạy cảm**, ví dụ về lương thưởng riêng → A thấy banner "AI đã chuyển hội thoại cho bạn." và nhãn "Ngoài knowledge đã duyệt".
4. A: bản nháp AI nằm cạnh ô soạn tin. Thử **Chỉnh sửa**, để nội dung toàn khoảng trắng rồi bấm **Gửi** → lỗi. Sửa nội dung hợp lệ rồi **Gửi**.
5. B chỉ thấy tin **đã được duyệt**, không bao giờ thấy bản nháp.
6. A bấm **Bật lại AI** → banner biến mất.

> Có sẵn: `candidate.minh@` có hội thoại đang cần NTD xử lý.

### MSG-03 · Trợ lý LaborLink (popup) & bộ nhớ — P1
1. Bấm nút **Mở trợ lý LaborLink** (góc màn hình), hỏi về cách dùng hệ thống hoặc gợi ý việc.
2. **Cài đặt** → tab **Bộ nhớ trợ lý** → chỉ hiện các mục đã xác nhận. Sửa một mục (có kiểm tra dữ liệu), xóa một mục → mục biến mất.

### NOTI-01 · Thông báo — **P0**
1. Sau các thao tác ở mục 5 (mời, đổi trạng thái, đặt lịch, offer, tin nhắn), mở chuông thông báo hoặc `/candidate/notifications` (`/employer/notifications` với NTD).
2. Có số chưa đọc. Bấm vào một thông báo → sang **đúng màn hình** liên quan, và thông báo chuyển sang đã đọc.
3. Đánh dấu đọc tất cả → "Bạn đã xem tất cả cập nhật mới."
4. Tài khoản chưa có thông báo nào → "Chưa có thông báo".
5. Mở Mailpit → có email cho các sự kiện quan trọng (lời mời, phỏng vấn, offer).

---

## 7. Dashboard & thống kê (SYS-EMP-02, SYS-CAN-13)

### DASH-01 · Dashboard nhà tuyển dụng — **P0**
`employer@` → **Tổng quan**:
- Các thẻ số (tin đang tuyển, ứng viên mới, phỏng vấn…) phải khớp với số thực tế ở các trang danh sách.
- Biểu đồ theo ngày, phễu tuyển dụng và top tin. Đổi **Khoảng thời gian** 7/30/90 ngày → số liệu thay đổi.

### DASH-02 · Dashboard ứng viên — **P0**
`candidate@` → **Tổng quan**: số đơn, lời mời, việc đã lưu, lịch phỏng vấn sắp tới khớp với các trang tương ứng; có khối gợi ý việc.

---

## 8. Admin / Back office (SYS-ADM-01 → 05)

**Tài khoản:** `admin@demo.local`

### ADM-01 · Duyệt doanh nghiệp — **P0**
1. Menu **Xác minh doanh nghiệp** (`/admin`) → thấy `pending@` (hoặc NTD ở PROF-02) với trạng thái "Đang chờ duyệt".
2. Xem giấy phép → **Duyệt** → xác nhận ở hộp "Duyệt doanh nghiệp?" → NTD chuyển sang "Đã xác minh" (tab **Đã xác minh**).
3. **Nhánh từ chối:** bấm **Từ chối** khi chưa nhập lý do → lỗi bắt buộc. Nhập lý do rồi xác nhận → NTD thấy lý do bị từ chối.

### ADM-02 · Quản lý tài khoản — **P0**
1. Menu **Tài khoản** → ô **Tìm kiếm tài khoản**, **Lọc theo vai trò**, **Lọc theo trạng thái**, phân trang.
2. Chọn một ứng viên, ví dụ `candidate.nam@` → **Khóa tài khoản** → nhập lý do → **Khóa**.
3. Mở trình duyệt khác, đăng nhập bằng tài khoản vừa khóa → bị chặn, có thông báo tiếng Việt.
4. Admin bấm **Mở khóa** → đăng nhập lại được.
5. Bấm **Lịch sử đăng nhập** → hộp hiện danh sách lần đăng nhập.

### ADM-03 · Kiểm duyệt tin — **P0**
1. Duyệt / từ chối tin đang chờ: xem JOB-04.
2. Menu **Tin tuyển dụng** → tìm một tin đang tuyển → **Ẩn tin** → bấm xác nhận khi chưa nhập **Lý do ẩn** → lỗi. Nhập lý do → toast "Đã ẩn tin …".
3. NTD sở hữu tin: `/employer/jobs` tab **Bị ẩn** → thấy tin kèm lý do. Khách không còn thấy tin trên `/jobs`.
4. Admin **Hiển thị lại** → tin hiện công khai trở lại.
5. `/admin/settings` → bật/tắt "Kiểm duyệt tin trước khi đăng" → toast xác nhận.

### ADM-04 · Xử lý báo cáo vi phạm — P1
Menu **Báo cáo vi phạm** → báo cáo từ APPLY-03 (hoặc báo cáo có sẵn trong seed) → **Xử lý** → nhập **Ghi chú xử lý (không bắt buộc)** → **Đánh dấu đã xử lý** → "Đã đánh dấu báo cáo là đã xử lý."

### ADM-05 · Quản lý tin ghim — P1
1. `/admin/settings` → chỉnh số vị trí ghim tối đa và số ngày ghim tối đa, bấm **Lưu cấu hình** (nhập giá trị sai → lỗi) → "Đã lưu cấu hình ghim tin."
2. Menu **Tin ghim** → sửa thời gian một lượt ghim đang chạy (kết thúc quá dài → lỗi dưới ô). **Hủy ghim** → xác nhận → "Đã hủy lượt ghim của tin …".

### ADM-06 · Danh mục & kỹ năng — P1
Menu **Danh mục** → **Thêm danh mục** (thử trùng tên → lỗi), sửa tên, xóa danh mục đang được tin dùng → bị chặn, xóa danh mục trống → được. Thêm/xóa kỹ năng.
Kiểm tra: ô chọn danh mục ở form tạo tin của NTD cập nhật theo.

### ADM-07 · Thống kê & quản trị AI — P1
- Menu **Thống kê**: có "Thống kê hệ thống" với các KPI (người dùng, tin, đơn). Đổi khoảng thời gian → số liệu tải lại.
- Menu **AI**: hiển thị provider đang dùng (mock hay thật) và số lượt gọi AI.
- Đăng nhập NTD hoặc ứng viên rồi vào `/admin/analytics` → bị chặn.

---

## 9. Gói dịch vụ & thanh toán — P1

> Hiện chỉ có **cổng thanh toán giả lập** (FakePaymentGateway), chưa tích hợp VNPay/MoMo thật. Demo cần nói rõ điểm này.

### PAY-01 · Bảng giá
1. Chưa đăng nhập vào `/pricing` → thấy các gói. Bấm **Chọn gói** → yêu cầu đăng nhập.
2. Đăng nhập ứng viên vào `/pricing` → hiện "Gói dịch vụ chỉ dành cho tài khoản nhà tuyển dụng." và nút bị vô hiệu.

### PAY-02 · Mua gói thành công / thất bại
**Tài khoản:** một NTD **chưa có gói**, ví dụ `cafe@` hoặc NTD mới đăng ký. `employer@` đã có PLUS còn hạn.

1. `/pricing` → gói PLUS → **Chọn gói** → chuyển sang trang thanh toán giả lập, số tiền đúng với giá gói.
2. Bấm **Thanh toán thành công** → quay về `/payment/result` → báo thành công. Menu **Thanh toán & gói dịch vụ** → gói đang hoạt động, còn nguyên sau khi F5.
3. Làm lại với một NTD khác, bấm **Thanh toán thất bại** → báo thất bại, **không** kích hoạt gói.
4. Bấm **Quay lại cửa hàng (chưa gửi xác nhận)** → trang kết quả hiện "đang chờ xử lý", không tự kích hoạt gói.

---

## 10. Kịch bản demo gợi ý (15–20 phút)

Thứ tự đi một vòng end-to-end để trình bày:

1. **Khách** tìm việc ở `/jobs`: lọc lương, xem bản đồ (SEARCH-01, 02).
2. **NTD** `employer@` tạo tin bằng AI → Gửi duyệt (JOB-02, 04).
3. **Admin** duyệt tin → tin lên `/jobs` (JOB-04).
4. **Ứng viên** `candidate@` ứng tuyển kèm lời nhắn (APPLY-01).
5. **NTD** chuyển Đã liên hệ → Đang phỏng vấn, đặt lịch phỏng vấn online (HIRE-01, 05).
6. **Ứng viên** xác nhận lịch, tải file .ics; hỏi NTD qua chat, AI trả lời FAQ (HIRE-05, MSG-02).
7. **NTD** gửi Offer → **Ứng viên** chấp nhận → Đã tuyển (HIRE-06).
8. Cả hai xem thông báo và dashboard (NOTI-01, DASH-01/02).
9. **Admin**: duyệt doanh nghiệp `pending@`, khóa/mở khóa tài khoản, xem thống kê (ADM-01, 02, 07).

**Điều kiện coi là MVP demo được:** tất cả case **P0** đạt. Các case P1 lỗi thì ghi lại, không chặn demo.

---

## 11. Ngoài phạm vi (không cần test)

- Ứng dụng mobile.
- Quản lý hợp đồng lao động, chấm công, trả lương.
- Cổng thanh toán thật (VNPay/MoMo).
- Đồng bộ Google Calendar, tự tạo phòng Google Meet/Zoom (chỉ có link Jitsi và file .ics).

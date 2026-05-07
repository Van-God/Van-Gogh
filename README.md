# Van-Gogh
# 🏪 BÀI TẬP VỀ NHÀ 03 — Hệ Thống Quản Lý Cầm Đồ

## Thông Tin Cá Nhân

| Thông tin | Nội dung |
|---|---|
| **Họ và tên** | Hoàng Đình Hiếu |
| **Mã sinh viên** | K235480106025 |
| **Lớp** | 59KMT |
| **Môn học** | Hệ Quản Trị Cơ Sở Dữ Liệu |
| **Giảng viên** | Đỗ Duy Cốp |
| **Tên Database** | `QuanLyCamDo_K235480106025` |
| **Deadline** | 23:59:59 ngày 12/05/2026 |

---

## Yêu Cầu Đầu Bài

Xây dựng hệ thống quản lý hợp đồng vay tiền thế chấp tài sản (cầm đồ) với các đặc thù:
- Cơ chế lãi suất **2 giai đoạn**: lãi đơn trước Deadline1, lãi kép sau Deadline1
- Quản lý **danh mục tài sản thế chấp** theo từng hợp đồng
- Xử lý **thanh lý đồ** khi quá hạn Deadline2
- **Audit Log** ghi lại từng lần khách trả tiền

---

## Giới Thiệu Hệ Thống

Hệ thống `QuanLyCamDo_K235480106025` được xây dựng trên SQL Server, gồm 5 phần chính:

1. **Thiết Kế CSDL** — 4 bảng có quan hệ với đầy đủ PK, FK, CHECK
2. **Function** — Hàm tính lãi đơn, lãi kép, tổng nợ theo hợp đồng và theo khách hàng
3. **Stored Procedure** — Tiếp nhận hợp đồng, xử lý trả nợ, gia hạn
4. **Trigger** — Tự động cập nhật trạng thái hợp đồng và tài sản
5. **Truy vấn nợ xấu** — Danh sách khách quá hạn kèm dự báo 1 tháng tới

---

## Phần 1: Thiết Kế CSDL

### Sơ Đồ Quan Hệ

```
KhachHang (1) ──────────< HopDong (N)
                               │
                               ├──────< TaiSan (N)
                               │
                               └──────< LichSuThanhToan (N)
```

### Mô Tả Các Bảng

| Bảng | Mục đích | PK | FK |
|---|---|---|---|
| `KhachHang` | Lưu hồ sơ khách hàng vay tiền | MaKhachHang | — |
| `HopDong` | Hợp đồng vay tiền, lưu Deadline1/2 | MaHopDong | MaKhachHang |
| `TaiSan` | Tài sản thế chấp theo từng hợp đồng | MaTaiSan | MaHopDong |
| `LichSuThanhToan` | Audit log mỗi lần khách trả tiền | MaLichSu | MaHopDong |

### Trạng Thái Hợp Đồng

```
'Đang vay' ──(quá Deadline1)──> 'Quá hạn (nợ xấu)'
                                        │
                              (quá Deadline2)
                                        │
                                        v
'Đang trả góp' <──(trả một phần)── [Xử lý trả nợ]
     │
     └──(trả đủ)──> 'Đã thanh toán'

'Quá hạn (nợ xấu)' ──(xử lý)──> 'Đã thanh lý'
```

### Trạng Thái Tài Sản

```
'Đang cầm cố' ──(quá Deadline2)──> 'Sẵn sàng thanh lý'
                                            │
                                    (HĐ -> Đã thanh lý)
                                            │
                                            v
                                    'Đã bán thanh lý'

'Đang cầm cố' ──(trả đủ tiền)──> 'Đã trả khách'
```

### Code SQL Tạo Bảng

```sql
CREATE TABLE [KhachHang] (
    [MaKhachHang]   INT             NOT NULL IDENTITY(1,1),
    [HoTen]         NVARCHAR(150)   NOT NULL,
    [SoDienThoai]   VARCHAR(15)     NOT NULL,
    [SoCCCD]        VARCHAR(20)     NULL,
    [DiaChi]        NVARCHAR(300)   NULL,
    [NgayTao]       DATETIME        NOT NULL DEFAULT GETDATE(),
    CONSTRAINT [PK_KhachHang] PRIMARY KEY ([MaKhachHang]),
    CONSTRAINT [UQ_KhachHang_SDT] UNIQUE ([SoDienThoai])
);

CREATE TABLE [HopDong] (
    [MaHopDong]     INT             NOT NULL IDENTITY(1,1),
    [MaKhachHang]   INT             NOT NULL,
    [SoTienVay]     MONEY           NOT NULL,
    [NgayVay]       DATE            NOT NULL DEFAULT CAST(GETDATE() AS DATE),
    [Deadline1]     DATE            NOT NULL,  -- Mốc hết lãi đơn
    [Deadline2]     DATE            NOT NULL,  -- Mốc thanh lý tài sản
    [TrangThai]     NVARCHAR(30)    NOT NULL DEFAULT N'Đang vay',
    ...
    CONSTRAINT [CK_HopDong_Deadline] CHECK (
        [Deadline2] > [Deadline1] AND [Deadline1] > [NgayVay]
    )
);
```

> 📸 **[CHÈN ẢNH: Tạo database và 4 bảng thành công]**
> *Chú thích: Ảnh cho thấy 4 bảng đã được tạo thành công trong Object Explorer. Bảng HopDong có 2 cột Deadline1/Deadline2 với ràng buộc CHECK đảm bảo Deadline2 > Deadline1 > NgayVay — logic nghiệp vụ quan trọng nhất của hệ thống.*

---

## Phần 2: Cơ Chế Lãi Suất và Function

### Giải Thích Thuật Toán Tính Lãi

Hệ thống áp dụng cơ chế lãi **2 giai đoạn**:

**Giai đoạn 1 — Lãi đơn (từ NgayVay đến Deadline1):**
```
Lãi 1 ngày = Gốc × 5.000 / 1.000.000 = Gốc × 0,005
Tổng lãi   = Gốc × 0,005 × SoNgayVay
Tổng nợ    = Gốc + Tổng lãi
```

**Giai đoạn 2 — Lãi kép (sau Deadline1):**
```
GốcMới     = Gốc + (Gốc × 0,005 × SoNgayDenDeadline1)
Tổng nợ    = GốcMới × (1,005) ^ SoNgaySauDeadline1
```

**Ví dụ minh họa** — Vay 10.000.000đ, Deadline1 = ngày 30:
```
Lãi đơn đến Deadline1  = 10.000.000 × 0,005 × 30 = 1.500.000đ
GốcMới tại Deadline1   = 11.500.000đ

Sau Deadline1 thêm 10 ngày:
Tổng nợ = 11.500.000 × (1,005)^10 = 12.089.750đ
```

### fn_TinhLaiDon — Hàm hỗ trợ tính lãi đơn

```sql
CREATE FUNCTION [dbo].[fn_TinhLaiDon]
(
    @SoTienGoc  MONEY,
    @SoNgay     INT
)
RETURNS MONEY
AS
BEGIN
    RETURN @SoTienGoc * 0.005 * @SoNgay;
END;
```

> 📸 **[CHÈN ẢNH: Tạo fn_TinhLaiDon thành công]**
> *Chú thích: Hàm hỗ trợ đơn giản, được fn_CalcMoneyTransaction gọi lại để tránh lặp code.*

### fn_CalcMoneyTransaction — Tính nợ 1 hợp đồng đến ngày cụ thể

**Luồng xử lý:**
- Bước 1. Lấy thông tin hợp đồng (SoTienVay, NgayVay, Deadline1)
- Bước 2. So sánh TargetDate với Deadline1
- Bước 3. Nếu TargetDate ≤ Deadline1 → tính lãi đơn thuần
- Bước 4. Nếu TargetDate > Deadline1 → tính nợ tại Deadline1, sau đó áp lãi kép
- Bước 5. Trừ đi tổng các khoản đã trả trong LichSuThanhToan
- Bước 6. Trả về dư nợ thực tế

```sql
CREATE FUNCTION [dbo].[fn_CalcMoneyTransaction]
(
    @MaHopDong  INT,
    @TargetDate DATE
)
RETURNS MONEY
AS
BEGIN
    -- Lấy thông tin hợp đồng
    SELECT @SoTienVay=SoTienVay, @NgayVay=NgayVay, @Deadline1=Deadline1
    FROM [HopDong] WHERE [MaHopDong] = @MaHopDong;

    IF @TargetDate <= @Deadline1
    BEGIN
        -- Giai đoạn 1: Lãi đơn
        DECLARE @SoNgayDon INT = DATEDIFF(DAY, @NgayVay, @TargetDate);
        SET @TongNo = @SoTienVay + dbo.fn_TinhLaiDon(@SoTienVay, @SoNgayDon);
    END
    ELSE
    BEGIN
        -- Giai đoạn 2: Lãi kép
        DECLARE @SoNgayDenD1 INT = DATEDIFF(DAY, @NgayVay, @Deadline1);
        DECLARE @NoDiD1 MONEY =
            @SoTienVay + dbo.fn_TinhLaiDon(@SoTienVay, @SoNgayDenD1);

        DECLARE @SoNgayKep INT = DATEDIFF(DAY, @Deadline1, @TargetDate);
        SET @TongNo = @NoDiD1 * POWER(CAST(1.005 AS FLOAT), @SoNgayKep);
    END

    -- Trừ số tiền đã trả
    SELECT @DaTra = ISNULL(SUM(SoTienTra), 0)
    FROM [LichSuThanhToan]
    WHERE [MaHopDong] = @MaHopDong AND CAST(NgayTra AS DATE) <= @TargetDate;

    RETURN CAST(@TongNo - @DaTra AS MONEY);
END;
```

> 📸 **[CHÈN ẢNH: Kết quả gọi fn_CalcMoneyTransaction với hợp đồng mẫu]**
> *Chú thích: Ảnh cho thấy hàm trả về đúng số tiền nợ tính đến ngày hôm nay, đã trừ các khoản đã trả trong LichSuThanhToan.*

### fn_CalcMoneyContract — Tính tổng nợ của 1 khách hàng

Dùng CURSOR để duyệt qua tất cả hợp đồng chưa kết thúc của khách hàng và cộng dồn dư nợ:

```sql
CREATE FUNCTION [dbo].[fn_CalcMoneyContract](@MaKhachHang INT, @TargetDate DATE)
RETURNS MONEY
AS
BEGIN
    -- Dùng CURSOR vì cần gọi fn_CalcMoneyTransaction cho từng hợp đồng
    DECLARE cur_HopDong CURSOR FOR
        SELECT MaHopDong FROM HopDong
        WHERE MaKhachHang = @MaKhachHang
          AND TrangThai NOT IN (N'Đã thanh toán', N'Đã thanh lý');

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @TongNo = @TongNo + dbo.fn_CalcMoneyTransaction(@MaHD, @TargetDate);
        FETCH NEXT FROM cur_HopDong INTO @MaHD;
    END;
    RETURN @TongNo;
END;
```

> 📸 **[CHÈN ẢNH: Kết quả fn_CalcMoneyContract tổng hợp nhiều hợp đồng]**
> *Chú thích: Hàm trả về tổng dư nợ của tất cả hợp đồng đang hoạt động của 1 khách — hữu ích khi khách có nhiều lần vay.*

---

## Phần 3: Stored Procedure

### sp_TiepNhanHopDong — Đăng Ký Hợp Đồng Vay Tiền Mới

**Ý tưởng (Scenario): "Quầy tiếp nhận đồ cầm"**

Tình huống: Khách mang đồ đến cầm, nhân viên chỉ cần nhập thông tin khách hàng, danh sách tài sản (tối đa 3 món), số tiền vay và 2 mốc deadline. SP tự động tạo hoặc tìm lại hồ sơ khách hàng theo số điện thoại.

**Luồng xử lý:**
- Bước 1. Kiểm tra số tiền vay > 0, Deadline1 > NgayVay, Deadline2 > Deadline1
- Bước 2. Tìm khách hàng theo SĐT — nếu chưa có thì tạo mới
- Bước 3. INSERT vào bảng HopDong
- Bước 4. INSERT từng tài sản vào bảng TaiSan
- Bước 5. Trả về thông tin hợp đồng vừa tạo

```sql
EXEC [dbo].[sp_TiepNhanHopDong]
    @HoTen       = N'Nguyễn Văn An',
    @SoDienThoai = '0912345678',
    @SoTienVay   = 5000000,
    @NgayVay     = '2026-04-01',
    @Deadline1   = '2026-05-01',
    @Deadline2   = '2026-06-01',
    @TenTaiSan1  = N'iPhone 14 Pro',
    @GiaTri1     = 8000000,
    @TenTaiSan2  = N'Đồng hồ Casio G-Shock',
    @GiaTri2     = 2500000;
```

> 📸 **[CHÈN ẢNH: Kết quả tạo hợp đồng thành công]**
> *Chú thích: SP in ra thông báo "✅ Tạo hợp đồng thành công! Mã HĐ: 1" kèm bảng tổng hợp thông tin hợp đồng và tổng giá trị tài sản thế chấp.*

---

### sp_XuLyTraNo — Xử Lý Khi Khách Mang Tiền Đến Trả

**Ý tưởng (Scenario): "Quầy thu tiền — mỗi xu đều được ghi lại"**

Tình huống: Khách đến trả tiền (có thể trả từng phần). SP phải xử lý nhiều trường hợp: tài sản đã thanh lý, trả đủ, trả một phần — và sau đó gợi ý tài sản nào có thể trả lại cho khách ngay.

**Luồng xử lý:**
- Bước 1. Kiểm tra trạng thái hợp đồng — nếu đã thanh lý thì từ chối
- Bước 2. Tính dư nợ thực tế đến hôm nay bằng `fn_CalcMoneyTransaction`
- Bước 3. Ghi vào `LichSuThanhToan` (không ghi đè, chỉ thêm mới)
- Bước 4. Nếu trả đủ → cập nhật HĐ sang "Đã thanh toán", trả lại tất cả tài sản
- Bước 5. Nếu trả một phần → cập nhật HĐ sang "Đang trả góp"
- Bước 6. Xuất danh sách tài sản có thể trả lại dựa trên điều kiện: **Giá trị tài sản còn lại ≥ Dư nợ còn lại**

```sql
EXEC [dbo].[sp_XuLyTraNo]
    @MaHopDong   = 1,
    @SoTienTra   = 2000000,
    @NhanVienThu = N'Hoàng Đình Hiếu';
```

> 📸 **[CHÈN ẢNH: Kết quả sp_XuLyTraNo — trả một phần]**
> *Chú thích: Tab Messages hiển thị dư nợ trước/sau khi trả, gợi ý tài sản có thể trả lại với cột KhaNangTraLai = "✅ Có thể trả" hoặc "❌ Không thể trả". Logic đảm bảo tài sản còn lại luôn đủ bảo đảm cho khoản nợ.*

> 📸 **[CHÈN ẢNH: Kiểm tra LichSuThanhToan sau khi trả]**
> *Chú thích: Bảng LichSuThanhToan ghi lại đầy đủ: ngày trả, số tiền trả, dư nợ trước/sau, nhân viên thu. Mỗi lần trả là 1 dòng mới — không bao giờ ghi đè.*

---

### sp_GiaHanHopDong — Gia Hạn Hợp Đồng

**Ý tưởng (Scenario): "Khách muốn né lãi kép"**

Tình huống: Khách gần đến Deadline1 nhưng chưa đủ tiền trả gốc. Họ có thể trả toàn bộ tiền lãi tích lũy để "làm mới" hợp đồng với 2 Deadline mới — tránh bị tính lãi kép.

**Luồng xử lý:**
- Bước 1. Tính lãi tích lũy đến hôm nay = TổngNợ - Gốc
- Bước 2. Ghi log khoản trả lãi vào LichSuThanhToan
- Bước 3. Cập nhật Deadline1 và Deadline2 mới
- Bước 4. Reset trạng thái về "Đang vay"

```sql
EXEC [dbo].[sp_GiaHanHopDong]
    @MaHopDong    = 1,
    @Deadline1Moi = '2026-06-01',
    @Deadline2Moi = '2026-07-01',
    @NhanVienThu  = N'Hoàng Đình Hiếu';
```

> 📸 **[CHÈN ẢNH: Kết quả gia hạn hợp đồng]**
> *Chú thích: SP in ra tiền lãi đã thu và 2 deadline mới. Hợp đồng được reset về "Đang vay" — từ đây lại bắt đầu tính lãi đơn từ đầu.*

---

## Phần 4: Trigger — Tự Động Hóa Trạng Thái

### Trigger 1: trg_CapNhatQuaHan

**Kịch bản:** Mỗi khi có thao tác INSERT/UPDATE trên bảng `HopDong`, trigger tự động kiểm tra xem hợp đồng đó đã vượt Deadline1 chưa — nếu có thì chuyển sang "Quá hạn (nợ xấu)".

```sql
CREATE TRIGGER [trg_CapNhatQuaHan]
ON [HopDong]
AFTER INSERT, UPDATE
AS
BEGIN
    UPDATE [HopDong]
    SET [TrangThai] = N'Quá hạn (nợ xấu)'
    WHERE [TrangThai] = N'Đang vay'
      AND [Deadline1] < CAST(GETDATE() AS DATE)
      AND [MaHopDong] IN (SELECT MaHopDong FROM inserted);
END;
```

> 📸 **[CHÈN ẢNH: Trigger tự động cập nhật TrangThai khi vượt Deadline1]**
> *Chú thích: Ảnh cho thấy sau khi tạo hợp đồng mẫu với Deadline1 trong quá khứ, trigger kích hoạt ngay và cập nhật trạng thái sang "Quá hạn (nợ xấu)" trong cùng transaction.*

---

### Trigger 2: trg_CapNhatSanSangThanhLy

**Kịch bản:** Khi hợp đồng chuyển sang "Quá hạn (nợ xấu)" **và** đã vượt Deadline2, trigger tự động chuyển các tài sản còn đang cầm cố sang "Sẵn sàng thanh lý".

> 📸 **[CHÈN ẢNH: Trigger chuyển tài sản sang Sẵn sàng thanh lý]**
> *Chú thích: Ảnh kiểm tra bảng TaiSan sau khi hợp đồng quá Deadline2 — cột TrangThai của tài sản tự động đổi từ "Đang cầm cố" sang "Sẵn sàng thanh lý" mà không cần can thiệp thủ công.*

---

### Trigger 3: trg_CapNhatDaBanThanhLy

**Kịch bản:** Khi nhân viên cập nhật hợp đồng thành "Đã thanh lý" (sau khi bán xong đồ), trigger tự động chuyển tất cả tài sản liên quan sang "Đã bán thanh lý".

**Phân tích chuỗi Trigger:**

```
Nhân viên UPDATE HopDong → 'Đã thanh lý'
    │
    └──> trg_CapNhatDaBanThanhLy kích hoạt
             │
             └──> TaiSan.TrangThai -> 'Đã bán thanh lý'
```

> 📸 **[CHÈN ẢNH: Chuỗi trigger hoạt động khi thanh lý tài sản]**
> *Chú thích: Tab Messages hiển thị "[Trigger 3]: Đã chuyển tài sản sang Đã bán thanh lý". Hệ thống 3 trigger hoạt động phối hợp để tự động quản lý vòng đời của hợp đồng và tài sản.*

---

## Phần 5: Truy Vấn Nợ Xấu (Event 4)

### fn_DanhSachNoXau — Danh Sách Khách Quá Hạn

**Ý tưởng:** Ban giám đốc cần xem hàng ngày danh sách các hợp đồng đã vượt Deadline1 nhưng chưa thanh toán, kèm theo **dự báo số tiền nợ sau 1 tháng nữa** để có phương án xử lý.

```sql
SELECT * FROM dbo.[fn_DanhSachNoXau]()
ORDER BY [SoNgayQuaHan] DESC;
```

| Cột | Ý nghĩa |
|---|---|
| TenKhachHang | Tên khách hàng |
| SoDienThoai | Số điện thoại liên lạc |
| SoTienVayGoc | Số tiền vay ban đầu |
| SoNgayQuaHan | Số ngày đã quá Deadline1 |
| TongTienPhaiTraHienTai | Dư nợ tính đến hôm nay (có lãi kép) |
| TongTienPhaiTraSau1Thang | Dự báo dư nợ sau 30 ngày nữa |

> 📸 **[CHÈN ẢNH: Kết quả fn_DanhSachNoXau]**
> *Chú thích: Ảnh cho thấy danh sách khách quá hạn được sắp xếp theo số ngày quá hạn giảm dần. Cột TongTienPhaiTraSau1Thang giúp nhân viên ước lượng mức độ rủi ro tăng thêm nếu không xử lý sớm — lãi kép khiến con số tăng nhanh hơn nhiều so với lãi đơn.*

---

## Phần 6: Dữ Liệu Mẫu và Demo

### Các hợp đồng mẫu đã tạo

| Mã HĐ | Khách hàng | Số tiền vay | Trạng thái | Tình huống |
|---|---|---|---|---|
| 1 | Nguyễn Văn An | 5.000.000đ | Đang vay | Hợp đồng bình thường, chưa quá hạn |
| 2 | Trần Thị Bình | 10.000.000đ | Quá hạn | Đã vượt Deadline1, đang tính lãi kép |
| 3 | Lê Minh Cường | 2.000.000đ | Đang vay | Hợp đồng nhỏ, 1 tài sản |
| 4 | Phạm Thị Dung | 15.000.000đ | Quá hạn | Đã vượt Deadline2, tài sản sẵn sàng thanh lý |

> 📸 **[CHÈN ẢNH: Bảng HopDong và TaiSan sau khi INSERT dữ liệu mẫu]**
> *Chú thích: Ảnh cho thấy 4 hợp đồng mẫu với đầy đủ tài sản thế chấp. Hợp đồng 2 và 4 đã tự động được Trigger cập nhật sang "Quá hạn (nợ xấu)" ngay khi INSERT vì Deadline1 đã trong quá khứ.*

> 📸 **[CHÈN ẢNH: Kết quả Demo trả nợ hợp đồng 1]**
> *Chú thích: Sau khi gọi sp_XuLyTraNo với 2.000.000đ cho hợp đồng 1, bảng LichSuThanhToan có thêm 1 dòng ghi lại khoản trả. Hợp đồng chuyển sang "Đang trả góp".*

> 📸 **[CHÈN ẢNH: Kết quả truy vấn danh sách nợ xấu]**
> *Chú thích: fn_DanhSachNoXau trả về hợp đồng 2 và 4 — cả 2 đều đã quá Deadline1. Cột TongTienPhaiTraSau1Thang cho thấy số tiền tăng đáng kể do lãi kép, tạo áp lực để khách hàng thanh toán sớm.*

---

## Tổng Kết

| Phần | Nội dung | Trạng thái |
|---|---|---|
| Phần 1 | 4 bảng (KhachHang, HopDong, TaiSan, LichSuThanhToan) | ✅ |
| Phần 2 | fn_TinhLaiDon + fn_CalcMoneyTransaction + fn_CalcMoneyContract | ✅ |
| Phần 3 | sp_TiepNhanHopDong + sp_XuLyTraNo + sp_GiaHanHopDong | ✅ |
| Phần 4 | 3 Trigger tự động quản lý trạng thái HĐ và tài sản | ✅ |
| Phần 5 | fn_DanhSachNoXau — truy vấn nợ xấu kèm dự báo | ✅ |
| Phần 6 | 4 hợp đồng mẫu + demo các nghiệp vụ chính | ✅ |

> **File script SQL đầy đủ:** [`baikiemtra3.sql`](./baikiemtra3.sql)

| [script.sql](./script.sql) | Toàn bộ cấu trúc bảng + dữ liệu mẫu + SP + Function + Trigger |
| [BaoCao_ERD.pdf](./BaoCao_ERD.pdf) | Sơ đồ ERD và giải thích thuật toán tính lãi |

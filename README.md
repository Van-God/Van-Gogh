# Van-Gogh
# Bài tập 03: Thiết kế và Cài đặt CSDL Quản lý Cầm Đồ

**Môn học:** Hệ quản trị Cơ sở Dữ liệu – TEE560  
**Lớp:** 59KMT  
**Giảng viên:** Đỗ Duy Cốp  
**Họ tên:** Hoàng [Tên của bạn]  
**Mã SV:** [Mã SV của bạn]  
**Deadline:** 23:59:59 ngày 12/05/2026  

---

## 1. Mô tả bài toán

Hệ thống quản lý các hợp đồng vay tiền thế chấp tài sản (cầm đồ).  
Đặc điểm chính:
- Một khách hàng có thể có nhiều hợp đồng cầm cố
- Một hợp đồng có thể bao gồm nhiều tài sản thế chấp
- Cơ chế tính lãi 2 giai đoạn: **lãi đơn** (trước Deadline1) và **lãi kép** (sau Deadline1)
- Quản lý trạng thái tài sản và xử lý thanh lý khi quá hạn

---

## 2. Sơ đồ ERD

> Vẽ trên draw.io — thể hiện đầy đủ thực thể, thuộc tính, khóa chính (PK), khóa ngoại (FK)

[PASTE ẢNH SƠ ĐỒ ERD VÀO ĐÂY]

### Các thực thể và mối quan hệ

| Thực thể | Vai trò | Quan hệ |
|---|---|---|
| `KhachHang` | Người đến cầm đồ | 1 KH → nhiều HĐ |
| `HopDong` | Hợp đồng vay tiền | 1 HĐ → nhiều TS |
| `TaiSan` | Tài sản thế chấp | N:M với HĐ |
| `HopDong_TaiSan` | Bảng trung gian (N:M) | Lưu giá trị định giá |
| `LichSuGiaoDich` | Audit log mỗi lần trả | 1 HĐ → nhiều GD |
| `NhanVien` | Người thu tiền | 1 NV → nhiều GD |

---

## 3. Thiết kế bảng (chuẩn 3NF)

### Bảng `KhachHang`
| Cột | Kiểu | Ràng buộc |
|---|---|---|
| KhachHangID | INT | PK, IDENTITY |
| HoTen | NVARCHAR(100) | NOT NULL |
| SoDienThoai | VARCHAR(15) | |
| CMND | VARCHAR(20) | |
| DiaChi | NVARCHAR(200) | |
| NgayTao | DATETIME | DEFAULT GETDATE() |

### Bảng `HopDong`
| Cột | Kiểu | Ràng buộc |
|---|---|---|
| HopDongID | INT | PK, IDENTITY |
| KhachHangID | INT | FK → KhachHang |
| SoTienVay | DECIMAL(18,0) | NOT NULL |
| NgayVay | DATE | NOT NULL |
| Deadline1 | DATE | NOT NULL |
| Deadline2 | DATE | NOT NULL |
| TrangThai | NVARCHAR(50) | DEFAULT 'Đang vay' |
| GhiChu | NVARCHAR(500) | |

### Bảng `TaiSan`
| Cột | Kiểu | Ràng buộc |
|---|---|---|
| TaiSanID | INT | PK, IDENTITY |
| TenTaiSan | NVARCHAR(200) | NOT NULL |
| MoTa | NVARCHAR(500) | |
| TrangThai | NVARCHAR(50) | DEFAULT 'Đang cầm cố' |

### Bảng `HopDong_TaiSan`
| Cột | Kiểu | Ràng buộc |
|---|---|---|
| HopDongID | INT | PK, FK → HopDong |
| TaiSanID | INT | PK, FK → TaiSan |
| GiaTriDinhGia | DECIMAL(18,0) | NOT NULL |

### Bảng `LichSuGiaoDich`
| Cột | Kiểu | Ràng buộc |
|---|---|---|
| GiaoDichID | INT | PK, IDENTITY |
| HopDongID | INT | FK → HopDong |
| NhanVienID | INT | FK → NhanVien |
| NgayGiaoDich | DATETIME | DEFAULT GETDATE() |
| SoTienTra | DECIMAL(18,0) | NOT NULL |
| SoTienConNo | DECIMAL(18,0) | |
| LoaiGiaoDich | NVARCHAR(50) | |
| GhiChu | NVARCHAR(500) | |

### Bảng `NhanVien`
| Cột | Kiểu | Ràng buộc |
|---|---|---|
| NhanVienID | INT | PK, IDENTITY |
| HoTen | NVARCHAR(100) | NOT NULL |
| SoDienThoai | VARCHAR(15) | |
| ChucVu | NVARCHAR(50) | |

---

## 4. Thuật toán tính lãi

### Thông số cơ bản
- Lãi suất: **5.000đ / 1.000.000đ gốc / ngày = 0,5%/ngày**

### Giai đoạn 1 — Lãi đơn (trước Deadline1)

```
Lãi đơn = Gốc × 0,5% × Số ngày
Tổng nợ = Gốc + Lãi đơn
```

**Ví dụ:** Vay 10.000.000đ, sau 30 ngày (chưa tới Deadline1):
```
Lãi = 10.000.000 × 0,005 × 30 = 1.500.000đ
Tổng nợ = 11.500.000đ
```

### Giai đoạn 2 — Lãi kép (sau Deadline1)

```
Gốc mới = Gốc + Lãi đơn tích lũy đến Deadline1
Tổng nợ = Gốc mới × (1 + 0,005)^n
          (n = số ngày sau Deadline1)
```

**Ví dụ:** Sau Deadline1 thêm 14 ngày:
```
Gốc mới = 11.500.000đ
Tổng nợ = 11.500.000 × (1,005)^14 ≈ 12.329.610đ
```

[PASTE ẢNH BIỂU ĐỒ THUẬT TOÁN TÍNH LÃI VÀO ĐÂY]

---

## 5. Cài đặt SQL

### 5.1 Tạo Database và các bảng

[PASTE ẢNH CHỤP MÀN HÌNH SSMS SAU KHI CHẠY CREATE TABLE VÀO ĐÂY]

```sql
CREATE DATABASE QuanLyCamDo;
GO
USE QuanLyCamDo;
-- Xem file script.sql để biết toàn bộ lệnh tạo bảng
```

### 5.2 Event 1 — Stored Procedure tiếp nhận hợp đồng mới

**Mô tả:** SP `sp_TiepNhanHopDong` nhận thông tin khách hàng, danh sách tài sản (kèm giá trị định giá), số tiền vay và thiết lập Deadline1, Deadline2. Có kiểm tra điều kiện tổng giá trị tài sản >= số tiền vay.

[PASTE ẢNH CHỤP KẾT QUẢ CHẠY SP THÀNH CÔNG VÀO ĐÂY]

```sql
EXEC sp_TiepNhanHopDong
    @KhachHangID = 1,
    @SoTienVay   = 10000000,
    @NgayVay     = '2026-04-01',
    @Deadline1   = '2026-05-01',
    @Deadline2   = '2026-06-01',
    @TenTaiSan1  = N'iPhone 14 Pro Max',
    @MoTaTaiSan1 = N'256GB, màu đen',
    @GiaTriTS1   = 18000000;
```

### 5.3 Event 2 — Function tính công nợ

**Mô tả:** Hàm `fn_CalcMoneyContract(HopDongID, TargetDate)` tính tổng số tiền phải trả đến ngày TargetDate, áp dụng đúng 2 giai đoạn lãi đơn/lãi kép.

[PASTE ẢNH CHỤP KẾT QUẢ CHẠY FUNCTION VÀO ĐÂY]

```sql
-- Kiểm tra tiền phải trả hôm nay
SELECT dbo.fn_CalcMoneyContract(1, CAST(GETDATE() AS DATE)) AS TienPhaiTra;

-- Kiểm tra tiền phải trả sau 1 tháng
SELECT dbo.fn_CalcMoneyContract(1, DATEADD(MONTH,1,GETDATE())) AS TienSau1Thang;
```

### 5.4 Event 3 — SP xử lý trả nợ và hoàn trả tài sản

**Mô tả:** SP `sp_XuLyTraNo` kiểm tra tài sản còn hay đã thanh lý → tính tổng nợ → ghi nhận vào log → gợi ý tài sản trả lại nếu điều kiện thoả mãn.

[PASTE ẢNH CHỤP KẾT QUẢ CHẠY SP TRẢ NỢ VÀO ĐÂY]

```sql
EXEC sp_XuLyTraNo
    @HopDongID  = 1,
    @SoTienTra  = 5000000,
    @NhanVienID = 1;
```

### 5.5 Event 4 — Query danh sách nợ xấu

**Mô tả:** Truy vấn xuất danh sách khách hàng quá Deadline1 chưa thanh toán, kèm tổng tiền phải trả hiện tại và sau 1 tháng.

[PASTE ẢNH CHỤP KẾT QUẢ QUERY NỢ XẤU VÀO ĐÂY]

```sql
SELECT
    kh.HoTen,
    kh.SoDienThoai,
    hd.SoTienVay,
    DATEDIFF(DAY, hd.Deadline1, GETDATE()) AS SoNgayQuaHan,
    dbo.fn_CalcMoneyContract(hd.HopDongID, CAST(GETDATE() AS DATE)) AS TienPhaiTraHienTai,
    dbo.fn_CalcMoneyContract(hd.HopDongID, DATEADD(MONTH,1,CAST(GETDATE() AS DATE))) AS TienSau1Thang
FROM HopDong hd
INNER JOIN KhachHang kh ON hd.KhachHangID = kh.KhachHangID
WHERE CAST(GETDATE() AS DATE) > hd.Deadline1
  AND hd.TrangThai NOT IN (N'Đã thanh toán', N'Đã thanh lý');
```

### 5.6 Event 5 — Trigger tự động cập nhật trạng thái

**Mô tả:** 3 trigger tự động:
- `trg_HopDong_QuaHan`: HĐ → "Quá hạn" khi vượt Deadline1
- `trg_TaiSan_SanSangThanhLy`: TS → "Sẵn sàng thanh lý" khi HĐ quá Deadline2
- `trg_TaiSan_DaBanThanhLy`: TS → "Đã bán thanh lý" khi HĐ → "Đã thanh lý"

[PASTE ẢNH CHỤP MÀN HÌNH TẠO TRIGGER THÀNH CÔNG VÀO ĐÂY]

```sql
-- Test trigger: cập nhật trạng thái HĐ và xem TS tự đổi
UPDATE HopDong SET TrangThai = N'Đã thanh lý' WHERE HopDongID = 2;
SELECT TaiSanID, TenTaiSan, TrangThai FROM TaiSan;
```

---

## 6. Dữ liệu mẫu (Sample Data)

[PASTE ẢNH CHỤP DỮ LIỆU MẪU TRONG SSMS VÀO ĐÂY]

Dữ liệu mẫu gồm:
- **3 nhân viên** (thu ngân, quản lý)
- **4 khách hàng** với các trạng thái hợp đồng khác nhau
- **6 tài sản** thế chấp
- **4 hợp đồng** (đang vay, quá hạn, trả góp, đã thanh toán)
- **2 giao dịch** trong lịch sử

---

## 7. Kiểm tra kết quả

### Xem toàn bộ hợp đồng + tiền phải trả hiện tại
[PASTE ẢNH KẾT QUẢ VÀO ĐÂY]

### Xem tài sản theo hợp đồng
[PASTE ẢNH KẾT QUẢ VÀO ĐÂY]

### Xem lịch sử giao dịch
[PASTE ẢNH KẾT QUẢ VÀO ĐÂY]

---

## 8. File đính kèm

| File | Mô tả |
|---|---|
| [script.sql](./script.sql) | Toàn bộ cấu trúc bảng + dữ liệu mẫu + SP + Function + Trigger |
| [BaoCao_ERD.pdf](./BaoCao_ERD.pdf) | Sơ đồ ERD và giải thích thuật toán tính lãi |

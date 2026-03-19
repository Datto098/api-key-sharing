# Shopee Ads Tool - Database Schema (MongoDB)

Tài liệu này cung cấp cái nhìn chi tiết nhất về cấu trúc dữ liệu, mục đích sử dụng và ý nghĩa của từng Collection trong hệ thống Shopee Ads Tool.

---

## 1. Hệ thống Định danh & Kết nối (Auth & Shops)

### `User` (Thông tin người dùng Dashboard)
*   **Mục đích**: Lưu trữ tài khoản Admin để truy cập vào giao diện web điều khiển.
*   **Sử dụng cho tính năng**: Đăng nhập (Login), Đăng ký (Register), Phân quyền người dùng, Quản lý phiên làm việc (Session Management).

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `name` | String | Tên hiển thị của người quản trị trên giao diện. |
| `email` | String | Địa chỉ email dùng làm ID đăng nhập (Duy nhất). |
| `password` | String | Mật khẩu đã được mã hóa (Hash Bcrypt) để đảm bảo bảo mật. |
| `refreshToken`| String | Dùng để cấp lại mã Access Token mới khi phiên làm việc cũ hết hạn. |
| `avatar` | String | Đường dẫn đến ảnh đại diện người dùng (Mặc định: null). |
| `createdAt` | Date | Thời điểm người dùng này được tạo ra trên hệ thống. |

### `Shop` (Kết nối Cửa hàng Shopee)
*   **Mục đích**: Lưu trữ thông tin định danh và "chìa khóa" (Tokens) để Tool có quyền thay mặt chủ shop gọi vào Shopee API.
*   **Sử dụng cho tính năng**: Kết nối shop (Auth v2.0), Tự động làm mới Token (Auto-refresh), Bật/Tắt tính năng đồng bộ dữ liệu cho từng Shop cụ thể.

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `shop_id` | Number | ID duy nhất của Shop do Shopee cấp (Dùng trong mọi cuộc gọi API). |
| `shop_name` | String | Tên Shop lấy từ Shopee để hiển thị trên Dashboard. |
| `region` | String | Mã vùng của Shop (VD: VN cho Việt Nam). |
| `access_token`| String | Mã tạm thời để gọi API (Hết hạn sau mỗi 4 tiếng). |
| `refresh_token`| String | Mã dùng để lấy lại access_token mới (Hết hạn sau 30 ngày). |
| `expire_in` | Number | Thời gian sống của token tính bằng giây. |
| `token_expiry_date`| Date | Thời điểm chính xác mà access_token sẽ bị vô hiệu lực. |
| `is_active` | Boolean | Nếu `false`, hệ thống sẽ ngừng cào dữ liệu và ngừng chạy Rule cho shop này. |
| `updated_at` | Date | Ghi lại thời điểm thông tin shop hoặc token được cập nhật lần cuối. |

---

## 2. Quản lý Sản phẩm & Cấu hình Tối ưu

### `Campaign` (Thông tin Chiến dịch)
*   **Mục đích**: Lưu trữ trạng thái và cấu hình hiện tại của các mẫu quảng cáo trên Shopee.
*   **Sử dụng cho tính năng**: Lọc chiến dịch trên Dashboard, Hiển thị tên/loại quảng cáo, Cung cấp dữ liệu `item_id` để khớp với giá vốn (COGS).

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `shop_id` | Number | Liên kết chiến dịch này với Shop nào. |
| `campaign_id` | Number | ID chiến dịch do Shopee cấp. |
| `campaign_name`| String | Tên chiến dịch. |
| `type` | String | Loại quảng cáo: `gms` (Shop Ads tự động) / `manual` (Liên quan đến từ khóa). |
| `status` | String | Trạng thái hiện tại trên Shopee (Đang chạy, Đã dừng, Hết ngân sách). |
| `daily_budget` | Number | Ngân sách giới hạn tối đa mỗi ngày. |
| `item_id` | Number | ID sản phẩm đại diện cho chiến dịch này (Dùng để tính Profit). |
| `last_synced` | Date | Lần cuối Tool cập nhật metadata của chiến dịch này từ Shopee. |

### `Cogs` (Giá vốn hàng bán)
*   **Mục đích**: Lưu trữ chi phí nhập hàng cho từng sản phẩm.
*   **Sử dụng cho tính năng**: Tính toán **Lợi nhuận (Profit)** = (Doanh thu - Chi phí Ads) - (Số đơn x Giá vốn). Để biết Shop đang "Lãi" hay "Lỗ" thật sự.

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `item_id` | Number | ID sản phẩm (Dùng để khớp với dữ liệu Ads). |
| `model_id` | Number | ID của phân loại (Size, Màu sắc) - mặc định 0 nếu không tách lẻ. |
| `item_name` | String | Tên sản phẩm để người dùng dễ nhận biết khi nhập giá vốn. |
| `cogs` | Number | Giá vốn nhập hàng (Đơn vị: VND). |
| `updated_at` | Date | Ngày cập nhật giá vốn gần nhất. |

---

## 3. Hệ thống Báo cáo & Caching (Module A - Dashboard)

### `Performance` (Dữ liệu Hiệu quả Lịch sử)
*   **Mục đích**: Lưu trữ các chỉ số kinh doanh theo từng ngày/giờ để phục vụ phân tích dài hạn.
*   **Sử dụng cho tính năng**: Vẽ biểu đồ đường (Charts) trên Dashboard, Báo cáo theo tuần/tháng, So sánh hiệu quả giữa các chiến dịch trong quá khứ. Cào mỗi 30 phút.

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `shop_id` | Number | ID Shop. |
| `campaign_id` | Number | ID Chiến dịch. |
| `campaign_name`| String | Tên chiến dịch tại thời điểm đồng bộ. |
| `date` | String | Ngày ghi nhận dữ liệu (Định dạng: `dd-mm-yyyy`). |
| `hour` | Number | Giờ ghi nhận dữ liệu (Dùng cho biểu đồ theo giờ). |
| `impression` | Number | Số lượt quảng cáo hiển thị tới khách hàng. |
| `clicks` | Number | Số lượt khách nhấp chuột vào quảng cáo. |
| `spend` | Number | Số tiền Shopee đã trừ vào tài khoản quảng cáo. |
| `conversion` | Number | Số đơn hàng (Tính cả Broad Match). |
| `gmv` | Number | Tổng giá trị đơn hàng (Doanh thu Ads). |
| `orders` | Number | Số lượng đơn hàng trực tiếp. |
| `roas` | Number | Hiệu quả chi phí (GMV / Spend). |
| `last_updated` | Date | Thời điểm chính xác Tool cào dữ liệu này về máy. |

### `AdsRealtime` (Dữ liệu "Nóng" - Đồng bộ mỗi 5 Phút)
*   **Mục đích**: Lưu trữ duy nhất số liệu **của ngày hôm nay** với tốc độ cập nhật siêu nhanh.
*   **Sử dụng cho tính năng**:
    1.  Hiển thị Tab "Hôm nay" trên Dashboard với độ trễ thấp nhất.
    2.  Làm căn cứ để **Automation Engine** ra quyết định tắt/mở quảng cáo ngay lập tức (Real-time Action).

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `shop_id` | Number | ID Shop. |
| `campaign_id` | Number | ID Chiến dịch. |
| `campaign_name`| String | Tên chiến dịch. |
| `impression_today`| Number | Tổng lượt hiển thị tính từ 0h sáng hôm nay. |
| `clicks_today` | Number | Tổng lượt Click tính từ 0h sáng. |
| `spend_today` | Number | Tổng tiền tiêu tính từ 0h sáng. |
| `rev_ads_today` | Number | Tổng doanh thu Ads tính từ 0h sáng. |
| `orders_today` | Number | Tổng số đơn hàng tính từ 0h sáng. |
| `synced_at` | Date | Thời điểm cập nhật cuối cùng (Cứ mỗi 5p cào 1 lần). |

### `StorePerformance` (Doanh thu Tổng thể Shop)
*   **Mục đích**: Lưu trữ tổng doanh thu (Ads + Tự nhiên) theo từng ngày của toàn Shop.
*   **Sử dụng cho tính năng**: Tính toán **Doanh thu tự nhiên (Organic Revenue)** = Doanh thu tổng - Doanh thu Ads. Giúp đo lường sức khỏe toàn diện của Shop.

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `shop_id` | Number | ID Shop. |
| `date` | String | Ngày ghi nhận dữ liệu (`dd-mm-yyyy`). |
| `rev_total` | Number | Tổng doanh số toàn shop (Lấy từ Order API). |
| `order_total` | Number | Tổng số đơn hàng toàn shop. |
| `last_updated` | Date | Thời điểm cập nhật cuối cùng. |

---

## 4. Hệ thống Tự động hóa (Module B)

### `AutomationRule` (Danh sách các Kịch bản Tự động)
*   **Mục đích**: Lưu trữ "bộ não" của hệ thống - các quy tắc do người dùng cài đặt.
*   **Sử dụng cho tính năng**: Động cơ Automation Engine sẽ quét qua bảng này để biết phải tối ưu quảng cáo nào.

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `shop_id` | Number | Quy tắc này áp dụng cho Shop nào. |
| `name` | String | Tên gợi nhớ của quy tắc. |
| `description` | String | Mô tả rule này dùng để làm gì. |
| `conditions` | Array | Các điều kiện kỹ thuật (VD: `roas < 1.0`, `spend > 200k`). |
| `action` | Object | Hành động thực hiện (VD: `pause_ad`, `update_budget`). |
| `isDryRun` | Boolean | Chạy thử: Hệ thống chỉ ghi log báo cáo, KHÔNG tắt/mở thật trên Shopee. |
| `isActive` | Boolean | Bật/Tắt quy tắc này. |
| `lastExecutedAt`| Date | Thời điểm gần nhất rule này thực hiện kiểm soát ads. |
| `createdAt` | Date | Ngày tạo rule. |
| `updatedAt` | Date | Ngày sửa đổi rule cuối cùng. |

### `ActionLog` (Nhật ký Thực thi Quảng cáo)
*   **Mục đích**: Ghi lại lịch sử hoạt động của Automation Engine.
*   **Sử dụng cho tính năng**: Audit (Kiểm soát), báo cáo hành động qua Telegram, truy vết xem tại sao một quảng cáo lại bị Tắt hoặc Tăng ngân sách.

| Trường (Field) | Kiểu dữ liệu | Ý nghĩa & Chức năng |
| :--- | :--- | :--- |
| `rule_id` | ObjectId | Rule nào đã thực hiện hành động này. |
| `shop_id` | Number | Shop bị tác động. |
| `campaign_id` | Number | Chiến dịch bị Tool thay đổi ngân sách/trạng thái. |
| `campaign_name`| String | Tên chiến dịch tại thời điểm đó. |
| `metric_snapshot`| Object | "Ảnh chụp" các chỉ số (ROAS, Spend...) ngay lúc đó để giải thích lý do thực thi. |
| `action_type` | String | Hành động đã làm: Bật, Tắt, hay Thay đổi ngân sách. |
| `action_value`| Mixed | Giá trị mới (VD: Ngân sách tăng lên 150.000đ). |
| `status` | String | Trạng thái thực hiện: `success` (Thành công), `error` (Thất bại). |
| `message` | String | Thông báo chi tiết lỗi hoặc lý do nếu bỏ qua (Skipped). |
| `api_response` | Object | Toàn bộ phản hồi từ Shopee API (Phục vụ kỹ thuật Fix lỗi). |
| `createdAt` | Date | Giờ thực hiện hành động. |

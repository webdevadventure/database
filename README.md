# database
### Dự kiến chia schema:

- **Schema**: Các bảng được nhóm thành các schema riêng biệt theo chức năng: `auth` (xác thực), `location` (địa điểm), `listing` (bài đăng), `transaction` (giao dịch), `content` (nội dung), và `admin` (quản trị).
- **Mục tiêu**: Tối ưu hóa cấu trúc, cải thiện bảo mật thông qua phân quyền schema, và đảm bảo tính toàn vẹn dữ liệu bằng cách sử dụng khóa chính, khóa ngoại, và kiểu dữ liệu phù hợp.

---

## Thiết Kế Chi Tiết

### 1. Schema `auth`

**Mục đích**: Quản lý thông tin người dùng và xác thực.

- **Bảng `users`**:
    - `user_id` (PK, serial): Khóa chính, tự động tăng.
    - `name` (text): Tên người dùng.
    - `email` (text, unique): Email duy nhất.
    - `phone` (text): Số điện thoại.
    - `password_hash` (text): Mật khẩu đã mã hóa.
    - `user_type` (enum: 'landlord', 'tenant'): Loại người dùng.
    - `kyc_status` (enum: 'pending', 'verified', 'rejected'): Trạng thái xác minh danh tính.
    - `deleted` (boolean, default false): Đánh dấu xóa mềm.
- **Bảng `landlords`**:
    - `landlord_id` (PK, FK, references `users.user_id`): Khóa ngoại liên kết với `users`.
    - `bank_info` (jsonb): Thông tin ngân hàng, dùng kiểu `jsonb` để linh hoạt.
    - `average_rating` (numeric(2,1)): Điểm đánh giá trung bình (ví dụ: 4.5).
    - `number_of_reviews` (integer): Số lượng đánh giá.
- **Bảng `tenants`**
    - `tenant_id` (PK, FK, references `users.user_id`): Khóa ngoại tham chiếu users.
    - `rental_history` (jsonb): Lịch sử thuê nhà (ví dụ: danh sách các listing_id đã thuê).

### 2. Schema `location`

**Mục đích**: Quản lý thông tin địa điểm theo cấp độ tỉnh, quận, phường, đường.

- **Bảng `provinces`**:
    - `province_id` (PK, serial): Mã tỉnh.
    - `name` (text): Tên tỉnh.
    - `zip_code` (integer): mã bư cục
- Bảng cities:
    - `city_id` (PK_serial): Mã thành phố
    - `name` (text): Tên thành phố
    - `province_id` (FK, references `provinces.province_id`): Liên kết với tỉnh.
- **Bảng `districts`**:
    - `district_id` (PK, serial): Mã quận.
    - `city_id` (FK_references `cties.city_id`): Liên kết thành phố
    - `name` (text): Tên quận.
- **Bảng `wards`**:
    - `ward_id` (PK, serial): Mã phường.
    - `district_id` (FK, references `districts.district_id`): Liên kết với quận.
    - `name` (text): Tên phường.
- **Bảng `streets`**:
    - `street_id` (PK, serial): Mã đường.
    - `ward_id` (FK, references `wards.ward_id`): Liên kết với phường.
    - `name` (text): Tên đường.

### 3. Schema `listing`

**Mục đích**: Quản lý bài đăng phòng trọ và các thông tin liên quan.

- **Bảng `listings`**:
    - `listing_id` (PK, serial): Mã bài đăng.
    - `landlord_id` (FK, references `auth.users.user_id`): Chủ nhà đăng bài.
    - `title` (text): Tiêu đề bài đăng.
    - `description` (text): Mô tả chi tiết.
    - `price` (numeric(12,2)): Giá thuê (ví dụ: 5,500,000 VNĐ).
    - `property_type` (enum: 'room', 'apartment', 'house'): Loại bất động sản.
    - `province_id` (FK, references `location.provinces.province_id`): Tỉnh.
    - `district_id` (FK, references `location.districts.district_id`): Quận.
    - `ward_id` (FK, references `location.wards.ward_id`): Phường.
    - `street_id` (FK, references `location.streets.street_id`): Đường.
    - `specific_address` (text): Địa chỉ cụ thể (ví dụ: số nhà).
    - `status` (enum: 'pending', 'approved', 'rejected'): Trạng thái bài đăng.
    - `posting_date` (timestamp): Ngày đăng bài.
    - `deleted` (boolean, default false): Đánh dấu xóa mềm.
- **Bảng `listing_images`**:
    - `image_id` (PK, serial): Mã ảnh.
    - `listing_id` (FK, references `listings.listing_id`): Liên kết với bài đăng.
    - `image_url` (text): Đường dẫn ảnh.
- **Bảng `reviews`**:
    - `review_id` (PK, serial): Mã đánh giá.
    - `tenant_id` (FK, references `auth.users.user_id`): Người thuê đánh giá.
    - `listing_id` (FK, references `listings.listing_id`): Bài đăng được đánh giá.
    - `review_text` (text): Nội dung đánh giá.
    - `rating` (integer, check (rating >= 1 AND rating <= 5)): Điểm số từ 1-5.
    - `review_date` (timestamp): Ngày đánh giá.

### 4. Schema `transaction`

**Mục đích**: Quản lý giao dịch thanh toán.

- **Bảng `payment_methods`**:
    - `method_id` (PK, serial): Mã phương thức thanh toán.
    - `name` (text): Tên phương thức (ví dụ: "Chuyển khoản", "Tiền mặt").
- **Bảng `transactions`**:
    - `transaction_id` (PK, serial): Mã giao dịch.
    - `sender_id` (FK, references `auth.users.user_id`): Người gửi tiền.
    - `receiver_id` (FK, references `auth.users.user_id`): Người nhận tiền.
    - `listing_id` (FK, references `listing.listings.listing_id`): Bài đăng liên quan.
    - `amount` (numeric(12,2)): Số tiền.
    - `status` (enum: 'pending', 'completed', 'failed'): Trạng thái giao dịch.
    - `transaction_date` (timestamp): Thời gian giao dịch.
    - `transaction_type` (enum: 'deposit', 'payment'): Loại giao dịch.
    - `method_id` (FK, references `payment_methods.method_id`): Phương thức thanh toán.

### 5. Schema `blog`

**Mục đích**: Quản lý nội dung blog và bình luận.

- **Bảng `categories`**:
    - `category_id` (PK, serial): Mã danh mục.
    - `name` (text): Tên danh mục (ví dụ: "Tin tức", "Mẹo thuê nhà").
- **Bảng `blog_posts`**:
    - `post_id` (PK, serial): Mã bài viết.
    - `author_id` (FK, references `auth.users.user_id`): Tác giả.
    - `title` (text): Tiêu đề bài viết.
    - `content` (text): Nội dung bài viết.
    - `posting_date` (timestamp): Ngày đăng.
    - `category_id` (FK, references `categories.category_id`): Danh mục.
- **Bảng `comments`**:
    - `comment_id` (PK, serial): Mã bình luận.
    - `user_id` (FK, references `auth.users.user_id`): Người bình luận.
    - `target_type` (enum: 'listing', 'blog'): Loại đối tượng được bình luận.
    - `target_id` (integer): Mã đối tượng (liên kết với `listings.listing_id` hoặc `blog_posts.blog_id`).
    - `content` (text): Nội dung bình luận.
    - `posting_date` (timestamp): Ngày bình luận.

### 6. Schema `admin`

**Mục đích**: Quản lý các hoạt động của admin.

- **Bảng `listing_approvals`**:
    - `approval_id` (PK, serial): Mã phê duyệt.
    - `listing_id` (FK, references `listing.listings.listing_id`): Bài đăng được phê duyệt.
    - `admin_id` (FK, references `auth.users.user_id`): Admin thực hiện.
    - `action` (enum: 'approve', 'reject'): Hành động.
    - `reason` (text): Lý do (nếu từ chối).
    - `approval_date` (timestamp): Thời gian phê duyệt.
- **Bảng `alert_types`**:
    - `alert_type_id` (PK, serial): Mã loại cảnh báo.
    - `name` (text): Tên loại (ví dụ: "Nội dung không phù hợp").
- **Bảng `alerts`**:
    - `alert_id` (PK, serial): Mã cảnh báo.
    - `listing_id` (FK, references `listing.listings.listing_id`): Bài đăng bị cảnh báo.
    - `alert_type_id` (FK, references `alert_types.alert_type_id`): Loại cảnh báo.
    - `description` (text): Mô tả chi tiết.
    - `detection_time` (timestamp): Thời gian phát hiện.

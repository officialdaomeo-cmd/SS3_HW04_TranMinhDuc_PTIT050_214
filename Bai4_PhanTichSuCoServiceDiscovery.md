# Bài 4 — Phân tích sự cố Service Discovery khi scale hệ thống

## 1. Phân tích vấn đề không nhất quán `spring.application.name`
Việc đặt tên ứng dụng (`spring.application.name`) không nhất quán (như `RestaurantService` sử dụng CamelCase trong khi `order-service`, `payment-service` dùng kebab-case) có thể gây ra các vấn đề sau dù Eureka Client vẫn hoạt động:
- **Khó khăn trong tra cứu và gọi API (Service-to-Service call):** Khi `order-service` sử dụng `RestTemplate` hoặc `FeignClient` để gọi `restaurant-service`, lập trình viên sẽ phải gọi đích danh theo tên đăng ký (`http://RestaurantService/...`). Việc nhầm lẫn giữa chữ hoa và chữ thường hoặc không đồng bộ chuẩn tên sẽ dễ dẫn đến lỗi không tìm thấy dịch vụ (`UnknownHostException` hoặc `404 Not Found`).
- **Chuẩn hóa URL trên Gateway:** Các service name thường được dùng để tự động cấu hình routing trên API Gateway (như Spring Cloud Gateway). URL trên môi trường web nên sử dụng chuẩn `kebab-case`. Dùng `RestaurantService` làm path sẽ khiến URL thiết kế thiếu tính chuẩn mực và kém thân thiện (ví dụ: `http://gateway/RestaurantService/...`).
- **Khả năng hiển thị trên Eureka Dashboard:** Mặc dù Eureka có xu hướng chuyển các tên dịch vụ thành in hoa (`RESTAURANTSERVICE`), nhưng việc duy trì sự thống nhất ngay từ đầu giúp toàn bộ hệ thống code, cấu hình, và tài liệu dễ đọc, dễ bảo trì hơn.
- **Giải pháp:** Nên thống nhất sử dụng định dạng `kebab-case` cho tất cả các service (Ví dụ: `restaurant-service`).

## 2. Lỗi cấu hình `defaultZone`
- **Nguyên nhân:** Cấu hình `defaultZone: http://eureka-server:8761/eureka` bị thiếu dấu `/` ở cuối. Tuỳ thuộc vào phiên bản thư viện Spring Cloud Netflix Eureka đang sử dụng, việc thiếu dấu `/` có thể khiến client nối URL bị sai khi gọi API đăng ký lên Eureka Server (ví dụ biến thành `/eurekaapps` thay vì `/eureka/apps`). Kết quả là request bị lỗi `HTTP 404`, khiến Eureka Client không thể đăng ký instance.
- **Cách khắc phục:** Thêm dấu `/` vào cuối URL, sửa thành: `http://eureka-server:8761/eureka/`.

## 3. Cơ chế Heartbeat của Eureka
Khi `restaurant-service-2` bị crash đột ngột (không tắt ứng dụng một cách "graceful" - không kịp gửi tín hiệu hủy đăng ký), hệ thống sẽ xử lý như sau:
- **Gửi Heartbeat định kỳ:** Mặc định, mỗi instance sẽ gửi tín hiệu "heartbeat" đến Eureka Server mỗi **30 giây** để báo cáo rằng nó vẫn khỏe mạnh.
- **Thời gian chờ hết hạn (Lease Expiration):** Nếu Eureka Server không nhận được heartbeat, nó **không xóa** instance ngay lập tức. Mặc định, nó sẽ chờ **90 giây** (3 lần heartbeat bị miss). Nếu qua 90 giây mà vẫn không nhận được phản hồi, nó mới loại (evict) instance đó khỏi danh sách.
- **Chế độ tự bảo vệ (Self-Preservation Mode):** Nếu một số lượng lớn instance cùng ngừng gửi heartbeat (ít hơn 85% số heartbeat thành công trong 15 phút), Eureka Server sẽ kích hoạt tính năng tự bảo vệ. Ở chế độ này, nó cho rằng mạng đang bị sự cố (Network Partition) và sẽ **tạm dừng việc loại bỏ bất kỳ instance nào** ra khỏi danh sách để phòng hờ xóa nhầm các instance vẫn đang chạy.
- **Kết luận:** Sẽ mất **90 giây** để Eureka loại bỏ `restaurant-service-2`. Trong khoảng thời gian 90 giây này, `order-service` vẫn có nguy cơ gọi tới instance bị chết và dẫn đến timeout hoặc connection refused.

## 4. Đề xuất cấu hình cho `order-service`
Để `order-service` tra cứu và gọi mượt mà tới các instance mới nhất của `restaurant-service`, nó cần được cấu hình như sau:
1. **Sử dụng Load Balancer:**
   - Dùng annotation `@LoadBalanced` khi khởi tạo bean `RestTemplate`, hoặc sử dụng `OpenFeign` (đã tích hợp sẵn tính năng cân bằng tải). Việc này sẽ giúp tự động phân phối request luân phiên (Round Robin) tới 4 instance.
2. **Gọi dịch vụ bằng tên (Service ID) thay vì địa chỉ cứng:**
   - Thay vì code cứng URL: `http://192.168.1.5:8081/api/...`, `order-service` phải dùng: `http://restaurant-service/api/...`.
3. **Cấu hình Eureka Client:**
   - Phải đảm bảo khai báo thuộc tính `fetch-registry: true` (đây là giá trị mặc định, nhưng nên khai báo rõ ràng) để client tự động kéo danh sách định kỳ.
4. **Cơ chế Retry/Fallback (Khuyến nghị thêm):**
   - Do độ trễ của cơ chế heartbeat (như đã nói ở phần 3), `order-service` cần được tích hợp **Resilience4j (Retry / Circuit Breaker)**. Nếu nó lỡ gọi trúng instance `restaurant-service-2` vừa mới crash chưa kịp bị xóa khỏi danh sách, cơ chế Retry sẽ lập tức gọi sang instance khác còn sống (`restaurant-service-1`, `3`, hoặc `4`).

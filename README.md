# FoodX - Hệ thống Microservices

Dự án này bao gồm 3 service chính:
- `restaurant-service`
- `order-service`
- `delivery-service`

## Chuẩn hóa dependency Spring Cloud

### Vấn đề của việc khai báo version cố định (hardcoded)
Trong tình huống ban đầu, các file `build.gradle` (ví dụ `order-service`) khai báo trực tiếp phiên bản cho từng thư viện Spring Cloud như sau:
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-config:3.1.4'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:3.1.5'
}
```
**Rủi ro:**
1. **Xung đột thư viện (Dependency Conflicts):** Spring Cloud là một tập hợp gồm hàng chục module khác nhau liên kết chặt chẽ. Việc gán cứng (hardcode) phiên bản khác nhau (`3.1.4`, `3.1.5`) cho các module dễ dẫn đến việc các thư viện này kéo theo các sub-dependency không tương thích, gây lỗi lúc build hoặc lỗi ngầm khi chạy (runtime exception).
2. **Khó bảo trì và nâng cấp:** Khi dự án cần tích hợp thêm các starter khác (như API Gateway, Feign, Resilience4j, v.v.), lập trình viên phải tự tìm kiếm bằng tay phiên bản nào tương thích với các phiên bản đang dùng. Việc này tốn thời gian và rất dễ xảy ra sai sót.
3. **Rủi ro không tương thích Spring Boot:** Các thư viện Spring Cloud cần tương thích hoàn toàn với lõi Spring Boot đang được sử dụng. Khai báo rời rạc làm mất đi sự đối chiếu tổng thể giữa Spring Boot và Spring Cloud.

### Giải pháp: Sử dụng Spring Cloud BOM (Bill of Materials)
Spring Cloud cung cấp một BOM dưới dạng `spring-cloud-dependencies`. Nó là một bảng tham chiếu trung tâm định nghĩa danh sách các phiên bản đã được test và đảm bảo hoạt động hoàn hảo cùng nhau cho cả một đợt phát hành (Release Train).

**BOM giải quyết vấn đề và vì sao nó cần thiết?**
1. **Quản lý phiên bản tập trung:** Việc thêm BOM vào khối `dependencyManagement` giúp khai báo tập trung phiên bản Spring Cloud của toàn hệ thống (ví dụ: `2022.0.4` hoặc `2023.0.0`). 
2. **Loại bỏ việc khai báo version thủ công:** Nhờ có BOM, trong khối `dependencies`, chúng ta có thể thoải mái thêm các starter của Spring Cloud (Config, Eureka, Gateway...) mà **không cần ghi version**. Maven/Gradle sẽ tự tra cứu trong BOM để lấy ra phiên bản chính xác, tương thích nhất với Release Train đã chọn.
3. **Đồng bộ hóa các service:** Việc cấu hình chung BOM ở cả 3 service (`restaurant-service`, `order-service`, `delivery-service`) đảm bảo toàn bộ hệ thống đang chạy chung một phiên bản Spring Cloud, tránh lỗi giao tiếp giữa các service do chênh lệch phiên bản (ví dụ Eureka Server dùng bản mới mà Client dùng bản quá cũ).

Điều này đặc biệt quan trọng **trước khi tích hợp Config Server và Eureka**, vì đây là các công cụ hạ tầng nền tảng, mọi service trong hệ thống đều phải giao tiếp với chúng. Việc cấu hình BOM đúng đắn từ đầu giúp quá trình tích hợp trơn tru, không gặp ác mộng xung đột classpath.

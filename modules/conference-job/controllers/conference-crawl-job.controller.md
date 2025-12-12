# ConferenceCrawlJobController

## 1. Tổng quan (Overview)

- **Mục đích chính**: Controller quản lý các tác vụ crawl/update conference thông qua API RESTful
- **Vai trò trong kiến trúc**: Là entry point cho tất cả operations liên quan đến conference crawl job, điều phối giữa client và service layer
- **Dependencies chính**: ConferenceCrawlJobService, ConferenceService, ConferenceOrganizationService, AdminConferenceService, NotificationService, EmailService, FollowConferenceService, UserService

## 2. Class/Component chính

### **ConferenceCrawlJobController**
- **Chức năng**: Điều phối các operations crawl, update, schedule conference jobs
- **Constructor dependencies**:
  - `conferenceCrawlJobService`: Xử lý logic crawl job chính
  - `conferenceService`: Quản lý conference data
  - `conferenceOrganizationService`: Quản lý organization data (thông tin tổ chức, địa điểm, lịch diễn ra của conference)
  - `adminConferenceService`: Admin operations cho conference (quản lý conference với quyền admin, import/export dữ liệu). Service này xử lý các tác vụ quản trị conference mà user thường không có quyền truy cập
  - `notificationService`: Gửi notifications
  - `emailService`: Xử lý email operations
  - `followService`: Quản lý follow relationships (user theo dõi conference, gửi thông báo khi có cập nhật)
  - `userService`: User management
- **Properties quan trọng**: Không có properties riêng, chỉ sử dụng injected services

## 3. API Endpoints

### **GET /conference-crawl-job**
- **Mục đích**: Lấy danh sách tất cả crawl jobs
- **Parameters**: None
- **Response**: Array của ConferenceCrawlJobDTO

### **GET /conference-crawl-job/status**
- **Mục đích**: Lấy status của crawl jobs với pagination
- **Parameters**: 
  - `page` (optional): Page number, default 1
  - `perPage` (optional): Items per page, default 10
  - `status` (optional): Filter by status
- **Response**: Paginated job status list

### **GET /conference-crawl-job/stats**
- **Mục đích**: Lấy thống kê crawl jobs
- **Parameters**: None
- **Response**: Job statistics (total, completed, failed, pending, running, successRate)

### **POST /conference-crawl-job/schedule-cron**
- **Mục đích**: Schedule cron job cho automatic conference updates
- **Parameters**:
  - `schedule` (required): Cron expression string - Chuỗi định dạng thời gian để lên lịch tự động cho tác vụ
    - **Cú pháp**: `* * * * *` (phút giờ ngày tháng ngày-trong-tuần)
    - **Ví dụ phổ biến**:
      - `0 0 * * *` - Mỗi ngày lúc nửa đêm
      - `0 9 * * 1-5` - Mỗi ngày trong tuần lúc 9 giờ sáng
      - `*/15 * * * *` - Mỗi 15 phút
      - `0 2 * * 0` - Mỗi Chủ nhật lúc 2 giờ sáng
  - `batchSize` (optional): Batch size, default 10
- **Response**: Xác nhận thông báo lên lịch

### **POST /conference-crawl-job/cancel-cron**
- **Mục đích**: Hủy tất cả cron jobs và stop operations
- **Parameters**: None
- **Response**: Kết quả hủy với thông tin chi tiết về trạng thái

### **POST /conference-crawl-job/start-cron-immediate**
- **Mục đích**: Thực thi cron job ngay lập tức
- **Authentication**: Bearer token required (Admin only)
- **Parameters**:
  - `batchSize` (optional): Batch size, default 10
  - `take` (optional): Number of conferences to fetch, default 10
- **Response**: Immediate execution results

### **GET /conference-crawl-job/cron-status**
- **Mục đích**: Lấy status của cron jobs
- **Parameters**: None
- **Response**: Cron job status information bao gồm:
  - `isActive`: Boolean - Cron job có đang active hay không
  - `registryExists`: Boolean - Cron job có tồn tại trong scheduler registry hay không
  - `schedule`: String - Biểu thức cron đang được sử dụng (null nếu không có)
  - `lastRun`: Date - Thời gian thực thi lần cuối cùng (null nếu chưa chạy)
  - `nextRun`: Date - Thời gian thực thi lần tiếp theo (null nếu không có)

### **GET /conference-crawl-job/start**
- **Mục đích**: Start crawl job với hardcoded conference data (testing)
- **Parameters**: None (requires authenticated user)
- **Response**: Crawl data from external service
- **Hardcoded data example**:
  ```json
  {
    "Title": "AAAI Conference on Human Computation and Crowdsourcing",
    "Acronym": "HCOMP"
  }
  ```
- **Ghi chú**: Endpoint này dùng conference data cố định trong code để test chức năng crawl mà không cần setup dữ liệu phức tạp

### **POST /conference-crawl-job/update/:id**
- **Mục đích**: Cập nhật specific conference
- **Authentication**: Bearer token required (User)
- **Parameters**:
  - `id` (path param): Conference ID
  - **Note**: Không cần mainLink, cfpLink và impLink vì đã được fetch từ conferenceOrganizationService
- **Response**: Update results with notifications sent

### **POST /conference-crawl-job/schedule-update**
- **Mục đích**: Schedule batch update cho tất cả conferences
- **Parameters**:
  - `batchSize` (optional): Batch size, default 10. Chia conferences thành các batch để xử lý, tránh quá tải
- **Response**: Batch update schedule results

### **POST /conference-crawl-job/schedule-delayed**
- **Mục đích**: Schedule delayed crawl job. Có hai loại crawl là New Crawl(tìm hội nghị mới) và Update Crawl(cập nhật hội nghị đã có). Endpoint này là Update Crawl. Nó lấy conferences cũ từ database đem đi cập nhật.
- **Authentication**: Bearer token required
- **Parameters**:
  - `delaySeconds` (optional): Delay in seconds, default 0
  - `delayMinutes` (optional): Delay in minutes, default 0
  - `delayHours` (optional): Delay in hours, default 0
  - `batchSize` (optional): Batch size, default 10
  - `take` (optional): Số lượng hội nghị, default 10
- **Response**: Delayed schedule confirmation

### **POST /conference-crawl-job/force-update-jobs**
- **Mục đích**: Force update tất cả các job về trạng thái cancelled. Sử dụng khi nhiều jobs bị kẹt ở trạng thái RUNNING.
- **Note**: Chỉ cập nhật trạng thái trong database, không dừng cron jobs đang chạy
- **Parameters**: None
- **Response**: Force update results

### **GET /conference-crawl-job/detailed-status**
- **Mục đích**: Lấy detailed status của jobs
- **Parameters**: None
- **Response**: Detailed job status information bao gồm:
  - **database**: Thông tin jobs trong database
    - `totalJobs`: Tổng số jobs
    - `pending`: Jobs đang chờ (count + danh sách chi tiết)
      - `id`, `conferenceId`, `status`, `progress`, `message`, `createdAt`, `updatedAt`
    - `running`: Jobs đang chạy (count + danh sách chi tiết)
      - `id`, `conferenceId`, `status`, `progress`, `message`, `createdAt`, `updatedAt`
    - `completed`: Số jobs đã hoàn thành
    - `failed`: Số jobs thất bại
    - `cancelled`: Số jobs đã hủy
  - **queue**: Trạng thái hàng đợi Bull Queue
    - `waiting`: Số jobs đang chờ trong queue
    - `active`: Số jobs đang active trong queue
    - `completed`: Số jobs đã hoàn thành trong queue
    - `failed`: Số jobs thất bại trong queue
    - `activeJobs`: Danh sách jobs đang active (id, name, progress)
  - **cronJob**: Trạng thái cron job (isActive, registryExists, schedule, lastRun, nextRun)

### **POST /conference-crawl-job/test-stop-operations**
- **Mục đích**: Test stop operations (debugging toàn diện). Kiểm tra từng component riêng biệt để tìm ra lỗi khi các thao tác dừng không hoạt động đúng.
- **Parameters**: None
- **Response**: Test operation results bao gồm 3 bước:
  - **Test 1**: Force update database jobs (`forceUpdateJobsToCancel`) - Chỉ cập nhật trạng thái trong database, không dừng cron jobs đang chạy
  - **Test 2**: Get detailed status (`getDetailedJobStatus`) 
  - **Test 3**: Force stop cron jobs (`forceStopAllCronJobs`) - Dừng runtime processes và queue, không cập nhật database
  - **Kết quả**: `{ success: boolean, updateResult, statusResult, cronResult }`
- **Sử dụng**: Khi không biết tại sao `cancel-cron` không hoạt động, cần kiểm tra từng method riêng lẻ

### **POST /conference-crawl-job/simple-cancel-test**
- **Mục đích**: Simple cancel test (debugging cơ bản). Kiểm tra nhanh chức năng cancel cơ bản của cron job scheduler.
- **Parameters**: None
- **Response**: Test results with timestamp:
  - **Kết quả**: `{ testName: 'simple-cancel-test', success: boolean, result, timestamp }`
- **Sử dụng**: Test nhanh method `cancelCronUpdate()` đơn giản, level debugging thấp nhất
- **Note**: `cancelCronUpdate()` chỉ hủy cron job instance, không xử lý queue hay database jobs

## 4. Business Logic

Controller này chủ yếu điều phối các operations:
- **Job Management**: Tạo, quản lý, hủy crawl jobs
- **Scheduling**: Cron job scheduling và management
- **Batch Processing**: Xử lý hàng loạt conferences
- **Real-time Operations**: Immediate execution và status tracking
- **Notification Integration**: Gửi notifications sau khi update
- **User Context**: Tracking user actions và permissions

## 5. Data Models/DTOs

### **Input Types**:
- `ConferenceCrawlJobInputDTO`: Input cho crawl job creation
- `RequestWithUser`: Extended Request interface với user info

### **Output Types**:
- `ConferenceCrawlJobDTO`: Job information response
- Paginated responses với metadata
- Status objects với progress tracking

### **Validation Rules**:
- JWT authentication cho protected endpoints
- Role-based access control (Admin vs User)
- Parameter validation (positive numbers, required fields)
- Error handling với appropriate HTTP status codes
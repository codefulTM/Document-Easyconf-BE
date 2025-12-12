# ConferenceCrawlJobService

## 1. Tổng quan (Overview)

- **Mục đích chính**: Service xử lý logic crawl/update conference data thông qua queue và cron jobs
- **Vai trò trong kiến trúc**: Là business logic layer cho conference crawl operations, quản lý queue jobs, cron scheduling, và database operations
- **Dependencies chính**:
  - `PrismaService`: Database operations
  - `ConferenceCrawlQueue`: BullMQ queue management
  - `HttpService`: External API calls
  - `SchedulerRegistry`: Cron job management
  - `AdminConferenceService`: Conference data management

## 2. Class/Component chính

### **ConferenceCrawlJobService**
- **Chức năng**: Quản lý toàn bộ lifecycle của conference crawl jobs
- **Constructor dependencies**:
  - `prismaService`: Database access layer
  - `conferenceCrawlQueue`: Queue system cho async processing
  - `httpService`: HTTP client cho external API calls
  - `schedulerRegistry`: Cron job registration và management
  - `adminConferenceService`: Conference CRUD operations
- **Properties quan trọng**:
  - `cronJob: CronJob | null`: Active cron job instance
- **Methods chính**:
  - Job management: `getListConferenceCrawlJob()`, `getUpdateStatus()`, `getUpdateStats()`
  - Queue operations: `scheduleUpdate()`, `createConferenceCrawlJob()`, `createConferenceCrawlJobs()`
  - Cron management: `scheduleCronUpdate()`, `cancelCronUpdate()`, `forceStopAllCronJobs()`
  - Debug/testing: `testStopOperations()`, `getDetailedJobStatus()`

## 3. API Endpoints
*(Không áp dụng - đây là Service layer)*

## 4. Business Logic

### **Core Methods và Chức năng**

#### **Job Management Methods**
- `getListConferenceCrawlJob()`: Get all crawl jobs từ database
- `getUpdateStatus()`: Paginated job status với filter by status
- `getUpdateStats()`: Thống kê jobs (total, completed, failed, pending, running, successRate)

#### **Queue Operations**
- `scheduleUpdate(batchSize, take?)`: Schedule batch update cho conferences
  - Chia conferences thành batches để tránh overload
  - Add jobs vào BullMQ queue với proper naming
  - Cụ thể step by step:
    - Bước 1: Lấy tất cả các hội nghị từ database, kèm theo thông tin tổ chức của chúng, với các điều kiện:
        - Lấy 1 tổ chức cũ nhất chưa được cập nhật
        - Chỉ lấy những hội nghị có ít nhất một tổ chức thỏa mãn: có đường link hoặc là phiên bản mới nhất
        - Sort: Sắp xếp theo thời gian cập nhật giảm dần(mới nhất trước)
        - Limit: Giới hạn số lượng kết quả theo tham số take

    - Bước 2: Tạo batches từ conferences để chuẩn bị cho queue processing
        - Khởi tạo batches array
        - Chia conferences thành batches
        - Filter để lấy conferences có organizations
        - Map để transform dữ liệu cho khớp với DTO
        - Thêm vào batches nếu có data

    - Bước 3: Với mỗi batch, tạo job records trong database cho từng conference trong batch. Mỗi job sẽ có status = PENDING, progress = 0. Sau khi tạo records, jobs sẽ được thêm vào BullMQ queue để xử lý. Tất cả những điều trên được thực hiện bởi hàm createBatchUpdateConferenceCrawlJob Kết quả tạo job sẽ được lưu vào biến results. 

- `createConferenceCrawlJob(input)`: Tạo conference crawl job
  - Cụ thể step by step:
    - Tạo job instance trong database
    - Xác định job type là update hay là crawl
    - Thêm job vào BullMQ queue để xử lý bất đồng bộ

- `createConferenceCrawlJobs(inputs)`: Tạo conference crawl jobs
  - Cụ thể step by step:
    - Nếu là một hội nghị duy nhất -> Gọi hàm createConferenceCrawlJob
    - Nếu là nhiều hội nghị -> Xử lý theo lô. Nếu có conference trong batch chứa link thì là update job, nếu không có link thì là crawl job. Update job -> gọi createBatchUpdateConferenceCrawlJob(). Crawl job -> gọi createBatchCrrawlJob()

- `fetchConferenceCrawlData(input: ConferenceCrawlNewRequestDto)`: Gọi external crawler service để lấy conference data
  - Cụ thể step by step:
    - Lấy crawler URL từ .env
    - Tạo HTTP POST request với:
      - URL: CRAWL_URL + '/crawl-conferences'
      - Body: input (ConferenceCrawlNewRequestDto)
      - Params: dataSource: 'client', mode: 'sync'. Nghĩa là request đến từ frontend, và xử lý đồng bộ chứ không phải bất đồng bộ.
      - Headers: Content-Type: 'application/json'
    - Xây dựng RxJS pipeline với catchError
    - Convert Observable thành Promise bằng firstValueFrom
    - Extract data field từ response
    - Return ConferenceCrawlNewResponseDto

- `fetchUpdateConferenceCrawlData(input: ConferenceCrawlUpdateRequestDto)`: Gọi external crawler service để update conference data
  - **Mục đích**: Fetch updated data cho conferences đã có sẵn
  - **Step by step**:
    - Lấy crawler URL từ .env
    - Log input data để debug
    - Tạo HTTP POST request tương tự fetchConferenceCrawlData
    - Xử lý lỗi với detailed logging
    - Return ConferenceCrawlNewResponseDto

- `importConferenceCrawlData(crawlData, jobId)`: Import conference data từ crawler vào database
  - **Mục đích**: Process và lưu conference data từ external crawler
  - **Step by step**:
    - Update job status thành RUNNING (progress 20%)
    - Loop qua từng conference trong crawlData.data
    - Transform crawl data sang ConferenceSaveDto format
    - Gọi adminConferenceService để save/update conference
    - Final update job status thành COMPLETED hoặc FAILED

- `fetchBatchUpdateConferenceCrawlData(inputs)`: Fetch data cho batch update operations
  - **Mục đích**: Gọi crawler service cho multiple conferences cùng lúc
  - **Step by step**:
    - Lấy crawler URL
    - Tạo HTTP POST request với batch inputs
    - Process response và return ConferenceCrawlNewResponseDto

- `createBatchUpdateConferenceCrawlJob(inputs)`: Tạo batch jobs cho update operations
  - **Mục đích**: Tạo multiple job records cho conference updates
  - **Step by step**:
    - Loop qua inputs array
    - Tạo job record trong database cho mỗi conference
    - Add jobs vào BullMQ queue với UPDATE job type
    - Return array của created job DTOs

- `createBatchCrawlJob(inputs)`: Tạo batch jobs cho crawl operations
  - **Mục đích**: Tạo multiple job records cho new conference crawling
  - **Step by step**:
    - Loop qua inputs array
    - Tạo job record trong database cho mỗi conference
    - Add jobs vào BullMQ queue với BATCH_CRAWL job type
    - Return array created job DTOs

- `updateConferenceCrawlJob(id, data)`: Update status và progress của job
  - **Mục đích**: Cập nhật job state trong database
  - **Step by step**:
    - Update job record với new data (status, progress, message,...)
    - Return updated job DTO

- `getConferenceToCrawl(take)`: Lấy conferences cần crawl
  - **Mục đích**: Fetch candidates cho crawling operations
  - Trả về danh sách các hội nghị từ database, với điều kiện là hội nghị phải có ít nhất một tổ chức liên kết và tổ chức đó phải có đường link không rỗng. Giới hạn số lượng kết quả theo tham số take.

#### **Cron Job Management**
- `scheduleCronUpdate(schedule, batchSize, take?)`: Dùng để tạo và quản lý cron job tự động cho conference updates
  - Cụ thể step by step:
    - Hủy cron job hiện tại
    - Tạo instance CronJob mới với hai tham số:
      - schedule: định nghĩa thời gian thực thi với format * * * * * (phút giờ ngày tháng ngày-trong-tuần). Mục đích là xác định khi nào job sẽ được triggerr
      - callback: function được thực thi mỗi khi cron trigger

    - Đăng ký với NestJS SchedulerRegistry. Note: NestJS SchedulerRegistry là service quản lý scheduled jobs trong NestJS
    - Bắt đầu cron job

- `cancelCronUpdate()`: Basic cron job cancellation
  - Stop cron job và xóa khỏi registry
  - **Scope**: Chỉ cron job, không xử lý queue/database
  
- `forceStopAllCronJobs()`: Huỷ bỏ hoàn toàn tất cả các cron job
  - Stop tất cả cron jobs liên quan đến conference
  - Terminate queue processes
  - Update database jobs to cancelled
  - **Scope**: Runtime processes và database

#### **Comprehensive Operations**

- `stopConferenceCrawlOperations()`: Dừng hoàn toàn tất cả các hoạt động crawl conference
  - **Mục đích**: Cleanup toàn diện khi cần dừng hệ thống - dừng queue, hủy cron jobs, cập nhật database
  - **Step by step**:
    - **Step 1**: Kiểm tra trạng thái hiện tại (database jobs, queue state)
    - **Step 2**: Dừng và terminate conference crawl queue
      - Pause queue → drain waiting jobs → obliterate active jobs
      - Update database status cho các jobs trong queue
    - **Step 3**: Hủy các cron jobs đang hoạt động
      - Gọi cancelCronUpdate() để dừng scheduled jobs
    - **Step 4**: Update tất cả pending/running jobs trong database thành cancelled
      - Gọi forceUpdateJobsToCancel() để đảm bảo data consistency
    - **Step 5**: Verification và summary
      - Kiểm tra final state, tạo report, return detailed results

 - `forceUpdateJobsToCancel()`: Database-level cleanup
  - Update PENDING/RUNNING jobs thành CANCELLED
  - Reset progress và update timestamps
  - **Scope**: Database only, không tác động runtime

- `getDetailedJobStatus()`: Báo cáo trạng thái chi tiết
  - Database jobs breakdown (pending, running, completed, failed, cancelled)
  - Queue status (waiting, active, completed, failed)
  - Cron job status (active, schedule, last/next run)

#### **Debug Methods**
- `testStopOperations()`: 3-step debugging process
  - Test 1: `forceUpdateJobsToCancel()` - Database cleanup
  - Test 2: `getDetailedJobStatus()` - Status verification
  - Test 3: `forceStopAllCronJobs()` - Runtime cleanup

### **Xử lý Dữ Liệu**
1. **Tạo Job**: Xác thực DTO input → Tạo bản ghi trong cơ sở dữ liệu → Thêm job vào hàng đợi
2. **Lên lịch Cron**: Xác thực lịch → Xóa job cũ → Đăng ký lịch mới
3. **Thực thi Job**: Xử lý hàng đợi → Gọi API bên ngoài → Cập nhật cơ sở dữ liệu → Theo dõi trạng thái
4. **Xử lý Lỗi**: Sử dụng try-catch chi tiết với ghi log và các biện pháp khôi phục

### **Tích hợp bên ngoài**
- **Tích hợp API bên ngoài**: Lấy dữ liệu hội nghị qua HttpService với xử lý lỗi thích hợp
- **Hệ thống xử lý hàng đợi**: Sử dụng BullMQ để đảm bảo xử lý đồng bộ đáng tin cậy với các chính sách lặp lại
- **Cơ sở dữ liệu**: Sử dụng Prisma ORM để thực hiện các thao tác truy vấn cơ sở dữ liệu hiệu quả
- **Lịch định kỳ**: Sử dụng NestJS SchedulerRegistry để quản lý việc lập lịch

## 5. Data Models/DTOs

### **Các kiểu dữ liệu đầu vào**
- `ConferenceCrawlJobInputDTO`: Đầu vào cho việc tạo job
- `ConferenceCrawlNewRequestDto`: Yêu cầu crawl hội nghị mới
- `ConferenceCrawlUpdateRequestDto`: Yêu cầu cập nhật hội nghị
- `ConferenceSaveDto`: Kiểu dữ liệu cho dữ liệu hội nghị cho các thao tác quản lý

### **Các kiểu dữ liệu đầu ra**
- `ConferenceCrawlJobDTO`: Thông tin về job
- `ConferenceCrawlNewResponseDto`: Phản hồi sau khi crawl mới
- Các phản hồi có phân trang (`jobs`, `meta` đối tượng)
- Các đối tượng trạng thái với chi tiết detai
- Kết quả của các thao tác với theo dõi thành công/thất bại

# ConferenceImportProcessor Documentation

## 1. Tổng quan (Overview)

### Mục đích chính của file/module
- **ConferenceImportProcessor** là một BullMQ processor xử lý các tác vụ liên quan đến việc crawl và import dữ liệu conference
- Chịu trách nhiệm thực thi các job trong queue `CRAWL_CONFERENCE_QUEUE` để tự động thu thập thông tin conference từ các nguồn bên ngoài
- Quản lý toàn bộ lifecycle của conference import jobs bao gồm single crawl, batch crawl, và update operations

### Vai trò trong kiến trúc hệ thống
- Là **worker processor** trong kiến trúc message queue BullMQ. Worker Processor là một service/process chuyên trách lắng nghe và xử lý các job từ một queue cụ thể.
- Kết nối giữa **queue system** và **conference management services**
- Bridge layer giữa **external crawler service** và **internal database operations**
- Cung cấp **real-time progress tracking** thông qua WebSocket messages

### Dependencies chính
- **@nestjs/bullmq**: BullMQ processor và WorkerHost. WorkerHost là base class cung cấp functionality để kết nối với BullMQ worker.
- **ConferenceCrawlJobService**: Service gọi external crawler API
- **ConferenceOrganizationService**: Import dữ liệu vào database
- **MessageService**: Gửi real-time updates qua WebSocket
- **PrismaService**: Database operations cho job status tracking
- **LoggerService**: Logging và monitoring

## 2. Class/Component chính

### Tên class và chức năng
```typescript
@Injectable()
@Processor(CONFERENCE_QUEUE_NAME.CRAWL)
export class ConferenceImportProcessor extends WorkerHost
```

**ConferenceImportProcessor** - Processor chính xử lý 4 loại job:
- **CRAWL**: Import dữ liệu conference đơn lẻ
- **BATCH_CRAWL**: Import hàng loạt conferences
- **UPDATE**: Cập nhật dữ liệu conference có sẵn
- **NOTIFY**: Gửi thông báo (placeholder)

### Constructor và injected dependencies
```typescript
constructor(
  private loggerService: LoggerService,
  private conferenceCrawlJobService: ConferenceCrawlJobService,
  private conferenceOrganizationService: ConferenceOrganizationService,
  private messageService: MessageService,
  private prismaService: PrismaService,
)
```

### Properties và Methods quan trọng

#### Public Methods
- **`process()`** - Main Dispatcher
  - **Chức năng**: Route jobs đến appropriate handler dựa trên job name
  - **Input**: `Job<ConferenceCrawlJobDTO | ConferenceBatchCrawlJobDTO>`. Data type này đại diện cho một job trong queue BullMQ, chứa thông tin về job. Job ở đây có thể là single conference job(ConferenceCrawlJobDTO) hoặc batch conference job(ConferenceBatchCrawlJobDTO)
  - **Output**: Processed job data

#### Private Methods

##### Database Status Management
- **`updateJobStatusInDatabase()`**: Cập nhật status job trong database
- **`updateIndividualJobInBatch()`**: Cập nhật status individual job trong batch. Note: hàm này và hàm updateJobStatusInDatabase() gần như là một, chỉ khác nhau về tên.

##### Job Processing Handlers
- **`handleCrawlConferenceJob()`** - Single Conference Processing
  - **Chức năng**: Xử lý import một conference vào database
  - **Key Features**:
    - Progress tracking với real-time updates
    - Link preservation (ưu tiên links đã tồn tại hơn crawler data). Ví dụ: Conference đã có mainLink trong database -> Giữ nguyên. Conference chưa có cfpLink -> Dùng link từ crawler.
    - Thực hiện việc import toàn diện (tổ chức, địa điểm, chủ đề, ngày)
  - **Flow**:
    ```
    - Đặt trạng thái của job thành RUNNING trong cả queue và database
    - Kiểm tra xem loại job là update hay crawl job. Nếu có ít nhất một hội nghị có link thì là update job
    - Gọi external crawler service và lưu kết quả
    - Với mỗi hội nghị trong dữ liệu của job:
      - Match crawler data với conference (title/acronym)
      - Giữ lại link cũ của các hội nghị nếu tồn tại hoặc lấy link mới từ crawler data nếu không tồn tại.
      - Import organization data mới, giữ lấy dữ liệu organizeData được trả về. Mỗi lần crawl phải tạo organization data mới vì thêm mới thì dễ dàng hơn tìm và update organization data cũ
      - Import place data từ organizeData.id
      - Import topics từ organizeData.id
      - Import dates từ crawler data
      - Đánh dấu organization là latest
    - Cập nhật trạng thái job thành COMPLETED
    ```

- **`handleBatchCrawlConferenceJob()`** - Batch Processing
  - **Chức năng**: Xử lý import hàng loạt conferences vào database. Tên hàm hơi gây hiểu lầm một xíu, batch ở đây nghĩa là hàng loạt job một lúc chứ không phải chia theo nhiều lô và xử lý từng lô
  - **Flow**:
    ```
    - Set tất cả các job thành trạng thái RUNNING
    - Kiểm tra xem loại job là update hay crawl job. Nếu có ít nhất một hội nghị có link thì là update job
    - Gọi external crawler service và lưu kết quả
    - Với từng hội nghị, import dữ liệu được crawl vào database.  
    - Cập nhật trạng thái của từng job thành COMPLETED

    ```

- **`handleUpdateConferenceJob()`**: Xử lý conference update (TODO - chưa implemented)

# CONFERENCE_CRAWL_JOB_NAME Enum

Các enum này định nghĩa tên của các loại công việc (job types) trong hệ thống xử lý conference của Easyconf-BE.

## CRAWL = 'CRAWL_CONFERENCE_JOB'

- **Mục đích**: Job crawl conference mới từ đầu
- **Sử dụng**: Khi conference chưa có dữ liệu, cần thu thập thông tin cơ bản
- **Processor**: `handleCrawlConferenceJob()` trong `ConferenceImportProcessor`

## UPDATE = 'UPDATE_CONFERENCE_JOB'

- **Mục đích**: Job cập nhật conference đã có sẵn
- **Sử dụng**: Khi conference đã tồn tại và cần cập nhật thông tin mới
- **Processor**: `handleUpdateConferenceJob()` (hiện tại TODO)

## NOTIFY = 'NOTIFY_CONFERENCE_JOB'

- **Mục đích**: Job thông báo sau khi xử lý
- **Sử dụng**: Gửi thông báo về trạng thái/kết quả của các job khác
- **Processor**: Chỉ log thông báo, chưa triển khai đầy đủ

## BATCH_CRAWL = 'BATCH_CRAWL_CONFERENCE_JOB'

- **Mục đích**: Job crawl hàng loạt nhiều conference
- **Sử dụng**: Xử lý nhiều conference cùng lúc để tối ưu performance
- **Processor**: `handleBatchCrawlConferenceJob()` trong `ConferenceImportProcessor`

**Flow hoạt động**: Các job này được thêm vào `CRAWL_CONFERENCE_QUEUE` và được xử lý bởi `ConferenceImportProcessor` dựa trên job name tương ứng.
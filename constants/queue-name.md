# CONFERENCE_QUEUE_NAME Enum

Các enum này định nghĩa tên của các hàng đợi (queue) trong hệ thống xử lý tác vụ bất đồng bộ của Easyconf-BE, sử dụng BullMQ.

## CRAWL = 'CRAWL_CONFERENCE_QUEUE'

- **Mục đích**: Hàng đợi xử lý tác vụ crawl/thu thập dữ liệu conference
- **Chức năng**: Xử lý các job như crawl conference mới, cập nhật conference có sẵn, batch crawl
- **Processor**: `ConferenceImportProcessor` xử lý các job type: CRAWL, UPDATE, BATCH_CRAWL

## NOTIFY = 'NOTIFY_CONFERENCE_QUEUE'

- **Mục đích**: Hàng đợi xử lý tác vụ thông báo/notify
- **Chức năng**: Gửi thông báo về trạng thái crawl, kết quả xử lý
- **Processor**: Hiện tại chỉ có log thông báo, chưa triển khai đầy đủ
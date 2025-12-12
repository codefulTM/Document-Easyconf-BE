# Overview

Các file liên quan đến việc crawl dữ liệu trong thư mục `Easyconf-BE`:

- Module Conference Job (chủ yếu xử lý crawl):
  - `src/modules/conference-job/controllers/conference-crawl-job.controller.ts` - Controller điều khiển job crawl
  - `src/modules/conference-job/services/conference-crawl-job.service.ts` - Service chính xử lý crawl
  - `src/modules/conference-job/queues/conference-import.processor.ts` - Queue processor xử lý import dữ liệu
  - `src/modules/conference-job/models/conference-crawl-job/` - Các DTO liên quan đến crawl job
  - `src/modules/conference-job/models/crawl-request/` - DTO cho request crawl
  - `src/modules/conference-job/models/crawl-response/` - DTO cho response crawl

- File cấu hình và constants:
  - `src/constants/job-name.ts` - Tên các job
  - `src/constants/queue-name.ts` - Tên các queue
  - `.env.example` - Biến môi trường (có thể chứa config crawl)

- Module Conference:
  - `src/modules/conference/models/conference-crawl/conference-crawl.ts` - Model crawl
  - `src/modules/conference/services/conference.service.ts` - Service xử lý conference

- Các file khác:
  - `prisma/schema.prisma` - Database schema (có thể chứa model crawl)
  - `src/modules/conference-job/conference-job.module.ts` - Module định nghĩa các service crawl

Hệ thống sử dụng Bull Queue để xử lý job crawl với kiến trúc controller-service-queue processor.

Mỗi file doc trong thư mục document sẽ bao gồm:
1. **Tổng quan (Overview)**
   - Mục đích chính của file/module
   - Vai trò trong kiến trúc hệ thống
   - Dependencies chính

2. **Class/Component chính**
   - Tên class và chức năng
   - Constructor và injected dependencies
   - Properties và methods quan trọng

3. **API Endpoints (nếu là Controller)**
   - Danh sách các routes với method (GET, POST, PUT, DELETE)
   - Mục đích của từng endpoint
   - Parameters và response types

4. **Business Logic (nếu là Service)**
   - Các methods chính và chức năng
   - Flow xử lý dữ liệu
   - Integration với external services
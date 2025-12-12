# Conference Service Documentation

## 1. Tổng quan (Overview)

### Mục đích chính của file/module
- Quản lý các thao tác liên quan đến hội nghị (conference) trong hệ thống
- Cung cấp các phương thức để lấy, tạo, cập nhật và xóa thông tin hội nghị
- Xử lý logic nghiệp vụ liên quan đến hội nghị

### Vai trò trong kiến trúc hệ thống
- Là service layer nằm giữa controller và database
- Tương tác với database thông qua Prisma Service
- Kết nối với các services khác như ConferenceOrganizationService, ConferenceRankService, RedisCacheService

### Dependencies chính
- `PrismaService`: Tương tác với database
- `ConferenceOrganizationService`: Quản lý thông tin tổ chức hội nghị
- `ConferenceRankService`: Xử lý xếp hạng hội nghị
- `RedisCacheService`: Cache dữ liệu để tối ưu hiệu năng
- `RecommendService`: Cung cấp các gợi ý liên quan

## 2. Class/Component chính

### Tên class và chức năng
```typescript
@Injectable()
export class ConferenceService
```
- Quản lý toàn bộ các hoạt động liên quan đến hội nghị
- Xử lý logic nghiệp vụ phức tạp
- Tương tác với nhiều service khác để hoàn thành các tác vụ

### Constructor và injected dependencies
```typescript
constructor(
  private readonly prismaService: PrismaService,
  private readonly conferenceOraganizationService: ConferenceOrganizationSerivce,
  private readonly conferenceRankService: ConferenceRankService,
  private readonly txHost: TransactionHost<TransactionalAdapterPrisma>,
  private readonly cacheService: RedisCacheService,
  private readonly recommendService: RecommendService,
)
```

### Properties và Methods quan trọng
- **Private Properties**:
  - `logger`: Ghi log các hoạt động
  - `paginate`: Hàm hỗ trợ phân trang

- **Public Methods**:
  - `getConferences()`: Lấy danh sách hội nghị với các bộ lọc và sắp xếp
  - `getConferencesFromDB()`: Lấy dữ liệu hội nghị từ database
  - `getConferenceById()`: Lấy thông tin chi tiết một hội nghị
  - `createConference()`: Tạo mới một hội nghị
  - `updateConference()`: Cập nhật thông tin hội nghị
  - `deleteConference()`: Xóa một hội nghị

## 3. Business Logic

### Các methods chính

#### `getConferences(conferenceFilter?, sortOptions?)`
- **Chức năng**: Lấy danh sách hội nghị với phân trang và lọc
- **Tham số**:
  - `conferenceFilter`: Bộ lọc theo các tiêu chí
  - `sortOptions`: Tùy chọn sắp xếp
- **Trả về**: `Promise<ConferencePaginationDTO>`

#### `getConferencesFromDB(conferenceFilter?, sortOptions?)`
- **Chức năng**: Truy vấn database để lấy dữ liệu hội nghị
- **Xử lý**:
  - Tạo cache key dựa trên filters và sort options
  - Kiểm tra dữ liệu trong cache trước
  - Nếu không có trong cache thì query database
  - Lưu kết quả vào cache

### Flow xử lý dữ liệu
1. Nhận request từ controller
2. Kiểm tra cache
3. Nếu có trong cache -> trả về kết quả
4. Nếu không có -> query database
5. Xử lý dữ liệu nếu cần
6. Lưu vào cache
7. Trả về kết quả

### Integration với external services
- **Prisma Service**: Tương tác với database
- **Redis Cache**: Lưu trữ tạm dữ liệu để tăng tốc độ truy vấn
- **Conference Organization Service**: Quản lý thông tin tổ chức
- **Recommendation Service**: Cung cấp các gợi ý liên quan

## 4. Ghi chú đặc biệt
- Sử dụng transaction cho các thao tác ghi dữ liệu quan trọng
- Có cơ chế cache để tối ưu hiệu năng
- Hỗ trợ phân trang và sắp xếp linh hoạt
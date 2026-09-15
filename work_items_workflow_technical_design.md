# TÀI LIỆU THIẾT KẾ KỸ THUẬT

# LÕI TIẾP NHẬN VÀ ĐIỀU PHỐI CÔNG VIỆC

## Lịch sử thay đổi

| Ngày       | Phiên bản | Nội dung                                                                                                                                                                                            |
| ---------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-09-11 | Pilot v1  | Thiết kế và triển khai local lõi `work-items`, camera gateway, chống trùng, giao việc và tích hợp mẫu với `urban-services`.                                                                         |
| 2026-09-14 | Pilot v2  | Thu hẹp lõi còn 5 bảng; bỏ timeline, bình luận và chi tiết nghiệp vụ chung. Mỗi phân hệ tiếp tục sở hữu toàn bộ dữ liệu và lịch sử xử lý riêng.                                                     |
| 2026-09-14 | Pilot v3  | Bỏ trạng thái vòng đời chung khỏi `work_items`; danh sách hiển thị trạng thái gốc của phân hệ và chỉ còn phạm vi Tất cả công việc/Việc của tôi.                                                     |
| 2026-09-15 | Pilot v4  | Refactor điểm tích hợp thành `WorkItemModuleRegistry` và adapter theo phân hệ; bỏ phụ thuộc trực tiếp `work-items -> urban-services`, bỏ giả định trạng thái chung và bổ sung `module_record_code`. |

## 1. Mục tiêu và phạm vi

`work-items` là lớp tiếp nhận và điều phối chung, không phải một phân hệ nghiệp vụ mới. Lõi chung chịu trách nhiệm:

- nhận dữ liệu từ camera, người dân và cán bộ;
- chuẩn hóa thông tin đầu vào;
- phát hiện và xử lý phản ánh nghi trùng;
- phân loại lĩnh vực, chọn phân hệ đích và giao việc;
- lưu trách nhiệm điều phối hiện tại và lịch sử giao/trả việc tối thiểu;
- nhận snapshot trạng thái gốc từ phân hệ để hiển thị danh sách chung mà không phải query nhiều bảng.

Lõi chung **không** chịu trách nhiệm:

- hiển thị hoặc quản lý chi tiết nghiệp vụ sau khi giao;
- quản lý timeline xử lý của phân hệ;
- quản lý bình luận, trao đổi nghiệp vụ;
- thay thế trạng thái, quy trình hoặc API hiện có của từng phân hệ;
- triển khai workflow động.

Pilot chỉ tích hợp hoàn chỉnh với `urban-services`. Bốn phân hệ còn lại chưa bị thay đổi trong giai đoạn này.

## 2. Ranh giới sở hữu dữ liệu

### 2.1. Lõi `work-items`

Lõi sở hữu dữ liệu đầu vào dùng chung, kết quả chống trùng, quyết định phân loại và thông tin điều phối. Lõi không sở hữu vòng đời trạng thái nghiệp vụ.

### 2.2. Phân hệ nghiệp vụ

Sau khi giao việc thành công, phân hệ đích là nguồn sự thật cho:

- hồ sơ chi tiết;
- trạng thái và quy trình chuyên ngành;
- timeline;
- bình luận;
- tài liệu, ảnh kết quả phát sinh trong quá trình xử lý;
- lịch sử thao tác và thông báo nghiệp vụ.

`urban-services` tiếp tục sử dụng nguyên trạng timeline, comment, assignment và API hiện có. Không backfill timeline/comment của phân hệ sang common và không tạo facade đọc ngược từ common.

## 3. Luồng nghiệp vụ

```text
Nguồn dữ liệu
  -> WorkItemIntakeService
  -> work_items + work_item_sources + work_item_attachments
  -> tạo gợi ý nghi trùng nếu đủ điều kiện
  -> cán bộ xác nhận/bỏ qua gợi ý
  -> phân loại lĩnh vực và phân hệ
  -> WorkItemAssignmentService
  -> adapter tạo hoặc tái sử dụng hồ sơ tại phân hệ
  -> phân hệ tạo assignment nghiệp vụ của chính nó
  -> common ghi work_item_assignments và module_status ban đầu của phân hệ
  -> người dùng mở màn hình phân hệ để xử lý
  -> phân hệ đồng bộ một chiều trạng thái gốc về work_items.module_status
```

Không tạo hồ sơ phân hệ ngay lúc tiếp nhận. Hồ sơ phân hệ chỉ được tạo lần đầu khi giao việc thành công. Nếu cán bộ cuối cùng từ chối, công việc quay lại trạng thái chưa giao nhưng giữ `module_record_id`; lần giao sau tái sử dụng hồ sơ đã có.

## 4. Mô hình dữ liệu chung

Sau quyết định Pilot v2, lõi có đúng **5 bảng**.

### 4.1. `work_items`

Một dòng đại diện cho một công việc chung. Đây là dữ liệu danh sách và báo cáo, không chứa chi tiết quy trình của phân hệ.

| Trường                      | Kiểu           | Bắt buộc | Ý nghĩa                                                                                      |
| --------------------------- | -------------- | -------- | -------------------------------------------------------------------------------------------- |
| `id`                        | `uuid`         | Có       | Khóa chính nội bộ.                                                                           |
| `code`                      | `varchar(50)`  | Có       | Mã công việc duy nhất để tra cứu.                                                            |
| `title`                     | `varchar(500)` | Có       | Tiêu đề chuẩn hóa.                                                                           |
| `description`               | `text`         | Có       | Nội dung đầu vào phục vụ phân loại và chống trùng. Không phải hồ sơ detail nghiệp vụ.        |
| `address`                   | `varchar(500)` | Không    | Địa chỉ mô tả.                                                                               |
| `latitude`                  | `decimal`      | Không    | Vĩ độ chuẩn hóa.                                                                             |
| `longitude`                 | `decimal`      | Không    | Kinh độ chuẩn hóa.                                                                           |
| `category_code`             | `varchar(50)`  | Không    | Lĩnh vực do adapter hoặc cán bộ xác nhận.                                                    |
| `module_code`               | `varchar(50)`  | Không    | Phân hệ nhận việc.                                                                           |
| `module_record_id`          | `uuid`         | Không    | Khóa hồ sơ đã tạo trong phân hệ.                                                             |
| `module_record_code`        | `varchar(100)` | Không    | Mã hiển thị do phân hệ đích sở hữu; không giả định trùng với `work_items.code`.              |
| `module_status`             | `varchar(50)`  | Không    | Snapshot trạng thái gốc gần nhất do phân hệ đồng bộ; không phải trạng thái riêng của common. |
| `priority`                  | `varchar(20)`  | Có       | `NORMAL`, `HIGH`, `URGENT`.                                                                  |
| `duplicate_of_work_item_id` | `uuid`         | Không    | Trỏ tới công việc gốc khi đã xác nhận trùng.                                                 |
| `created_at`                | `timestamptz`  | Có       | Thời gian tiếp nhận.                                                                         |
| `updated_at`                | `timestamptz`  | Có       | Thời gian cập nhật gần nhất.                                                                 |

`work_items` không có cột `status`. Trạng thái giao việc được suy ra từ `work_item_assignments`: có assignment người dùng còn hiệu lực là đã giao; không có là chưa giao. API chỉ trả `module_status` nguyên bản của phân hệ và `module_status_label` do adapter của phân hệ đó trả về. Khi chưa giao, `module_status = null` và nhãn trình bày là **Chưa giao**. Không có enum, mapping hoặc quy trình trạng thái dùng chung trong common.

### 4.2. `work_item_sources`

Lưu từng lần tiếp nhận tạo nên công việc. Một công việc có thể có nhiều nguồn khi nhiều camera/người dân/cán bộ phản ánh cùng sự kiện.

| Trường                                                | Kiểu           | Ý nghĩa                                       |
| ----------------------------------------------------- | -------------- | --------------------------------------------- |
| `id`, `work_item_id`                                  | `uuid`         | Định danh và liên kết công việc.              |
| `source_type`                                         | `varchar(30)`  | `CAMERA`, `CITIZEN`, `OFFICER`.               |
| `source_system`                                       | `varchar(100)` | Hệ thống gửi dữ liệu.                         |
| `source_reference`                                    | `varchar(255)` | Idempotency key duy nhất theo nguồn.          |
| `source_code`                                         | `varchar(100)` | Mã hiển thị từ nguồn nếu có.                  |
| `reporter_name`, `is_anonymous`                       | hỗn hợp        | Snapshot người phản ánh.                      |
| `camera_id`, `camera_name`, `task_type`, `confidence` | hỗn hợp        | Metadata camera đã chuẩn hóa.                 |
| `occurred_at`, `created_at`                           | `timestamptz`  | Thời điểm xảy ra và tiếp nhận.                |
| `raw_payload`                                         | `jsonb`        | Payload gốc có kiểm soát để đối soát adapter. |

Bảng này vẫn cần cho idempotency, chống trùng và thống kê nguồn dù màn Công việc không có detail.

### 4.3. `work_item_attachments`

Chỉ lưu ảnh/video/tệp **đầu vào** dùng cho tiếp nhận và chống trùng. Tài liệu kết quả sau xử lý thuộc phân hệ.

Các trường chính: `id`, `work_item_id`, `source_id`, `file_url` hoặc `object_key`, `thumbnail_url`, `media_type`, `checksum`, `note`, `created_at`. Mỗi attachment phải có đúng một vị trí lưu trữ hợp lệ.

### 4.4. `work_item_assignments`

Bảng lưu trách nhiệm điều phối và lịch sử giao/trả việc tối thiểu. Bảng này vẫn cần thiết để xác định người/phòng ban đang chịu trách nhiệm, hỗ trợ thống kê và đưa việc về hàng chưa giao khi bị từ chối.

| Trường                | Kiểu                   | Ý nghĩa                                                    |
| --------------------- | ---------------------- | ---------------------------------------------------------- |
| `id`, `work_item_id`  | `uuid`                 | Định danh và liên kết công việc.                           |
| `assignment_batch_id` | `uuid`                 | Gom các dòng của cùng một lần giao.                        |
| `department_id`       | `uuid`                 | Phòng ban nhận việc.                                       |
| `user_id`             | `uuid` nullable        | Cán bộ nhận việc; dòng phạm vi phòng ban có thể để `null`. |
| `assignment_source`   | `varchar(30)`          | `MANUAL`, `TRANSFER`, `MIGRATION`.                         |
| `assigned_by_user_id` | `uuid`                 | Người thực hiện giao việc.                                 |
| `assignee_snapshot`   | `jsonb`                | Snapshot tên phòng ban/cán bộ tại thời điểm giao.          |
| `note`                | `text`                 | Ghi chú giao việc.                                         |
| `assigned_at`         | `timestamptz`          | Thời điểm bắt đầu chịu trách nhiệm.                        |
| `released_at`         | `timestamptz` nullable | `null` nghĩa là assignment còn hiệu lực.                   |
| `released_by_user_id` | `uuid` nullable        | Người kết thúc assignment.                                 |
| `release_reason`      | `text` nullable        | Lý do trả/từ chối/chuyển việc.                             |

Không xóa dòng cũ khi chuyển hoặc từ chối; đặt `released_at` rồi tạo batch mới. Đây chỉ là audit điều phối, không thay thế timeline và assignment nghiệp vụ của phân hệ.

### 4.5. `work_item_merge_suggestions`

Lưu ứng viên nghi trùng và quyết định của cán bộ. Các trường chính gồm `id`, `candidate_work_item_id`, `suggested_master_work_item_id`, `score`, `distance_meters`, `time_difference_seconds`, `matched_signals`, `status`, người/thời gian xử lý và `resolution_note`.

Nếu một trong hai việc đã giao thì việc đã giao là gốc. Xác nhận trùng đặt `duplicate_of_work_item_id` cho việc phụ và đóng assignment còn hiệu lực; không ghi trạng thái `DUPLICATE` vào `work_items`. Bỏ qua chỉ đóng suggestion. Tách bản ghi trùng xóa liên kết gốc; assignment cũ không tự khôi phục.

## 5. Các bảng bị loại bỏ

- `work_item_timelines`.
- `work_item_comments`.

Migration điều chỉnh phải xóa hai bảng common này sau khi bảo đảm `urban-services` vẫn sử dụng bảng timeline/comment riêng. Không xóa, backfill ngược hoặc thay đổi dữ liệu lịch sử hiện có của phân hệ.

## 6. API common mục tiêu

| Method | Endpoint                           | Mục đích                                             |
| ------ | ---------------------------------- | ---------------------------------------------------- |
| `GET`  | `/work-items`                      | Danh sách và bộ lọc chung.                           |
| `POST` | `/work-items/classify`             | Xác nhận lĩnh vực/phân hệ/ưu tiên.                   |
| `POST` | `/work-items/assign`               | Giao việc qua adapter common.                        |
| `POST` | `/work-items/decline`              | Kết thúc trách nhiệm của cán bộ và trả việc khi cần. |
| `GET`  | `/work-items/duplicate-candidates` | Lấy đề xuất nghi trùng.                              |
| `POST` | `/work-items/duplicates/confirm`   | Xác nhận trùng.                                      |
| `POST` | `/work-items/duplicates/dismiss`   | Bỏ qua đề xuất.                                      |
| `POST` | `/work-items/duplicates/unlink`    | Tách khỏi công việc gốc.                             |
| `POST` | `/work-items/intake/camera`        | Cổng tiếp nhận camera.                               |

Không còn API common `/work-items/detail` và `/work-items/comments`. Danh sách không trả timeline, comment, attachment hoặc lịch sử xử lý. Sau khi giao, FE điều hướng sang màn hình phân hệ bằng `module_code` và định danh phân hệ.

### 6.1. Contract danh sách Pilot v3

`GET /work-items` nhận `page`, `limit`, `scope=ALL|MINE`, `module_code`, `source_type` và `search`.
`module_code=UNCLASSIFIED` lọc các việc chưa có phân hệ; `scope=MINE` chỉ lấy việc có assignment người dùng hiện tại còn hiệu lực. Không nhận bộ lọc trạng thái common.

Mỗi phần tử danh sách trả thêm dữ liệu trình bày đã có trong lõi:

- `description`, `address`, `category_name`;
- `module_record_id`, `module_record_code`;
- `sources[]` và `assignees[]` đang còn hiệu lực;
- `has_duplicate_suggestion`, `duplicate_of_work_item_id`, `duplicate_of_code`, `duplicate_count`;
- `deadline` được chiếu từ `intake_deadline`, `processing_deadline`, `completed_at`;
- `is_assigned`, `is_duplicate`, `is_rejected` để FE không phải đoán từ enum của nhiều phân hệ.

Response không có trường `status` hoặc `status_label`. `module_status` là trạng thái nguyên bản khi việc đang được giao và không ánh xạ sang enum vòng đời common. `module_status_label` do adapter của `module_code` trả về; trạng thái chưa biết giữ nguyên code để không hiển thị sai nghĩa.

Các yêu cầu `reject`, `reassign`, intake cán bộ, gộp trùng thủ công, adapter và danh mục của bốn phân hệ còn lại chưa thuộc pilot hiện tại. FE không được dùng sự tồn tại của mockup để coi các API này đã sẵn sàng.

## 7. Common service và adapter

- `WorkItemIntakeService`: tạo work item, source, attachment và chạy detector.
- `WorkItemDuplicateService`: xác nhận, bỏ qua và tách trùng.
- `WorkItemAssignmentService`: cổng duy nhất giao/từ chối việc.
- `WorkItemAssignmentLedgerService`: quản lý assignment common còn hiệu lực và lịch sử release.
- `WorkItemModuleProjectionService`: điểm ghi duy nhất cho snapshot danh sách. Nó chỉ đồng bộ `module_record_id`, `module_record_code`, `module_status`, hạn xử lý, thời điểm hoàn tất và kết quả/từ chối khi phân hệ yêu cầu; không diễn giải trạng thái.
- `WorkItemModuleRegistry`: chọn adapter theo `module_code`, lấy nhãn/trạng thái kết thúc theo adapter và lấy tên lĩnh vực để trả danh sách. Registry không import entity hay enum nghiệp vụ của phân hệ.
- `UrbanServicesWorkItemAdapter`: adapter mẫu, nằm trong module `urban-services`; tạo/tái sử dụng `urban_report`, đồng bộ assignment cũ, thực hiện duplicate/unlink tương thích và tự sở hữu mapping trạng thái Dịch vụ đô thị số.

Không còn `WorkItemTimelineService` và `WorkItemCommentService` trong lõi common.

### 7.1. Hợp đồng adapter khi ghép một phân hệ mới

Mỗi phân hệ tự sở hữu một class triển khai `WorkItemModuleAdapter`, đặt trong thư mục của chính phân hệ đó. Common chỉ gọi các hàm sau:

| Hàm                                    | Trách nhiệm của adapter                                                                                                              |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `assign(input, manager)`               | Convert dữ liệu common sang DTO/entity của phân hệ; tạo hoặc tái sử dụng hồ sơ, giao trách nhiệm theo logic hiện có và trả snapshot. |
| `releaseAssignment(input, manager)`    | Đồng bộ việc cán bộ từ chối/trả trách nhiệm về assignment của phân hệ.                                                               |
| `getStatusMetadata(status)`            | Trả nhãn hiển thị, `isTerminal`, `isRejected` cho **mã trạng thái nguyên bản** của riêng phân hệ.                                    |
| `getCategoryNames(codes)`              | Tùy chọn; trả tên lĩnh vực để màn danh sách không query trực tiếp bảng danh mục của phân hệ.                                         |
| `confirmDuplicate` / `unlinkDuplicate` | Tùy chọn; chỉ triển khai nếu phân hệ cần phản chiếu quyết định trùng common vào hồ sơ nội bộ.                                        |

Adapter trả `WorkItemModuleProjection` gồm `moduleRecordId`, `moduleRecordCode`, `moduleStatus`, `processingDeadline`, `completedAt` và các snapshot kết quả nếu có. `moduleStatus` được lưu nguyên chuỗi của phân hệ; ví dụ `DONE`, `pending`, `ESCALATED` đều hợp lệ. Lõi common tuyệt đối không so sánh trực tiếp các chuỗi này.

Để đăng ký một adapter, composition root `WorkItemsModule` import module nghiệp vụ, nhận adapter đã được module đó export và thêm vào provider `WORK_ITEM_MODULE_ADAPTERS`. Đây là thay đổi có chủ đích duy nhất ở common khi bổ sung module; không sửa `WorkItemAssignmentService`, `WorkItemDuplicateService` hoặc API common.

## 8. Tính nhất quán giao dịch

Khi chung database, tạo/tái sử dụng hồ sơ phân hệ, assignment phân hệ, assignment common và cập nhật snapshot phải nằm trong cùng transaction. Nếu giao việc tại phân hệ thất bại thì không được tạo assignment common hiệu lực và không cập nhật `module_status`.

Khi phân hệ cập nhật nghiệp vụ thành công, nó gọi `WorkItemModuleProjectionService.synchronize(...)` trong transaction phù hợp, truyền `workItemId`, `moduleCode` và snapshot. Lõi common không được ghi timeline/comment thay cho phân hệ. Nếu chưa tồn tại hoặc không còn liên kết cùng `module_record_id`, projection bị bỏ qua để không ghi đè sai hồ sơ.

## 9. Yêu cầu FE

Màn **Danh sách công việc** gồm một danh sách, bộ lọc phân hệ, tìm kiếm, phân loại, xử lý nghi trùng và giao việc. Không có KPI/tab/bộ lọc theo trạng thái common; không có drawer/trang detail nghiệp vụ, timeline hoặc bình luận chung.

- Tab **Tất cả công việc** gửi `scope=ALL`; tab **Việc của tôi** gửi `scope=MINE` và BE lọc theo assignment người dùng còn hiệu lực.

- Việc chưa giao: cho phép xử lý trùng, phân loại và giao việc.
- Việc đã giao: hành động chính là mở màn hình phân hệ.
- Badge dùng `module_status_label` từ BE. Với việc đã giao, `module_status/module_status_label` thuộc phân hệ; với việc chưa giao, `module_status = null` và nhãn **Chưa giao**.
- FE không suy luận trạng thái chuyên ngành và không ghép timeline từ nhiều nguồn.

## 10. Kế hoạch điều chỉnh pilot

1. Cập nhật migration local để xóa `work_item_timelines` và `work_item_comments`.
2. Khôi phục/giữ nguyên bảng timeline, comment và logic hiện có của `urban-services` nếu pilot cũ đã chuyển sang common.
3. Xóa entity, repository, service và provider common liên quan timeline/comment.
4. Xóa API `/work-items/detail`, `/work-items/comments` và DTO không còn sử dụng.
5. Loại bỏ việc ghi timeline common khỏi intake, classify, assignment và duplicate; thay bằng log kỹ thuật phù hợp khi cần.
6. Giữ `work_item_assignments` và logic release lịch sử.
7. Sửa FE bỏ detail sheet, timeline/comment chung; giữ modal/panel thao tác tối thiểu cho việc chưa giao.
8. Thêm migration mới xóa cột `work_items.status` và thay index hàng đợi bằng `module_code/module_status`.
9. Sửa API/FE bỏ stats, tab và filter trạng thái common; thêm phạm vi `ALL/MINE`.
10. Typecheck, chạy migration local và kiểm tra contract `urban-services` trước khi đưa lên dev.

## 11. Tiêu chí chấp nhận

- Schema common chỉ còn đúng 5 bảng đã nêu.
- Không còn code đọc/ghi timeline hoặc comment common.
- API hiện tại của `urban-services` về timeline, comment và xử lý nghiệp vụ không thay đổi contract.
- Tiếp nhận không tạo hồ sơ phân hệ; giao việc mới tạo hoặc tái sử dụng hồ sơ.
- Từ chối cán bộ cuối cùng đóng assignment common hiệu lực và đưa `module_status` về `null`; hồ sơ phân hệ vẫn được giữ để tái sử dụng.
- `work_items` không còn cột hoặc enum trạng thái vòng đời chung.
- Trạng thái phân hệ đồng bộ nguyên bản sang `module_status`; API danh sách ánh xạ nhãn theo từng phân hệ.
- Không còn import trực tiếp service/entity/enum của một phân hệ trong `work-items`; mọi thao tác materialize, trả việc, duplicate và nhãn trạng thái đi qua adapter/registry.
- Màn danh sách chỉ có phạm vi `ALL/MINE`, không quản lý theo trạng thái common.
- Chống trùng hoạt động xuyên nguồn và không tạo cây trùng nhiều cấp.
- Không chạy migration trên dev/production trong giai đoạn điều chỉnh; chỉ xác minh local.

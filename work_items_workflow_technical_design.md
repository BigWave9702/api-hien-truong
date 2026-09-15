# TÀI LIỆU THIẾT KẾ KỸ THUẬT

# LÕI TIẾP NHẬN VÀ ĐIỀU PHỐI CÔNG VIỆC

## Lịch sử thay đổi

| Ngày       | Phiên bản | Nội dung                                                                                                                                                                                                                                                                                 |
| ---------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-09-11 | Pilot v1  | Thiết kế và triển khai local lõi `work-items`, camera gateway, chống trùng, giao việc và tích hợp mẫu với `urban-services`.                                                                                                                                                              |
| 2026-09-14 | Pilot v2  | Thu hẹp lõi còn 5 bảng; bỏ timeline, bình luận và chi tiết nghiệp vụ chung. Mỗi phân hệ tiếp tục sở hữu toàn bộ dữ liệu và lịch sử xử lý riêng.                                                                                                                                          |
| 2026-09-14 | Pilot v3  | Bỏ trạng thái vòng đời chung khỏi `work_items`; danh sách hiển thị trạng thái gốc của phân hệ và chỉ còn phạm vi Tất cả công việc/Việc của tôi.                                                                                                                                          |
| 2026-09-15 | Pilot v4  | Refactor điểm tích hợp thành `WorkItemModuleRegistry` và adapter theo phân hệ; bỏ phụ thuộc trực tiếp `work-items -> urban-services` và bổ sung `module_record_code`.                                                                                                                    |
| 2026-09-15 | v5        | Lõi trở thành cổng tiếp nhận duy nhất. Một công việc có nhiều nguồn: tách chống trùng thành ba mức (gửi lặp / tương quan tự động / soát trùng thủ công). Bổ sung hợp đồng adapter `canAccept` - `create` - `attach` - `appendSource` - `listStatuses` và bộ lọc trạng thái theo phân hệ. |
| 2026-09-15 | v5.1      | Đã triển khai `POST /work-items/intake`, `GET /work-items/module-statuses`, bộ lọc `module_status`, kiểm tra `canAccept` ngay khi phân loại, API đọc/tách source và chuyển source/attachment khi xác nhận trùng. Urban Services ghi timeline khởi tạo khi hồ sơ được tạo từ Work Item.   |
| 2026-09-15 | v5.2      | Hoàn thiện contract intake với attachment và giới hạn dữ liệu đầu vào; response camera phản ánh assignment thực tế. Chặn tách source khi chỉ có một nguồn, còn assignment hoặc đã materialize vào phân hệ. Khi hợp nhất nguồn, evidence được chuyển tiếp sang adapter của hồ sơ gốc. |

## 1. Mục tiêu và phạm vi

`work-items` là lớp tiếp nhận và điều phối chung, không phải một phân hệ nghiệp vụ mới.

Từ v5, lõi là **cổng tiếp nhận duy nhất** của toàn hệ thống. Mọi dữ liệu từ camera, người dân và cán bộ đều vào `work_items` trước, sau đó mới tới phân hệ nghiệp vụ. Các cổng tiếp nhận riêng hiện có của từng phân hệ sẽ được gỡ bỏ theo lộ trình ở mục 13.

Lõi chung chịu trách nhiệm:

- nhận dữ liệu từ mọi nguồn và chuẩn hóa thông tin đầu vào;
- bảo đảm một sự việc có thật ngoài đời tương ứng với đúng một công việc, dù được phản ánh nhiều lần từ nhiều nguồn;
- phân loại lĩnh vực, chọn phân hệ đích và giao việc;
- lưu trách nhiệm điều phối hiện tại và lịch sử giao/trả việc tối thiểu;
- giữ danh sách đầy đủ người phản ánh để phản hồi khi xử lý xong;
- nhận snapshot trạng thái gốc từ phân hệ để hiển thị danh sách chung mà không phải query nhiều bảng.

Lõi chung **không** chịu trách nhiệm:

- hiển thị hoặc quản lý chi tiết nghiệp vụ sau khi giao;
- quản lý timeline, bình luận, trao đổi nghiệp vụ;
- thay thế trạng thái, quy trình hoặc API hiện có của từng phân hệ;
- triển khai workflow động.

## 2. Ranh giới sở hữu dữ liệu

### 2.1. Lõi `work-items`

Lõi sở hữu dữ liệu đầu vào dùng chung, danh sách nguồn phản ánh, kết quả chống trùng, quyết định phân loại và thông tin điều phối. Lõi không sở hữu vòng đời trạng thái nghiệp vụ.

### 2.2. Phân hệ nghiệp vụ

Sau khi giao việc thành công, phân hệ đích là nguồn sự thật cho hồ sơ chi tiết, trạng thái và quy trình chuyên ngành, timeline, bình luận, tài liệu và ảnh kết quả phát sinh trong quá trình xử lý, lịch sử thao tác và thông báo nghiệp vụ.

Không backfill timeline/comment của phân hệ sang common và không tạo facade đọc ngược từ common.

## 3. Ba mức chống trùng

Đây là thay đổi cốt lõi của v5. Trước đây hệ thống gộp hai việc khác bản chất - _một sự việc được phản ánh nhiều lần_ và _một công việc thừa cần loại bỏ_ - vào cùng một cơ chế `duplicate_of_work_item_id`. Hệ quả là camera báo lại một sự cố đang kéo dài sẽ sinh ra hàng chục công việc, mỗi cái cần một thao tác tay để đóng lại.

Từ v5, ba mức được tách bạch:

| Mức               | Cơ chế nhận biết                           | Ai quyết định | Kết quả                                                                        |
| ----------------- | ------------------------------------------ | ------------- | ------------------------------------------------------------------------------ |
| **1. Gửi lặp**    | Trùng `(source_system, source_reference)`  | Máy           | Trả về công việc đã có, không ghi thêm gì.                                     |
| **2. Tương quan** | Khóa tương quan **xác định**, việc còn mở  | Máy           | Thêm một dòng `work_item_sources` vào công việc đang có. Không tạo việc mới.   |
| **3. Soát trùng** | Tương đồng **mờ** về cự ly, thời gian, chữ | Người         | Gợi ý trong `work_item_merge_suggestions`; cán bộ xác nhận thì hợp nhất nguồn. |

### 3.1. Quy tắc bắt buộc của mức 2

Khóa tương quan **phải xác định**. Tuyệt đối không dùng độ tương đồng văn bản, bán kính địa lý hay bất kỳ ngưỡng nào để tự động gộp. Tương đồng mờ luôn thuộc mức 3 và luôn cần người duyệt. Vi phạm quy tắc này sẽ khiến hai sự cố khác nhau tại cùng một camera bị nhập làm một mà không ai phát hiện.

Khóa tương quan theo loại nguồn:

- `CAMERA`: `camera_id` + `task_type`. Đây là tín hiệu máy, lặp lại có chu kỳ, và hai giá trị này do hệ thống nguồn cấp nên không mơ hồ.
- `CITIZEN`, `OFFICER`, `HOTLINE`: **không có** khóa tương quan. Mô tả do người viết tự do nên không thể so khớp xác định. Các nguồn này luôn tạo công việc mới và chỉ được hợp nhất qua mức 3.

Cửa sổ tương quan bị chặn bởi trạng thái, không phải bởi thời gian: chỉ gắn thêm nguồn vào công việc **còn mở**. Khi công việc kết thúc tại phân hệ, khóa tương quan được giải phóng; lần camera báo tiếp theo là một sự cố mới, không phải nguồn bổ sung của sự cố cũ.

### 3.2. Xác nhận trùng ở mức 3 là hợp nhất nguồn

Khi cán bộ xác nhận một gợi ý trùng, hệ thống **chuyển toàn bộ `work_item_sources` và `work_item_attachments` của việc phụ sang việc gốc**, chứ không chỉ gắn nhãn.

Việc phụ được giữ lại làm **bia mộ**: dòng `work_items` vẫn tồn tại với `duplicate_of_work_item_id` trỏ tới việc gốc, nhưng không còn nguồn nào và không bao giờ xuất hiện trong hàng đợi. Lý do giữ lại: mã công việc của nó có thể đã được trả cho người dân để tra cứu, xóa đi là làm hỏng mã đó. Tra cứu theo mã bia mộ phải chuyển hướng sang việc gốc.

Không hỗ trợ trùng nhiều cấp: việc gốc phải là việc chưa từng bị gắn bia mộ.

### 3.3. Tách nguồn

Tương quan tự động có thể sai. Phải có thao tác rút một `work_item_source` ra khỏi công việc để tạo công việc mới độc lập. `work_item_sources.origin_work_item_id` ghi lại công việc mà nguồn đó thuộc về lần đầu, phục vụ đối soát khi tách.

Tách nguồn khác với tách bia mộ. Tách bia mộ khôi phục một công việc đã bị hợp nhất ở mức 3; tách nguồn xử lý sai sót của mức 2.

## 4. Luồng nghiệp vụ

```text
Camera / Người dân / Cán bộ
  -> WorkItemIntakeService
       |- trùng (source_system, source_reference)? -> trả về nguyên trạng
       |- có khóa tương quan và công việc còn mở?  -> thêm work_item_sources vào việc đó
       |- còn lại                                   -> tạo work_items + sources + attachments
  -> chạy dò nghi trùng mức 3 nếu là công việc mới
  -> cán bộ phân loại lĩnh vực và phân hệ đích
  -> WorkItemAssignmentService
       |- adapter.canAccept() -> thiếu dữ liệu thì chặn ngay, báo rõ thiếu gì
       |- adapter.create() hoặc adapter.attach() -> hồ sơ tại phân hệ
  -> phân hệ tạo assignment nghiệp vụ của chính nó
  -> common ghi work_item_assignments và module_status ban đầu
  -> nguồn đến sau khi đã giao -> adapter.appendSource()
  -> phân hệ đồng bộ một chiều trạng thái gốc về work_items.module_status
  -> phân hệ xử lý xong -> phản hồi mọi người phản ánh trong work_item_sources
```

Không tạo hồ sơ phân hệ lúc tiếp nhận. Hồ sơ phân hệ chỉ được tạo lần đầu khi giao việc thành công. Nếu cán bộ cuối cùng từ chối, công việc quay lại trạng thái chưa giao nhưng giữ `module_record_id`; lần giao sau tái sử dụng hồ sơ đã có.

## 5. Mô hình dữ liệu chung

Lõi có đúng **5 bảng**.

### 5.1. `work_items`

Một dòng đại diện cho một sự việc có thật, bất kể được phản ánh bao nhiêu lần.

| Trường                      | Kiểu            | Bắt buộc | Ý nghĩa                                                                      |
| --------------------------- | --------------- | -------- | ---------------------------------------------------------------------------- |
| `id`                        | `uuid`          | Có       | Khóa chính nội bộ.                                                           |
| `code`                      | `varchar(50)`   | Có       | Mã công việc duy nhất để tra cứu. Có thể đã được trả cho người dân.          |
| `title`                     | `varchar(500)`  | Có       | Tiêu đề chuẩn hóa, lấy từ nguồn đầu tiên.                                    |
| `description`               | `text`          | Có       | Nội dung đầu vào phục vụ phân loại và chống trùng.                           |
| `address`                   | `varchar(500)`  | Không    | Địa chỉ mô tả.                                                               |
| `latitude`, `longitude`     | `numeric(10,7)` | Không    | Tọa độ chuẩn hóa.                                                            |
| `category_code`             | `varchar(100)`  | Không    | Mã lĩnh vực trong danh mục của phân hệ đích.                                 |
| `module_payload`            | `jsonb`         | Không    | Dữ liệu bắt buộc riêng của phân hệ mà một mã lĩnh vực không chở nổi.         |
| `module_code`               | `varchar(50)`   | Không    | Phân hệ nhận việc.                                                           |
| `module_record_id`          | `varchar(100)`  | Không    | Khóa hồ sơ trong phân hệ, lưu dạng chuỗi.                                    |
| `module_record_code`        | `varchar(100)`  | Không    | Mã hiển thị do phân hệ đích sở hữu.                                          |
| `module_status`             | `varchar(50)`   | Không    | Snapshot trạng thái gốc gần nhất do phân hệ đồng bộ.                         |
| `priority`                  | `varchar(20)`   | Có       | `NORMAL`, `HIGH`, `URGENT`.                                                  |
| `correlation_key`           | `varchar(200)`  | Không    | Khóa tương quan mức 2. `null` với nguồn không tương quan được.               |
| `correlation_closed_at`     | `timestamptz`   | Không    | Thời điểm khóa tương quan hết hiệu lực. `null` nghĩa là còn nhận thêm nguồn. |
| `source_count`              | `integer`       | Có       | Số nguồn đang thuộc công việc. Mặc định 1.                                   |
| `first_reported_at`         | `timestamptz`   | Có       | Thời điểm nguồn đầu tiên xảy ra.                                             |
| `last_reported_at`          | `timestamptz`   | Có       | Thời điểm nguồn gần nhất xảy ra.                                             |
| `duplicate_of_work_item_id` | `uuid`          | Không    | Khác `null` nghĩa là dòng này là bia mộ, nội dung đã chuyển sang việc gốc.   |
| `created_at`, `updated_at`  | `timestamptz`   | Có       | Thời gian tiếp nhận và cập nhật gần nhất.                                    |

`work_items` không có cột `status`. Trạng thái giao việc suy ra từ `work_item_assignments`: có assignment người dùng còn hiệu lực là đã giao, không có là chưa giao. Không có enum, mapping hoặc quy trình trạng thái dùng chung trong common.

Ràng buộc bắt buộc: chỉ mục duy nhất từng phần trên `correlation_key` với điều kiện `correlation_key IS NOT NULL AND correlation_closed_at IS NULL`. Ràng buộc này là thứ bảo đảm mức 2 không bao giờ sinh ra hai công việc mở cho cùng một khóa, kể cả khi nhiều tiến trình tiếp nhận chạy song song.

`module_record_id` giữ nguyên tên nhưng đổi kiểu từ `uuid` sang `varchar(100)`, vì khóa chính của các phân hệ không đồng kiểu: `sanitation_waste_point_reports.id` là `integer`, các bảng còn lại là `uuid`. Giữ nguyên tên cột và tên trường API để FE và các consumer hiện có không phải sửa. Xem mục 12.

### 5.2. `work_item_sources`

Mỗi lần tiếp nhận là một dòng. **Một công việc có nhiều nguồn.** Đây là quan hệ N-1 thật sự, không phải 1-1 như cách triển khai trước v5.

| Trường                                                                | Kiểu            | Ý nghĩa                                                                    |
| --------------------------------------------------------------------- | --------------- | -------------------------------------------------------------------------- |
| `id`, `work_item_id`                                                  | `uuid`          | Định danh và liên kết công việc đang sống.                                 |
| `origin_work_item_id`                                                 | `uuid` nullable | Công việc mà nguồn này thuộc về lần đầu, nếu đã bị chuyển do hợp nhất.     |
| `source_type`                                                         | `varchar(30)`   | `CAMERA`, `CITIZEN`, `OFFICER`, `HOTLINE`, `INTEGRATION`.                  |
| `source_system`                                                       | `varchar(100)`  | Hệ thống gửi dữ liệu.                                                      |
| `source_reference`                                                    | `varchar(200)`  | Idempotency key duy nhất theo nguồn.                                       |
| `source_code`                                                         | `varchar(100)`  | Mã hiển thị từ nguồn nếu có.                                               |
| `reporter_name`, `reporter_phone`, `reporter_user_id`, `is_anonymous` | hỗn hợp         | Snapshot người phản ánh, dùng để phản hồi khi xử lý xong.                  |
| `camera_id`, `camera_name`, `task_type`, `model_id`, `confidence`     | hỗn hợp         | Metadata camera đã chuẩn hóa.                                              |
| `occurred_at`                                                         | `timestamptz`   | Thời điểm sự việc xảy ra tại nguồn. Bắt buộc; thiếu thì lấy lúc tiếp nhận. |
| `created_at`                                                          | `timestamptz`   | Thời điểm tiếp nhận.                                                       |
| `adapter_name`, `adapter_version`                                     | `varchar`       | Bộ chuyển đổi đã tạo dòng này.                                             |
| `raw_payload`                                                         | `jsonb`         | Payload gốc có kiểm soát để đối soát adapter.                              |

Nguồn đến sau **không được ghi đè** `title`, `description`, `address` của công việc. Chi tiết riêng của từng lần phản ánh nằm lại ở dòng nguồn. Nguồn đến sau chỉ cập nhật `source_count` và `last_reported_at`.

### 5.3. `work_item_attachments`

Chỉ lưu ảnh/video/tệp **đầu vào** dùng cho tiếp nhận và chống trùng. Tài liệu kết quả sau xử lý thuộc phân hệ.

Các trường chính: `id`, `work_item_id`, `source_id`, `file_url` hoặc `object_key`, `thumbnail_url`, `media_type`, `captured_at`, `note`, `created_at`. Mỗi attachment phải có đúng một vị trí lưu trữ hợp lệ.

Khi hợp nhất hoặc tách, attachment đi theo `source_id` của nó.

### 5.4. `work_item_assignments`

Lưu trách nhiệm điều phối và lịch sử giao/trả việc tối thiểu.

| Trường                | Kiểu                   | Ý nghĩa                                             |
| --------------------- | ---------------------- | --------------------------------------------------- |
| `id`, `work_item_id`  | `uuid`                 | Định danh và liên kết công việc.                    |
| `assignment_batch_id` | `uuid`                 | Gom các dòng của cùng một lần giao.                 |
| `department_id`       | `uuid`                 | Phòng ban nhận việc.                                |
| `user_id`             | `uuid` nullable        | Cán bộ nhận việc; dòng phạm vi phòng ban để `null`. |
| `assignment_source`   | `varchar(30)`          | `MANUAL`, `TRANSFER`, `MODULE_API`, `MIGRATION`.    |
| `assigned_by_user_id` | `uuid`                 | Người thực hiện giao việc.                          |
| `assignee_snapshot`   | `jsonb`                | Snapshot tên phòng ban/cán bộ tại thời điểm giao.   |
| `assigned_at`         | `timestamptz`          | Thời điểm bắt đầu chịu trách nhiệm.                 |
| `accepted_at`         | `timestamptz` nullable | Thời điểm cán bộ tiếp nhận.                         |
| `released_at`         | `timestamptz` nullable | `null` nghĩa là assignment còn hiệu lực.            |
| `release_reason`      | `text` nullable        | Lý do trả/từ chối/chuyển việc.                      |

Không xóa dòng cũ khi chuyển hoặc từ chối; đặt `released_at` rồi tạo batch mới.

Số lượng người nhận do adapter quyết định, không do lõi áp đặt. `adapter.canAccept()` khai báo `maxAssignees`; lõi kiểm tra trước khi gọi `create`/`attach`. Lý do: `urban_reports` hỗ trợ nhiều phòng ban và nhiều cán bộ, trong khi `flood_events.assigned_user_id`, `market_incidents.assigned_to_user_id` và `urban_order_events.received_by_id` mỗi bảng chỉ có một người. Nếu lõi cứ gửi nhiều người thì phần dư bị mất im lặng.

### 5.5. `work_item_merge_suggestions`

Lưu ứng viên nghi trùng mức 3 và quyết định của cán bộ: `id`, `work_item_id`, `suggested_master_id`, `score`, `distance_meters`, `time_difference_seconds`, `matched_signals`, `algorithm_version`, `status`, người/thời gian xử lý và `resolution_note`.

Nếu một trong hai việc đã giao thì việc đã giao là gốc. Xác nhận thì hợp nhất nguồn theo mục 3.2. Bỏ qua chỉ đóng suggestion.

Chỉ tạo gợi ý cho công việc **mới**. Nguồn được gắn thêm qua mức 2 không kích hoạt dò trùng, vì nó đã thuộc đúng công việc rồi.

## 6. Hợp đồng adapter

Mỗi phân hệ sở hữu một class triển khai `WorkItemModuleAdapter`, đặt trong thư mục của chính phân hệ đó. Lõi chỉ gọi các hàm dưới đây và không import entity, enum hay service nghiệp vụ của bất kỳ phân hệ nào.

| Hàm                                                    | Bắt buộc | Trách nhiệm                                                                                                 |
| ------------------------------------------------------ | -------- | ----------------------------------------------------------------------------------------------------------- |
| `moduleCode`                                           | Có       | Mã phân hệ mà adapter phục vụ.                                                                              |
| `canAccept(workItem)`                                  | Có       | Trả `{ ok, missing[], maxAssignees }`. Khai báo phân hệ cần thêm dữ liệu gì và nhận tối đa bao nhiêu người. |
| `create(input, manager)`                               | Có       | Tạo hồ sơ mới tại phân hệ và trả snapshot.                                                                  |
| `attach(workItem, recordRef, manager)`                 | Có       | Gắn công việc vào hồ sơ **đã tồn tại** tại phân hệ và trả snapshot.                                         |
| `appendSource(workItem, source, attachments, manager)` | Có       | Đẩy một nguồn đến sau sang hồ sơ đã giao.                                                                   |
| `listStatuses()`                                       | Có       | Liệt kê toàn bộ trạng thái nguyên bản của phân hệ kèm nhãn, để FE đổ option bộ lọc.                         |
| `getStatusMetadata(status)`                            | Có       | Trả nhãn hiển thị, `isTerminal`, `isRejected` cho một mã trạng thái nguyên bản.                             |
| `releaseAssignment(input, manager)`                    | Có       | Đồng bộ việc cán bộ từ chối hoặc trả trách nhiệm về assignment của phân hệ.                                 |
| `getCategoryNames(codes)`                              | Không    | Trả tên lĩnh vực để màn danh sách không query bảng danh mục của phân hệ.                                    |
| `confirmDuplicate` / `unlinkDuplicate`                 | Không    | Phản chiếu quyết định hợp nhất của lõi vào hồ sơ nội bộ nếu phân hệ cần.                                    |

`canAccept` là hàm quan trọng nhất của v5. Nó phải được gọi ở bước **phân loại**, không phải bước giao việc. Nếu chỉ kiểm tra lúc giao, cán bộ sẽ phân loại xong rồi mới biết phân hệ đích không nhận được, và công việc kẹt lại không đi tiếp được.

`appendSource` là thứ ngăn dữ liệu làm giàu chỉ giàu ở lõi. Không có nó, ảnh và phản ánh đến sau khi giao sẽ vô hình với cán bộ đang xử lý.

`attach` là thứ cho phép nối một công việc vào hồ sơ có sẵn. Không có nó, lõi chỉ biết tạo mới, và mọi hồ sơ đã tồn tại tại phân hệ sẽ vĩnh viễn không có công việc tương ứng.

Adapter trả `WorkItemModuleProjection` gồm `moduleRecordId`, `moduleRecordCode`, `moduleStatus`, `processingDeadline`, `completedAt` và snapshot kết quả nếu có. `moduleStatus` lưu nguyên chuỗi của phân hệ; `DONE`, `pending`, `ESCALATED` đều hợp lệ. Lõi tuyệt đối không so sánh trực tiếp các chuỗi này.

Để đăng ký, `WorkItemsModule` import module nghiệp vụ, nhận adapter đã được module đó export và thêm vào provider `WORK_ITEM_MODULE_ADAPTERS`. Đây là thay đổi có chủ đích duy nhất ở lõi khi bổ sung phân hệ.

## 7. Common service

- `WorkItemIntakeService`: thực thi ba mức chống trùng ở mục 3; tạo hoặc gắn thêm nguồn, attachment và cập nhật roll-up.
- `WorkItemCorrelationService`: dựng khóa tương quan theo loại nguồn và tra công việc còn mở theo khóa đó.
- `WorkItemDuplicateDetectorService`: chỉ sinh gợi ý mức 3, không bao giờ tự thay đổi liên kết.
- `WorkItemMergeService`: hợp nhất nguồn, tách bia mộ và tách nguồn.
- `WorkItemAssignmentService`: cổng duy nhất giao và từ chối việc.
- `WorkItemAssignmentLedgerService`: điểm ghi duy nhất vào `work_item_assignments`.
- `WorkItemModuleProjectionService`: điểm ghi duy nhất cho snapshot danh sách.
- `WorkItemModuleRegistry`: chọn adapter theo `module_code` và tổng hợp nhãn, trạng thái.

## 8. API

| Method | Endpoint                           | Mục đích                                                 |
| ------ | ---------------------------------- | -------------------------------------------------------- |
| `POST` | `/work-items/intake`               | Cổng tiếp nhận duy nhất cho mọi nguồn.                   |
| `GET`  | `/work-items`                      | Danh sách và bộ lọc.                                     |
| `GET`  | `/work-items/module-statuses`      | Danh sách trạng thái của một phân hệ để FE đổ option.    |
| `POST` | `/work-items/classify`             | Xác nhận lĩnh vực, phân hệ, ưu tiên và `module_payload`. |
| `POST` | `/work-items/assign`               | Giao việc qua adapter.                                   |
| `POST` | `/work-items/decline`              | Kết thúc trách nhiệm của cán bộ và trả việc khi cần.     |
| `GET`  | `/work-items/sources`              | Danh sách nguồn của một công việc.                       |
| `POST` | `/work-items/sources/split`        | Tách một nguồn thành công việc mới.                      |
| `GET`  | `/work-items/duplicate-candidates` | Lấy gợi ý nghi trùng mức 3.                              |
| `POST` | `/work-items/duplicates/confirm`   | Xác nhận trùng và hợp nhất nguồn.                        |
| `POST` | `/work-items/duplicates/dismiss`   | Bỏ qua gợi ý.                                            |
| `POST` | `/work-items/duplicates/unlink`    | Tách khỏi công việc gốc.                                 |

Không có API common `/work-items/detail` và `/work-items/comments`. Danh sách không trả timeline, comment hoặc lịch sử xử lý. Sau khi giao, FE điều hướng sang màn hình phân hệ bằng `module_code` và `module_record_id`.

### 8.1. Contract danh sách

`GET /work-items` nhận `page`, `limit`, `scope=ALL|MINE`, `module_code`, `module_status`, `source_type` và `search`.

- `module_code=UNCLASSIFIED` lọc các việc chưa có phân hệ.
- `module_status` **chỉ hợp lệ khi có `module_code`** là một phân hệ cụ thể; gửi thiếu `module_code` trả 400. Lý do: trạng thái thuộc về phân hệ, không có nghĩa khi đứng một mình.
- `module_status=UNASSIGNED` là giá trị đặc biệt, lọc các việc chưa giao trong phân hệ đó.
- `scope=MINE` chỉ lấy việc có assignment người dùng hiện tại còn hiệu lực.
- Mặc định danh sách ẩn bia mộ; muốn xem thì gửi tham số riêng.

Mỗi phần tử trả thêm: `description`, `address`, `category_name`, `module_record_id`, `module_record_code`, `sources[]`, `assignees[]` còn hiệu lực, `source_count`, `first_reported_at`, `last_reported_at`, `has_duplicate_suggestion`, `duplicate_of_work_item_id`, `duplicate_of_code`, `deadline`, `is_assigned`, `is_duplicate`, `is_rejected`, `priority`, `created_at`.

Response không có trường `status` hoặc `status_label`. `module_status_label` do adapter của `module_code` trả về; trạng thái chưa biết giữ nguyên code để không hiển thị sai nghĩa.

### 8.2. Contract danh sách trạng thái

`GET /work-items/module-statuses?module_code=...` trả danh sách `{ value, label, is_terminal }` lấy từ `adapter.listStatuses()`, cộng thêm mục `UNASSIGNED` với nhãn **Chưa giao**.

FE không được hardcode enum trạng thái của bất kỳ phân hệ nào.

## 9. Tính nhất quán giao dịch

Khi chung database, việc tạo hoặc gắn hồ sơ phân hệ, assignment phân hệ, assignment common và cập nhật snapshot phải nằm trong cùng transaction. Nếu thao tác tại phân hệ thất bại thì không được tạo assignment common hiệu lực và không cập nhật `module_status`.

Tiếp nhận mức 2 phải chạy trong transaction có khóa, và phải dựa vào chỉ mục duy nhất từng phần ở mục 5.1 làm lớp bảo vệ cuối. Khóa ứng dụng một mình không đủ khi có nhiều tiến trình.

Hợp nhất nguồn phải chuyển nguồn, chuyển attachment, cập nhật roll-up của cả hai bên, đóng assignment của việc phụ và đóng suggestion trong cùng một transaction.

Khi phân hệ cập nhật nghiệp vụ thành công, nó gọi `WorkItemModuleProjectionService.synchronize(...)` trong transaction phù hợp. Nếu không còn liên kết cùng `module_record_id`, projection bị bỏ qua để không ghi đè sai hồ sơ.

## 10. Yêu cầu FE

Màn **Danh sách công việc** gồm một danh sách, bộ lọc phân hệ, bộ lọc trạng thái theo phân hệ, tìm kiếm, phân loại, xử lý nghi trùng và giao việc.

- Tab **Tất cả công việc** gửi `scope=ALL`; tab **Việc của tôi** gửi `scope=MINE`.
- Bộ lọc trạng thái **bị vô hiệu hóa** khi chưa chọn phân hệ. Khi chọn một phân hệ, FE gọi `GET /work-items/module-statuses` để đổ option.
- Việc chưa giao: cho phép xử lý trùng, phân loại và giao việc.
- Việc đã giao: hành động chính là mở màn hình phân hệ.
- Badge dùng `module_status_label` từ BE; việc chưa giao hiển thị **Chưa giao**.
- Một công việc hiển thị `source_count` và danh sách nguồn. Nhiều nguồn là dấu hiệu sự việc được nhiều người xác nhận, không phải lỗi dữ liệu.
- FE không suy luận trạng thái chuyên ngành và không hardcode enum của phân hệ.

## 11. Nghĩa vụ phản hồi người phản ánh

Đây là yêu cầu mới của v5, trước đây chưa được ghi ở đâu.

Khi một công việc được xử lý xong tại phân hệ, **mọi người dân đã phản ánh về sự việc đó đều phải được trả lời**, không chỉ người phản ánh đầu tiên. Danh sách này là toàn bộ `work_item_sources` của công việc có `source_type` thuộc nhóm người dân và có thông tin liên hệ.

Đây chính là lý do thực tế để hợp nhất nguồn thay vì gắn nhãn trùng: sau khi hợp nhất, câu truy vấn là một lệnh trên `work_item_sources`; nếu để rải rác ở các công việc bị gắn nhãn thì phải đi vòng qua bảng con và rất dễ bỏ sót.

Lõi cung cấp danh sách; việc phát thông báo thuộc về module thông báo. Cần chốt chủ sở hữu trước khi triển khai.

## 12. Quyết định cần chốt

| #   | Quyết định                                    | Đề xuất                                                                             | Lý do                                                                                                                                                                                                                                                                            |
| --- | --------------------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Kiểu của khóa hồ sơ phân hệ                   | **Đã chốt**: giữ nguyên tên `module_record_id`, đổi kiểu `uuid` sang `varchar(100)` | `sanitation_waste_point_reports.id` là `integer`, không lưu được vào cột `uuid`. Giữ `uuid` thì Vệ sinh môi trường vĩnh viễn không ghép được, mà đây lại là một trong hai đích chính của luồng camera IOC. Giữ nguyên tên cột và tên trường API để không phá contract đang chạy. |
| 2   | Chủ sở hữu việc phản hồi người phản ánh       | Module thông báo                                                                    | Lõi không nên tự phát thông báo cho người dân.                                                                                                                                                                                                                                   |
| 3   | Thời điểm gỡ cổng tiếp nhận riêng của phân hệ | Sau khi adapter của phân hệ đó đã chạy                                              | Xem mục 13.                                                                                                                                                                                                                                                                      |

## 13. Kế hoạch triển khai

Thứ tự này bắt buộc. Đảo thứ tự sẽ làm đứt dữ liệu của các phân hệ.

1. Viết migration đổi kiểu `work_items.module_record_id` từ `uuid` sang `varchar(100)`, giữ nguyên tên cột và chỉ mục duy nhất `(module_code, module_record_id)`.
2. Bổ sung `correlation_key`, `correlation_closed_at`, `source_count`, `first_reported_at`, `last_reported_at`, `module_payload` vào `work_items`; bổ sung `origin_work_item_id` vào `work_item_sources`; tạo chỉ mục duy nhất từng phần cho khóa tương quan.
3. Mở rộng `WorkItemModuleAdapter` theo mục 6 và cập nhật `UrbanServicesWorkItemAdapter` cho khớp. Đây là adapter mẫu mà bốn phân hệ còn lại sẽ nhìn vào.
4. Triển khai `WorkItemCorrelationService` và sửa `WorkItemIntakeService` theo ba mức ở mục 3.
5. Chuyển `confirm` sang hợp nhất nguồn; bổ sung tách nguồn.
6. Bổ sung `module_status` vào bộ lọc danh sách và endpoint `module-statuses`.
7. Hoàn thiện cổng tiếp nhận: giới hạn độ dài mọi trường chuỗi theo đúng kích thước cột, bắt buộc có `occurred_at`.
8. **Chỉ tới bước này** mới bàn giao cho bốn phân hệ còn lại viết adapter của mình. Lõi đã đúng và ổn định thì phân hệ mới sửa một lần là xong.
9. Sau khi adapter của một phân hệ chạy được, mới gỡ cổng tiếp nhận camera riêng của phân hệ đó.
10. Chuyển `ioc-camera-events` sang gọi `WorkItemIntakeService`; nhánh sự kiện không nhận diện được phải tạo công việc **Chưa phân loại** thay vì bỏ đi.
11. Gỡ webhook camera riêng của Markets và Sanitation sau khi bước 10 chạy ổn định.

Bước 1 đến 7 nằm trọn trong `src/modules/work-items` và không đụng code của phân hệ khác.

## 14. Tiêu chí chấp nhận

- Schema common chỉ còn đúng 5 bảng đã nêu.
- Không còn code đọc hoặc ghi timeline, comment trong lõi.
- Không còn import trực tiếp service, entity, enum của một phân hệ trong `work-items`.
- Camera báo lại cùng `camera_id` và `task_type` khi việc còn mở chỉ sinh thêm một dòng `work_item_sources`, không sinh công việc mới và không sinh gợi ý trùng.
- Camera báo lại sau khi việc đã kết thúc sinh ra công việc mới.
- Xác nhận trùng chuyển hết nguồn và attachment sang việc gốc; tra cứu theo mã việc phụ vẫn chuyển hướng được sang việc gốc.
- Truy vấn toàn bộ người phản ánh của một công việc chỉ cần một lệnh trên `work_item_sources`.
- Tách một nguồn khỏi công việc tạo ra công việc mới độc lập với đúng attachment của nguồn đó.
- `classify` chặn ngay khi phân hệ đích không nhận được công việc, và nêu rõ còn thiếu dữ liệu gì.
- Giao nhiều người cho phân hệ chỉ nhận một người bị chặn với thông báo rõ ràng, không im lặng bỏ bớt.
- Nguồn đến sau khi đã giao được đẩy sang phân hệ qua `appendSource`.
- Tiếp nhận không tạo hồ sơ phân hệ; giao việc mới tạo hoặc gắn hồ sơ.
- Từ chối cán bộ cuối cùng đóng assignment hiệu lực và đưa `module_status` về `null`; hồ sơ phân hệ vẫn được giữ để tái sử dụng.
- `work_items` không còn cột hoặc enum trạng thái vòng đời chung.
- Bộ lọc `module_status` không kèm `module_code` trả 400.
- FE lấy danh sách trạng thái từ API, không hardcode.
- Chống trùng không tạo cây trùng nhiều cấp.
- Không chạy migration trên dev hoặc production trong giai đoạn điều chỉnh; chỉ xác minh local.

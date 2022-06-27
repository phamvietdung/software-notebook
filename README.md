## Application
 - DDD phải tradeoff giữa performance và tiêu chí của ddd( là phải save cả domain lên cùng lúc, thay vì seprate các thành phần của domain)
 
## Database
 - Nên có metadata database: collect các thông tin như region, gender, location. Thông tin này có thể chia theo tenant. Các lựa chọn không có tenantId được mặc định hiểu là default.
 - Dynamic field không poor performance như dự tính, khi deploy trên localhost load 1k record tốn 6s. Nhưng khi deploy lên server ~ 0.6s (Test trên khoảng 20 field dynamic)
 - Cần dựng cơ chế indexing/syncing data phục vụ việc searching dữ liệu. 

## Database Sync
 - CQRS rất tốt, nhưng implement khá rắc rối
 - Nên chia database Insert + Update vào main DB, các thao tác search => sync sang 1 con db ES hoặc tổ chức dưới dạng 1 static records(mongodb)

## AWS + GG
 - Nên sử dụng 1 số services như SQS, SNS, Email serivces. Thay vì dựng hẳn 1 con queue hoặc notification services.


## Coding
 - For + Parallel performance: Trên tập dữ liệu nhỏ dạng static không có nhiều sự thay đổi < 1k loop. Tuy nhiên > 1k loop sẽ chênh nhau 1s khi xử lý. Chưa test trên việc async/await dữ liệu. DBcontext ko share trên nhiều theard được. Nên khi chạy parallel sẽ bị crash.

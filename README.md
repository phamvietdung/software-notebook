## Application
 - DDD phải tradeoff giữa performance và tiêu chí của ddd( là phải save cả domain lên cùng lúc, thay vì seprate các thành phần của domain)
 
 
## Database
 - Nên có metadata database: collect các thông tin như region, gender, location. Thông tin này có thể chia theo tenant. Các lựa chọn không có tenantId được mặc định hiểu là default.


## Database Sync
 - CQRS rất tốt, nhưng implement khá rắc rối
 - Nên chia database Insert + Update vào main DB, các thao tác search => sync sang 1 con db ES hoặc tổ chức dưới dạng 1 static records(mongodb)

## AWS + GG
 - Nên sử dụng 1 số services như SQS, SNS, Email serivces. Thay vì dựng hẳn 1 con queue hoặc notification services.

## Application
 - DDD phải tradeoff giữa performance và tiêu chí của ddd( là phải save cả domain lên cùng lúc, thay vì seprate các thành phần của domain)
 
 
## Database
 - Nên có metadata database: collect các thông tin như region, gender, location. Thông tin này có thể chia theo tenant. Các lựa chọn không có tenantId được mặc định hiểu là default.

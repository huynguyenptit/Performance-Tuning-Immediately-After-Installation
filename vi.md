# MySQL 5.7 điều chỉnh hiệu suất ngay sau khi cài đặt
Blog này cập nhật [blog của Stephane Combaudon về điều chỉnh hiệu suất MySQL][1], và bao gồm cả điều chỉnh hiệu suất ngay sau khi cài đặt của MySQL 5.7.

Vài năm trước đây, Stephane Combaudon đã viết 1 blog trên [10 cài đặt tùy chỉnh hiệu suất MySQL sau khi cài đặt ][1] bao gồm (cả hiện tại) các bản cũ hơn của MySQL: 5.1, 5.5 và 5.6. Trong bài đăng này, tôi sẽ xem xét các điều chỉnh trong MYSQL 5.7 ( với trọng tâm vào InnoDB).

Tin tốt là MySQL đã có những giá trị mặc định tốt hơn đáng kể. Morgan Tocker đã tạo 1 [trang với một danh sách đầy đủ các tính năng trong MySQL 5.7][2], và sẽ là 1 nơi tham khảo tuyệt vời. Ví dụ, các biến sau đây sẽ được đặt mặc định:

- innodb_file_per_table=ON
- innodb_stats_on_metadata = OFF
- innodb_buffer_pool_instances = 8 (hoặc 1 nếu innodb_buffer_pool_size < 1GB)
- query_cache_type = 0; query_cache_size = 0; (vô hiệu hóa mutex)


Trong MySQL 5.7. chỉ có duy nhất 4 biến quan trong cần thay đổi. Tuy nhiên, có các biến InnoDB và MySQL toàn cục khác có thể cần điều chỉnh cho các khối lượng công việc và phần cứng cụ thể.

Để bắt đầu, thêm các cài đặt sau vào my.cnf dưới phần [mysqld]. Bạn sẽ cần khởi động lại MySQL:
[mysqld] 
# other variables here 
innodb_buffer_pool_size = 1G # (chỉnh sửa dữ liệu ở đây, 50%-70% của toàn bộ RAM)
innodb_log_file_size = 256M 
innodb_flush_log_at_trx_commit = 1 # có thể đổi thành 2 hoặc 0 
innodb_flush_method = O_DIRECT
Mô tả:

| Biến |  Giá trị | 
| ------------- |:-------------:| 
| innodb_buffer_pool_size |  Bắt đầu với 50% 70% tổng RAM. Không nhất thiết phải lớn hơn kích thước cơ sở dữ liệu|  
| innodb_flush_log_at_trx_commit | * 1   (Mặc định) 
|                                |0/2 hiệu suất cao hơn,kém tin cậy hơn)|  
| innodb_log_file_size |  128M – 2G (không cần thiết phải lớn hơn vùng đệm) |  
| innodb_flush_method |  O_DIRECT (tránh buffering 2 lần) | 

 

Tiếp theo là gì?

Đây là những xuất phát điểm tốt cho bất kỳ cài đặt mới nào. 1 số các biến khác có thể cải thiện hiệu năng của MySQL cho 1 vài khối lượng công việc. Thông thường, tôi sẽ thiết lập  công cụ giám sát/đồ thị hóa MySQL (ví dụ, [nền tảng giảm sát và quản lý Percona][3]) và sau đó kiểm tra bảng điều khiển MySQL để điều chỉnh hiệu năng sau này.

Liệu chúng ta có thể điều chỉnh thêm gì nữa dựa trên các đồ thị không?

Kích thước vùng đệm InnoDB. Hãy xem những đồ thị sau:

![MySQL 5.7 Performance Tuning][4]

![MySQL 5.7 Performance Tuning][5]

Như chúng ta có thể thấy, chúng ta có thể hưởng lợi từ việc tăng kích thước vùng đệm InnoDB 1 chút lên ~10G, vì chúng ta có RAM và số trang miễn phí nhỏ so với tổng số vùng đệm.

Kích thước file log InnoDB. Xem đồ thị sau:

![MySQL 5.7 Performance Tuning][6]


Như chúng ta có thể thấy, InnoDB thường ghi 2.26 GB dữ liệu mỗi giờ, vượt quá kích thước tổng của các file log (2G). Bây giờ chúng ta có thể tăng biến innodb_log_file_size và khởi động lại MySQL. Một cách khác, sử dụng "hiển thị trạng thái engine InnoDB" để [tính toán kích thước file log InnoDB hợp lý][7]

Các biến khác

Có 1 số các biến InnoDB khác bạn có thể điều chỉnh thêm:

innodb_autoinc_lock_mode

Cài đặt [innodb_autoinc_lock_mode][8] =2 (chế độ interleaved) có thể sẽ loại bỏ những thứ cần thiết cho khóa AUTO_INC mức-bảng ( và có thể tăng hiệu năng khi lệnh insert nhiều dòng được sử dụng để thêm các giá trị vào bảng mới khóa chỉnh tự_tăng).  Điều này yêu cầu binlog_format=ROW  hoặc MIXED  (và ROW thì mặc định trong MySQL 5.7).


innodb_io_capacity và innodb_io_capacity_max

Đây là một điều chỉnh nâng cao hơn, và chỉ có ý nghĩa khi bạn đang thực hiện việc viết rất nhiều trong tất cả các thời gian ( nó không áp dụng cho việc đọc, ví dụ các lệnh SELECT). Nếu bạn thực sự cần điều chỉnh nó, cách tốt nhất là bạn phải biết có bao nhiêu IOPS mà hệ thống có thể thực hiện. Ví dụ, nếu server có 1 drive SSD, chúng ta có thể đặt innodb_io_capacity_max=6000 và innodb_io_capacity=3000 (50% của tối đa) . Đây là 1 ý tưởng tốt để chạy sysbench hay bất cứ công cụ benchmark nào để đánh giá thông lượng đĩa.

Nhưng liệu chúng ta có cần phải lo lắng về cài đặt này?Xem biểu đồ về [những trang bẩn][9] của vùng đệm":

![screen-shot-2016-10-03-at-7-19-47-pm][10]


Trong trường hợp này, tổng số lượng các trang bẩn là rất cao, và trông như là InnoDB không thể duy trì việc làm sạch chúng. Nếu chúng ta có 1 hệ thống đĩa phụ tốc độ cao ( ví dụ SSD), chúng ta có thể hưởng lợi bằng cách tăng innodb_io_capacity và innodb_io_capacity_max.

Kết luận hoặc TL;phiên bản DR 

Những mặc định mới của MySQL 5.7 tốt hơn nhiều cho những khối lượng công việc với mục đích chung. Tại cùng thời điểm, chúng ta vẫn cần cấu hình các biến InnoDB để tận dụng lượng RAM chúng ta có. Sau quá trình cài đặt, thực hiện những bước sau

1. Thêm các biến InnoDB vào my.cnf(như mô tả bên trên) và khởi động lại MySQL
2. Cài đặt các hệ thống kiểm soát, (ví dụ nền tảng kiểm soát và quản lý Percona)
3. Xem biểu đồ và xác định xem MySQL có cần điều chỉnh hơn nữa không

[1]: https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/
[2]: http://www.thecompletelistoffeatures.com/
[3]: http://pmmdemo.percona.com
[4]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png
[5]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png
[6]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png
[7]: https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/
[8]: http://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html
[9]: http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page
[10]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png
[11]: https://secure.gravatar.com/avatar/79877aeedbd68531a30468cd771d5d07?s=84&d=mm&r=g
[12]: https://www.percona.com/blog/author/alexanderrubin/

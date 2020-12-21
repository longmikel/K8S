## Giới thiệu về Kubernetes

`Kubernetes chỉ là một phần trong hệ sinh thái của container ngày nay`, do vậy trước khi đi vào tìm hiểu về Kubernetes, chúng ta cần biết tổng quan về hệ sinh thái của container. Trong hệ sinh thái của container có 03 phần chính, bao gồm:

-- bổ sung bằng hình sau --

 - Phần 1: Đây là thành phần cơ bản và cốt lõi nhất trong hệ sinh thái container, các thành phần này bao gồm: Kiến trúc lõi của container (các khái niệm về runtime spec, image format speck); khái niệm images, network và storage.
 - Phần 2: Các công nghệ về platform liên quan tới container, bao gồm: các sản phẩm để orchestration container (sử dụng để khởi tạo các container theo các cơ chế điều phối), nền tảng để quản lý các container và các nền tảng PaaS dựa trên container.
 - Phần 3: Các công nghệ hỗ trợ container, bao gồm: công nghệ hỗ trợ khi triển khai container trên nhiều máy chủ vật lý, các giải pháp phụ trợ về thu thập log và gíam sát.

 - Kubernetes là một sản phẩm nằm trong phần 2 của hệ sinh thái của container hiện nay. Nó đóng vai trò là là một công cụ orchestration (tạo ra, điều phối và chỉ huy các container) - nôm na mà nói thì kubernetes là thằng túm đầu các máy vật lý hoặc máy ảo được cài đặt môi trường để triển khai container.

## Các thông tin khởi đầu với Kubernetes

### Cần chuẩn bị gì khi tìm hiểu về Kubernetes
- Cần vừa đọc khái niệm lý thuyết vừa thực hành.
- Để tìm hiểu tốt về Kubernetes thì cần tìm hiểu về hệ sinh thái container và docker ở mức cở bản (có nghĩa là cài cắm và thực hành được với docker cơ bản). Tài liệu về docker cơ bản ở đây: [Ghichep-docker](https://github.com/hocchudong/ghichep-docker/)
- Kỹ năng và kiến thức tối quan trọng khi làm việc với Kubernetes là: Linux, Linux và Linux.
- Kỹ năng bổ trợ khác: Network (TCP/IP), một vài kỹ năng cài đặt các ứng dụng cơ bản như web server, database.
- Đối tượng người dùng thích hợp khi tìm hiểu Kubernetes: Sinh viên IT, các sysadmin, các lập trình viên và xa hơn nữa là bất kỳ ai có nhu cầu tìm hiểu.
- Tài liệu chuẩn của Kubernetes tất nhiên là ở trang chủ của họ https://kubernetes.io/, ngoài ra github này cũng coi như là một tham khảo tốt.

### Các thông tin tóm tắt về Kubernetes

- Khơi mào cho Kubernetes là anh Google. Anh ấy sinh ra nó vì anh ấy bá và tất nhiên trong hệ thống của anh Google có ứng dụng container (theo mật báo thì có khoảng 2 tỉ container trong datacenter của google.
- Kubernetes được dân chuyên môn hay viết tắt là K8S, tra từ điển ENG-VIETNAM thì không có nghĩa của từ này :). Trong các tài liệu này sẽ dùng từ K8S cho nhanh gọn nhé.
- Kubernetes là một `orchestration tools` trong hệ sinh thái của container (có nghĩa còn các thứ khác tương đương hay là đối thủ của nó). Tóm lại nó là thằng để tạo, sửa, xóa, thay đổi, thêm bớt ... các container. Sử dụng nó thì skill về docker của bạn sẽ lên một level mới ;).

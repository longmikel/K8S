## Kiến trúc và các thành phần trong K8S
Phần này sẽ mô tả về kiến trúc và các thành phần trong K8S. Muốn hiểu phần này được tốt nhất thì nên đọc phần bài về [Giới thiệu về Kubernetes](./gioithieu.md/) và [Cài đặt Kubernetes Cluster với RKE](./rke-k8s.md/)

Như trong phần [Cài đặt Kubernetes Cluster với RKE](./rke-k8s.md/) chúng ta có thể nhìn thấy các thành phần `kubeadm, kubelet, kube-proxy, ectd, flannet` nằm trên các node. Trong phần này ta sẽ làm rõ hơn về chúng.

Kubernetes được xếp vào một trong các orchestration tools nằm trong hệ sinh thái của container. Do vậy K8S có các thành phần để đảm bảo thực hiện được việc tự động triển khai, mở rộng và vận hành container trên các cụm cluster (chính là các máy chạy các container sau này).

Kubernetes có thể được triển khai trên một hoặc nhiều máy vật lý, máy ảo hoặc cả máy vật lý và máy ảo để tạo thành cụm cluster. Cụm cluster này chịu sự điều khiển của Kubernetes và sinh ra các container khi người dùng yêu cầu. Kiến trúc logic của Kubernetes bao gồm 02 thành phần chính dựa theo vai trò của các node, đó là: `Master node` và `Worker node`

- `Master node`: Đóng vai trò là thành phần Control plane, điều khiển toàn bộ các hoạt động chung và kiểm soát các container trên `node worker`. Các thành phần chính trên `master node` bao gồm: `API-server, Controller-manager, Schedule, Etcd và cả Docker Engine`. Lưu ý: Có thể trong hình vẽ dưới bạn không nhìn thấy thành phần là docker được hiển thị ra nhưng trên `master node`  cần có docker, lý do là để chạy các thành phần của K8S trên các container.

- `Worker node`: Vai trò chính của `worker node` là môi trường để chạy các container mà người dùng yêu cầu, do vậy thành phần chính của `worker node` bao gồm: `kubelet, kube-proxy` và chắc chắn là `Docker`

Thường thì khi triển khai thực tế thì số lượng `node worker` sẽ nhiều hơn số lượng `node master`. Do vậy `node master` hay chính xác là K8S cần hoàn thành tốt nhiệm vụ liên quan tới việc quản lý, xử lý các container sao cho linh hoạt và trơn tru nhất. Ngoài ra, nếu như với các hệ thống thực tế cần có khả năng `High Availability` thì chúng ta cần triển khai nhiều `node master`

-- bổ sung hình kiến trúc k8s sau --

Các thành phần chính trong cụm cluster K8S bao gồm:

- ectd
- API-server
- Controller-manager
- Schedule
- Agent
- Proxy
- CLI

Tuy một số thành phần kể tên ở trên không được liệt kê trong hình vẽ nhưng chúng là thành phần cần phải có để đảm bảo hoạt động của cụm Cluseter K8S. Sau đây chúng ta sẽ mô tả về vài trò của các thành phần chính trong kiến trúc của K8S. Sau khi hoàn tất phần này, chúng ta sẽ tìm hiểu về các khái niệm chính, mục tiêu là để người đọc hiểu được các thuật ngữ khi làm việc này này với K8S.


#### etcd
- Etcd là một thành phần database phân tán, sử dụng ghi dữ liệu theo cơ chế `key/value` trong K8S cluster. Etcd được cài trên node master và lưu tất cả các thông tin trong Cluser. Etcd sử dụng port 2380 để listening từng request và port 2379 để client gửi request tới.
- Ectd nằm trên node master.

#### API-server
- API server là thành phần tiếp nhận yêu cầu của hệ thống K8S thông qua REST, tức là nó tiếp nhận các chỉ thị từ người dùng cho đến các services trong hệ thống Cluster thông qua API - có nghĩa là người dùng hoặc các service khác trong cụm cluster có thể tương tác tới K8S thông qua HTTP/HTTPS.

- API-server hoạt động trên port 6443 (HTTPS) và 8080 (HTTP).
- API-server nằm trên node master.

#### Controller-manager
- Thành phần controller-manager là thành phần quản lý trong K8S, nó có nhiệm vụ xử lý các tác vụ trong cụm cluster để đảm bảo hoạt động của các tài nguyên trong cụm cluster. Controller-manager có các thành phần bên trong như sau:

  - Node Controller: Tiếp nhận và trả lời các thông báo khi có một node bị down.
  - Replication Controller: Đảm bảo các công việc duy trì chính xác số lượng bản replicate và phân phối các container trong pod (Pod tạm hình dung là một tập hợp các container khi người dùng có nhu cầu tạo ra và cùng thực hiện chạy một ứng dụng).
  - Endpoints Controller: Populates the Endpoints object (i.e., join Services & Pods).
  - Service Account & Token Controllers:  Tạo ra các accounts và token để có thể sử dụng được các API cho các namespaces.

- Thành phần controller-manager hoạt động trên node master và sử dụng port 10252.

#### Scheduler

kube-scheduler có nhiệm vụ quan sát để lựa chọn ra các node mỗi khi có yêu cầu tạo pod. Nó sẽ lựa chọn các node sao cho phù hợp nhất dựa vào các cơ chế lập lịch mà nó có. Kube-scheduler được cài đặt trên node master và sử dụng port 10251.

#### Agent - kubelet
- Agent hay chính là kubelet, một thành phần chạy chính trên các node worker. Khi kube-scheduler đã xác định được một pod được chạy trên node nào thì nó sẽ gửi các thông tin về cấu hình (images, volume ...) tới kubelet trên node đó. Dựa vào thông tin nhận được thì kubelet sẽ tiến hành tạo các container theo yêu cầu.
- Vai trò chính của kubelet là:
  - Dõi theo các pod trên node được gán để hoạt động.
  - Mount các volume cho pod
  - Đảm bảo hoạt động của các container của pod hoạt động tốt trên node đó (node worker có cài docker đó).
  - Report về trạng thái của các pod để cụm cluster biết được xem các container còn hoạt động tốt hay không.

- Kubelet chạy trên các node worker và sử dụng port 10250 và 10255.

#### Proxy
- Các service chỉ hoạt động ở chế độ logic, do vậy muốn bên ngoài có thể truy cập được vào các service này thì cần có thành phần chuyển tiếp các request từ bên ngoài và bên trong.
- Kube-proxy được cài đặt trên các node worker, sử dụng port 31080

#### CLI

- kubectl là thành phần cung cấp câu lệnh để người dùng tương tác với K8S. kubectl có thể chạy trên bất cứ máy nào, miễn là có kết nối được với K8S API-server

### Pod network

Như trong các phần trước, chúng ta còn thấy một thành phần khác đó là pod network, đây là thành phần xử lý về network trong cụm K8S cluster. Pod network đảm bảo cho các container có thể truyền thông được với nhau. Có nhiều lựa chọn về pod network, nhưng trong tài liệu này chỉ giới thiệu về `flannet`


### Tại sao trên node master cũng có các thành phần `kubelet` và `kube-proxy`
- Trong hình dưới ta có thể thấy trên node master có các thành phần là `kubelet` và `kube-proxy`. Tại sao lại như vậy.

-- bổ sung hình thành phần sau ---

Câu trả lời là: Do các node master cũng có các service (ứng dụng) được sử dụng để đảm bảo hoạt động của K8S, do vậy chúng được chạy trong các container và thuộc một pod với namespaces là `kube-system`.

Ta có thể sử dụng lệnh `kubectl get pod --all-namespaces -o wide` để quan sát việc này.

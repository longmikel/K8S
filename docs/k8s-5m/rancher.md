# Cài đặt Rancher 2 trên Kubernetes cluster
## Cài đặt công cụ CLI
Các công cụ CLI sau đây được yêu cầu để thiết lập Kubernetes Cluster. Đảm bảo rằng các công cụ này đã được cài đặt và có sẵn trong:
kubectl: Kubernetes command-line tool
helm: Package management for Kubernetes

## Thêm kho lưu trữ biểu đồ Helm
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

## Khởi tạo namespace của Rancher
Cần phải xác định namespace Kubernetes nơi các tài nguyên được tạo bởi Biểu đồ sẽ được cài đặt. Điều này phải luôn luôn là hệ thống gia súc:
```bash
kubectl create namespace cattle-system
```

## Cài đặt cert-manager
Bước này chỉ thực hiện để sử dụng các chứng chỉ do Rancher tạo ra (ingress.tls.source=rancher) hoặc để yêu cầu chứng chỉ của Let’s Encrypt (ingress.tls.source=letsEncrypt).

P/s: Bỏ qua bước này nếu bạn sử dụng các tệp chứng chỉ riêng (tùy chọn ingress.tls.source=secret) hoặc nếu bạn sử dụng TLS termination on an external load balancer.

### Install the CustomResourceDefinition resources separately
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.crds.yaml
```

### Khởi tạo namespace của cert-manager
```bash
kubectl create namespace cert-manager
```

### Add the Jetstack Helm repository
```bash
helm repo add jetstack https://charts.jetstack.io
```

### Update your local Helm chart repository cache
```bash
helm repo update
```

### Install the cert-manager Helm chart
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.0
```

Khi bạn đã cài đặt cert-manager, bạn có thể xác minh nó được triển khai chính xác bằng cách kiểm tra cert-manager namespace để chạy các nhóm:
```bash
kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-766d5c494b-2p8sv              1/1     Running   0          2d20h
cert-manager-cainjector-6649bbb695-l6ddv   1/1     Running   0          2d20h
cert-manager-webhook-68d464c8b-kgbhm       1/1     Running   0          2d20h
```

## Cài đặt Rancher với Helm và tùy chọn chứng chỉ được chọn
### Cài đặt Rancher với chứng chỉ cấp bởi Let’s Encrypt
Tùy chọn này sử dụng trình cert-manager để tự động yêu cầu và gia hạn các chứng chỉ Let’s Encrypt. Đây là một dịch vụ miễn phí cung cấp cho bạn một chứng chỉ hợp lệ vì Let’s Encrypt là một CA đáng tin cậy.

Trong lệnh sau:
- hostname: đặt theo bản ghi DNS (đường dẫn dùng để truy cập Rancher UI)
- ingress.tls.source: letsEncrypt
- letsEncrypt.email: được đặt thành địa chỉ email được sử dụng để liên lạc liên quan chứng chỉ đã tạo (ví dụ: thông báo hết hạn)
- Nếu bạn đang cài đặt phiên bản alpha, Helm yêu cầu thêm tùy chọn --devel vào lệnh (Nếu phiên bản khác thì bạn bỏ qua bước này)
```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rke.matbao.io \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=longhm1@matbao.com
```

### Cài đặt Rancher với chứng chỉ riêng
Tùy chọn này không sử dụng trình cert-manager để tự động yêu cầu và gia hạn các chứng chỉ riêng vì không hỗ trợ. Đây là chứng chỉ bạn đăng ký riêng và được cấp bởi những tổ chức có uy tín như: Sectigo, Symantec ...
```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rke.matbao.io \
  --set ingress.tls.source=secret \
```

Nếu bạn đang sử dụng chứng chỉ có CA riêng, hãy thêm --set privateCA=true vào lệnh:
```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rke.matbao.io \
  --set ingress.tls.source=secret \
  --set privateCA=true
```

Bây giờ Rancher đã được triển khai, tiếp theo bạn thực hiện [Adding TLS Secrets](https://rancher.com/docs/rancher/v2.x/en/installation/options/tls-secrets/) để xuất bản các tệp chứng chỉ để Rancher và bộ điều khiển Ingress có thể sử dụng chúng.

## Xác minh rằng Rancher đã được triển khai thành công
```bash
kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
```

Nếu bạn thấy lỗi sau: error: deployment "rancher" exceeded its progress deadline, bạn có thể kiểm tra trạng thái của việc triển khai bằng cách chạy lệnh sau:
```bash
kubectl -n cattle-system get deploy rancher
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rancher   3         3         3            3           3m
```

Nó sẽ hiển thị cùng một số lượng cho DESIRED và AVAILABLE.

## Cài đặt Load Balancer với HA Proxy
### Cài đặt HA Proxy
```bash
--- Ubuntu / Debian ---
sudo add-apt-repository ppa:vbernat/haproxy-2.1
sudo apt-get update
sudo apt-get install haproxy
```

### Cấu hình HA Proxy
Cấu hình trong tệp /etc/haproxy/haproxy.cfg với nội dung:
```bash
global
        log 127.0.0.1   local0
        log 127.0.0.1   local1 debug
        maxconn   45000 # Total Max Connections.
        daemon
        nbproc      1 # Number of processing cores.

defaults
        timeout server 86400000
        timeout connect 86400000
        timeout client 86400000
        timeout queue   1000s

frontend k8s_frontend_http
	mode tcp
	bind IP-Public-RKE:80
	default_backend k8s_backend_http

frontend k8s_frontend_https
	mode tcp
	bind IP-Public-RKE:443
	default_backend k8s_backend_https

backend k8s_backend_http
	mode tcp
	server x-node01 IP-Private-Worker:80 weight 1 maxconn 5000 inter 2s rise 2 fall 3 check
	server x-node02 IP-Private-Worker:80 weight 1 maxconn 5000 inter 2s rise 2 fall 3 check

backend k8s_backend_https
	mode tcp
	server x-node01 IP-Private-Worker:443 weight 1 maxconn 5000 inter 2s rise 2 fall 3 check
	server x-node02 IP-Private-Worker:443 weight 1 maxconn 5000 inter 2s rise 2 fall 3 check
```

## Khởi động dịch vụ HA Proxy sau khi đã khai báo cấu hình
```bash
systemctl enable haproxy
systemctl restart haproxy
```

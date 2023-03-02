# Services: expose traffic cho pod 

## 1. Giới thiệu
Ở bài này chúng ta sẽ nói về Kubernetes Services, một resource cho phép chúng ta ra expose traffic cho Pod, để Pod có thể giao tiếp và xử lý request từ client.

## 2. Kubernetes Services là gì?

Là một resouce sẽ tạo ra một single, constant point của một nhóm Pod phía sau nó. Mỗi service sẽ có một địa chỉ IP và port không đổi, trừ khi ta xóa nó đi và tạo lại. Client sẽ mở connection tới service, và connection đó sẽ được dẫn tới một trong những Pod ở phía sau.

![](../imgs/service.png)

Vậy service giúp ta những vấn đề gì? Mỗi thằng Pod nó cũng có địa chỉ IP riêng của nó, sao ta không gọi thẳng nó luôn mà thông qua service chi cho mất công? Để trả lời những câu hỏi này thì ta sẽ nói qua một vài vấn đề.

### 2.1 Pods are ephemeral

Có nghĩa Pod là một resource rất phù du, nó sẽ được tạo ra, bị xóa, và thay thế bằng một thằng khác bất cứ lúc nào, có thể là lúc ta muốn đổi một template khác cho Pod, Pod cũ sẽ bị xóa đi và Pod mới sẽ được tạo ra. Hoặc là trường hợp một woker node die, Pod trên worker node đó cũng sẽ die theo, và một Pod mới sẽ được tạo ra trên woker node khác.

Khi tạo thằng pod mới tạo ra, nó sẽ có một IP khác với thằng cũ. Nếu ta dùng IP của Pod để tạo connection tới client thì lúc Pod được thay thế với IP khác thì ta phải update lại code.

### 2.2 Multiple Pod run same application

Có nghĩa là ta sẽ có nhiều pod đang chạy một ứng dụng của chúng ta để tăng performance. Ví dụ khi ta dùng ReplicaSet với replicas = 3, nó sẽ tạo ra 3 Pod. Vậy làm sao ta biết được nên gửi request tới Pod nào?

Thì để giải quyết những vấn để trên thì Kubernetes cung cấp cho chúng ta Services resource, Service sẽ tạo ra một endpoint không đổi cho các Pod phía sau, client chỉ cần tương tác với endpoint này.

## 3. Service quản lý connection như thế nào?

Làm sao Service biết được nó sẽ chọn Pod nào để quản lý connection tới những Pod đó, nếu bạn còn nhớ label selectors ở những bài trước và cách ReplicaSet sử dụng label để quản lý Pod. Thì Services cũng sẽ sử dụng label selectors để chọn Pod mà nó quản lý connection.
![](../imgs/service_label.png)

Service sẽ có 4 loại cơ bản là:

- ClusterIP
- NodePort
- ExternalName
- LoadBalancer

### 3.1 ClusterIP

Đây là loại service sẽ tạo một IP và local DNS mà sẽ có thể truy cập ở bên trong cluster, không thể truy cập từ bên ngoài, được dùng chủ yếu cho các Pod ở bên trong cluster dễ dàng giao tiếp với nhau.

Ví dụ như sau, ta có một application cần xài redis, ta sẽ deploy một Pod redis và application của ta cần kết nối với Pod redis đó. Với redis thì connection string của nó sẽ có dạng như sau `redis://<host-name>:<port>`, vậy ứng dụng ta sẽ kết nối với redis thế nào? Sau đây ta sẽ làm ví dụ tạo Pod redis và Service cho nó.

```
kubectl apply -f redis/redis-replicaset
kubectl apply -f redis/redis-service-clusterIP.yaml
kubectl get svc
>>
  NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
  kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP    29h
  redis        ClusterIP     <none>        6379/TCP   33s
```
Bây redis của chúng ta sẽ có địa chỉ truy cập như sau `redis://10.97.55.37:6379`. Ta có thể thấy hostname của service của chúng ta ở đây là `10.97.55.37`, ta có thể dùng địa chỉ này để các ứng ta deploy về sau truy cập được tới redis host.

Vậy nếu ta có các ứng dụng đã chạy sẵn, thì làm sao ta kết nối tới host này được, không lẻ chúng ta phải update lại code? Thì để giải quyết vấn đề này, thay vì truy cập redis host thằng IP, ta có thể truy cập thông qua DNS, kubernes có cung cấp cho chúng ta một local DNS bên trong cluster, giúp chúng ta có thể connect được tới host ta muốn thông qua DNS. Ta có thể connect redis với địa chỉ sau `redis://redis:6379`, với host name là tên của Service chúng ta đặt trong trường metadata.

![](../imgs/service_clusterIP.png)

Tạo một container tron Pod rồi access tới container redis.
```
kubectl run hello-redis --image=080196/hello-redis
```

Đây là code của application trong `080196/hello-redis image`
```
const redis = require("redis");

const client = redis.createClient("redis://redis:6379")

client.on("connect", () => {
  console.log("Connect redis success");
})
```
Kiểm tra log của pod, nếu in ra Connect redis success thì chúng ta đã kết nối được với redis host bằng dns.

```
kubectl logs hello-redis
>>
  Connect redis success
```
Clear resource
```
kubectl delete pod hello-redis
kubectl delete -f redis/redis-replicaset.yaml
```
ClusterIP giúp các ứng dụng deploy trong cluster của ta giao tiếp với nhau dễ dàng hơn nhiều. Còn nếu chúng ta muốn client từ bên ngoài có thể truy cập vào ứng dụng của chúng ta thì sao? Có 3 cách đó là dùng `NodePort`, `LoadBalancer` (hỗ trợ clound), `Ingress`.

### 3.2 NodePort

Đây là cách đầu tiên để expose Pod cho client bên ngoài có thể truy cập vào được Pod bên trong cluster. Giống như `ClusterIP`, `NodePort` cũng sẽ tạo endpoint có thể truy cập được bên trong cluster bằng IP và DNS, đồng thời nó sẽ sử dụng một port của toàn bộ worker node để client bên ngoài có thể giao tiếp được với Pod của chúng ta thông qua port đó. NodePort sẽ có range mặc định từ `30000-32767`. Ví dụ:
```
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello-kube
  type: NodePort # type NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30123 # port of the worker node
```
![](../imgs/service_nodeport.png)

Client có thể gửi request tới Pod bằng địa chỉ `130.211.97.55:30123` hoặc `130.211.99.206:30123`. Và application bên trong cluster có thể gửi request tới Pod với địa chỉ `http://hello-kube`.

Giờ ta tạo thử NodePort service. Tạo một file tên là `hello-nodeport.yaml` trong folder hello-kube
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hello-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-kube
  template:
    metadata:
      labels:
        app: hello-kube
    spec:
      containers:
      - image: 080196/hello-kube
        name: hello-kube
        ports:
          - containerPort: 3000

---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello-kube
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 31000
```

Chạy pod và service:
```
kubectl apply -f hello-kube/hello-nodeport.yaml
```
Test gửi request tới Pod với địa chỉ `http://<your_worker_node_ip>:<node_port>`
```
# get NodeportIP
kubectl get nodes -o wide
>>
  NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    ...
  minikube   Ready    control-plane   45h   v1.24.1   192.168.49.2   ...

curl http://192.168.49.2:31000
>>
  Hello kube
```

![](../imgs/service_nodeport_ip.png)

### 3.3 LoadBalancer

Khi bạn chạy kubernetes trên cloud, nó sẽ hỗ trợ LoadBalancer Service, nếu bạn chạy trên môi trường không có hỗ trợ LoadBalancer thì bạn không thể tạo được loại Service này. Khi bạn tạo LoadBalancer Service, nó sẽ tạo ra cho chúng ta một public IP, mà client có thể truy cập Pod bên trong Cluster bằng địa chỉ public IP này. Ví dụ
```
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  selector:
    app: kubia
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```
![](../imgs/service_loadbalancer.png)

### 3.4 Ingress resource

`Ingress` là một resource cho phép chúng ta expose `HTTP` and `HTTPS` routes từ bên ngoài cluster tới service bên trong cluster của chúng ta. Ingress sẽ giúp chúng ta gán một domain thực tế với service bên trong cluster. Ví dụ:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    - host: kubia.example.com # domain name
  http:
    paths:
      - path: /
        backend:
          serviceName: kubia-nodeport # name of the service inside cluster
          servicePort: 80
```
![](../imgs/service_ingress.png)

Ta cũng có thể mapping nhiều service với cùng một domain. Ví dụ:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    - host: kubia.example.com
  http:
    paths:
      - path: /bar
        backend:
          serviceName: kubia-bar
          servicePort: 80
      - path: /foo
        backend:
          serviceName: kubia-foo
          servicePort: 80
```

Ingress là resource hữu dụng nhất để expose HTTP và HTTPS của Pod ra ngoài. Bạn có thể đọc nhiều hơn về Ingress ở [đây](https://kubernetes.io/docs/concepts/services-networking/ingress/).

## 4. Kết luận

Vậy là chúng ta đã tìm hiểu xong về Kubernetes Servies. Sử dụng nó khi bạn muốn expose traffic của Pod và mở connection cho client bên ngoài truy cập vào Pod của chúng ta.
- `ClusterIP`: Pod giao tiếp bên trong cùng một Node. Để giao tiếp bên ngoài cần port-forward
- `NodeIP`: giao tiếp với Pod thông qua Node IP
- `Loadbalance`: dùng cho cloud computing
- `Ingress`: expose HTTP và HTTPS
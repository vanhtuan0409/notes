# (WIP) QUIC

### Lí do dẫn đến QUIC

Chúng ta đã quen với 2 protocol được sử dụng phổ biến ở tầng Transport là UDP và TCP protocol. Trước khi nói về những lí do dẫn đến sự ra đời của QUIC, mình sẽ nhắc lại 1 vài tính chất của UDP và TCP

|                   | TCP                                                                                     | UDP                                                     |
| ----------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| Phân loại         | Connection/Stateful                                                                     | Connectionless/Stateless                                |
| Thứ tự            | Packet được đảm bảo thứ tự gửi/nhận                                                     | Packet không đảm bảo thứ tự gửi/nhận                    |
| Đảm bảo           | Packet được đảm bảo gửi thành công. Nếu packet loss xảy ra, protocol sẽ thực hiện retry | Packet không được đảm bảo gửi thành công. Fire n Forget |
| Checksum          | Có checksum                                                                             | Không có checksum                                       |
| Handshake         | 3-way handshake                                                                         | Không có handshake                                      |
| Built-in security | Không có                                                                                | Không có                                                |

Trong thực tế, các application thường chọn TCP làm transport protocol bởi vì tính thứ tự và đảm bảo việc gửi packet luôn luôn thành công. Lựa chọn này đồng nghĩa với việc application đã lựa chọn tradeoff đánh đổi performance cho reliablility. Để bù đắp lại cho tradeoff trên, 1 vài kĩ thuật thường được sử dụng để tối ưu hóa cho performance:
- Reuse connection cho nhiều request để giảm thiểu chi phí cho 3-way handshake
- Tạo nhiều connection cùng lúc để tối ưu hóa bandwidth (bù cho việc TCP không hỗ trợ multiplex)
- Hiện thực multiplexing ở tầng application (HTTP2, sẽ được đề cập cụ thể hơn ở phần sau)

Tuy nhiên với những tối ưu đó, vẫn tồn tại 1 vài hạn chế đối với TCP protocol

#### Security

TCP không định nghĩa bất kì cơ chế security nào trong protocol. Vì vậy khi có nhu cầu về bảo mật, chúng ta thường phải sử dụng thêm 1 layer nữa ở phía trên TCP, và đó chính là TLS

<p align="center">
    <img alt="layers" src="https://i.imgur.com/nwWWeMc.png"/>
</p>

Tuy nhiên bản thân TLS protocol cũng yêu cầu quy trình 3-way handshake riêng để trao đổi cryptographic parameters giữa client và server. Điều này khiến cho việc sử dụng TLS over TCP thực chất cần đến **4 round-trip** để tạo kết nối trước khi application thật sự có thể trao đổi request

<p align="center">
    <img alt="hand-shake" width="70%" src="https://blog.cloudflare.com/content/images/2018/07/http-request-over-tcp-tls@2x.png"/>
</p>

#### Head of line blocking

Trước khi nói về bài toán Head-of-line blocking, mình sẽ review lại 1 ít về multiplexing request. Thông thường việc gửi và nhận request trên 1 connection phải diễn ra 1 cách tuần tự. Giao thức này tương tự với việc sử dụng bộ đàm, tại 1 thời điểm chỉ có 1 người được nói. Multiplexing là kĩ thuật cho phép kết hợp nhiều nhiều request/response lại và gửi đi trên cùng 1 connection (gần giống vs batching)

<p align="center">
    <img alt="http2" width="70%" src="https://blog.hostvn.net/wp-content/uploads/2017/09/HTTP2-Multiplexing-1024x1015.png"/>
</p>

Multiplexing đặc biệt hiệu quả khi client/server có khả năng xử lý concurrent cao. HTTP2 đã tự implement multiplexing trên TCP connection bằng cách đưa ra khái niệm **stream** và đính kèm thông tin về stream trong HTTP request

<p align="center">
    <img alt="http2-header" width="70%" src="https://miro.medium.com/max/1400/1*Ys85poyVRTszancKYmx9eQ.png"/>
</p>

Khi sử dụng HTTP2, application có thể fetch nhiều object cùng 1 lúc trên 1 TCP connection thay vì phải sử dụng nhiều TCP connection như trước đây. Điều này giúp giảm chi phí của việc handshake (cả TCP và TLS) đồng thời sử dụng network bandwidth tốt hơn

<p align="center">
    <img alt="http2-header" width="70%" src="https://www.cloudflare.com/img/products/website-optimization/http2/multiplexing.svg"/>
</p

### Reference

- https://blog.cloudflare.com/the-road-to-quic/
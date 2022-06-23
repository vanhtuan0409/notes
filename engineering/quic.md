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

Trước khi nói về bài toán Head-of-line blocking, mình sẽ review lại 1 ít về multiplexing request. Thông thường việc gửi và nhận request trên 1 connection phải diễn ra 1 cách tuần tự. Giao thức này tương tự với việc sử dụng bộ đàm, tại 1 thời điểm chỉ có 1 người được nói. Multiplexing là kĩ thuật cho phép kết hợp nhiều nhiều request/response lại và gửi đi trên cùng 1 connection. Multiplexing gần giống vs pipelining/batching ở chỗ request có thể gửi liên tục mà không cần chờ response, tuy nhiên khác ở chỗ pipelining ko yêu cầu response phải trả về theo đúng thứ tự của request, trong khi pipelining thì yêu cầu việc đó

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

Tuy nhiên việc multiplexing trên TCP connection gặp phải 1 bài toán rất lớn, đó là dù cho request/response đã có thể dược gửi nhận 1 cách độc lập nhưng khi có packet loss xảy ra, **tất cả request stream** đều bị block lại để TCP thực hiện retransmission. Nếu trong điều kiện lí tưởng, chỉ 1 request stream bị ảnh hưởng bới packet loss và các stream còn lại vẫn hoạt động bình thường. Nguyên nhân dẫn đến điều này là vì các Stream không thật sự độc lập với nhau mà vẫn chịu sự ràng buộc của TCP là packet phải được gửi nhận theo thứ tự.

#### Network migration

Một trong những điểm khác nhau giữa TCP và UDP đó là TCP không *stateless*. Một TCP connection có thể có 1 trong các trạng thái như LISTEN, SYN-SENT, ESTABLISHED, CLOSING, .... OS cần phải theo dõi trạng thái 1 connection bằng 1 bảng mapping riêng trong kernel state, hay chính là *conntrack table*. 1 dòng trong conntrack table được xác định (primary key) bởi tuple gồm 4 yếu tố `(src_ip, src_port, dst_ip, dst_port)`

Chính vì việc TCP connection là *stateful* dẫn đến việc nếu client hoặc server thay đổi 1 trong 4 yếu tố trong tuple kể trên thì sẽ không thể sử dụng connection đã tạo được nữa. Việc này thường xảy ra khi có 1 sự thay đổi ở layer thấp hơn (link level) như việc chuyển đổi từ mạng dây sang WiFi hoặc từ WiFi sang 3G/4G. Khi có sự thay đổi về network, application bắt buộc phải tạo lại connection mới, ngoài ra bài toán này khiến TCP không thể hỗ trợ 1 vài kĩ thuật routing ở IP protocol level nhằm tránh nghẽn mạng (network congestion) như ECMP

Tóm gọn lại, dù ổn định và đã được sử dụng lâu đời, TCP vẫn tồn tại nhiều nhược điểm về performance trong nhu cầu hiện đại

### QUIC

### Reference

- https://blog.cloudflare.com/the-road-to-quic/
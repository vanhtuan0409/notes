# (WIP) Linux TUN device

Trong quá trình tìm hiểu Userspace Networking và Wireguard thì mình phát hiện ra TUN/TAP device. Đây là 1 interface của *Unix kernel cho phép implement 1 phần của network stack bằng program chạy trong Userspace

Thông thường, user program sẽ gửi/nhận request qua mạng thông qua việc đọc/ghi vào tcp/udp socket (everything is a file). Packet sau đó sẽ vào kernel space, đi qua network stack (thêm tcp/udp/ip header, routing, ...), rồi cuối cùng vào network interface để được gửi qua máy khác

![traditional](https://www.packetcoders.io/content/images/2020/10/image3.png)

Tuy nhiên với sự phát triển của internet, nhu cầu chuyển 1 phần của network stack vào user space ra đời. 1 ví dụ của nhu cầu đó là việc tạo kết nối private giữa 2 mạng LAN tại Layer 2/Layer 3 thông qua mạng Internet, hay dễ hiểu thì gọi là VPN :]]]. Kết nối private này thường phát triển theo các nhu cầu riêng về Authentication, Authorization, Encryption, ... nên khó có thể phát triển 1 standard protocol, nên việc đặt trong kernel space là không phù hợp.

Đáp ứng với nhu cầu trên, *Unix kernel cung cấp interface để tạo ra network interface ảo. Khi packet được gửi tới những interface ảo này thì sẽ được kernel gửi lên userspace để được xử lý bởi VPN. VPN sau đó có thể quyết định việc encrypt và routing tới 1 máy khác qua mạng internet. Cụ thể hơn thì kernel cung cấp 3 loại interface ảo là:
- TUN device: để gửi/nhận packet ở Layer 3 (IP protocol)
- TAP device: để gửi/nhận packet ở Layer 2 (Ethernet protocol)
- Veth pair: để kết nối 2 userspace program

![tuntap](https://www.packetcoders.io/content/images/2020/10/image2.png)
![veth](https://www.packetcoders.io/content/images/2020/10/image4.png)

### References

- https://www.kernel.org/doc/html/v5.8/networking/tuntap.html
- https://github.com/WireGuard/wireguard-go/tree/master/tun
- https://www.packetcoders.io/virtual-networking-devices-tun-tap-and-veth-pairs-explained/
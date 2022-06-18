# Linux TUN device

Trong quá trình tìm hiểu Userspace Networking và Wireguard thì mình phát hiện ra TUN/TAP device. Đây là 1 interface của \*Unix kernel cho phép implement 1 phần của network stack bằng program chạy trong Userspace

Thông thường, user program sẽ gửi/nhận request qua mạng thông qua việc đọc/ghi vào tcp/udp socket (everything is a file). Packet sau đó sẽ vào kernel space, đi qua network stack (thêm tcp/udp/ip header, routing, ...), rồi cuối cùng vào network interface để được gửi qua máy khác

![traditional](https://www.packetcoders.io/content/images/2020/10/image3.png)

Tuy nhiên với sự phát triển của internet, nhu cầu chuyển 1 phần của network stack vào user space ra đời. 1 ví dụ của nhu cầu đó là việc tạo kết nối private giữa 2 mạng LAN tại Layer 2/Layer 3 thông qua mạng Internet, hay dễ hiểu thì gọi là VPN :]]]. Kết nối private này thường phát triển theo các nhu cầu riêng về Authentication, Authorization, Encryption, ... nên khó có thể phát triển 1 standard protocol, nên việc đặt trong kernel space là không phù hợp.

Đáp ứng với nhu cầu trên, \*Unix kernel cung cấp interface để tạo ra network interface ảo. Khi packet được gửi tới những interface ảo này thì sẽ được kernel gửi lên userspace để được xử lý bởi VPN. VPN sau đó có thể quyết định việc encrypt và routing tới 1 máy khác qua mạng internet. Cụ thể hơn thì kernel cung cấp 3 loại interface ảo là:

- TUN device: để gửi/nhận packet ở Layer 3 (IP protocol)
- TAP device: để gửi/nhận packet ở Layer 2 (Ethernet protocol)
- Veth pair: để kết nối 2 userspace program (ko đề cập trong bài này)

![tuntap](https://www.packetcoders.io/content/images/2020/10/image2.png)

### Programing interface

Để tạo được TUN/TAP device thì ứng dụng phải được quyền access vào `/dev/net/tun` đồng thời có priviledge `CAP_NET_ADMIN`. Vì vậy mà 1 số container wireguard hoặc openvpn bạn thường thấy snippet như sau

```
services:
    wireguard:
        image: ...
        cap_add:
            - NET_ADMIN
        devices:
            - /dev/net/tun:/dev/net/tun
```

Tạo TUN interface ảo với `go` như sau (nguồn từ [wireguard-go](https://github.com/WireGuard/wireguard-go)):

```
import (
	"fmt"
	"os"
	"unsafe"
	"golang.org/x/sys/unix"
)

const (
    ifReqSize = unix.IFNAMSIZ + 64
)

func CreateTUN(name string, mtu int) (*os.File, error) {
	nfd, err := unix.Open("/dev/net/tun", os.O_RDWR, 0)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, fmt.Errorf("CreateTUN(%q) failed; /dev/net/tun does not exist", name)
		}
		return nil, err
	}

	var ifr [ifReqSize]byte
	var flags uint16 = unix.IFF_TUN // | unix.IFF_NO_PI (disabled for TUN status hack)
	nameBytes := []byte(name)
	if len(nameBytes) >= unix.IFNAMSIZ {
		unix.Close(nfd)
		return nil, fmt.Errorf("interface name too long: %w", unix.ENAMETOOLONG)
	}
	copy(ifr[:], nameBytes)
	*(*uint16)(unsafe.Pointer(&ifr[unix.IFNAMSIZ])) = flags

	_, _, errno := unix.Syscall(
		unix.SYS_IOCTL,
		uintptr(nfd),
		uintptr(unix.TUNSETIFF),
		uintptr(unsafe.Pointer(&ifr[0])),
	)
	if errno != 0 {
		unix.Close(nfd)
		return nil, errno
	}

	err = unix.SetNonblock(nfd, true)
	if err != nil {
		unix.Close(nfd)
		return nil, err
	}

	// Note that the above -- open,ioctl,nonblock -- must happen prior to handing it to netpoll as below this line.

	fd := os.NewFile(uintptr(nfd), cloneDevicePath)
    return fd, nil
}
```

_TLDR:_ mở file `/dev/net/tun` sau đó gọi lệnh `ioctl` với tham số đặc biệt

Sau đó việc đọc/ghi từ TUN interface thông qua việc tương tác với `*os.File` như bình thường. **Note:** đối với darwin, tên của interface **bắt buộc** có format `utun%d`. Nếu tạo interface với tên là `utun`, kernel sẽ tự tạo interface có tên từ `utun0` đến `utun9`

### References

- https://www.kernel.org/doc/html/v5.8/networking/tuntap.html
- https://github.com/WireGuard/wireguard-go/tree/master/tun
- https://www.packetcoders.io/virtual-networking-devices-tun-tap-and-veth-pairs-explained/

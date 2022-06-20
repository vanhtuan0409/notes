# (WIP) QUIC

## Lí do dẫn đến QUIC

Chúng ta đã quen với 2 protocol được sử dụng phổ biến ở tầng Transport là UDP và TCP protocol. Trước khi nói về những lí do dẫn đến sự ra đời của QUIC, mình sẽ nhắc lại 1 vài tính chất của UDP và TCP

|           | TCP                                                                                     | UDP                                                     |
| --------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| Phân loại | Connection/Stateful                                                                     | Connectionless/Stateless                                |
| Thứ tự    | Packet được đảm bảo thứ tự gửi/nhận                                                     | Packet không đảm bảo thứ tự gửi/nhận                    |
| Đảm bảo   | Packet được đảm bảo gửi thành công. Nếu packet loss xảy ra, protocol sẽ thực hiện retry | Packet không được đảm bảo gửi thành công. Fire n Forget |
| Checksum  | Có checksum                                                                             | Không có checksum                                       |
| Tốc độ    | Nhanh                                                                                   | Chậm                                                    |
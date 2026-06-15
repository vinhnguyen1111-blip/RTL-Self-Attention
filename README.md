# FPGA AI Attention Accelerator (Verilog)

Hệ thống FPGA hiện thực một lõi **Self-Attention** (Q, K, V, Score, Softmax, Z = Attention × V) với kích thước ma trận **64×64 (Q8.8 fixed-point, 16-bit)**, giao tiếp với máy tính qua **UART**, và hiển thị kết quả qua **LED 7 đoạn**.

---

## 1. Sơ đồ khối tổng quát


<img width="5096" height="3839" alt="image" src="https://github.com/user-attachments/assets/45092090-7094-4f3c-900f-d5bf55a24c14" />


### Sơ đồ chi tiết bên trong `attention_top` (Lõi AI)


<img width="1280" height="936" alt="image" src="https://github.com/user-attachments/assets/718b9e41-df2e-4d48-95e4-dfd4bdbc9eea" />


---

## 2. Mục lục các module (Module Index)

| # | Module | File | Vai trò |
|---|--------|------|---------|
| 1 | `system_top` | `system_top.v` | Module top cấp cao nhất, kết nối toàn bộ hệ thống lên FPGA |
| 2 | `system_controller` | `system_controller.v` | FSM trung tâm: nhận dữ liệu UART, nạp RAM, điều khiển lõi AI, gửi kết quả về PC, điều khiển LED/7-đoạn |
| 3 | `attention_top` | `attention_top.v` | Lõi tính toán Self-Attention 64x64 (Q, K, V, Score, Softmax, Z) |
| 4 | `softmax_top` | `softmax_top.v` | Tính Softmax cho 1 hàng của ma trận Score (dùng ROM e^x + chia) |
| 5 | `calc_sum_exp` | `calc_sum_exp.v` | Tính tổng Σe^(x - max) cho mẫu số Softmax |
| 6 | `softmax_divide` | `softmax_divide.v` | Bộ chia (numer/denom) dùng cho phép chia trong Softmax |
| 7 | `mac_unit` | `mac_unit.v` | Đơn vị nhân–cộng dồn (Multiply-Accumulate), số Q8.8 có dấu |
| 8 | `rom_exp` | `rom_exp.v` | ROM tra bảng giá trị e^x (Q8.8), 4097 word |
| 9 | `ram_1p` | `ram_1p.v` | RAM 1 cổng (IP `altsyncram`), dùng cho RAM Softmax |
| 10 | `ram_2p` | `ram_2p.v` | RAM giả 2 cổng (Simple Dual-Port, IP `altsyncram`), dùng cho Q/K/V/Wq/Wk/Wv/Score/Z |
| 11 | `ram_X` | `ram_X.v` | RAM lưu ma trận đầu vào X (64x64), 1 cổng đọc + 1 cổng ghi riêng |
| 12 | `uart_rx` | `uart_rx.v` | Bộ nhận UART (8-N-1), baud rate cấu hình được |
| 13 | `uart_tx` | `uart_tx.v` | Bộ truyền UART (8-N-1), baud rate cấu hình được |
| 14 | `button_debouncer` | `ui_modules.v` | Mạch chống dội cho nút nhấn, tạo xung 1 nhịp clock |
| 15 | `hex_decoder` | `ui_modules.v` | Giải mã 4-bit nhị phân sang mã LED 7 đoạn |

---

## 3. Chi tiết chân tín hiệu I/O từng module

### 3.1 `system_top`
Module top cấp cao nhất nạp lên FPGA.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `KEY[1:0]` | input | 2 | `KEY[0]` = `rst_n` (reset tích cực mức thấp), `KEY[1]` = nút Next |
| `uart_rx_pin` | input | 1 | Chân RX UART (nhận từ PC) |
| `uart_tx_pin` | output | 1 | Chân TX UART (gửi đến PC) |
| `led[3:0]` | output | 4 | LED hiển thị trạng thái hệ thống |
| `HEX0` | output | 7 | LED 7 đoạn — nibble thấp nhất của `z_display` |
| `HEX1` | output | 7 | LED 7 đoạn — nibble kế tiếp |
| `HEX2` | output | 7 | LED 7 đoạn — nibble kế tiếp |
| `HEX3` | output | 7 | LED 7 đoạn — nibble cao nhất của `z_display` |

---

### 3.2 `system_controller`
FSM trung tâm: nhận ma trận từ UART, nạp RAM, kích hoạt lõi AI, gửi kết quả Z, điều khiển hiển thị.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `rx_data` | input | 8 | Byte nhận được từ `uart_rx` |
| `rx_done` | input | 1 | Xung báo đã nhận xong 1 byte |
| `tx_data` | output | 8 | Byte cần gửi tới `uart_tx` |
| `tx_start` | output | 1 | Xung kích hoạt gửi |
| `tx_busy` | input | 1 | Báo `uart_tx` đang bận |
| `tx_done` | input | 1 | Báo đã gửi xong 1 byte |
| `we_X` | output | 1 | Cho phép ghi RAM X (nạp ma trận đầu vào) |
| `uart_addr` | output | 12 | Địa chỉ bus chung cho RAM X / Wq / Wk / Wv / Z |
| `uart_data_in` | output | 16 | Dữ liệu ghi vào RAM (Q8.8) |
| `uart_we_Wq` | output | 1 | Cho phép ghi RAM Wq |
| `uart_we_Wk` | output | 1 | Cho phép ghi RAM Wk |
| `uart_we_Wv` | output | 1 | Cho phép ghi RAM Wv |
| `uart_data_Z_out` | input | 16 | Dữ liệu đọc ra từ RAM Z (kết quả Attention) |
| `ai_start` | output | 1 | Xung khởi động lõi `attention_top` |
| `ai_done` | input | 1 | Báo lõi AI đã tính xong |
| `mode_ai_running` | output | 1 | Cờ báo lõi AI đang chạy (gạt MUX nội bộ trong `attention_top`) |
| `led_status` | output | 4 | Trạng thái hệ thống hiển thị lên LED |
| `btn_next` | input | 1 | Xung từ nút nhấn (đã debounce) để xem phần tử Z tiếp theo |
| `display_data` | output | 16 | Giá trị Z hiện tại đưa ra LED 7 đoạn |

---

### 3.3 `attention_top`
Lõi tính toán Self-Attention 64×64.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `start` | input | 1 | Xung khởi động tính toán |
| `done` | output | 1 | Báo đã tính xong toàn bộ |
| `addr_X` | output | 12 | Địa chỉ đọc ma trận X từ RAM ngoài |
| `data_X` | input | 16 | Dữ liệu X đọc về (Q8.8) |
| `mode_ai_running` | input | 1 | Cờ điều khiển MUX (1 = AI đang chạy, 0 = cho phép UART nạp/đọc trọng số & kết quả) |
| `uart_addr` | input | 12 | Địa chỉ bus UART (dùng khi nạp Wq/Wk/Wv hoặc đọc Z) |
| `uart_data_in` | input | 16 | Dữ liệu ghi từ UART (Q8.8) |
| `uart_we_Wq` | input | 1 | Cho phép ghi RAM Wq từ UART |
| `uart_we_Wk` | input | 1 | Cho phép ghi RAM Wk từ UART |
| `uart_we_Wv` | input | 1 | Cho phép ghi RAM Wv từ UART |
| `uart_data_Z_out` | output | 16 | Dữ liệu kết quả Z đọc ra cho UART |

---

### 3.4 `softmax_top`
Tính Softmax cho một hàng (64 phần tử) của ma trận Score.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `start` | input | 1 | Xung khởi động tính softmax cho 1 hàng |
| `done` | output | 1 | Báo đã tính softmax xong cho hàng hiện tại |
| `addr_Score` | output | 12 | Địa chỉ đọc RAM Score |
| `data_Score` | input | 16 (signed) | Giá trị Score đọc về |
| `max_val_in` | input | 16 (signed) | Giá trị max của hàng hiện tại (đã tính sẵn ở pha trước) |
| `addr_Exp` | output | 13 | Địa chỉ tra ROM e^x |
| `data_Exp` | input | 16 | Giá trị e^x đọc về (Q8.8, luôn dương) |
| `we_Softmax` | output | 1 | Cho phép ghi RAM Softmax |
| `addr_Softmax` | output | 12 | Địa chỉ ghi RAM Softmax |
| `data_Softmax` | output | 16 | Giá trị softmax kết quả (Q8.8) |

---

### 3.5 `calc_sum_exp`
Tính mẫu số Softmax: Σe^(x − max) cho 64 phần tử của một hàng.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `start` | input | 1 | Xung khởi động |
| `row_idx` | input | 6 | Chỉ số hàng đang xử lý |
| `max_val` | input | 16 (signed) | Giá trị max của hàng |
| `done` | output | 1 | Báo đã tính xong tổng |
| `sum_val` | output | 32 | Tổng Σe^(x−max) (mẫu số) |
| `addr_Score` | output | 12 | Địa chỉ đọc RAM Score |
| `data_Score` | input | 16 (signed) | Giá trị Score đọc về |
| `addr_Exp` | output | 13 | Địa chỉ tra ROM e^x |
| `data_Exp` | input | 16 | Giá trị e^x đọc về (Q8.8) |

---

### 3.6 `softmax_divide`
Bộ chia tổ hợp (combinational) numer / denom.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `numer` | input | 32 | Tử số (e^x) |
| `denom` | input | 32 | Mẫu số (Σe^x) |
| `quotient` | output | 32 | Kết quả phép chia |
| `remain` | output | 32 | Số dư phép chia |

---

### 3.7 `mac_unit`
Đơn vị nhân–cộng dồn số có dấu Q8.8.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `en` | input | 1 | Cho phép tích lũy (accumulate) |
| `clr_acc` | input | 1 | Xóa accumulator (độc lập với `en`) |
| `a_in` | input | 16 (signed Q8.8) | Toán hạng A |
| `b_in` | input | 16 (signed Q8.8) | Toán hạng B |
| `mac_out` | output | 16 (signed Q8.8) | Kết quả MAC (tổ hợp, combinational) |

---

### 3.8 `rom_exp`
ROM tra bảng giá trị e^x (Altera/Intel `altsyncram` ROM IP).

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clock` | input | 1 | Clock |
| `address` | input | 12 | Địa chỉ tra ROM (kẹp tối đa 4095) |
| `q` | output | 16 | Giá trị e^x (Q8.8) |

---

### 3.9 `ram_1p`
RAM 1 cổng (Single-Port, IP `altsyncram`) — dùng cho RAM Softmax.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clock` | input | 1 | Clock |
| `address` | input | 12 | Địa chỉ đọc/ghi |
| `data` | input | 16 | Dữ liệu ghi vào |
| `wren` | input | 1 | Cho phép ghi |
| `q` | output | 16 | Dữ liệu đọc ra |

---

### 3.10 `ram_2p`
RAM giả 2 cổng (Simple Dual-Port, IP `altsyncram`) — dùng cho Q, K, V, Wq, Wk, Wv, Score, Z.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clock` | input | 1 | Clock |
| `address_a` | input | 12 | Địa chỉ cổng A |
| `address_b` | input | 12 | Địa chỉ cổng B |
| `data_a` | input | 16 | Dữ liệu ghi cổng A |
| `data_b` | input | 16 | Dữ liệu ghi cổng B |
| `wren_a` | input | 1 | Cho phép ghi cổng A |
| `wren_b` | input | 1 | Cho phép ghi cổng B |
| `q_a` | output | 16 | Dữ liệu đọc cổng A |
| `q_b` | output | 16 | Dữ liệu đọc cổng B |

---

### 3.11 `ram_X`
RAM lưu ma trận đầu vào X (64×64), 1 cổng đọc riêng + 1 cổng ghi riêng (IP `altsyncram`).

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clock` | input | 1 | Clock |
| `data` | input | 16 | Dữ liệu ghi vào (từ UART) |
| `wraddress` | input | 12 | Địa chỉ ghi |
| `wren` | input | 1 | Cho phép ghi |
| `rdaddress` | input | 12 | Địa chỉ đọc (từ lõi AI) |
| `q` | output | 16 | Dữ liệu đọc ra (cho lõi AI) |

---

### 3.12 `uart_rx`
Bộ nhận UART 8-N-1.

**Tham số:** `CLK_FREQ = 50_000_000`, `BAUD_RATE = 115200`

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `rx_pin` | input | 1 | Chân vật lý nối với TX của Laptop |
| `rx_data` | output | 8 | Byte nhận được |
| `rx_done` | output | 1 | Xung 1 nhịp báo nhận xong 1 byte |

---

### 3.13 `uart_tx`
Bộ truyền UART 8-N-1.

**Tham số:** `CLK_FREQ = 50_000_000`, `BAUD_RATE = 115200`

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `rst_n` | input | 1 | Reset tích cực mức thấp |
| `tx_start` | input | 1 | Xung kích hoạt (1 clock) để bắt đầu gửi |
| `tx_data` | input | 8 | Byte cần gửi đi |
| `tx_pin` | output | 1 | Chân vật lý nối với RX của Laptop |
| `tx_busy` | output | 1 | Cờ báo bận (đang gửi) |
| `tx_done` | output | 1 | Xung 1 nhịp báo gửi xong 1 byte |

---

### 3.14 `button_debouncer` *(trong `ui_modules.v`)*
Mạch chống dội cho nút nhấn.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `clk` | input | 1 | Clock hệ thống |
| `btn_in` | input | 1 | Tín hiệu thô từ nút nhấn |
| `btn_edge` | output | 1 | Xung sạch 1 nhịp clock khi nút được nhấn xuống |

---

### 3.15 `hex_decoder` *(trong `ui_modules.v`)*
Giải mã 4-bit nhị phân sang mã LED 7 đoạn.

| Chân | Hướng | Bit-width | Mô tả |
|------|-------|-----------|-------|
| `hex_in` | input | 4 | Giá trị nhị phân (0–F) |
| `segments` | output | 7 | Mã điều khiển LED 7 đoạn |

---

## 4. Luồng hoạt động tổng quát

1. **Nạp dữ liệu**: PC gửi qua UART lần lượt 4 ma trận 64×64 (X, Wq, Wk, Wv), `system_controller` ghi từng ma trận vào `ram_X`, `Wq`, `Wk`, `Wv` bên trong `attention_top`.
2. **Tính toán Attention**: `system_controller` phát `ai_start`, `attention_top` thực hiện pipeline 4 lõi MAC để tính `Q = X·Wq`, `K = X·Wk`, `V = X·Wv`, `Score = Q·K^T` (có scale), tìm max online, tính `Softmax(Score)` qua `softmax_top` (dùng `rom_exp`, `calc_sum_exp`, `softmax_divide`), rồi `Z = Softmax · V`.
3. **Trả kết quả**: `attention_top` báo `ai_done`, `system_controller` đọc RAM Z và gửi 4096 word kết quả về PC qua UART, đồng thời LED báo trạng thái.
4. **Hiển thị**: Sau khi hoàn tất, giá trị Z hiện tại được đưa ra `HEX0..HEX3`; nhấn `KEY[1]` để duyệt qua các phần tử Z tiếp theo.

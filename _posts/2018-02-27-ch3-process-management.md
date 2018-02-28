---
layout: post
title: Chapter 3. Process Management
subtitle: Chương 3. Quản lý tiến trình
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [test]
---

#### **Bài viết được dịch từ cuốn Linux Kernel Development**

Chương này sẽ giới thiệu về khái niệm của **tiến trình**. Bộ quản lý tiến trình là một yếu tố cốt lõi của bất kì hệ điều hành nào, trong đó có cả Linux.

## Tiến trình 

Tiền trình là một chương trình (object code stored on some media) .
Bên cạnh việc thi hành đoạn mã chương trình (text section in Unix), các tiến trình còn bao gồm:
- Mở các file.
- Chờ các tín hiệu.
- Dữ liệu bên trong kernel.
- Trạng thái bộ xử lý.
- Không gian địa chỉ trong bộ nhớ (with one or more memory mappings).
- **Các thread**.
- Vùng dữ liệu chứa biến toàn cục.

### <span style="color:red">Các thread</span>

Các thread là các hoạt động bên trong một tiến trình. Mỗi thread chứa:
- Bộ đếm chương trình (program counter)
- Ngăn nhớ tiến trình (process stack)
- Tập các thanh ghi bộ xử lý.

Kernel lập lịch cho các thread, chứ không phải là các tiến trình (process). Linux không phân biệt giữa thread và process. Với Linux, một thread là một loại tiến trình đặc biệt.


### <span style="color:red">Bộ xử lý ảo và bộ nhớ ảo</span>

Ở những hệ điều hành hiện đại, các tiến trình cung cấp 2 cái **ảo** : một bộ xử lý ảo và một bộ nhớ ảo:
- Bộ xử lý ảo cung cấp cho tiến trình cảm giác rằng tiến trình đó là độc quyền trên hệ thống, thay vì sự thật là nó chia sẻ bộ xử lý chính với hàng trăm các tiến trình khác. Xem chương 4.
- Bộ nhớ ảo để c ho các tiến trình cư ngụ và quản lý như là cái tiến trình đấy chiếm giữ toàn bộ bộ nhớ vậy. Xem chương 12.

Các threads chia sẻ bộ nhớ ảo __trừu tượng__, trong khi chỉ nhận bộ xử lý ảo của riêng nó.

### <span style="color:red">Vòng đời của một tiến trình</span>

Một tiến trình (process) là một chương trình được kích hoạt và liên hệ tới các tài nguyên:
- Hai hoặc nhiều tiến trình có thể tồn tại và chạy cùng một chương trình.
- Hai hoặc nhiều tiến trình có thể tồn tại và chia sẻ tài nguyên, ví dụ như là file hoặc một không gian địa chỉ.

### <span style="color:red">fork, exec, exit và wait</span>

Trên linux, lời gọi hệ thống `fork()` tạo một tiến trình mới bằng cách nhân đôi (duplicate) một tiến trình có sẵn.
- Tiến trình gọi `fork()` là tiến trình cha, trong khi tiến trình mới sinh ra là con.
- Nơi mà tiến trình cha tiếp tục chạy lại (resume) và nơi tiến trình con bắt đầu chạy là  1 chỗ: chính là nơi gọi `fork()`.
- Lệnh gọi hệ thống `fork()` return cho kernel 2 lần: một lần là tiến trình cha và một lần tiến trình con.

Họ lời gọi hàm `exec()` thì tạo một không gian địa chỉ mới rồi load ngay lập tức chương trình mới vào tiến trình con sau một `fork`. Trong cùng Linux kernel, `fork()` thực ra thi hành bằng lời gọi hệ thống `clone()`, `clone()` sẽ được thảo luận sau.

Lời gọi `exit()` kết thúc một tiến trình và giải phóng tài nguyên. Một tiến trình cha có thể dò xem trạng thái của một tiến trình con được kết thúc bằng lời gọi `wait4()`. Một tiến trình có thể đợi một tiến trình cụ thể nào đó kết thúc. Khi một tiến trình kết thúc, nó được đặt vào một trạng thái đặt biệt, gọi là __zombie__, trạng thái này đại diện cho các tiến trình được kết thúc cho đến khi cha của gọi lệnh `wait()` hoặc `waitpid()`. Kernel sẽ thi hành lời gọi `wait4()`. Hệ thống Linux, thông qua thư viện C, cung cấp các hàm `wait()`, `waitpid()`, `wait3()` và `wait4()`.

## Descriptor của tiến trình và Task Structure 

Một cái tên khác của tiến trình là task (tác vụ). Trong cuốn sách này, 2 thuật ngữ được luân phiên nay, mặc dù "task" thông thường để chỉ một "process" từ cái nhìn của kernel 0.0

Kernel lưu danh sách các tiến trình dưới dạng một danh sách liên kết đôi vòng (circular doubly linked list), và gọi nó là **danh sách tác vụ** (task list). Nó chứa tất cả các thông tin về một tiến trình cụ thể.

(Còn nữa) 

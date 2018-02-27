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

Các threads chia sẻ bộ nhớ ảo _trừu tượng_, trong khi chỉ nhận bộ xử lý ảo của riêng nó.

### <span style="color:red">Vòng đời của một tiến trình</span>
(Còn nữa)

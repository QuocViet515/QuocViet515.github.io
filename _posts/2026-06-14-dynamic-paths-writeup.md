---
title: "CTF Writeup: Dynamic Paths - Thoát Khỏi Behemoth"
excerpt: "Bài viết chia sẻ cách áp dụng thuật toán Quy hoạch động (Dynamic Programming) và Pwntools để tự động hóa và giải quyết thử thách lập trình socket."
header:
  overlay_image: /assets/images/catppuccin_mocha-skin-archive-large.png
  overlay_filter: 0.5
  teaser: /assets/images/catppuccin_mocha-skin-archive-large.png
categories:
  - CTF Writeup
tags:
  - Python
  - Pwntools
  - Dynamic Programming
  - Socket
---

Chào các bạn, bài viết này sẽ trình bày writeup cho challenge **Dynamic Paths**. Đây là một bài tập kết hợp khá thú vị giữa kiến thức về cấu trúc dữ liệu, thuật toán tìm đường đi tối ưu và kỹ năng lập trình tương tác qua socket. 

## 1. Phân tích đề bài (Scenario)

Cốt truyện của bài toán đưa chúng ta vào một tình huống chạy trốn: bạn đang bị một con siêu mutant Behemoth truy đuổi trong hệ thống đường hầm ngầm. Để trốn thoát, bạn phải tìm được đường đi có khoảng cách ngắn nhất băng qua 100 khu vực bản đồ.

**Thông tin đầu vào:**
- Server sẽ yêu cầu chúng ta vượt qua `t = 100` vòng (grids).
- Mỗi vòng cung cấp kích thước ma trận `i x j` và một mảng các con số đại diện cho khoảng cách giữa các khối.

**Luật di chuyển:**
- Bạn bắt đầu tại góc trên cùng bên trái.
- Tại mỗi ô, bạn chỉ được phép di chuyển **xuống dưới** hoặc sang **phải**.
- **Mục tiêu:** Tìm một lộ trình đến góc dưới cùng bên phải sao cho tổng các con số trên đường đi là **nhỏ nhất**, sau đó gửi kết quả lại cho server.

## 2. Hướng giải quyết bằng Quy hoạch động (DP)

Đây là biến thể của bài toán kinh điển [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/) trong khoa học máy tính. Vì chúng ta bị giới hạn hướng đi (chỉ xuống và phải), nên tổng chi phí tối ưu để đến một ô bất kỳ sẽ phụ thuộc hoàn toàn vào ô ngay phía trên hoặc ô ngay bên trái nó.

**Công thức truy hồi:**
Gọi $DP[r][c]$ là tổng chi phí nhỏ nhất để đi từ điểm xuất phát đến tọa độ $(r, c)$. Tại một ô bất kỳ ở giữa ma trận, giá trị nhỏ nhất sẽ là:

$$DP[r][c] = \min(DP[r-1][c], DP[r][c-1]) + Grid[r][c]$$

![Minh họa quy hoạch động ma trận](/assets/images/catppuccin_mocha-skin-archive-large.png)

## 3. Tự động hóa với Pwntools

Vì phải giải quyết liên tục 100 vòng, việc tính tay là bất khả thi do giới hạn thời gian (timeout) của server. Chúng ta sẽ dùng thư viện `pwntools` trong Python để kết nối socket, bắt gói tin, tính toán và gửi trả đáp án.

**Lưu ý quan trọng:** Khi vừa mở kết nối, server sẽ gửi một đoạn cốt truyện khá dài. Nếu cố gắng ép kiểu dữ liệu ngay từ những dòng đầu tiên, script sẽ crash với lỗi `ValueError`. Cách khắc phục là dùng hàm `recvuntil(b'Test ')` để ép script bỏ qua phần rác và chỉ bắt đầu đọc khi vào đúng chu kỳ test.

Dưới đây là đoạn script Python hoàn chỉnh:

```python
from pwn import *

def solve_min_path(rows, cols, grid):
    dp = [[0] * cols for _ in range(rows)]
    dp[0][0] = grid[0][0]
    
    # Khởi tạo rìa trên cùng và rìa trái
    for c in range(1, cols):
        dp[0][c] = dp[0][c-1] + grid[0][c]
    for r in range(1, rows):
        dp[r][0] = dp[r-1][0] + grid[r][0]
        
    # Tính toán chi phí cho các ô còn lại theo công thức DP
    for r in range(1, rows):
        for c in range(1, cols):
            dp[r][c] = min(dp[r-1][c], dp[r][c-1]) + grid[r][c]
            
    return dp[rows-1][cols-1]

def main():
    HOST = '154.57.164.83'
    PORT = 32338
    
    context.log_level = 'info'
    r = remote(HOST, PORT)
    
    for i in range(100):
        # Bỏ qua đoạn text cốt truyện, tìm đến chính xác test case
        r.recvuntil(b'Test ')
        test_info = r.recvline().decode().strip()
        log.info(f"Đang giải quyết Test {test_info}")
        
        # Nhận dòng kích thước
        dim_line = r.recvline().decode().strip()
        rows, cols = map(int, dim_line.split())
        
        # Nhận mảng 1 chiều và chuyển về dạng lưới ma trận 2 chiều
        grid_line = r.recvline().decode().strip()
        nums = list(map(int, grid_line.split()))
        grid = [nums[k * cols:(k + 1) * cols] for k in range(rows)]
        
        # Tính toán và in ra đáp án
        ans = solve_min_path(rows, cols, grid)
        log.success(f"Đáp án: {ans}")
        
        # Chờ dấu nhắc lệnh và gửi kết quả
        r.recvuntil(b'> ')
        r.sendline(str(ans).encode())
    
    log.info("Hoàn tất 100 tests. Chờ nhận Flag...")
    print(r.recvall(timeout=5).decode())

if __name__ == '__main__':
    main()
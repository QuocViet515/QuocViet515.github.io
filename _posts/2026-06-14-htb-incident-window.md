---
title: "HTB Writeup: Incident Window"
excerpt: "Giải quyết bài toán đếm cửa sổ thời gian (Sliding Window) trong thử thách Incident Window trên Hack Thê Box."
header:
  overlay_image: /assets/images/posts/htb-incident-window/Screenshot_2026-05-16_092345.png
  overlay_filter: 0.5
  teaser: /assets/images/posts/htb-incident-window/Screenshot_2026-05-16_092345.png
categories:
  - CTF Writeup
tags:
  - HTB
  - Python
  - Sliding Window
  - Algorithm
mathjax: true
---

## Incident Window — HTB

<div class="notice--info">
  <strong>Mục tiêu:</strong> Đếm số “cửa sổ thời gian” dài <strong>W</strong> giây có số sự kiện <strong>S</strong> (Suspicious) <strong>&ge; K</strong>.
</div>

### Bài toán

- **Vấn đề:** Nếu duyệt từng sự kiện và kiểm tra mọi khung thời gian chứa nó (cách ngây thơ/naive $O(N^2)$), chương trình sẽ rất chậm vì có thể có tới **50.000** sự kiện.
- **Ràng buộc quan trọng:** mốc thời gian lớn nhất $ts \le 10.000$.

### Giải pháp — Sliding Window (Cửa sổ trượt)

Vì $Max\_TS = 10.000$, ta có thể đếm số sự kiện **S** theo từng giây bằng một mảng:

```python
s_counts = [0] * (MAX_TS + 1)
```

- Với mỗi sự kiện `S` ở giây `X`: tăng `s_counts[X] += 1`.
- Tính tổng số `S` trong cửa sổ đầu tiên `[0, W)`.
- Trượt cửa sổ thêm 1 giây mỗi bước:
    - **Trừ** số sự kiện ở mép trái (vừa ra khỏi cửa sổ)
    - **Cộng** số sự kiện ở mép phải (vừa vào cửa sổ)

Độ phức tạp: $O(Max\_TS)$, phù hợp với dữ liệu lớn.

---

## Python reference implementation

```python
import sys

def main():
    input_data = sys.stdin.read().split()
    if not input_data:
        return

    N = int(input_data[0])
    W = int(input_data[1])
    K = int(input_data[2])

    MAX_TS = 10000
    s_counts = [0] * (MAX_TS + 1)

    idx = 3
    for _ in range(N):
        ts = int(input_data[idx])
        event_type = input_data[idx + 1]
        idx += 2

        if event_type == 'S' and ts <= MAX_TS:
            s_counts[ts] += 1

    alerts = 0

    # First window [0, W)
    current_s = sum(s_counts[0:W])
    if current_s >= K:
        alerts += 1

    # Slide windows t = 1..MAX_TS-W
    for t in range(1, MAX_TS - W + 1):
        current_s = current_s - s_counts[t - 1] + s_counts[t + W - 1]
        if current_s >= K:
            alerts += 1

    print(alerts)

if __name__ == '__main__':
    main()
```

![Giải pháp Incident Window](/assets/images/posts/htb-incident-window/Screenshot_2026-05-16_092345.png)

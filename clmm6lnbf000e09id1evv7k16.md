---
title: "Ứng dụng Build Heavy-write application vào task import data"
datePublished: Sat Sep 16 2023 15:26:40 GMT+0000 (Coordinated Universal Time)
cuid: clmm6lnbf000e09id1evv7k16
slug: ung-dung-build-heavy-write-application-vao-task-import-data
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1694874344690/a57f0a33-05bc-49dc-9898-bfe985996848.jpeg
tags: oracle, performance, nodejs, typescript

---

Sau một thời gian dài im hơi lặng tiếng và chia tay với Sec thì lần này mình quay trở lại với một vai trò là Dev quèn không thể quèn hơn :( . Dạo này mình hay lang thang trên mạng để đọc các Techblog mong góp nhặt thêm kiến thức để bỏ chữ quèn ở trên đi và trở thành Dev thực thụ =)).

Bắt đầu vào cái chủ đề ngày hôm nay là "*Làm sao để tối ưu hoá Performance trong task import data*? :v". Về cái vấn đề này thì dự án trên công ty của mình có một tính năng đó là import dữ liệu từ file Excel vào database. Lúc đầu thì bọn mình làm theo kiểu chỉ đọc file excel và ghi vào database Oracle theo cách thông thường mọi người đều làm thôi. Workflow của nó thì như hình dưới đây:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1694875321857/54e590d6-b60f-4c13-8985-469ff0c19486.png align="center")

Mọi thứ nó vẫn chạy ngon lành và chẳng ai để ý đến Performance (vì lúc test chỉ thử với file ít dữ liệu thôi). Cho đến một ngày,.... mình nhận file excel data từ khách hàng. Bùm =))) file dữ liệu 10Mb. Data to thì không phải là to nhưng mà code của mình nó chạy hết 15p / 100000 row (●'◡'●). Không ổn!!!!!!!!!!!!!. Đang loay hoay tìm cách fixx thì mình đọc được cái blog của anh ***Minh Monmen*** về đúng cái vấn đề này luôn ([<mark>link nè</mark>](https://viblo.asia/s/chuyen-anh-tho-xay-va-write-heavy-application-24lJDz46KPM)) và mình quyết định thử áp dụng với cái code hổ lốn của mình 😊.

Bắt đầu check code nào &gt;&lt;

```typescript

const createInfoArr = [];
let rowStart = 1;
let cellStart = 1;
while (dataRow) {
    createInfoArr.push(
        InfoEntity.create({
            firstName: wSheet
                .getRow(rowStart)
                .getCell(cellStart)
                .value?.toString(),
            lastName: wSheet
                .getRow(rowStart)
                .getCell(cellStart + 1)
                .value?.toString(),
            address: wSheet
                .getRow(rowStart)
                .getCell(cellStart + 2)
                .value?.toString(),
        })
    );
    rowStart += 1;
    dataRow = wSheet.getRow(rowStart).getCell(cellStart).value;
}
await transactionManager.save(createInfoArr);
```

### Vấn đề đầu tiên: **Sử dụng save transaction**

Theo cái nguồn [này](https://dev.to/rishit/optimizing-typeorm-tips-from-experience-part-1-dont-use-save-4ke9) mình đọc được (mn confirm giúp em nhé =))) thì khi sử dụng save transaction nó sẽ thực hiện 2 tác vụ lên database

* B1: sẽ query select để tìm xem bản ghi có tồn tại trong db hay không
    
* B2: Nếu tồn tại thì thực hiện update, còn không thực hiện Insert
    

Việc sử dụng save transaction sẽ mất nhiều thời gian hơn vì

* Mỗi query sẽ gọi 2 lần vào database → đồng nghĩa với việc chịu thêm **network latency**
    
* Yêu cầu tác vụ chỉ thực hiện Insert hoặc Update → thực hiện cả hai là dư thừa
    
* Câu lệnh *SELECT* được tạo ra bởi TypeORM thường bao gồm các subquery, khi dữ liệu trong database lớn thì sẽ mất thêm thời gian
    

Với các task insert dữ liệu từ file excel thì mình thường xoá dữ liệu trước đó và insert thêm dữ liệu hoàn toàn mới vào. Nên giải pháp ở đây là sẽ dùng *INSERT TRANSACTION* thay cho *SAVE TRANSACTION*.
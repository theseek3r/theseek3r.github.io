+++
title = "Memory consistency model: Sequential consistency và Total Store Order"
date = "2021-08-21T20:28:40+07:00"
description = "Sơ lược về Sequential consistency và Total Store Order memory model"
author = "blue"
showFullContent = false
+++

## Memory consistency model

Ban đầu X = Y = 0
```
CPU 1                                   CPU 2

X = 1                                   Y = 1
R1 = Y                                  R2 = X
```

2 CPU này thực thi độc lập, thứ tự ngẫu nhiên, liệu có khi nào chúng ta thu được kết quả `R1 = 0 and R2 = 0` hay không?

Theo những cảm nhận thông thường, chúng ta có thể trả lời là không vì với `R1 = 0` ta có thể suy ra lệnh `R1 = Y` ở CPU 1 phải được thực thi trước `Y = 1` ở CPU 2. Điều này dẫn đến `X = 1` sẽ được thi sẽ được thực thi trước `R2 = X` (do `X = 1` thực thi trước `R1 = Y` theo trật tự chương trình (program order) ở CPU 1 và `Y = 1` thực thi trước `R2 = X` theo trật tự chương trình ở CPU 2). Vì thế, nếu `R1 = 0` thì `R2 = 1` và ngược lại nếu `R2 = 1` thì `R1 = 0`. Ở đây, chúng ta cũng có thể có cặp `R1 = R2 = 1` ví dụ như ở trật tự này `X = 1` -> `Y = 1` -> `R1 = Y` -> `R2 = X`. Tuy nhiên, việc thu được kết quả `R1 = 0 and R2 = 0` là vô lý.

Một điều bất ngờ là ở hầu hết cái kiến trúc CPU hiện đại ngày nay kết quả `R1 = 0 and R2 = 0` là hoàn toàn có thể xảy ra. Ở lập luận ở trên, chúng ta có sử dụng trật tự chương trình (program order) để lập luận về trật tự truy xuất bộ nhớ (memory order). Tuy nhiên, ở phần lớn kiến trúc CPU, trật tự truy xuất bộ nhớ có thể không giống với trật tự của chương trình.

Việc hiểu về trật tự truy xuất bộ nhớ là vô cùng quan trọng đối với người lập trình để có thể viết ra những chương trình đem lại kết quả như mong muốn. Vì thế, các nhà sản xuất thường xây dựng **memory consistency model** để quy định các thay đổi trong trật tự truy xuất bộ nhớ (memory reordering) được cho phép và những thay đổi không được cho phép.

## Sequential consistency (SC)
Ở memory consistency model này, trật tự truy xuất bộ nhớ được đảm bảo giống với trật tự chương trình của từng CPU.
```
(LOAD(x)  <p  LOAD(y))  -> (LOAD(x)  <m  LOAD(y))
(LOAD(x)  <p  STORE(y)) -> (LOAD(x)  <m  STORE(y))
(STORE(x) <p  LOAD(y))  -> (STORE(x) <m  LOAD(y))
(STORE(x) <p  STORE(y)) -> (STORE(x) <m  STORE(y))

Với 
LOAD(x), STORE(x) là việc đọc và ghi bộ nhớ ở địa chỉ x
<p thể hiện trật tự chương trình
<m thể hiện trật tự truy xuất bộ nhớ 
```

Trở lại với ví dụ ở đầu, ta có
```
(X = 1 <p R1 = Y) -> (X = 1 <m R1 = Y)          (1)
R1 = 0            -> (R1 = Y <m Y = 1)          (2)
(Y = 1 <p R2 = X) -> (Y = 1 <m R2 = X)          (3)
(1), (2), (3)     -> (X = 1 <m R2 = X)          (4)
(4)               -> R2 = 1
```
Tương tự nếu `R2 = 0`
Từ đó, chúng ta có thể thấy trong SC, kết quả `R1= 0 and R2 = 0` là không thể

## Total store order (TSO)
SC mang đến một mô hình dễ hiểu cho người lập trình tuy nhiên hiệu năng không tốt. Một tác vụ truy xuất bộ nhớ thực thi lâu hơn mong đợi có thể làm gián đoạn việc thực thi trên CPU. Trong 2 tác vụ truy xuất bộ nhớ, tác vụ STORE thường có thời gian thực thi lâu hơn.

Để giải thích cho điều này giả sử mỗi CPU sẽ có 1 bộ cache riêng và sử dụng giao thức MESI để quản lý cache. Trong giao thức này, CPU có thể đọc từ cacheline khi cacheline trong trạng thái Modified (M), Exclusive (E), Shared (S), tuy nhiên chỉ có thể ghi vào cacheline khi cacheline trong trạng thái Modified (M), Exclusive (E). Khi thực thi tác vụ LOAD(x) nếu x không trong cache của CPU A (cache miss), 1 lời nhắn sẽ được gửi tới cache controller. Nếu x nằm trong cache của CPU khác, dữ liệu của cacheline đó sẽ được gửi lại cho CPU A, và trạng thái của tất cả các cacheline có chứa x ở các CPU sẽ là Shared. Nếu x không nằm trong cache của CPU khác, cacheline sẽ được đọc lên từ bộ nhớ chính. Đối với tác vụ STORE(x), điểm khác biệt đến khi x không nằm trong cache của CPU hiện tại nhưng nằm trong cache của CPU khác. Khi đó, để có được cacheline ở trạng thái Exclusive (E) có thể ghi được, CPU phải đợi các CPU khác xóa cacheline ở cache của các CPU đó. Để giải quyết vấn đề này, mỗi CPU sẽ có thêm 1 first-in-first-out (FIFO) store buffer, thay vì phải đợi khi cacheline được trả về có trạng thái Exclusive (E), CPU sẽ ghi vào store buffer và tiếp tục thực thi các lệnh ở dưới. Khi cacheline đó đã sẵn sàng, giá trị ở store buffer sẽ được ghi vào cacheline.

Với cách hiện thực này, memory consistency model được nới lỏng hơn so với SC, khi đó trật tự chương trình STORE -> LOAD không đảm bảo thứ tự STORE -> LOAD ở trật tự truy xuất bộ nhớ. Tuy nhiên, trật tự STORE -> STORE vẫn được đảm bảo, khi store buffer được hiện thực với mô hình FIFO và khi store buffer không rỗng, các tác vụ STORE tiếp theo phải ghi giá trị vào store buffer dù cacheline đang ở trạng thái có thể ghi được ở CPU đó.

Để cho phép người lập trình đảm bảo trật tự STORE -> LOAD khi cần thiết, memory barrier (hay còn gọi là fence) được cung cấp, lệnh này sẽ ... store buffer

```
(LOAD(x)  <p  LOAD(y))  -> (LOAD(x)  <m  LOAD(y))
(LOAD(x)  <p  STORE(y)) -> (LOAD(x)  <m  STORE(y))
(STORE(x) <p  STORE(y)) -> (STORE(x) <m  STORE(y))
(FENCE <p LOAD(x)/STORE(x)/FENCE) -> (FENCE <m LOAD(x)/STORE(x)/FENCE)
(LOAD(x)/STORE(x)/FENCE <p FENCE) -> (LOAD(x)/STORE(x)/FENCE <m FENCE)
```

Trở lại với ví dụ ở đầu, do trật tự chương trình STORE -> LOAD không đảm bảo việc truy xuất bộ nhớ có trật tự tương tự, trật tự truy xuất bộ nhớ như sau hoàn toàn có thể xảy ra

```
(R1 = Y) <m (Y = 1) <m (R2 = X) <m (X = 1)
```

Khi đó, chúng ta sẽ thu được kết quả `R1 = 0 and R2 = 0`

Tình huống ví dụ về việc xảy ra trật tự trên:
- Giả sử ban đầu cacheline chứa X đang ở trạng thái Exclusive ở CPU 2, cacheline chứa Y đang ở trạng thái Shared và ở cả 2 CPU
- CPU 1 thực thi `X = 1` nhưng bị cache miss nên gửi yêu cầu sỡ hữu cacheline chứa X ở trạng thái Exclusive đến các CPU và lưu `X = 1` vào store buffer
- CPU 1 thực thi `R1 = Y` và đọc được Y từ cacheline có giá trị là 0
- CPU 2 thực thi `Y = 1` nhưng bị cache miss nên gửi yêu cầu sỡ hữu cacheline chứa Y ở trạng thái Exclusive đến các CPU và lưu `Y = 1` vào store buffer
- CPU 1 nhận được yêu cầu của CPU 2 về cacheline chứa Y, thực hiện xóa cacheline này trong cache của mình và gửi lời nhắn đã xóa (acknowledge) cho CPU 1
- CPU 2 nhận được thông tin từ CPU 1 viết `Y = 1` từ store buffer xuống cache
- CPU 2 thực thi `R2 = X` và đọc được X từ cacheline có giá trị là 0
- CPU 2 nhận được yêu cầu của CPU 1 về cacheline chứa X, thực hiện xóa cacheline này trong cache của mình và gửi lời nhắn đã xóa (acknowledge) cho CPU 1
- CPU 1 nhận được thông tin từ CPU 2 viết `X = 1` từ store buffer xuống cache. Tuy nhiên, lúc này thì việc đọc R1, R2 hoàn tất và chúng ta có kết quả `R1 = 0 and R2 = 0`

### Tài liệu tham khảo:
- Vijay Nagarajan, Daniel J. Sorin, Mark D. Hill, David A. Wood's book [A Primer on Memory Consistency and Cache Coherence, Second Edition](https://www.morganclaypool.com/doi/abs/10.2200/S00962ED2V01Y201910CAC049)
- Peter Sewell, Susmit Sarkar, Scott Owens, Francesco Zappa Nardelli, Magnus O. Myreen's paper [x86-TSO: A Rigorous and Usable Programmer’s Model for x86 Multiprocessors](https://www.cl.cam.ac.uk/~pes20/weakmemory/cacm.pdf)



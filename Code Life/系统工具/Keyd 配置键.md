1. 下载
```bash
git clone https://github.com/rvaiya/keyd
cd keyd
make && sudo make install
sudo systemctl enable --now keyd
```
2. 编译配置文件`sudo vim /etc/keyd/default.conf`
```conf
# /etc/keyd/default.conf
# 96键键盘配置 - CapsLock 切换上档字符层
# 安装: sudo apt install keyd
# 生效: sudo keyd reload

[ids]
*

[main]
capslock = toggle(shift_layer)

[shift_layer:toggle]
# 字母键保持大写（等同于按住 Shift）
a = A
b = B
c = C
d = D
e = E
f = F
g = G
h = H
i = I
j = J
k = K
l = L
m = M
n = N
o = O
p = P
q = Q
r = R
s = S
t = T
u = U
v = V
w = W
x = X
y = Y
z = Z

# 数字行上档字符
1 = !
2 = @
3 = #
4 = $
5 = %
6 = ^
7 = &
8 = *
9 = (
0 = )

# 符号键上档字符
equal = +
minus = _
leftbrace = {
rightbrace = }
backslash = |
semicolon = :
apostrophe = "
grave = ~
comma = <
dot = >
slash = ?
```
3. 执行reload `sudo keyd reload` 
4. 完全移除`keyd`插件
```bash
sudo systemctl stop keyd
sudo systemctl disable keyd
sudo apt remove --purge keyd
sudo rm -rf /etc/keyd/
```